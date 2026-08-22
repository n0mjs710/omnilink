# ROUTING.md — Routing Core Semantics

The core owns the bridge table, the stream table, per-bridge
arbitration, the dynamic-rule engine, and the event bus. It
reads the DMRD header and the `bits` byte (FRAME §2) and knows nothing
else about any protocol: no FEC, no framing, no sockets, no slot state,
no pacing.

**hblink3 is the specification** (D-03). Where this document is silent,
`bridge.py` is the answer. Departures are marked **[departure]**; there
are three — §3.1, §4.3, §5.1.

## 1. Vocabulary

- **System** — the set of endpoints that hear each other without the
  router. One `[[system]]` stanza; immutable after startup; indexed by a
  `uint16_t`.

  The term comes from IPSC, where a system *is* the peer-to-peer mesh. It
  carries to HBP as a server and its connected clients, which hear each
  other through that server's local repeat. Same meaning, different
  mechanism — which is why local repeat is an HBP adapter duty and does
  not exist on IPSC (D-18). Bound endpoints are the degenerate case: one
  connection, nothing to circulate.
- **Bridge** — a named conference, and the unit of routing. The only way
  traffic moves between systems.
- **Member** — one system's participation in one bridge:
  `(system, slot, tgid)` plus dynamic-rule state. Members live in the
  reloadable rules table.
- **Stream** — one transmission, keyed by its tag (FRAME §3), bounded by
  a voice header … terminator, or by timeout.
- **Endpoint** — a device below a system: a client on an HBP server, a
  peer on an IPSC network. The neutral term used where the core must span
  protocols. The core routes to *members*, never to endpoints.

## 2. Ingress

**Ingress is unconditional.** Adapters forward every *admitted* stream to
the core, including streams local repeat also delivered in-system and
streams that match no bridge. An unmatched stream still feeds the unit
route cache (§6) and emits a rate-limited `unbridged` event before being
dropped.

A router that sees only the traffic it routes cannot tell an operator
what their network is doing, and the dashboard, accounting, and route
cache would all be lying.

**ACLs are admission, not routing, and the core has no ACL layer**
(D-31). Traffic an adapter refuses never reaches the core; the adapter
events the denial. Enforcement is at the edge because the thing ACLs
exist to control — local in-system repeat — never transits the core
(ADAPTERS §1.3).

**Bridge rules are the routing whitelist.** A TGID that is no member's
TGID on that `(system, slot)` matches nothing and goes nowhere. There is
no second gate to configure and no way for the two to disagree.

### 2.1 Bridge lookup

```
BRIDGE_SRC_INDEX[(system u16, slot u8, tgid u32)] -> member list
```

Built once per rules load, each entry carrying its bridge back-pointer
and sibling member list. The value is a **list**: one ingress stream may
match several bridges and is routed into every one (D-04).

Two gates, both hblink3's:

- The matched **source member** must be `active` (§4).
- Each **target member** must be `active` and must not belong to the
  origin system. Loop prevention here is by *system*, not by member,
  which is what makes a same-system TS1↔TS2 bridge entry safe to write.

The core hands each surviving target's `(tgid, slot, endpoint)` to its
adapter as delivery parameters. **The core does not modify the packet**
— the adapter rewrites the destination, in the header and the LC
together, when it copies (FRAME §4.1).

## 3. Stream lifecycle

State per active stream, from a fixed pool sized `max_streams` (default
200, sized to the D-20 ceiling and not below it):

```
tag, delivery_set[], bridges_held[], origin_system, src_id, dst_id, slot,
t_start, t_last_packet, packets_forwarded, packets_dropped,
rules_generation, state
```

- **Stream open — any traffic packet** with an unknown tag. A real voice
  header when the origin delivered one; a bare voice burst on late entry,
  in which case the stream is headerless and flows headerless end to end
  (D-14).
- Activation triggers fire (§4.2), then bridge lookup runs, then
  arbitration is attempted per matched bridge (§5.1), then the delivery
  set is resolved (§3.1). At least one hold → ACTIVE, forward, emit
  `call_start`. No holds → CONTENDED: record it, forward nothing, emit
  `stream_contention` **once**; later packets of a CONTENDED stream are
  dropped silently. One log line per lost call, not fifty.
- **Traffic on an ACTIVE stream** updates `t_last_packet` and forwards to
  the delivery set.
- **A real terminator**: forward, fire deactivation triggers (§4.3),
  release every held bridge, emit `call_end`, free the slot.
- **Timeout.** The core scans the pool on a 500 ms timer;
  `now - t_last_packet > stream_timeout` (default 2.0 s) frees the slot,
  releases the bridges, fires deactivation triggers (§4.3, D-30), emits
  `call_end` with `reason=timeout` — and sends **nothing** downstream
  (D-14).

**`rules_generation`** pins the stream to the rules arena it opened
under, so a reload mid-call cannot pull a member out from under a live
transmission (D-09).

**Pool exhaustion** — refuse the new arrival, never evict a live one,
emit a rate-limited event, keep running (D-23):

| pool | limit | on exhaustion |
|---|---|---|
| core stream table | `max_streams` (200) | drop the opening packet, `stream_pool_full` |
| unit route cache | `unit_cache_max` (4096) | LRU-evict oldest (§6) |
| per-peer stream pool (OBP) | per-adapter config | drop, `stream_pool_full` with system + peer |

### 3.1 The delivery set is resolved once, and only grows **[departure]**

The delivery set — the members a stream is forwarded to — is resolved at
stream open and cached against the tag. Later packets replay it rather
than re-deriving. hblink3 re-derives per packet (D-29).

- **Removals are not applied in flight.** A member going inactive
  mid-transmission keeps receiving until the call ends.
- **Additions are applied immediately.** A member becoming active
  mid-transmission is added to every in-flight stream on its bridges and
  begins receiving as a headerless late entry.

> KANSAS has a call in progress and WEST is not an active member. A user
> on WEST keys the trigger TGID to bring WEST in. Their own transmission
> is refused, because KANSAS is held. Splice WEST into the call already
> running and they unkey into the middle of someone else's transmission
> and know to wait. Do not, and they hear silence, conclude the bridge is
> idle, key up and talk, and are dropped again with no indication that
> nobody heard them.

So: **a stream's delivery set never shrinks during its lifetime.**

## 4. Dynamic rules

Per-member state, reproducing hblink3's engine including its asymmetries.
These are not conveniences layered on routing; they *are* routing, and
they are built in phase 1 (D-12).

```
active   bool      # gates ingress and egress (§2.1)
to_type  NONE|ON|OFF
timeout  seconds   # config states MINUTES; ×60 at load
timer    time_t
on[]     trigger TGIDs that activate this member
off[]    trigger TGIDs that deactivate it
reset[]  trigger TGIDs that reset its timer without changing state
```

**At load:** `timer = now + timeout` if the member loads `active`,
otherwise `timer = now`.

**Bound endpoints carry no trigger state.** An `obp`, `cc`, or `xlx`
member loads with `to_type = NONE` and empty lists, and the validator
rejects any attempt to configure otherwise (CONFIG §4). A trunk, conduit,
or reflector link has no user to key a trigger TGID, and §4.2–§4.3 only
act on members of the originating system, so nothing could fire them.
Such members are never visited by the sweep. Their `active` flag still
works and is still operator-controllable (§4.5).

### 4.1 Trigger timing is asymmetric

**Activation fires at call start, so the operator's identification is not
truncated.** The common configuration has trigger TGID equal to member
TGID — trigger on 314, talk on 314. An operator brings the bridge up the
way operators actually do: they key up and say "N0XYZ monitoring."
Firing at call start passes that transmission through, and the bridge
hears who arrived. Firing at unkey would eat it entirely.

Not an FCC identification question — the repeater identifies itself — but
the human convention, which is the one that matters operationally.

The principle generalizes: **kerchunking is poor form and it exists, but
addressing it here would break the polite operator's standard procedure.**
Optimize for correct operating practice, not against incorrect practice.

The concern that this lets kerchunks flood the routing system mostly
cannot apply, because routing is by TGID: a trigger on a TGID no member
carries fires the trigger and then finds nothing to route to. The only
case that forwards is trigger-TGID equals member-TGID, which is the case
above.

**Deactivation fires at call end**, for the mirror reason: a transmission
that tears the bridge down should complete first.

Both are in-band signalling and have nothing to do with routing that
packet. Keep the two paths distinct.

**Triggers fire even when the triggering stream is itself dropped.** A
stream that loses arbitration forwards nothing, but its triggers still
run — that is exactly the §3.1 case, where an operator keys up to bring a
bridge in while it is already busy. Skipping trigger processing for a
dropped stream is a plausible optimization and it breaks this.

### 4.2 At call start, for each member of the origin system

Matching requires `slot == member.slot`. For each member where
`dst_id ∈ member.on ∪ member.reset`:

- If `dst_id ∈ member.on` and the member is inactive: set `active`, set
  `timer = pkt_time + timeout`, emit a bridge-state event. If
  `to_type == OFF`, additionally cancel the timer (`timer = pkt_time`) —
  an OFF-type timeout counts down toward *activation*, and the member is
  now already active.
- Then, if the member is active and `to_type == ON`, reset
  `timer = pkt_time + timeout`.

### 4.3 At call end, for each member of the origin system

Matching requires `slot == member.slot`.

**"Call end" means end of stream from any cause — a real terminator or a
timeout. [departure]** (D-30.)

- **Traffic on the member's own TGID resets its timer.** If
  `dst_id == member.tgid` and either (`to_type == ON` and active) or
  (`to_type == OFF` and inactive): `timer = pkt_time + timeout`. This is
  what makes "stays up while people are talking on it" work.
- For each member where `dst_id ∈ member.off ∪ member.reset`:
  - If `dst_id ∈ member.off` and the member is active: clear `active`,
    emit a bridge-state event, and if `to_type == ON` cancel the timer.
  - If the member is now inactive and `to_type == OFF`, reset
    `timer = pkt_time + timeout`.
  - If the member is still active, `to_type == ON`, and
    `dst_id ∈ member.off`, cancel the timer.

### 4.4 The timer sweep

Runs on a **10 s** period (hblink3's `run_periodic(10, rule_timer_loop)`;
its "run every minute" comment is stale — match the code).

- `to_type == ON`, active, `timer < now`: **deactivate** — *unless a call
  is in progress on that member's `(system, slot)` whose RX TGID equals
  the member's TGID*, in which case defer by extending
  `timer = now + timeout` and emit a deferral event. Timing a bridge out
  from under a live conversation is the most user-visible way to get this
  wrong.
- `to_type == OFF`, inactive, `timer < now`: **activate**.
- `to_type == NONE`: never touched.

The deferral checks the member's own **RX** state, so it protects the
member where the RF user is talking, not members receiving bridged
traffic — which is one of the ways a delivery set can change mid-stream
(§3.1).

Any state change emits a bridge-state event; a no-op sweep pushes
nothing.

### 4.5 Operator control

The control socket can `enable` and `disable` a member or a whole bridge
at runtime — the same `active` flag the trigger engine drives, one
mechanism with two drivers. An override is evented like any other state
change and persists only until the next reload or restart; the file is
the source of truth.

## 5. Arbitration, slots, and hang time

### 5.1 Per-bridge talker holder **[departure]**

Exactly one stream holds a bridge at a time (D-16).

- **Acquire** when the stream opens, per matched bridge. A stream
  matching several bridges takes what it can get and forwards to the
  bridges it holds; refusals are noted in the `stream_contention` event.
- **Release** at call end or timeout, immediately and unconditionally.
- **No queueing.** DMR has no floor control. A refused stream is reported
  and discarded, and if the holder ends mid-contender the contender stays
  lost — its call start has already passed and streams are never spliced
  mid-call. Both ancestors behave this way.
- **No hang time on a bridge, ever** (D-16).

### 5.2 Slot arbitration and hang live at the adapter

The egress adapter owns all slot truth (D-17), because an HBP or IPSC
system carries local traffic that occupies slots without transiting the
core.

- Per `(system, slot)` the adapter tracks the current outbound stream.
  Two bridges egressing to one `(system, slot)` concurrently: first wins,
  second is dropped with `slot_busy`. Invisible to the core by design.
- **Hang time lives here and only here.** Per `(system, slot)` the
  adapter keeps `hang_tgid`, `hang_src`, `hang_until`; on stream end it
  sets `hang_until = now + system.group_hangtime` (default 4.0 s, 0
  disables). Admission during hang:

  | contender | admitted? | why |
  |---|---|---|
  | same TGID, any source | **yes** | the conversation the hang protects |
  | same source, different TGID | **yes** | one user switching talkgroups |
  | different TGID, different source | **no** — `slot_busy` | a foreign talkgroup seizing a held channel |

  Getting the second row wrong locks out the net.
- **The contention horizon is a separate timer from stream timeout.** A
  stream whose packets stop must free its slot fast enough that the next
  talker is not refused: the adapter releases a silent outbound stream
  after `stream_to` (default **0.36 s**, hblink3's `STREAM_TO`), well
  before the core's 2.0 s `stream_timeout`. Two timers, two jobs;
  conflating them holds a slot for two seconds after every dropped
  stream.
- Unslotted egress has no such arbitration.
- **Local traffic ties down slots too**, participates in the same
  arbitration state, and is evented `local:true` (D-17).

### 5.3 Delivery filtering happens below the member level

A repeater's subscription may decline delivery of traffic the core routed
to its system (D-03). That is a limiter at the edge, invisible to the
core, which routes to **members** — systems — and never to endpoints.
Bridge rules remain the sole routing authority; a subscription can only
narrow what a system already carries.

## 6. Unit (private) calls — target route cache (phase 6)

hblink4's `user_cache.py` is the reference, promoted core-wide (D-26).

```
src_id -> { system, endpoint, slot, last_tgid, last_heard }
```

- Fed passively from **every** ingress traffic packet — headers and voice
  bursts alike, so a long transmission keeps its talker routable.
- Entry timeout `unit_cache_timeout` (default 600 s); fixed pool, LRU
  eviction at `unit_cache_max` (default 4096); swept on the 10 s tick.
- Routing a unit stream: look up `dst_id`. **Hit** → forward to that one
  system with the cached endpoint and slot as delivery parameters and
  `tgid = dst_id` unchanged. That is the only path by which a remembered
  slot reaches an adapter, and it travels as a delivery parameter, never
  as a routing key. **Miss or expired** → drop, emit `unit_no_route`.
  Never flooded.
- Bridges are not consulted and §5.1 does not apply; unit streams contend
  only at the egress edge.
- CC-CC and XLX are rejected as unit targets at rules load (D-07).
- The cache also backs the dashboard's last-heard view.

Until phase 6: unit streams drop with an event.

## 7. LC coherence on retarget

Source and destination appear **twice** — in the DMRD header and inside
the Link Control — and every retarget must keep them in agreement. A
mismatch is inaudible to every test that does not decode audio, and wrong
on the air.

This is a property of DMR routing, not of one protocol: HBP carries the
LC inside the burst and IPSC carries a DMR LC of its own that dmrlink3
rewrites in place.

**The egress adapter owns this**, because it is the component that copies
the packet (FRAME §4.1). Generate the replacement LCs once per stream per
target and splice per burst by position; never re-derive per packet, and
never let the header and the LC be rewritten by different code paths.
hblink3's `gen_lcs()` / `embed_lc()` pair is the reference.

A standing bug class, not a one-time task, and the reason every adapter
owes a loopback-identity vector (D-11, FRAME §6).

## 8. Loop prevention

1. **Never forward a packet to its origin system.** Structural, in the
   core, by system rather than by member.
2. **Edge dedupe.** The OBP adapter drops re-received copies of a stream
   it is already handling, per-peer by wire stream ID — the common
   reflection case, caught where it happens.
3. **Same `(src, dst)` is not loop evidence** (D-19). Overlapping
   same-source streams are ordinary contention; the event notes
   `same_src: true`.
4. **Loops are a cadence.** The core counts consecutive same-source
   hang-window re-captures whose turnaround is sub-human (`loop_gap`,
   default 0.5 s) and raises `loop_suspected` at `loop_count` (default 4)
   — alarm class, surfaced prominently.
5. **Federation preserves origin metadata.** OBP with
   `preserve_source_peer` keeps the originating repeater ID and `src_id`
   intact across instances, so these observations hold across federated
   cores. TTL fields stay absent.

## 9. What the core does not do

- Enforce ACLs (D-31, ADAPTERS §1.3).
- Parse the burst (D-25).
- Modify the packet (FRAME §4.1).
- Track per-member delivery outcomes (D-17).
- Hold slot, hang, or endpoint state (D-16, D-17).
- Resolve names or IDs (D-10).
- Synthesize a packet, a header, or a terminator (D-14).
- Allocate on the datapath (D-23).
