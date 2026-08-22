# DECISIONS.md — Design Decisions

One record per decision: what it is, and why where the why is not
obvious. Mechanism lives in the normative documents; this file does not
repeat it.

Records are revisable. Changing one is a design change, made
deliberately, with every document that depends on it updated in the same
pass.

> The design preceding 2026-08-21 was withdrawn in full. If a document,
> comment, or issue describes a representation-neutral core, an internal
> frame struct, TGID-only routing, a PORT trunk, a Playback adapter,
> foreign-mode adapters, or "no config reload," it is a leftover.

---

## D-01 — Name, family, license

**OmniLink**, in the hblink/dmrlink family. GPLv3, copyright Cortney T.
Buffington (N0MJS) — matching every ancestor repo, and required by the
GPL'd modules it lifts.

## D-02 — OmniLink is a DMR application, end to end

Not a multi-mode gateway and not mode-agnostic. The core's language is
DMR: 24-bit IDs, DMR Link Control, A–F burst positions, 60 ms voice
frames. No transcoding, no vocoder work, no non-DMR mode anywhere.

A proposal requiring OmniLink to understand a second air interface is out
of scope, not deferred.

## D-03 — hblink3 is the specification; OmniLink replaces hblink3 and
## dmrlink3; HBlink4 stays

**hblink3's routing semantics are the specification**, not an
inspiration: they are what an operator's existing `rules.py` *means*, and
OmniLink is only useful if those files carry across. Where the documents
are silent on a routing question, `bridge.py` is the answer. Departures
are marked `[departure]` at their site in ROUTING.md; there are three
(D-16, D-29, D-30).

OmniLink takes the routing-core role that hblink3 and dmrlink3 fill
today. Both are retired from it once the parity gate passes (D-12).

**HBlink4 stays** as the device-facing edge access server, interconnected
over OpenBridge. Two riders:

- A repeater's subscription filters **everything delivered to that
  repeater** — local repeat and bridge egress alike — and is a **limiter
  only**: it can decline traffic its system already carries, never cause
  bridging. Documentation must state the footgun plainly: subscribing to
  a TGID buys delivery *if the system carries it*, not a bridge to it.
- Endpoint classification (hotspot vs. repeater) comes from HBP login
  software/package fields only. Dashboard colour; no behaviour.

## D-04 — The routing key is `(system, slot, TGID)`, and one ingress may
## feed several bridges

hblink3 indexes `BRIDGE_SRC_INDEX[(system, slot, dst_id)]` to a **list**
and fans one ingress stream into every matching bridge. OmniLink does the
same.

Slot belongs in the key because it is a real distinction: TG 3120 on TS1
and on TS2 of one HBP server are two logical channels, separately ACL'd
(`TGID_TS1_ACL` / `TGID_TS2_ACL`), and operators route them differently.

Keying on TGID alone would require rejecting a member that appears in two
bridges, and the only remedy such a rejection could offer — "model it as
two systems" — is impossible for an HBP server whose repeaters are
connected to one socket.

Unslotted systems get a **nominal** slot at rules load (D-07).

## D-05 — The interchange unit is the DMRD packet, untouched

What crosses the adapter/core boundary is the HBP DMRD packet itself,
unmodified, plus two scalars the wire has no room for:

```c
void nx_core_ingress(uint16_t sys, uint32_t tag, const uint8_t *dmrd, int len);
```

Layout, the `bits` byte, the stream tag, and the burst rules are in
FRAME.md.

**The core is not protocol-agnostic, by design.** Abstraction away from
DMRD was only ever justified by a claim we do not make. OmniLink is DMR
only (D-02), so there is no second air interface to stay neutral toward;
HBP/MMDVM is dominant, OBP and XLX are DMRD-shaped, IPSC and CC-CC are
legacy support. We favour DMRD because the network does. Abstracting cost
cycles on the dominant path and added a translation layer for errors to
hide in.

Consequences:

- HBP, OBP, and XLX hand the core exactly what arrived. IPSC re-packs the
  elements its own wire carries — `ipsc2hbpc` already composes this
  packet. CC-CC assembles one.
- The core reads the `bits` byte for slot, call type, frame type, and
  vseq. hblink3 has always worked this way.
- **The two hardest adapters lift verbatim.**
  `ipsc2hbpc/src/translate.c` already builds a DMRD and calls
  `hbp_send_dmrd()`; that call becomes `nx_core_ingress()`.
- No byte-exact internal format to specify, test, and keep correct.

**The tag travels beside the packet, not inside it.** Overloading DMRD's
`stream_id` field would make a packet mean two things depending on where
it is, with no compiler help and a silent failure when an untagged packet
reaches the core. Passing it as an argument also preserves the
originating wire `stream_id` for cross-system debugging.

Two costs, accepted:

- **LC-in-payload duplication is permanent.** Source and destination
  appear in both the header and the Link Control, and every retarget must
  keep them in sync. Not an HBP quirk — IPSC carries a DMR LC too.
  ROUTING §7 is the standing rule.
- **Retargeting clobbers Talker Alias**, which rides in the embedded LC
  on bursts B–E. A real problem to solve on its own merits; no internal
  representation would have avoided it.

## D-06 — OpenBridge is the trunk

Instance-to-instance federation is OpenBridge, not a bespoke protocol. An
OBP hop is near-passthrough, so a purpose-built trunk's one selling point
— zero translation loss — is not a differentiator. It also deletes an
adapter, a wire format, and an authentication scheme from a project whose
scope is modest by design (D-20).

The origin system index and the stream tag do not cross an OBP hop; the
far end derives the system from which peer the traffic arrived on and
allocates its own tag.

**Both ends of an OmniLink↔OmniLink link are ours**, so OBP can be
extended there without anyone's cooperation — as `PRESERVE_SOURCE_PEER`
already was. Repeater-facing OBP, which must interoperate with
BrandMeister and DMR+, gets no such liberties.

## D-07 — Bound endpoints: one member syntax, protocol-determined
## validation

> **A bound endpoint is a system whose bridge identity is carried by the
> connection rather than by anything in the packet.**

OBP, XLX, and CC-CC are one pattern, previously special-cased separately.
The config expression is one member syntax; the named system's protocol
decides which fields are required, optional, or an error (CONFIG §4). The
special-casing lives in the validator, where it can produce a specific
message with a remedy, rather than in the grammar, where it becomes
boilerplate.

**Injected versus exposed is about observability.** A value whose wrong
setting fails silently is injected and supplying it is an error; a value
whose wrong setting is visible is exposed.

- **XLX TS2/TG9** — injected. No acknowledgement exists and no packet
  carries module identity, so mis-addressed traffic vanishes without
  symptom.
- **OBP slot 1** — injected. It is the OpenBridge specification, and
  BrandMeister, DMR+, IPSC2, TGIF, FreeDMR and anything else conforming
  will discard a call on slot 2. The `both_slots` override exists only
  because both ends of an OmniLink↔OmniLink or OmniLink↔HBlink link are
  ours (D-06); without that flag, `slot` on an OBP member is a config
  error rather than a default to override.
- **CC-CC local TGID** — exposed. A wrong value appears immediately as
  traffic under an unexpected talkgroup, and the specification expects
  each end to configure it independently.

**XLX and CC-CC join exactly one bridge**, enforced at rules load. Two
failures follow otherwise, and the second is why the rule exists:

1. Ingress is ambiguous — nothing on the wire says which bridge a stream
   belongs to, so one stream would have to become two.
2. Coherent egress would require U-turn replication: traffic arriving
   from bridge A belongs on bridge B as well, which means reversing
   direction inside the core. Decline it and the topology is incoherent
   in a way no operator would predict — A and B both reach the far
   reflector but never each other. Accept it and the endpoint has become
   a bridge-joiner, which is what a bridge already is.

Both are also excluded as unit-call targets. The CC-CC specification puts
private calls out of scope; for XLX the stakes are higher, since module
selection *is* a private call to 4001–4026 and a stray one could move the
reflector for every user on it.

**The single-member-syntax half of this decision is provisional.** The
alternative is hblink3's shape — separate tables per kind, where a wrong
value is inexpressible rather than rejected. One syntax is being run
first because a bridge's membership stays in one readable place, which is
the daily operation. Revisit on operator experience and settle before
wide deployment; the validator's remedy text is the mitigation meanwhile.

## D-08 — Single-threaded by default, because the work is I/O bound

One thread, one `ev_loop`, direct function calls.

**The reason is the workload, not a principle.** OmniLink waits on
network I/O; it is not CPU bound. At D-20 scope, even the egress fan-out
that dominates the work (ARCHITECTURE §1) sits comfortably inside one
core. A thread would buy stall isolation and nothing else, and that
benefit is already bounded by a systemd watchdog plus instance
federation: a stalled adapter stalls the daemon, the watchdog restarts it
in about a second, and calls in flight are lost exactly as they would be
at an RF site taking a power hit.

**This is a preference with fallbacks, not a prohibition** — the same
shape as D-32. Threads are not the thing being avoided, and neither is
lock-free coordination. **Locking is.** Order of preference:

1. **One thread on the event loop.** The default, and correct while the
   daemon is I/O bound.
2. **Lock-free threading** — atomics for state, a
   single-producer/single-consumer ring for packets. Both are fine when a
   thread earns its place.
3. **Locks.** Where it starts to get ugly: contention, ordering, and
   deadlock reasoning that spreads well beyond the code that took the
   lock. Last resort, and usually a sign the split is in the wrong place.

What is rejected is the everything-is-a-thread habit — reaching for
concurrency by default and paying its coordination cost whether or not
the workload asks for it.

**Adding a thread is a design change**, decided deliberately with the
docs updated, not something introduced mid-task because it seemed
convenient. The adapter contract is a clean seam if that day comes:
adapters could sit behind rings without the core changing.

libpthread is excluded today by this decision, not by the dependency
policy (D-32).

**Why C:** neither operational pain being solved here (unforgiving rules
files, chained daemons) is a language problem — D-09 and D-03 fix those.
C is chosen for a single binary with no fragile dependencies (D-32) and
the direct lift of field-proven code that already exists in the right
shape. The cost is plain: the ACL grammar, the dynamic-rule engine, and
the validator are where C buys least and costs most, and D-12 puts all
three on the critical path.

## D-09 — Two config files; rules and ACLs reload live

Config splits along the line hblink3 already draws with `hblink.cfg` and
`rules.py`: `omnilink.toml` (systems, ports, credentials) requires a
restart; `rules.toml` (bridges, members, ACLs) reloads live. Rebinding
sockets and re-authenticating peers mid-flight drops calls anyway, so
runtime system changes would buy an operator nothing a restart does not.

Three binding properties, mechanism in CONFIG §6:

1. **Validate, then swap.** A file that fails validation changes
   *nothing*. This is what makes reload safe to use, and it is the actual
   cure for what hurts about `rules.py`.
2. **`omnilink --check-config`** runs the identical validator against
   files on disk, so a bad rules file is caught in a pre-commit hook
   rather than on a live network.
3. **In-flight streams finish on the rules they started under.**

**TOML**, for consistency: already the house format across ipsc2hbpc,
cc2obp, and dmr-talkback, with the parser already vendored. YAML has no
parser meeting D-32 worth vendoring. JSON has no comments.
SQLite would end git-tracked, diffable configs.

**The web editor is a separate application and the file stays
authoritative** (CONFIG §7). A database-backed editor would end diffable
history, break the SSH workflow operators use, and create a merge problem
where none exists.

## D-10 — Bidirectional control socket; the event contract is normative,
## the dashboard is not

One Unix socket, JSON-lines, carrying events out and commands in
(EVENTS.md). Bidirectional from the start because the same channel serves
reload (D-09), runtime bridge control, and any future editor —
retrofitting a control path later would touch every consumer.

**Binding:** the event schema, vocabulary, ordering guarantee, command
set, and socket contract. **Arbitrary:** the dashboard application. Any
consumer that speaks the socket is legitimate. Name and ID resolution is
consumer-side — no SQL, no ID database, no lookups in the daemon.

## D-11 — Conformance vectors are mandatory per adapter

Bit-representation mismatches between protocols are the dominant
cross-protocol failure mode and are invisible to every test that does not
decode audio. Every adapter ships ingress vectors, egress vectors, and
the loopback identity (FRAME §6) before it is called done.

Bought with debugging time: the c-Bridge's AMBE is a 3-lane block
interleave of the ETSI d-bit order, resembling nothing in the ETSI
documents. Finding that cost a full cycle in cc2obp; the fix
(`cccc_ambe.c`) ports verbatim and is locked in by vectors.

## D-12 — hblink3 feature parity is a cutover gate, not a backlog

Because OmniLink retires hblink3, the features an existing network
depends on are day-one scope:

- **The full ACL model** — four types, the two-part grammar,
  global-then-system layering, first-denial-wins, fail-closed, `use_acl`
  and the one ACL it cannot disable (ADAPTERS §1.3, D-31).
- **The dynamic-rule engine** — `active`, `to_type`, `timeout`, and the
  `on`/`off`/`reset` trigger lists, with hblink3's exact activation,
  deactivation, timer-reset, and timeout-deferral behaviour (ROUTING §4).

A network cannot cut over to a router that will not express its current
rules or admit its current repeaters. Both are built in phase 1; the
parity gate (PLAN phase 3) runs OmniLink shadowed against a live hblink3
config and demonstrates identical routing decisions.

## D-13 — The RF-equivalence test (standing design filter)

DMR's air interface already survives late entry, gaps, vanished signals,
and missing headers *at the receiver*. A weird stream through OmniLink
should therefore degrade exactly like a weird signal on RF.

Intervene only where an IP failure mode has **no RF analog**: loops,
cross-network floor control, identity and admission, and operator truth.
Every feature proposal gets this test first.

## D-14 — Preserve call structure; synthesize nothing

Forward what was received; rebuild nothing.

- Real headers and terminators cross as real headers and terminators.
- A headerless (late-entry) stream flows headerless end to end.
- Burst position is preserved, never re-counted. Relabelling after loss
  would be editing the stream.
- On timeout the core frees state, releases arbitration, emits the event,
  and sends **nothing** downstream. Loss of signal is the RF world's
  steady state and every far edge handles it natively (D-13).

Routing rewrites addresses everywhere they appear (TGID, slot, LC
destination) and leaves structure and timing untouched.

**One carve-out: the IPSC egress wire.** `ipsc2hbpc` closes a drained or
vanished stream with a terminator (`translate.c`, `on_stream_timeout` →
`emit_term`), and that is retained because **it is what an IPSC peer
does**: a MOTOTRBO repeater that loses a stream generates a terminator
toward its own IPSC peers. Emitting one is conforming to the mesh's
behaviour, not inventing material — and the IPSC adapter *is* a peer in
that mesh (D-18).

(A repeater left without a terminator times out on its own; it does not
stay keyed. The reason for this behaviour is protocol convention, not a
stuck transmitter.)

It stays on the wire: never a routed packet, never back into the core. No
other adapter emits a terminator it did not receive.

## D-15 — No pacing, no loss concealment; the IPSC egress clock is the
## sole exception

Every endpoint stack owns its own jitter buffering. Forward on arrival;
gaps present as real gaps.

MOTOTRBO repeaters carry only a ~2-packet buffer, so timing must arrive
correct, and correcting timing requires a small buffer to correct it in.
That buffer is **position-preserving** — playout scheduled by sequence,
so a missing burst leaves its gap at the right instant. Per-system
configurable.

**On the IPSC wire only, an empty playout slot is filled with silence**
(`translate.c`, `tr->ambe_silence`). IPSC cannot express "nothing" at a
scheduled instant, so the choice is silence or a timing lie; the
position-preserving buffer exists to make it silence *at the right
moment*. Nothing is reconstructed and the synthesis count is evented. No
other adapter may do this.

## D-16 — Per-bridge talker holder; hang lives at the adapter, never on a
## bridge **[departure]**

hblink3 resolves contention only at the target `(system, slot)`, which is
complete for slotted targets and leaves unslotted members — OBP, CC-CC,
XLX — with no contention control at all, so two sources on one bridge
interleave into one conference.

OmniLink adds a **per-bridge holder**: one stream holds a bridge at a
time. Mechanism in ROUTING §5.1.

Per-`(system, slot)` arbitration and hang time stay at the adapter,
because they must: an HBP or IPSC system carries local traffic that
occupies a slot without transiting the core, so core-side slot state
would sometimes lie (D-17).

### There is no hang time on a bridge

Hang time holds a radio access channel, and a bridge has no channel. Its
purpose on RF is to keep a physical timeslot assigned to a talkgroup so a
conversation is not trampled by a *different* talkgroup. A bridge carries
one conversation by definition.

A per-bridge hang timer would refuse **every** contender, because every
contender on a bridge is same-talkgroup by construction — the case both
ancestors admit:

```python
# hblink4/hblink.py — group-call hang
if current_stream.rf_src == rf_src:
    return False  # Same user, allow through (TG switch)
if current_stream.dst_id == dst_id:
    return False  # Same TG, different user — allow
```

A round-table net under a per-bridge hang would lose every second
station. It also has no RF analog — a repeater's hang keeps the repeater
keyed, it never locks out another user — so D-13 rejects it
independently.

## D-17 — Slot truth at the adapter; delivery truth as events

The adapter owns all slot truth and arbitration (D-16). Operator
visibility is reconstructed from events rather than protocol: the core's
`call_start` states **intent** (members forwarded to); adapters event
their **outcomes** (`slot_busy`, `endpoint_down`,
`payload_unsupported`); the log and dashboard join the two. The core does
not track per-member outcomes and keeps forwarding — wasted frames cost
nothing at this scale, and slot scarcity is mechanism, not policy (D-21).

Two riders that are easy to get wrong:

- **Egress is suppressed to a system that is not ready**, and its name is
  omitted from the intent list. hblink3 does this (`egress_ready()`). A
  list claiming delivery to a dead system is worse than no list.
- **A locally repeated stream produces two `call_start` events**, one
  from the adapter with `local:true` and one from the core for the
  bridged copy. Same transmission, same tag. Consumers must fold them or
  last-heard double-counts every locally repeated stream.

## D-18 — Local repeat lives in the HBP adapter; IPSC has none

A *system* is the set of endpoints that hear each other without the
router (ROUTING §1). How that is achieved is protocol-specific, and this
decision is that difference.

An HBP server must repeat among its own clients — the client/server
obligation — inside the adapter, never via the core. The core still sees
every stream exactly once, because ingress is unconditional.

**IPSC is peer-to-peer**: each speaker replicates to all peers and the
master is only the peer-list bootstrap, so the IPSC adapter has no
local-repeat duty in any mode. It replicates its own core-egress traffic
like any speaking peer and tracks slot occupancy from observed peer
traffic so it never transmits into a busy slot.

## D-19 — Same `(src, dst)` is not loop evidence; loop detection is a
## pattern alarm

Hams share one radio ID across all their radios, so two radios with one
ID on two systems legitimately produce same-source, same-TG streams in
overlap. By header metadata alone, an echo returning within the hang
window is indistinguishable from such a reply.

So the datapath never guesses. An overlapping same-source stream is
ordinary contention, and the `stream_contention` event notes
`same_src: true` — honest data, no inference.

Loops are caught by the one signature no human conversation produces:
**cadence**. Mechanism in ROUTING §8. Loops are misconfigurations; the
fix is operator action. A configurable circuit breaker is available later
as policy (D-21).

## D-20 — Scope: state / regional / national-group; ~100 systems

Design ceiling of about 100 configured systems per instance. Explicitly
not a platform for a worldwide network. Past the ceiling, federate over
OBP (D-06) — for resilience, not performance; too many eggs in one basket
is an operational failure mode regardless of CPU headroom.

This scope is what makes D-08 correct rather than merely acceptable.
ARCHITECTURE §1 does the sizing.

## D-21 — Policy in the core, mechanism at the edge

The core decides *whether and where* a stream goes: bridge rules,
dynamic-rule timers and triggers, arbitration, the unit route cache,
loop observation. Single-copy policy is what makes one rules
file, one dashboard, and one truth possible.

Adapters decide *how* it goes: TS/TGID rewrite execution, slot
arbitration, hang, cadence, pacing, protocol state, local repeat, and
ACL admission — which is policy the operator writes but which can only
be enforced at the edge, because what it protects never transits the
core (D-31).

The same line explains why an adapter that drops on slot contention
merely events it, and why adapters never hold rule fragments.

## D-22 — Resolve hostnames at startup only

Both ancestors resolve hostnames *inside* the connect path, reached from
reconnect timers. At a hundred systems, many outbound by hostname, one
unresponsive resolver blocks the entire daemon for the resolver timeout
and drops every call on every system — and the systemd watchdog does not
fire, because the process is alive, just deaf.

Config load turns every hostname into a `sockaddr` once; the datapath and
reconnect timers never call `getaddrinfo`. An address change needs a
restart, consistent with D-09.

**A name that will not resolve at startup disables that one system and
nothing else.** Log it at error, mark the system disabled exactly as
`enabled = false` would, and carry on starting the rest. DNS is not ours
and it is not the operator's mistake; one unreachable peer must not take
down a router carrying ninety-nine working systems. Telemetry and
infrastructure the router does not control should never become
prerequisites for it to run.

**Restart is the retry mechanism.** In-process retry would bring a system
up after startup — mutating the immutable system table (D-09) and
allocating that system's sockets and pools mid-run, which is what the
startup-allocation model exists to prevent (D-23). A restart gets a
clean, bounded allocation. Reusing `enabled` also means no new state to
reason about.

**Do not make it exit, and do not restart it on a schedule.** A daemon
that quits because one system failed to resolve, under `Restart=`,
becomes a loop dropping every call on every system each time round — and
an automatic restart to chase a name that may still not resolve does the
same thing more slowly. Start degraded and stay up; a restart is an
operator action taken when the name is known to be fixed.

This does not argue against `Restart=` itself. Restarting a process that
has crashed or stalled is what bounds a wedged adapter (D-08). Restarting
a *healthy* daemon to re-read something external is the case that costs
more than it recovers.

Using a hostname is the operator's choice and carries this consequence.
Configure addresses instead and it cannot arise.

This is the single-thread model's real hazard, not a spinning protocol
bug — the watchdog covers those.

## D-23 — No allocation on the datapath; reload allocates on the control
## plane

All tables, stream slots, and per-endpoint pools are sized from config
and allocated once at startup. Steady state runs `malloc`-free.

**A rules reload may allocate** (D-09): it builds a new table arena,
validates, swaps a pointer, and releases the old arena when no in-flight
stream references it. The distinction must stay explicit in the code so
it is not "cleaned up" by someone reading only the headline.

Every fixed pool needs a stated behaviour at its limit, or an implementer
invents one: refuse the *new* arrival, never evict a live one, emit a
rate-limited event, keep running. Sustained exhaustion is a
misconfiguration or an attack, not a traffic condition, so those events
are alarm-worthy.

## D-24 — *(withdrawn)* Migration policy

Specified how the author's own networks would cut over from hblink3,
dmrlink3, and ipsc2hbpc. That is deployment practice, not project
design, and it belongs to whoever is doing the deploying. The number is
retired rather than reused.

Each phase still earns its role by passing its gate (PLAN.md); that is
the plan's own rule and needs no decision record.

## D-25 — The core routes on the DMRD header and never parses the burst

The core reads source, destination, repeater ID, and the `bits` byte, and
treats the burst as opaque (FRAME §2). Any feature requiring the core to
look inside a burst is a design violation: extend an adapter instead.
Talker alias, GPS, and data payloads all live in the burst.

Forward compatibility follows for free: an unrecognized
`frame_type`/`dtype_vseq` combination on a stream the core already has
open is forwarded like any other traffic.

IDs are DMR's 24 bits. Packed fields use explicit shift/mask accessors,
never C struct bitfields.

## D-26 — Unit (private) calls route from a target route cache

hblink4's `user_cache.py` is the reference, promoted from per-server to
core-wide. Fed passively from every ingress packet; a hit forwards to one
system, a miss drops with an event. Never flooded. Mechanism in ROUTING
§6; built in phase 6.

**A federated design cannot consume identifiers from a namespace it does
not administer** — a standing constraint, not a scope cut. TGID space is
operator-scoped and bounded by explicit bridge membership. Radio-ID space
is globally administered and unit routing is unbounded by construction.
Any future in-daemon virtual endpoint — recorder, announcer — must be
reachable by talkgroup, never by radio ID. BrandMeister does the opposite
and it works for them because they are a single administered network that
is the sole authority over its own namespace; OmniLink assumes an
internet of independent routers. The precedent does not transfer.

*Talkback/echo is not an OmniLink feature.* `dmr-talkback` is a finished
standalone product that connects as an ordinary HBP client.

## D-27 — Data calls ride opaquely, later

Voice is the first-class payload. Data rides as an ordinary DMRD packet
with a data-sync frame type, deliverable only between adapters that carry
bursts natively, and is built after the parity gate. Because the core
never parses the burst (D-25), this needs no core change when it lands.

No data-call reassembly, ever, in the daemon.

## D-28 — Colour code is RF-local: never read, never rewrite, 1 when
## invented

Colour code is an RF air-interface interference-mitigation mechanism: it
lets a receiver ignore a co-channel repeater it is not meant to hear. It
has no meaning between repeaters, and each applies its own before
transmitting.

Three rules, not one:

- **Never read it.** Not a routing key, not an ACL input, not a match
  condition, not a filter.
- **Never rewrite it.** A passed-through burst keeps whatever the origin
  stamped, so a stream may legitimately carry a foreign colour code end
  to end. Normalizing would re-encode the slot type and both EMB halves
  on every burst on the dominant path, to change a value nobody reads.
- **Use 1 when you must invent one**, from `dmr/`'s precomputed tables.

Colour code is not timeslot. A repeater must be *told* which slot a call
belongs in, because it has two and cannot infer the choice. It is never
told its colour code; it already knows it. Slot is information the
network owes the repeater; colour code is information the repeater owes
itself.

Consequences: no `colorcode` key exists for burst construction — the
value appears only in a system's login self-description (CONFIG §2.3),
and only on systems that log in.
`dmr/` is complete as vendored; no Golay(20,8) slot-type encoder is
needed and hblink3's `xlx_slot_type()` is not ported.

## D-29 — A stream's delivery set is resolved once and only grows
## **[departure]**

The core resolves a stream's delivery set at stream open and caches it
against the stream tag; later packets replay it. hblink3 re-derives per
packet.

- **Removals do not apply in flight.** Truncating a transmission because
  a timer expired has no RF analog (D-13).
- **Additions apply immediately**, as a headerless late entry.

A member brought into a busy bridge that
was *not* spliced into the call already running would hear silence,
conclude the bridge is idle, transmit, and be refused — with no
indication that nobody heard them. Splicing it in is the RF behaviour:
the operator unkeys into the middle of a transmission and knows to wait.
Silent-but-busy is an IP-only failure mode.

Ordering at call start: activation triggers fire **before** the delivery
set is resolved, so the transmission that brings a member in reaches it.

Resolving once also makes `call_start`'s intent list (D-17) a floor
rather than a snapshot.

## D-30 — Call end is call end, whatever caused it **[departure]**

Deactivation triggers and dynamic-rule timer resets fire when a stream
ends from **any** cause — a real terminator or a timeout.

hblink3 fires them only inside its real-terminator branch, so a stream
whose terminator is lost never tears its bridge down. DMR loses
terminators routinely — that is why stream timeout exists — and a bridge
an operator asked to drop should drop whether or not their last burst
survived the path.

The core already emits `call_end` with `reason=timeout` on that path, so
this costs nothing beyond running the same trigger block from both exits.
Nothing is synthesized downstream (D-14).

## D-31 — Traffic ACLs are enforced at the adapter; TGID ACLs are HBP-only

The core has no ACL layer. Bridge rules are already the routing
whitelist — a TGID that is no member's TGID on that `(system, slot)`
matches nothing and goes nowhere — so an ACL adds nothing to routing.

What an ACL does add is control over traffic that never reaches the core:
**local in-system repeat**. Without it, a user on one repeater can key an
invented TGID and have it repeated to every other repeater on that system,
and no bridge rule can stop that, because local repeat is not
bridge-driven (D-18). Enforcement therefore belongs where local repeat
happens — the HBP adapter.

TGID ACLs are HBP-only for the same reason:

- **IPSC cannot enforce it.** Peers hear each other directly and the
  router is not in that path, so there is nothing to filter.
- **OBP** has no need: its permitted TGIDs are exactly its bridge
  memberships, already fail-closed (D-07).
- **CC-CC and XLX** carry one talkgroup by construction.

`sub_acl` is enforced by every adapter at ingress, since keeping a banned
radio ID off the network applies wherever it enters, and hblink3 applies
it on OpenBridge too. `reg_acl` stays at login (HBP server, IPSC master).

hblink3 already works this way: `dmrd_acl_check` lives in `HBSYSTEM`, and
`bridge.py` never checks an ACL. Grammar, layering, and enforcement order
are in ADAPTERS §1.3.

**ACLs are admission, not routing.** Refused traffic never reaches the
core, so the core's view is of admitted traffic. The adapter events the
denial, and the dashboard shows it as a denial rather than as an
unbridged call.

## D-32 — Depend only on what is effectively part of C

Libraries are not avoided on principle. What is avoided is depending on
anything that cannot be counted on to still be there, still behave the
same way, and still compile, years from now.

**Acceptable:** the C11 standard library, POSIX 2008, and glibc — things
that are effectively part of the C environment, that change slowly if at
all, and that break backward compatibility only with a long and
well-signposted runway.

**Not acceptable:** everything else, however convenient.

Two reasons. The second is the more common failure:

1. **Bloat and unmeasured cost.** A library gets chosen on its API. What
   it does internally is rarely evaluated and almost never measured, so
   its cost lands somewhere in the hot path unexamined.
2. **Supply risk.** A third-party dependency can change its interface,
   change its behaviour, or disappear. Then the build breaks with no
   warning and no recourse, on someone else's schedule.

Three specific exclusions, none of which are judgements about quality —
all three are simply unnecessary here, and each one not taken is one
fewer thing to track:

- **OpenSSL** — `crypto.c` carries its own SHA/HMAC, small and already
  proven in the ancestors. OpenSSL is mainstream and *still* broke
  compilation across 1.0 → 1.1 → 3.0, which is reason 2 in its purest
  form: mainstream is not the same as stable.
- **cJSON or similar** — events are `printf`-formatted. The daemon only
  ever *emits* JSON and never parses it (D-10).
- **libevent** — `eventloop.c` is lifted from ipsc2hbpc and proven.

The result is a single binary an operator can build on any reasonably
current Linux and expect to keep building.

**Adding a dependency is a decision, not an implementation detail**
(D-33). Propose it, with what it buys and what it costs against the two
tests above; do not add one and mention it in a commit message.

## D-33 — Implementers recommend; the lead author approves

**The lead author is the single approving authority for this project.**
OmniLink is written with substantial help from coding tools, and those
tools produce good recommendations — but a recommendation is not a
decision, and the difference is deliberate.

- **Anyone implementing may propose anything**: a design change, a
  dependency, a departure from hblink3, a different structure. Proposals
  are welcome and expected; a tool that spots a real problem and says
  nothing is worse than one that argues.
- **Only the lead author approves.** Approval means the decision is
  recorded here, the documents that depend on it are updated in the same
  pass, and the change lands afterwards — not before.
- **Nothing lands on "it seemed better."** The docs are normative
  (STYLE.md). An implementer who finds them wrong writes a
  `docs/DEVIATIONS.md` entry, implements the smallest reasonable
  reading, and keeps going — that entry *is* the proposal.

The reason is not process for its own sake. Several tools contribute, at
different times, without shared memory of what was already weighed and
rejected. A decision that one of them makes unilaterally is invisible to
the next, and the corpus stops being a single coherent design. Routing it
through one person is what keeps it one design rather than a merge of
several.

This applies to everything with a decision record: routing semantics,
timers, protocol behaviour, dependencies (D-32), threading (D-08), config
shape. When in doubt, it is a decision.
