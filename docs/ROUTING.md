# ROUTING.md — Routing Core Semantics

The core owns the bridge table, the ACL layer, the active-stream table,
per-bridge talker arbitration, the dynamic-rule engine, and the event
bus. It knows nothing about any protocol, FEC, peers-as-sockets, pacing,
or wire framing. Everything in this file happens in the core module, on
the daemon's single thread (ARCHITECTURE.md §2).

**hblink3 is the specification** (D-03). Where this document is silent,
hblink3's `bridge.py` behavior is the answer. Where OmniLink departs
from it, the departure is called out explicitly and marked
**[departure]**; there is exactly one, in §5.

## 1. Vocabulary

- **System** — one configured protocol instance: an HBP master, an HBP
  peer, an IPSC network, an OBP link, a CC-CC conduit, an XLX module
  connection. One `[[system]]` stanza. Immutable after startup (D-09),
  indexed by a `uint16_t` used as `origin_system` in the frame.
- **Bridge** — a named conference, and the unit of routing. A bridge is
  the *only* way traffic moves between systems.
- **Member** — one system's participation in one bridge:
  `(system, slot, tgid)` plus its dynamic-rule state. Members live in
  the reloadable rules table (D-09).
- **Stream** — one transmission, keyed `(origin_system, stream_tag)`,
  bounded by CALL_START … CALL_END or by timeout.
- **Endpoint** — a device below a system: one repeater or hotspot on an
  HBP master, one peer on an IPSC network. The core routes to *members*,
  never to endpoints (§5).

## 2. Ingress

**Ingress is unconditional.** Adapters forward every stream to the core —
including streams also delivered in-system by local repeat (D-18) and
streams that match no bridge at all. An unmatched stream still feeds the
unit route cache (§6) and emits a rate-limited `unbridged` event before
being dropped.

This matters more than it looks: it is what makes the core's view
complete, and therefore what makes the unified dashboard, the accounting,
and the route cache trustworthy. A router that only sees the traffic it
routes cannot tell an operator what their network is doing.

### 2.1 ACL admission (D-12)

Every ingress traffic frame passes the ACL layer before anything else.
The model is hblink3's exactly, because operators' existing ACL strings
must keep meaning what they meant.

**Grammar.** One string, one action: `PERMIT:` or `DENY:` followed by a
list of IDs and/or inclusive ranges, or the keyword `ALL`. There is
exactly one ACTION per ACL — mixing permit and deny within one string is
not expressible and is a config error. An ACL simultaneously defines the
action for the IDs it lists **and the opposite action for every ID it
does not**; that second half is the part operators get wrong, and the
validator's error text says so.

**The four types.**

| ACL | Applies to | Range | Notes |
|---|---|---|---|
| `reg_acl` | endpoint registration (login) | 1 … 4294967295 | master-mode systems only; outbound and OBP links accept no registrations. **Always enforced** — `use_acl` cannot disable it. |
| `sub_acl` | source radio ID of a call | 1 … 16776415 | every voice and data stream |
| `tgid_ts1_acl` | destination TGID on slot 1 | 1 … 16776415 | |
| `tgid_ts2_acl` | destination TGID on slot 2 | 1 … 16776415 | |

**Order of enforcement**, first denial wins, all must pass:

1. global `sub_acl`
2. global `tgid_ts1_acl` / `tgid_ts2_acl`, selected by the stream's slot
3. system `sub_acl`
4. system `tgid_ts1_acl` / `tgid_ts2_acl`

Registration is separately gated: global `reg_acl` **and** system
`reg_acl` must both permit the endpoint or the login is refused.

**Fail closed.** A malformed ACL is a startup or reload failure, never a
silently-ignored ACL. hblink3 exits at startup on `ACL CREATION ERROR`
and that is correct; OmniLink refuses to start, or — on reload — refuses
the swap and keeps the previous rules (D-09).

**OpenBridge rider.** OBP systems do not register, so `reg_acl` does not
apply, and all OBP traffic is carried on slot 1 — so the **global
`tgid_ts1_acl`** is the control that governs it. Documentation must say
this plainly; an operator who filters OBP traffic with `tgid_ts2_acl` has
built a filter that can never fire.

ACL denials emit rate-limited events (`acl_denied` with the ACL that
fired), because an ACL that silently eats traffic is the hardest class of
misconfiguration to diagnose.

### 2.2 Bridge lookup

```
BRIDGE_SRC_INDEX[(system u16, slot u8, tgid u32)] -> member list
```

Built once per rules load, each entry carrying its bridge back-pointer
and its sibling member list. This is hblink3's index exactly, including
that the value is a **list**: one ingress stream may match several
bridges, and it is routed into every one of them (D-04).

Two gates apply, and both are hblink3's:

- The matched **source member** must be `active` (§4). An inactive
  source member routes nothing.
- Each **target member** must be `active`, and must not belong to the
  origin system. Loop prevention here is by *system*, not by member — a
  bridge containing two members on one system never delivers back into
  that system, which is what makes a same-system TS1↔TS2 bridge entry
  safe to write.

Egress rewrites `dst_id` to the target member's TGID and hands the
adapter the target's `(tgid, slot)` as its delivery parameters. The
adapter is *told* where to put the traffic; it never looks it up (D-21).

## 3. Stream lifecycle

State per active stream, from a fixed pool sized `max_streams`
(default 200, sized to the D-20 ceiling and not below it):

```
key(origin_system, stream_tag), bridges_held[], src_id, dst_id, slot,
flags, lc[9], t_start, t_last_frame, frames_forwarded, frames_dropped,
rules_generation, state
```

- **Stream open — any traffic frame** with an unknown key opens a
  stream. A real CALL_START when the origin delivered a header; a bare
  VOICE frame on late entry, in which case the stream is headerless and
  flows headerless end to end (D-14). The precomputed LC is built here,
  once, and reused for every retarget (§7).
- Bridge lookup runs, arbitration is attempted per matched bridge (§5),
  and the stream records which bridges it holds. At least one hold →
  ACTIVE, forward, emit `call_start` naming the members forwarded to and
  noting headerless where true. No holds → CONTENDED: record it, forward
  nothing, emit `stream_contention` **once**; subsequent frames of a
  CONTENDED stream are dropped silently. One log line per lost call, not
  fifty.
- **Traffic on an ACTIVE stream** — VOICE, plus unknown in-stream types
  per FRAME.md §6 — updates `t_last_frame` and forwards to the held
  bridges' members.
- **CALL_END**, real terminator only: forward, fire deactivation triggers
  (§4), release every held bridge, emit `call_end` with duration, frame
  counts, and reason, free the slot.
- **Timeout.** The core scans the pool on a 500 ms timer;
  `now - t_last_frame > stream_timeout` (default 2.0 s) frees the slot,
  releases the bridges, emits `call_end` with `reason=timeout` — and
  sends **nothing** downstream (D-14). Adapters likewise never
  synthesize terminators into the routing path; ingress expires its
  `stream_tag` mappings silently and egress expires per-stream state on
  its own inactivity timer.

**`rules_generation`** pins the stream to the rules arena it opened
under, so a reload mid-call cannot pull a bridge or member out from under
an in-flight transmission (D-09, D-23).

**Pool exhaustion** (D-23) — refuse the new arrival, never evict a live
one, emit a rate-limited event, keep running:

| pool | limit | on exhaustion |
|---|---|---|
| core stream table | `max_streams` (200) | drop the opening frame, `stream_pool_full` |
| unit route cache | `unit_cache_max` (4096) | LRU-evict oldest (§6) |
| per-peer stream pool (OBP) | per-adapter config | drop, `stream_pool_full` with system + peer |

## 4. Dynamic rules (D-12)

Per-member state, reproducing hblink3's engine including its asymmetries.
These are not conveniences layered on top of routing; they *are* routing,
and they are built in phase 1.

```
active   bool      # gates ingress and egress (§2.2)
to_type  NONE|ON|OFF
timeout  seconds   # config states MINUTES; ×60 at load
timer    time_t
on[]     trigger TGIDs that activate this member
off[]    trigger TGIDs that deactivate it
reset[]  trigger TGIDs that reset its timer without changing state
```

**At load:** `timer = now + timeout` if the member loads `active`,
otherwise `timer = now`.

### 4.1 Trigger timing is asymmetric, and deliberately so

**Activation triggers fire on CALL_START.** They must, so that the header
frame and every subsequent frame of the *triggering transmission* route
to the newly connected targets. Firing them at call end would connect the
bridge exactly one transmission too late — the operator keys up to bring
the link up and their own call is the one that gets dropped.

**Deactivation triggers fire on CALL_END**, for the mirror reason: a
transmission that tears the bridge down should itself complete first.

Both are in-band signalling and have nothing to do with routing that
frame. Implementations must keep the two paths distinct.

### 4.2 On CALL_START, for each member of the origin system

Matching requires `slot == member.slot`. For each member where
`dst_id ∈ member.on ∪ member.reset`:

- If `dst_id ∈ member.on` and the member is inactive: set `active`, set
  `timer = pkt_time + timeout`, emit a bridge-state event. If
  `to_type == OFF`, additionally cancel the timer (`timer = pkt_time`) —
  an OFF-type timeout counts down toward *activation*, and the member is
  now already active.
- Then, if the member is active and `to_type == ON`, reset
  `timer = pkt_time + timeout`.

### 4.3 On CALL_END, for each member of the origin system

Matching requires `slot == member.slot`.

- **Traffic on the member's own TGID resets its timer.** If
  `dst_id == member.tgid` and either (`to_type == ON` and active) or
  (`to_type == OFF` and inactive): `timer = pkt_time + timeout`. This is
  what makes "stays up while people are talking on it" work.
- For each member where `dst_id ∈ member.off ∪ member.reset`:
  - If `dst_id ∈ member.off` and the member is active: clear `active`,
    emit a bridge-state event, and if `to_type == ON` cancel the timer
    (`timer = pkt_time`).
  - If the member is now inactive and `to_type == OFF`, reset
    `timer = pkt_time + timeout`.
  - If the member is still active, `to_type == ON`, and
    `dst_id ∈ member.off`, cancel the timer.

### 4.4 The timer sweep

Runs on a **10 s** period (hblink3's `run_periodic(10, rule_timer_loop)`;
its "run every minute" comment is stale — match the code, not the
comment).

- `to_type == ON` and active and `timer < now`: **deactivate** — *unless
  a call is in progress on that member's `(system, slot)` whose RX TGID
  equals the member's TGID*, in which case defer by extending
  `timer = now + timeout` and emit a deferral event. Timing a bridge out
  from under a live conversation is the single most user-visible way to
  get this wrong.
- `to_type == OFF` and inactive and `timer < now`: **activate**.
- `to_type == NONE`: never touched by the sweep.

Any state change emits a bridge-state event so the dashboard reflects it
immediately; the periodic snapshot covers the rest, so a no-op sweep
pushes nothing (D-10).

### 4.5 Operator control

The control socket (D-10) can `enable` and `disable` a member or a whole
bridge at runtime. That is the same `active` flag the trigger engine
drives — one mechanism, two drivers — and an operator override is
reported on the event bus like any other state change. A `disable`
persists only until the next reload or restart; the file is the source of
truth (D-09).

## 5. Arbitration, slots, and hang time

### 5.1 Per-bridge talker holder **[departure]**

Exactly one stream holds a bridge at a time. This is OmniLink's one
deliberate departure from hblink3 (D-16): hblink3 arbitrates only at the
target `(system, slot)`, which is complete for slotted targets and leaves
unslotted members — OBP, CC, XLX — with no contention control at all, so
two sources on one bridge interleave into one conference.

- **Acquire** when the stream opens, per matched bridge. A stream
  matching several bridges (D-04) takes what it can get and forwards to
  the bridges it holds; bridges it was refused are noted in the
  `stream_contention` event.
- **Release** on CALL_END or timeout, immediately and unconditionally.
- **No queueing.** DMR has no floor control. A refused stream is reported
  and discarded, and if the holder ends mid-contender the contender stays
  lost — its CALL_START has already passed and streams are never spliced
  mid-call. Both ancestors behave this way.
- **No hang time on a bridge, ever.** The full reasoning is in D-16 and
  it is not a nuance: a per-bridge hang timer would refuse *every*
  contender, because every contender on a bridge is same-talkgroup by
  construction. A round-table net would lose every second station.

### 5.2 Slot arbitration and hang live at the adapter

For each egress member on a slotted protocol the member config carries
`slot`. The core stamps nothing — the egress **adapter** owns all slot
truth (D-17), because an HBP or IPSC system carries local in-system
traffic that occupies slots without ever transiting the core.

- Per `(system, slot)` the adapter tracks the current outbound stream. If
  two bridges egress to one `(system, slot)` concurrently, the first
  wins and the second is dropped with a `slot_busy` event. This
  contention is invisible to the core by design.
- **Hang time lives here and only here.** Per `(system, slot)` the
  adapter keeps `hang_tgid`, `hang_src`, `hang_until`; on stream end it
  sets `hang_until = now + system.group_hangtime` (default 4.0 s, 0
  disables). Admission during hang is the D-16 table, whose middle rows
  are the ones that matter: **same TGID from any source is admitted**,
  and **same source on a different TGID is admitted**. Only a different
  talkgroup from a different source is refused.
- **The contention horizon is a separate timer from stream timeout.** A
  stream whose frames simply stop must free its slot fast enough that the
  next talker is not refused: the adapter releases a silent outbound
  stream after `stream_to` (default **0.36 s**, hblink3's `STREAM_TO`),
  well before the core's 2.0 s `stream_timeout` frees core-side state.
  Two timers, two jobs. Conflating them holds a slot for two seconds
  after every dropped stream.
- Unslotted egress has no such arbitration; OBP carries many concurrent
  talkgroups natively and its wire slot field is fixed at 1 by
  convention.
- **Local traffic ties down slots too**, participates in slot
  arbitration exactly like core egress, and is reported on the event bus
  tagged `local:true` (D-17).

### 5.3 Delivery filtering happens below the member level

Within an HBP system, a repeater's subscription (D-03) may decline
delivery of traffic the core routed to that system. That is a limiter at
the edge, invisible to the core, which routes to **members** — that is,
to systems — and never to endpoints. Bridge rules remain the sole
routing authority; a subscription can only narrow what a system already
carries.

## 6. Unit (private) calls — target route cache (phase 6)

hblink4's `user_cache.py` is the reference, promoted from per-server to
core-wide (D-26).

```
src_id -> { system, endpoint, slot, last_tgid, last_heard }
```

- Fed passively from **every** ingress traffic frame — CALL_START and
  VOICE alike, so a long transmission keeps its talker routable — using
  metadata the frame already carries. No adapter cooperation required
  beyond honest metadata.
- Entry timeout `unit_cache_timeout` (default 600 s); fixed pool, LRU
  eviction at `unit_cache_max` (default 4096); swept on the core
  housekeeping timer.
- Routing a `UNIT` stream: look up `dst_id`. **Hit** → forward to that
  one system with the cached endpoint and slot as delivery parameters and
  `tgid = dst_id` unchanged. That struct is the only path by which a
  remembered slot reaches an adapter, and it travels as a delivery
  parameter, never as a routing key. **Miss or expired** → drop, emit
  `unit_no_route`. Never flooded.
- Bridges are not consulted and §5.1 does not apply; unit streams contend
  only at the egress edge. Loop safety comes from single-target
  forwarding plus §7.
- Bound endpoints (OBP aside) are rejected as unit targets at rules load
  (D-07).
- The cache also backs the dashboard's last-heard view, so it earns its
  memory twice.

Until phase 6: `UNIT` streams drop with an event.

## 7. LC coherence on retarget

Because the frame carries the on-air burst verbatim (D-05), source and
destination appear **twice** — in the frame header and inside the burst's
Link Control — and every retarget must keep them in agreement. A
mismatch is inaudible to every test that does not decode audio, and
wrong on the air.

The rule: **build the stream's LC once, at stream open, and splice it on
every egress with the target member's TGID.** hblink3 does exactly this
(`STATUS[slot]['RX_LC']`, built from the voice header when one arrived
and synthesized from the HBP header when the stream was headerless) and
that is the reference implementation. Never re-derive the LC per frame,
and never let the header and the LC be rewritten by different code paths.

This is a standing bug class, not a one-time task. It is why every
adapter owes a loopback-identity conformance vector (D-11, FRAME.md §7).

## 8. Loop prevention (D-19)

1. **Never forward a frame to its origin system.** Structural, in the
   core, by system rather than by member.
2. **Edge dedupe.** The OBP adapter drops re-received copies of a stream
   it is already handling, per-peer by stream ID — the common reflection
   case, caught where it happens.
3. **Same `(src, dst)` is not loop evidence.** Hams share one radio ID
   across their radios, so same-source same-TG streams in overlap are
   ordinary contention: arbitration drops the newcomer and the
   `stream_contention` event notes `same_src: true`. Honest data, no
   inference.
4. **Loops are a cadence, detected as a pattern.** The core counts
   consecutive same-source hang-window re-captures whose turnaround is
   sub-human (`loop_gap`, default 0.5 s) and raises `loop_suspected` at
   `loop_count` (default 4) — alarm class, surfaced prominently. Loops
   are misconfigurations; the fix is operator action. A configurable
   circuit breaker that temporarily disables the offending member is
   available later as policy (D-21).
5. **Federation preserves origin metadata.** OBP with
   `PRESERVE_SOURCE_PEER` keeps `origin_peer` and `src_id` intact across
   instances (D-06), so these observations hold across federated cores.
   TTL fields remain deliberately absent.

## 9. What the core does not do

- Parse payloads. Ever. It routes on the frame header (D-25).
- Track per-member delivery outcomes (D-17 — that is what events are
  for).
- Hold slot state, hang state, or peer state (D-16, D-17).
- Resolve names or IDs (D-10).
- Synthesize a frame, a header, or a terminator (D-14).
- Allocate on the datapath (D-23).
