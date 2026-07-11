# ROUTING.md — Routing Core Semantics

The core is deliberately small: it owns the bridge table, the active-stream
table, per-bridge talker arbitration, and the event bus. It knows nothing
about any protocol, FEC, slots, pacing, or peers-as-sockets. Everything in
this file happens in the core module, on the daemon's single thread
(ARCHITECTURE.md §2).

## 1. Vocabulary

- **System** — one configured protocol instance (an HBP master, an IPSC
  network, an OBP peer, a CC conduit, a PORT link). Immutable table,
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
  the member's TGID. The egress adapter assigns the member's configured
  slot where its protocol needs one (§5).

Lookup structure: a hash map `(system u16, tgid u32) → member*` built once
at config load; members carry their bridge back-pointer.

Ingress is **unconditional**: adapters forward every stream to the core,
including streams also delivered in-system by local repeat (ADAPTERS.md)
and streams that match no bridge. An unbridged stream still updates the
target route cache (§6) and emits a rate-limited `unbridged` event before
being dropped — the core sees everything happening at every edge, which is
what makes the unified dashboard, accounting, and route cache trustworthy.

## 3. Stream lifecycle

State per active stream (fixed pool, default 64 concurrent):

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
  FRAME.md §6.2): update `t_last_frame`, forward.
- **CALL_END** (real terminator only) for an ACTIVE stream: forward,
  release the talker lock into hang time, emit `call_end` event
  (duration, frames, reason), free the stream slot.
- **Timeout.** Core scans the stream pool on a 500 ms timer;
  `now - t_last_frame > 2.0 s` → free the stream slot, release the
  bridge into hang time, emit `call_end` with `reason=timeout` — and
  send **nothing** downstream. Loss of signal is the RF world's steady
  state: every far edge has native stream-timeout/hang handling of its
  own (D-21). Adapters likewise never synthesize timeout terminators;
  ingress expires its stream_tag mappings silently, egress expires its
  per-stream state on its own inactivity timer.

Unit calls (`NXF_UNIT`): phase 5, see §6. Until then: drop + event.

## 4. Per-bridge talker arbitration and hang time

Exactly one stream may hold a bridge at a time.

```
bridge runtime: holder(stream key | none), hang_src, hang_until
```

- Acquire: granted if no holder AND (`now >= hang_until` OR
  `src_id == hang_src`). The hang-time exception lets the same talker
  continue a conversation across transmissions without being trumped, and
  lets anyone in after the hang expires — the hblink4 hang-time semantic,
  promoted from per-slot to per-bridge.
- Release (CALL_END or timeout): `hang_src = src_id`,
  `hang_until = now + bridge.hang_time` (config, default 4.0 s, 0 disables).
- A CONTENDED ingress stream does **not** queue; DMR has no floor-control
  queueing. It is reported and discarded. If the holder ends while the
  contender is still transmitting, the contender stays lost (its
  CALL_START already passed; we do not splice mid-call — matches
  hblink3/4 behavior).

## 5. Slot is an egress property

For each egress member on a slotted protocol (HBP, IPSC), the member config
carries `slot = 1 | 2`. The core stamps nothing — the egress **adapter**
reads the member slot (passed at config time as part of its per-system TG
map) and owns slot arbitration:

- Per (system, slot) the adapter tracks the current outbound stream + its
  own hang time mirroring the repeater's behavior. If bridge A and bridge B
  both try to egress on system S slot 1 concurrently, the adapter forwards
  the first and drops the second with a `slot_busy` event. This contention
  is invisible to the core *by design* — slot scarcity is an edge problem,
  and pushing it to the edge is what keeps the core protocol-agnostic.
- Unslotted egress (OBP, PORT, CC-per-its-rules) has no such arbitration;
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
  fields the canonical frame already carries: `origin_system`,
  `origin_peer`, `origin_slot`. No adapter cooperation needed beyond
  honest metadata.
- Entry timeout 600 s (hblink4's default), config `unit_cache_timeout`;
  fixed pool, LRU eviction (default 4096 entries); periodic sweep on the
  core's housekeeping timer.
- Routing a stream with NXF_UNIT: look up `dst_id`. Hit → forward to that
  single system, `dst_id` unchanged; the egress adapter delivers toward
  the cached `peer` and, on a slotted protocol, uses the cached `slot` —
  the one legitimate use of remembered slot, and it travels via the
  cache, never as a routing key. Miss or expired → drop + `unit_no_route`
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
6. PORT carries `origin_peer`/`src_id` intact, so the same observations
   hold across federated cores. TTL fields remain deliberately absent.

## 8. Config shape (TOML, parsed by lifted toml.c)

```toml
[core]
stream_timeout = 2.0
event_socket   = "/var/run/omnilink.sock"
log_file       = "/var/log/omnilink.log"
log_level      = "info"

[[system]]
name     = "KS-DMR-HBP"
protocol = "hbp"
mode     = "master"
bind     = "0.0.0.0:62031"
passphrase = "s3cr3t"
# ... per-protocol keys per ADAPTERS.md

[[system]]
name     = "BM-3102"
protocol = "obp"
# ...

[[bridge]]
name      = "STATEWIDE"
hang_time = 4.0
member    = [
  { system = "KS-DMR-HBP", tgid = 3120,  slot = 1 },
  { system = "BM-3102",    tgid = 3120 },
  { system = "MOTO-EAST",  tgid = 2,     slot = 2 },
]
```

Config load validates: unique system names, unique `(system, tgid)` across
all bridges, slot present iff the member's protocol is slotted, and builds
the immutable tables of ARCHITECTURE.md §5.

Static rules only in phase 1. The hblink3 rules engine's dynamic layer
(ACTIVE/ON/OFF, timeouts, trigger TGIDs) is a known, wanted feature —
`enabled` exists on members from day one so the dynamic layer (phase 6)
is additive: timers and triggers flip `enabled`, nothing else changes.
