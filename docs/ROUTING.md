# ROUTING.md — Routing Core Semantics

The core is deliberately small: it owns the bridge table, the active-stream
table, per-bridge talker arbitration, and the event bus. It knows nothing
about any protocol, FEC, slots, pacing, or peers-as-sockets. Everything in
this file happens in the core module, on the daemon's single thread
(ARCHITECTURE.md §2).

## 1. Vocabulary

- **System** — one configured protocol instance (an HBP master, an IPSC
  network, an OBP peer, a CC conduit, an XLX module connection). Immutable table,
  indexed by `origin_system` u16.
- **Bridge** — a named conference. The unit of routing. Members are
  `(system, tgid, slot, enabled)` tuples. A bridge is the *only* way
  traffic moves between systems.
- **Stream** — one transmission: keyed `(origin_system, stream_tag)`,
  bounded by CALL_START … CALL_END (or timeout).

## 2. The routing key is TGID alone

Within the network, timeslot is meaningless — it is an air-interface TDMA
constraint local to slotted edges. Therefore:

- Ingress matching: a group-call stream from system S with destination
  TGID T selects the bridge that has member `(S, T)`. `origin_slot` plays
  no part. Config validation **rejects** two bridges both containing
  member `(S, T)` — that lookup must be unambiguous. (Note this is looser
  than hblink3, where `(S, T, TS1)` and `(S, T, TS2)` could feed different
  bridges. That trick is intentionally not supported; see D-05.)
- TGID translation is inherent: each member states its own local TGID, so
  bridge "STATEWIDE" can be TG 3120 on the IPSC side and TG 31201 on an
  OBP peer.
- Egress: forward to every *other* enabled member, rewriting `dst_id` to
  the member's TGID and passing the member's `(tgid, slot)` as the
  `nx_egress_target` (ADAPTERS.md §1). The adapter is told its delivery
  parameters; it never looks them up.

Lookup structure: a hash map `(system u16, tgid u32) → member*` built once
at config load; members carry their bridge back-pointer.

Ingress is **unconditional**: adapters forward every stream to the core,
including streams also delivered in-system by local repeat (ADAPTERS.md)
and streams that match no bridge. An unbridged stream still updates the
target route cache (§6) and emits a rate-limited `unbridged` event before
being dropped — the core sees everything happening at every edge, which is
what makes the unified dashboard, accounting, and route cache trustworthy.

## 3. Stream lifecycle

State per active stream (fixed pool, `max_streams`, default 200 — sized
to the ARCHITECTURE.md §1 ceiling, not below it):

```
key(origin_system, stream_tag), bridge*, src_id, dst_id, flags,
t_start, t_last_frame, frames_forwarded, frames_dropped, state
```

- **Stream open — any traffic frame** with an unknown key opens the
  stream: a real CALL_START when the origin delivered a header, or a
  bare VOICE frame on late entry (headerless streams flow headerless
  end-to-end — FRAME.md §4.2). Try to acquire the bridge talker lock
  (§4). Acquired → ACTIVE, forward, emit `call_start` event (noting
  headerless where true). Refused → stream CONTENDED: record it, forward
  nothing, emit `stream_contention` once; subsequent frames of a
  CONTENDED stream are silently dropped (one log line per lost call,
  not 50).
- **Traffic on an ACTIVE stream** (VOICE, and unknown in-stream types per
  FRAME.md §6 rule 2): update `t_last_frame`, forward.
- **CALL_END** (real terminator only) for an ACTIVE stream: forward,
  release the talker lock, emit `call_end` event (duration, frames,
  reason), free the stream slot.
- **Timeout.** Core scans the stream pool on a 500 ms timer;
  `now - t_last_frame > 2.0 s` → free the stream slot, release the
  bridge, emit `call_end` with `reason=timeout` — and
  send **nothing** downstream. Loss of signal is the RF world's steady
  state: every far edge has native stream-timeout/hang handling of its
  own (D-21). Adapters likewise never synthesize timeout terminators;
  ingress expires its stream_tag mappings silently, egress expires its
  per-stream state on its own inactivity timer.

**Pool exhaustion.** Every fixed pool in a no-malloc daemon needs a
stated behavior at the limit, or an implementer invents one. In all
cases: refuse the *new* arrival, never evict a live one, emit a
rate-limited event, and keep running.

| pool | limit | on exhaustion |
|---|---|---|
| core stream table | `max_streams` (200) | drop the opening frame, `stream_pool_full` |
| unit route cache | `unit_cache_max` (4096) | LRU-evict the oldest entry (§6) |
| per-peer stream pool (OBP) | per-adapter config | drop, `stream_pool_full` with system+peer |

Sustained exhaustion is a misconfiguration or an attack, not a traffic
condition — at D-22 scope 200 concurrent streams is already generous —
so the event is alarm-worthy and the dashboard should surface it.

Unit calls (`NXF_UNIT`): phase 5, see §6. Until then: drop + event.

## 4. Per-bridge talker arbitration

Exactly one stream may hold a bridge at a time.

```
bridge runtime: holder(stream key | none)
```

- Acquire: granted if there is no holder. Released on CALL_END or
  timeout, immediately and unconditionally.
- A CONTENDED ingress stream does **not** queue; DMR has no floor-control
  queueing. It is reported and discarded. If the holder ends while the
  contender is still transmitting, the contender stays lost (its
  CALL_START already passed; we do not splice mid-call — matches
  hblink3/4 behavior).

### There is no hang time on a bridge

**Hang time holds a radio access channel, and a bridge has no channel.**
Its purpose on RF is to keep a physical timeslot assigned to a talkgroup
so a conversation is not trampled by a *different* talkgroup competing
for the same slot. A bridge carries exactly one conversation by
definition, so there is nothing for it to hold and nothing to hold it
against. Hang time lives at the adapter, on `(system, slot)` — the point
where we actually interface with a radio channel (§5, D-15, D-23).

Getting this wrong is not a subtle bug. A per-bridge hang timer that
refuses a new talker would refuse **every** talker: every contender on a
bridge is "same talkgroup, different user," which is precisely the case
both ancestors admit —

```python
# hblink4/hblink.py:1757-1762 — group-call hang
if current_stream.rf_src == rf_src:
    return False  # Same user, allow through (TG switch)
if current_stream.dst_id == dst_id:
    return False  # Same TG, different user — allow
```

hblink3 is the same shape: its `GROUP_HANGTIME` checks fire only when
`_target['TGID'] != RX_TGID`, and a same-TGID contender is gated by
`STREAM_TO` (0.36 s), not by hang. A round-table net under a per-bridge
hang would lose every second station for the hang duration. It also has
no RF analog — a repeater's hang keeps the repeater keyed, it never locks
out another user — so D-21 rejects it independently.

## 5. Slot is an egress property

For each egress member on a slotted protocol (HBP, IPSC), the member config
carries `slot = 1 | 2`. The core stamps nothing — the egress **adapter**
reads the member slot (passed at config time as part of its per-system TG
map) and owns slot arbitration:

- Per (system, slot) the adapter tracks the current outbound stream. If
  bridge A and bridge B both try to egress on system S slot 1
  concurrently, the adapter forwards the first and drops the second with a
  `slot_busy` event. This contention is invisible to the core *by design* —
  slot scarcity is an edge problem, and pushing it to the edge is what
  keeps the core protocol-agnostic.
- **Hang time lives here and only here** (§4). Per (system, slot) the
  adapter keeps `hang_tgid`, `hang_src`, `hang_until`; on stream end it
  sets `hang_until = now + system.hang_time` (config, per `[[system]]`,
  default 4.0 s, 0 disables). While in hang, admission mirrors the
  ancestors exactly:

  | contender | admitted? | why |
  |---|---|---|
  | same TGID, any source | **yes** | the conversation the hang exists to protect |
  | same source, different TGID | **yes** | one user switching talkgroups |
  | different TGID, different source | **no** — `slot_busy` | a foreign talkgroup trying to seize a held channel |

  This is the whole point of hang time: it protects a talkgroup's grip on
  a physical channel from *other talkgroups*, never from other members of
  that talkgroup. Getting the second row wrong locks out the net (§4).
- **A short contention horizon is a separate timer from stream timeout.**
  A stream whose frames simply stop must free its slot fast enough that
  the next talker is not refused: the adapter releases a silent outbound
  stream after `stream_to` (default **0.36 s**, hblink3's `STREAM_TO`),
  well before the core's 2.0 s `stream_timeout` frees the core-side stream
  state. Two timers, two jobs — conflating them holds a slot for 2 s after
  every dropped stream.
- Unslotted egress (OBP, CC-per-its-rules) has no such arbitration;
  OBP carries many concurrent streams natively (wire slot field fixed per
  OBP convention, slot 1).
- **Delivery truth (D-15).** The operator must always be
  able to see what actually happened, and the mechanism is events, not
  protocol: the core's `call_start`/`call_end` events state which
  members a stream was forwarded to (intent); an egress adapter that
  drops emits `slot_busy` / `pfmt_unsupported` / `peer_down` events
  (outcome); the unified log and dashboard join the two. The core does
  not track per-member outcomes and keeps forwarding regardless — the
  wasted frames cost nothing at our scale, and slot scarcity is
  mechanism, not policy (D-23): the core has nothing useful to say
  about it.
- **Per-endpoint delivery filtering happens below the member level.**
  Within an HBP system, a repeater's subscription (D-03) may decline
  delivery of traffic the core routed to that system — a limiter at the
  edge, invisible to the core, which routes to *members* (systems),
  never to endpoints. Bridge rules remain the sole routing authority.
- **A locally repeated stream produces two `call_start` events, and that
  is intended.** The adapter emits one with `local:true` (what the system
  did) and the core emits one for the bridged copy (what the router did).
  They are the same transmission and carry the same
  `(origin_system, stream_tag)`; consumers **must** key on that pair and
  treat `local:true` as a delivery fact rather than a second call, or
  last-heard double-counts every locally repeated stream (DASHBOARD.md §2).
- **Egress is suppressed to a system that is not ready.** A system whose
  adapter has reported down via `nx_core_system_state` — or, for an
  outbound client, has not completed login — is skipped, and its name is
  omitted from the `call_start` "members forwarded to" list. hblink3 does
  this (`egress_ready()`), and it matters for more than wasted frames: the
  intent list is what D-15 asks operators to trust, and a list that claims
  delivery to a dead system is worse than no list.
- **Local traffic ties down slots too.** An HBP or IPSC system can carry
  in-system traffic (local repeat) that occupies a slot without ever
  transiting the core. This is precisely why slot state cannot live in
  the core: the adapter is the only component with the whole truth about
  its slots. Local streams participate in slot arbitration exactly like
  core egress, and are reported on the event bus like any call, tagged
  `local:true`.

## 6. Unit (private) calls — target route cache (phase 5 build, designed now)

The reference implementation is hblink4's `user_cache.py` — the most
robust unit handling in the family — promoted from per-server to
core-wide. A **target route cache** in the core:

```
src_id → { system, peer, slot, last_tgid, last_heard }
```

- Fed passively from **every** ingress traffic frame (CALL_START and
  VOICE alike — a long transmission keeps its talker routable), using
  fields the frame already carries: `origin_system`,
  `origin_peer`, `origin_slot`. No adapter cooperation needed beyond
  honest metadata.
- Entry timeout 600 s (hblink4's default), config `unit_cache_timeout`;
  fixed pool, LRU eviction (default 4096 entries); periodic sweep on the
  core's housekeeping timer.
- Routing a stream with NXF_UNIT: look up `dst_id`. Hit → forward to that
  single system with an `nx_egress_target` carrying the cached `peer` and
  `slot` (ADAPTERS.md §1) and `tgid = dst_id` unchanged. That struct is
  the only path by which a remembered slot reaches an adapter — it travels
  as a delivery parameter, never as a routing key. Miss or expired → drop + `unit_no_route`
  event (matches hblink4: never flood unit traffic).
- Bridges are not consulted for unit calls; talker arbitration §4 does
  not apply (unit streams contend only at the egress edge, §5). Loop
  safety comes from single-target forwarding plus §7.
- The cache is also what the dashboard's last-heard view renders (via
  `call_start` events), so it earns its memory twice.

## 7. Loop prevention

1. Never forward a frame to its `origin_system` (structural, core).
2. Never forward a frame to the member it just came from via another
   route: impossible by §2's unique-member rule within one core.
3. Edge dedupe (mechanism): the OBP adapter drops re-received copies of
   a stream it is already handling (per-peer stream-id dedupe) — the
   common OBP reflection case, caught at the edge.
4. **Same (src, dst) is NOT loop evidence (D-25).** Radio IDs are not
   unique transmitter identities — current ham practice encourages one
   ID across all of an operator's radios, so two radios sharing an ID on
   two systems (a couple in conversation, say) legitimately produce
   same-src, same-TG streams in close succession or overlap. An
   overlapping same-src stream is therefore ordinary contention: bridge
   arbitration drops the newcomer (as it drops any contender) and the
   `stream_contention` event notes `same_src: true` — honest data, no
   guessing. (Automatic same-src suppression would be traffic-redundant
   with arbitration anyway and could only mislabel events; it is
   deliberately absent.)
5. **Loops are detected as a pattern, never per-frame.** By header
   metadata alone, an echo returning in the hang window is
   indistinguishable from a same-ID reply — same src, same TG, from a
   just-delivered-to system, shortly after stream end — so the datapath
   does not guess. What no human conversation produces is the *cadence*:
   a circulating loop re-captures the bridge within network RTT of each
   stream end, repeatedly. The core counts consecutive same-src
   hang-window re-captures whose turnaround is sub-human
   (`loop_gap`, default 0.5 s); at `loop_count` (default 4) it raises
   `loop_suspected` — an alarm-class event the dashboard must surface
   prominently. Loops are misconfigurations; the fix is operator
   action. (Phase 6 option, per D-23 policy: a configurable circuit
   breaker that temporarily disables the offending member.)
6. OBP federation preserves `origin_peer`/`src_id` (D-07, PRESERVE_SOURCE_PEER
   semantics), so the same observations
   hold across federated cores. TTL fields remain deliberately absent.

## 8. Config shape (TOML, parsed by lifted toml.c)

```toml
[core]
stream_timeout = 2.0
event_socket   = "/var/run/omnilink.sock"
log_file       = "/var/log/omnilink.log"
log_level      = "info"

[[system]]
name      = "KS-DMR-HBP"
protocol  = "hbp"
mode      = "master"
bind      = "0.0.0.0:62031"
passphrase = "s3cr3t"
hang_time = 4.0      # per-(system,slot) channel hang -- §4, §5
stream_to = 0.36     # silent-stream slot release; distinct from [core] stream_timeout
# ... per-protocol keys per ADAPTERS.md

[[system]]
name     = "BM-3102"
protocol = "obp"
# ...

[[bridge]]
name      = "STATEWIDE"
member    = [
  { system = "KS-DMR-HBP", tgid = 3120,  slot = 1 },
  { system = "BM-3102",    tgid = 3120 },
  { system = "MOTO-EAST",  tgid = 2,     slot = 2 },
]
```

Config load validates: unique system names, unique `(system, tgid)` across
all bridges, slot present iff the member's protocol is slotted,
**single-talkgroup systems (CC conduits, XLX connections) appearing in at
most one bridge and never as a unit-call target (D-27)**, **XLX members
carrying no explicit `tgid`/`slot` (injected as 9/2, ADAPTERS.md §6.2)**,
and builds the immutable tables of ARCHITECTURE.md §5. Validation errors must name
the offending line *and the remedy* — in particular, the duplicate
`(system, tgid)` rejection (D-05) must say plainly that one TGID on one
system belongs to exactly one bridge.

Be careful with the suggested remedy. hblink3 indexes `(system, ts, tgid)`
to a *list* and fans an ingress stream to every matching bridge, so one
TGID on one system feeding several bridges is legal there and migrating
configs will contain it. "Model it as two systems" is sound advice for a
per-slot split, and **wrong** for this case: an HBP master serving real
repeaters cannot be duplicated — the repeaters are connected to one
socket. The honest remedy is to merge those bridges, or to renumber the
TGID on one of them; the rejection message must say that rather than
sending the operator after an impossible split.

Static rules only in phase 1. The hblink3 rules engine's dynamic layer
(ACTIVE/ON/OFF, timeouts, trigger TGIDs) is a known, wanted feature —
`enabled` exists on members from day one so the dynamic layer (phase 6)
is additive: timers and triggers flip `enabled`, nothing else changes.
