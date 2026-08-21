# DECISIONS.md — Design Rationale

Each record states a design decision and the reasoning behind it. The
numbering is stable and referenced throughout the docs.

**These records are revisable.** A decision that meets reality and loses
gets superseded here, with the old reasoning kept and dated, because the
reasoning is what stops a settled question from being re-opened by
accident. What they are not is a discussion prompt: changing one is a
design change, made deliberately, with the docs that depend on it
updated in the same pass.

> **The design that preceded 2026-08-21 was withdrawn in full.** It made
> the core representation-neutral, routed on TGID alone, forbade config
> reload, and treated hblink3's ACLs and dynamic rules as optional
> extras. Every one of those is reversed below. Nothing in this corpus
> refers back to it; if you find a document that does, it is a miss in
> the rewrite.

---

## D-01 — Name, family, license

The project is **OmniLink**, in the hblink/dmrlink family tradition.
GPLv3, copyright Cortney T. Buffington (N0MJS) — matching every ancestor
repo, and required in any case by the GPL'd modules it lifts.

## D-02 — OmniLink is a DMR application, end to end

Not a multi-mode gateway and not digital-mode-agnostic. The core's
native language is DMR: 24-bit IDs, DMR Link Control, A–F burst
positions, ETSI payload kinds, 60 ms voice frames. There is no
transcoding, no vocoder work of any kind, and no non-DMR mode in the
core or at any edge.

This is a scope declaration with teeth: it is the reason the frame can
carry an on-air burst verbatim (D-05), the reason "translation" at the
legacy edges means bit-representation work rather than media work, and
the reason no adapter ever needs a DSP dependency. A proposal that
requires OmniLink to understand a second air interface is out of scope,
not deferred.

## D-03 — hblink3 is the semantic model; OmniLink replaces hblink3 and
## dmrlink3; HBlink4 stays

**hblink3's routing semantics are the specification**, not an
inspiration. Where this document set is silent on a routing question,
hblink3's behavior is the answer, and where OmniLink deliberately
departs from it the departure is written down as such (D-16 is the only
one so far). The reason is not sentiment: hblink3's semantics are what
the operator's existing `rules.py` files *mean*, and OmniLink is only
useful if those files can be carried across.

OmniLink takes over the **routing core** role — the conference-bridge
router that hblink3 and dmrlink3 fill today — and both are retired from
that role once the parity gate passes (D-12).

**HBlink4 stays** as the device-facing edge access server, interconnected
with OmniLink over OpenBridge. Two riders follow from that split:

- Within OmniLink's own HBP systems, a repeater's subscription (static
  ACL now, HBP `options` at login later) filters **everything delivered
  to that repeater** — local repeat and bridge egress alike. It can only
  *decline* traffic its system already carries; it can never cause
  bridging. Documentation must state this footgun plainly: subscribing
  to a TGID buys delivery *if the system carries it*, not a bridge to
  it. Bridge rules remain the sole routing authority.
- Endpoint classification (hotspot vs. repeater) comes from HBP login
  software/package fields only. It is dashboard color; it drives no
  behavior.

## D-04 — The routing key is `(system, slot, TGID)`, and one ingress may
## feed several bridges

hblink3 indexes `BRIDGE_SRC_INDEX[(system, slot, dst_id)]` to a **list**
of matching bridges and fans one ingress stream to every one of them
(`bridge.py`). OmniLink does the same, for two reasons.

**Slot is a real distinction on slotted systems.** TG 3120 on TS1 and TG
3120 on TS2 of one HBP master are two logical channels to the operator,
they are separately ACL'd (hblink3 has `TGID_TS1_ACL` and
`TGID_TS2_ACL`, not one TGID ACL), and operators route them differently.
A router that cannot express that cannot express what it is replacing.

**The alternative was tried on paper and its own escape hatch does not
exist.** Keying on TGID alone requires rejecting a member that appears
in two bridges, and the remedy such a rejection has to offer the
operator is "model it as two systems." For an HBP master serving real
repeaters that is impossible — the repeaters are connected to one
socket. A validation rule whose fix-it text cannot be followed is not a
constraint, it is a wall.

The stated cost of slot-in-the-key was that it pollutes the core with an
air-interface concept. That cost is already paid and already proven in
the field: hblink3 gives OpenBridge a nominal TS1 (overridable per
bridge) and XLX a fixed TS2, and has done so in production for years.
D-07 makes that pattern explicit and uniform rather than a set of
special cases.

Consequences carried forward:

- Lookup is `(system u16, slot u8, tgid u32) → member list`, built at
  rules load, each member carrying a back-pointer to its bridge.
- One ingress stream that matches *n* bridges produces *n* routed
  copies. This is intended. It is also why the per-bridge holder in
  D-16 acquires per bridge and forwards to whichever bridges it holds.
- Unslotted systems get a **nominal** slot at rules load, not a
  configured one (D-07).

## D-05 — The frame wraps the untouched 33-byte DMR burst

Voice rides the core as the **33-byte on-air burst**, unmodified — the
representation HBP and OBP already carry on the wire. IPSC and CC-CC
translate to and from it at their own edges. HBP is the dominant traffic
source, so the dominant path re-encodes nothing:

| path | conversions |
|---|---|
| HBP→HBP, HBP→OBP, OBP→OBP, HBP→XLX | **0** |
| IPSC→HBP, CC→HBP, IPSC→OBP | 1 |
| IPSC→IPSC, CC→CC, IPSC→CC | 2 |

The rejected alternative was a neutral internal representation (49-bit
AMBE with FEC and interleave stripped) that every adapter converted into.
It made the common path pay a full burst teardown and rebuild in order
to make the rarest path free, and it gave the core the shape of a
general-purpose media router rather than a DMR call router.

**The frame is not the DMRD packet, and that distinction is load-bearing.**
Carrying HBP's 55-byte wire packet as the internal unit would look like
an even purer version of this decision and is a trap. DMRD has no place
to put the things the core needs and the wire does not carry: which
configured system a frame arrived on, the core's own stream tag,
late-entry/headerless status, or call-type flags that outlive one
protocol's encoding. Worse, it would make IPSC and CC-CC adapters
fabricate a repeater ID and a stream ID just to say anything at all, and
it would promote OpenBridge's `PRESERVE_SOURCE_PEER` wart from a flag
into the structure of the daemon.

So: an explicit frame (FRAME.md), carrying explicit metadata, wrapping
the burst **byte-for-byte untouched**. Zero re-processing on the dominant
path, without the core speaking one protocol's header.

Accepted knowingly:

- Legacy↔legacy paths pay two conversions where a neutral core would
  have paid none. Lossless — the 72-bit FEC is derived from the 49 bits,
  so 49→72→49 recovers exactly — so this is wasted microseconds, not
  degraded audio, and it is the shrinking c-Bridge-to-c-Bridge case.
- **HBP's LC-in-payload duplication becomes a core concern.** Source and
  destination appear both in the frame header and inside the burst's Link
  Control, and every retarget must keep them in sync. This is a real,
  recurring bug class. hblink3 solves it with per-stream precomputed LCs
  and that approach is the reference implementation.
- A legacy→legacy path synthesizes burst structure (sync, EMB, slot
  type) that no downstream consumer reads. Bounded, and it comes from
  `dmr/`'s fixed tables (D-28) — never from a canned blob lifted out of
  a capture. The LC inside it is built from the real stream.

## D-06 — OpenBridge is the trunk

Instance-to-instance federation is **OpenBridge**, not a bespoke
protocol. With the burst as the frame payload (D-05) an OBP hop is
near-passthrough, so a purpose-built trunk's one selling point — zero
translation loss — is not a differentiator. What OBP does not carry
(`origin_system`, `vseq`, frame flags) the far end re-derives: the system
from which OBP peer the traffic arrived on, `vseq` from burst structure,
flags from the LC. Nothing is lost; some things are recomputed.

This also deletes an adapter, a wire format, and an authentication
scheme from a project whose scope is deliberately modest (D-20).

The extension door stays open on purpose: **both ends of an
OmniLink↔OmniLink link are ours**, so OBP can be extended there without
anyone's cooperation, exactly as `PRESERVE_SOURCE_PEER` already was in
hblink3 and cc2obp. Repeater-facing OBP, which must interoperate with
BrandMeister and DMR+, gets no such liberties.

## D-07 — Bound endpoints: one member syntax, protocol-determined
## validation

Three transports cannot express a full `(system, slot, tgid)` member the
way an HBP or IPSC system can, and each has been special-cased
separately in the past. They are one pattern:

> **A bound endpoint is a system whose bridge identity is carried by the
> connection rather than by anything in the frame.**

| system | binds | slot | TGID |
|---|---|---|---|
| OBP | per-TGID, many bridges per link | nominal TS1, overridable | **exposed** — it *is* the route, in both directions |
| XLX | whole connection, exactly one bridge | **injected** TS2 | **injected** TG 9 |
| CC-CC | whole conduit, exactly one bridge | **injected** nominal | **exposed** — the conduit's local TGID |

The config expression is **one member syntax for everything** (CONFIG.md
§4). The operator writes what is meaningful and omits what is not; the
named system's protocol determines which fields are required, optional,
or an error:

```toml
members = [
  { system = "KS-DMR",   slot = 1, tgid = 3120 },   # HBP: both required
  { system = "MOTO-EAST", slot = 2, tgid = 2 },     # IPSC: both required
  { system = "BM-3102",  tgid = 3120 },             # OBP: slot optional (nominal 1)
  { system = "CC-KSDMR", tgid = 3120 },             # CC: slot is an error
  { system = "XLX307-D" },                          # XLX: both are an error
]
```

No second table, no per-protocol syntax, nothing extra to remember. The
special-casing lives in the validator, where it can produce a specific
message with a remedy, instead of in the grammar, where it produces
boilerplate the operator has to get right by rote.

**Injected vs. exposed is about observability, not taste.** XLX's TS2/TG9
is a protocol constant whose wrong value is *silently* fatal — no
acknowledgement exists and no frame carries module identity, so
mis-addressed traffic vanishes without symptom. It is therefore injected,
and supplying it explicitly is an error. CC's local TGID is the opposite:
a wrong value is immediately visible as traffic under a consistent but
unexpected talkgroup, and the CC-CC specification expects each end to
configure it independently. So it is exposed.

**Single-bridge endpoints join exactly one bridge**, enforced at rules
load, because two independent failures follow otherwise and the second is
the load-bearing one:

1. *Ingress is ambiguous.* A stream arrives on the conduit with nothing
   to say which bridge it belongs to. One stream would have to become
   two.
2. *Coherent egress would require U-turn replication.* Traffic arriving
   from bridge A belongs, if the endpoint also sits on bridge B, on B as
   well — which means replicating and reversing direction inside the
   core. Decline that and the topology is incoherent in a way no
   operator would predict: A and B both reach the far reflector but never
   each other. Accept it and the endpoint has become a bridge-joiner,
   which is what a bridge already is.

Operators wanting two bridges joined join them. They do not do it
implicitly through a shared single-talkgroup endpoint.

Both are also excluded as unit-call targets. For CC the specification
puts private calls out of scope entirely. For XLX the stakes are higher:
module selection *is* a private call to 4001–4026, so a unit call
reaching an XLX system could silently move the reflector into a
different room for every user on it.

## D-08 — Single-threaded C11, one event loop, libc only

One thread, one `ev_loop`, direct function calls. No libpthread, no
atomics, no locks, no queues, no rings. Zero external library
dependencies.

At D-20 scope there is no performance or scaling argument for
concurrency (ARCHITECTURE.md §1 sizes it), and the one real thing
threads would buy — stall isolation, so a spinning protocol bug freezes
only its own protocol — is already bounded by a systemd watchdog plus
instance federation. A stalled adapter stalls the daemon; the watchdog
restarts it in about a second; calls in flight are lost exactly as they
would be at an RF site taking a power hit.

The adapter contract stays a clean seam — adapters could move behind
rings or sockets without core changes if the calculus ever changed — but
that is not a supported build target.

**Why C at all, stated honestly:** neither of the two real operational
pains being solved here (unforgiving rules files, chained daemons) is a
language problem, and C is not what fixes them — D-09 and D-03 are. C is
chosen for two things it genuinely provides: a dependency-free single
binary to deploy, and the direct lift of field-proven code that already
exists in exactly the right shape (ipsc2hbpc's IPSC engine and playout
clock, cc2obp's CC-CC link and its AMBE de-interleave fix,
`crypto`/`toml`/`net`/`log`/`eventloop`/`dmr`). The cost is equally
plain: the ACL grammar, the dynamic-rules engine, and the config
validator are where C buys the least and costs the most, and D-12 puts
all three on the critical path. That is a known, accepted trade.

## D-09 — Two config files; rules and ACLs reload live; validate-then-swap

Config splits along the line hblink3 already draws with `hblink.cfg` and
`rules.py`:

- **`omnilink.toml`** — core settings and `[[system]]` stanzas: binds,
  ports, credentials, protocol parameters, per-system timers. **Changes
  require a restart.**
- **`rules.toml`** — `[[bridge]]` definitions, their members, and the ACL
  layer. **Reloads live**, on `SIGHUP` or a `reload` command on the
  control socket (D-10).

The reload boundary is drawn there because rebinding sockets and
re-authenticating peers mid-flight drops calls anyway, so runtime system
changes would buy an operator nothing they cannot get from a restart —
while runtime *rules* changes are the thing operators actually want
hourly.

**Three properties are binding:**

1. **Validate, then swap.** The new rules are parsed and fully validated
   into a second table arena before anything is exchanged. A file that
   fails validation changes *nothing* — the daemon keeps routing on the
   old rules and reports the failure with the offending line and a
   remedy. This is the property that makes reload safe to use, and it is
   the actual cure for what hurts about `rules.py` today.
2. **`omnilink --check-config` is a first-class subcommand**, running the
   identical validator against files on disk and exiting non-zero on
   failure, so a bad rules file can be caught in a pre-commit hook or a
   deploy script instead of on a live network.
3. **In-flight streams finish on the rules they started under.** Table
   arenas are generation-tracked; the old arena is released when the last
   stream referencing it ends. Nothing is torn out from under a call.

**Format is TOML**, decided on consistency rather than syntax
preference: it is already the house format across ipsc2hbpc, cc2obp, and
dmr-talkback, and its parser is already vendored and proven. YAML has no
dependency-free C parser worth vendoring and would break D-08. JSON has
no comments and is unpleasant to hand-edit. SQLite is a dependency and
would end git-tracked, diffable, reviewable configs, which is how these
networks are actually operated.

**A migration tool ships with the parity gate.** `rules.py` is importable
Python; the converter imports an operator's existing file and emits
`rules.toml`, reporting anything it cannot represent rather than guessing
(PLAN.md phase 3).

**The web rules editor is a separate application, and the file stays
authoritative.** It reads and writes the same `rules.toml`, shells the
daemon's own validator, and asks for a reload over the control socket. It
never becomes a second source of truth — a database-backed editor would
end diffable configs, break the SSH workflow these operators actually
use, and create a merge problem where none exists. Because it is
decoupled this way, it can be built at any time and blocks nothing.

## D-10 — Bidirectional control socket; the event contract is normative,
## the dashboard is not

One Unix socket, JSON-lines, carrying two directions:

- **Out:** the unified event stream — every adapter and the core emit
  into a single ordered feed that drives the log and the dashboard.
  Single-threading makes strict total order trivial: emission order *is*
  the order.
- **In:** commands — `reload`, `check`, bridge/member `enable`/`disable`,
  `snapshot`, `stats`.

Making the socket bidirectional from the start is a deliberately cheap
decision taken early: the same channel serves config reload (D-09),
runtime dynamic-bridge control (D-12), and any future editor or GUI.
Retrofitting a control path onto a one-way event socket later would touch
every consumer.

**Binding:** the event schema, its vocabulary, its ordering guarantee,
the command set, and the socket contract. **Deliberately arbitrary:** the
dashboard application itself. Gen-1 is Python; any consumer that speaks
the socket is legitimate. Name and ID resolution is consumer-side, never
in the daemon — no SQL, no ID database, no lookups on the datapath.

## D-11 — Conformance vectors are mandatory per adapter

Bit-representation mismatches between protocols are the dominant
cross-protocol failure mode and are invisible to every test that does not
decode audio. Every adapter ships **ingress vectors, egress vectors, and
the loopback identity** (FRAME.md §7) before it is called done.

This rule was bought with debugging time. The c-Bridge's AMBE is a
**3-lane block interleave of the ETSI d-bit order**, resembling nothing
in the ETSI documents; finding that cost a full debugging cycle in
cc2obp, and the fix (`cccc_ambe.c`) ports into OmniLink verbatim and is
locked in by vectors so it can never silently regress.

## D-12 — hblink3 feature parity is a cutover gate, not a backlog

Because OmniLink retires hblink3 (D-03), the features an existing hblink3
network depends on are **day-one scope**, not "wanted extras":

- **The full ACL model** — `REG_ACL`, `SUB_ACL`, `TGID_TS1_ACL`,
  `TGID_TS2_ACL`; the two-part `PERMIT:`/`DENY:` grammar; global-then-
  system layering, drop at the first denial, fail-closed on a malformed
  ACL; `USE_ACL` and the one ACL it cannot turn off.
- **The dynamic-rules engine** — per-member `ACTIVE`, `TO_TYPE` ON/OFF,
  `TIMEOUT`, and the `ON`/`OFF`/`RESET` trigger-TGID lists, with
  hblink3's exact activation, deactivation, timer-reset, and
  timeout-deferred-during-a-call behavior.

Neither is optional. A network cannot cut over to a router that will not
express its current rules or admit its current repeaters. Both are built
in phase 1 as core semantics, and the parity gate (PLAN.md phase 3) is
passed by running OmniLink shadowed against a live hblink3 config and
demonstrating identical routing decisions.

## D-13 — The RF-equivalence test (standing design filter)

DMR's air interface already survives late entry, gaps, vanished signals,
and missing headers *at the receiver* — every consumer downstream of
OmniLink was built for weird RF. Therefore a weird stream through
OmniLink should degrade exactly like a weird signal on RF.

We intervene only where an IP failure mode has **no RF analog**: loops,
cross-network floor control, identity and admission, and operator truth.
Every feature proposal gets this test first. Re-solving ETSI's
receiver-side problems in the middle of the network is tail-chasing by
design.

## D-14 — Preserve call structure; synthesize nothing; the IPSC
## terminator is the one carve-out

Forward what was received; rebuild nothing.

- Real headers and terminators cross as real CALL_START and CALL_END.
- A headerless (late-entry) stream flows **headerless end to end** —
  egress builds a minimal LC from frame fields and emits no wire header.
- `vseq` is read once at ingress and followed at egress. Relabeling burst
  positions after loss would be editing the stream; missing bursts stay
  missing.
- On stream timeout the core frees state, releases arbitration, emits the
  event, and sends **nothing** downstream. Loss of signal is the RF
  world's steady state and every far edge handles it natively (D-13).

Addressing versus structure: routing rewrites addresses everywhere they
appear (TGID, slot, LC destination), and leaves structure and timing
untouched.

**One carved exception: the IPSC egress wire.** `ipsc2hbpc`, which is in
production and which the IPSC phase lifts, ends a drained or vanished
stream by emitting a terminator (`translate.c`, `on_stream_timeout` →
`emit_term`). That is not a defect to port out. IPSC has no
representation for "nothing," and a MOTOTRBO repeater left without a
terminator **stays keyed**. The rule above governs *frames*: nothing
fabricates a frame, no CALL_END is invented into the routing path, and no
other adapter emits a terminator it did not receive. What an IPSC egress
puts on its own wire to close a transmission the far repeater is
physically holding open is protocol machinery, and it stays.

## D-15 — No pacing, no loss concealment; the IPSC egress clock is the
## sole exception

Every endpoint stack owns its own jitter buffering. Forward on arrival;
gaps present as real gaps.

The exception exists because MOTOTRBO repeaters carry only a ~2-packet
buffer: timing must arrive correct, and correcting timing requires a
small buffer to correct it in. That buffer is **position-preserving** —
playout is scheduled by sequence (stream start + n × 60 ms) so a missing
burst leaves its gap at the right instant. Per-system configurable.

**And on the IPSC wire only, an empty playout slot is filled with
silence.** The production code this lifts emits a comfort-silence AMBE
burst when a scheduled slot comes up empty (`translate.c`,
`tr->ambe_silence`), because IPSC's wire cannot express "nothing" at a
scheduled instant — the choice is silence or a timing lie, and the
position-preserving buffer exists precisely to make it silence *at the
right moment*. Calling this concealment would be a stretch: nothing is
reconstructed, and the synthesis count is logged and evented, so no loss
is hidden from the operator. It applies nowhere else, and no other
adapter may do it.

## D-16 — Arbitration: a per-bridge talker holder, plus per-`(system,
## slot)` hang at the adapter — and never hang on a bridge

**This is OmniLink's one deliberate departure from hblink3 (D-03).**

hblink3 resolves contention only at the target: the first stream to
reach a `(system, slot)` holds it and later arrivals are dropped there.
That works because every hblink3 target of consequence is slotted. It
leaves a real hole once unslotted members are first-class: two sources
on one bridge have no slot to contend for on an OBP, CC, or XLX member,
so both are forwarded and interleave into one conference. That is
audible garble, and no amount of edge arbitration catches it.

So OmniLink adds a **per-bridge holder**: one stream holds a bridge at a
time, acquired when the stream opens, released unconditionally on
CALL_END or timeout. A stream matching several bridges (D-04) acquires
per bridge and forwards to whichever it holds; the rest are reported as
contention. A refused stream is **not queued** — DMR has no floor-control
queueing — it is reported and discarded, and if the holder ends while the
contender is still transmitting the contender stays lost. Splicing
mid-call is not done, matching both ancestors.

Per-`(system, slot)` arbitration and hang time remain at the adapter,
because they must: an HBP or IPSC system carries local in-system traffic
that occupies a slot without ever transiting the core, so core-side slot
state would sometimes lie. The adapter is the only component with the
whole truth about its slots (D-17).

### There is no hang time on a bridge

**Hang time holds a radio access channel, and a bridge has no channel.**
Its purpose on RF is to keep a physical timeslot assigned to a talkgroup
so a conversation is not trampled by a *different* talkgroup competing
for that slot. A bridge carries exactly one conversation by definition,
so there is nothing to hold and nothing to hold it against.

Getting this wrong is not a subtle bug. A per-bridge hang timer that
refused a new talker would refuse **every** talker, because every
contender on a bridge is "same talkgroup, different user" — precisely the
case both ancestors explicitly admit:

```python
# hblink4/hblink.py — group-call hang
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
out another user — so D-13 rejects it independently.

The adapter's `(system, slot)` hang admission table is therefore, exactly:

| contender | admitted? | why |
|---|---|---|
| same TGID, any source | **yes** | the conversation the hang exists to protect |
| same source, different TGID | **yes** | one user switching talkgroups |
| different TGID, different source | **no** — `slot_busy` | a foreign talkgroup seizing a held channel |

Getting the second row wrong locks out the net.

## D-17 — Slot truth lives at the adapter; delivery truth travels as
## events

Slot state cannot live in the core (D-16). The adapter owns all slot
truth and all slot arbitration.

Operator visibility is therefore reconstructed from the event stream
rather than from protocol: the core's `call_start` states **intent** (the
members a stream was forwarded to); adapters event their **outcomes**
(`slot_busy`, `peer_down`, `pfmt_unsupported`); the log and dashboard
join the two. The core does not track per-member outcomes and keeps
forwarding regardless — wasted frames cost nothing at this scale, and
slot scarcity is mechanism, not policy (D-21).

Two riders that are easy to get wrong:

- **Egress is suppressed to a system that is not ready.** A system whose
  adapter has reported down, or an outbound client that has not completed
  login, is skipped *and its name is omitted from the intent list*.
  hblink3 does this (`egress_ready()`), and it matters for more than
  wasted frames: the intent list is what operators are asked to trust,
  and a list claiming delivery to a dead system is worse than no list.
- **A locally repeated stream produces two `call_start` events, and that
  is intended.** The adapter emits one with `local:true` (what the system
  did) and the core emits one for the bridged copy (what the router did).
  Same transmission, same `(origin_system, stream_tag)`. Consumers
  **must** key on that pair and treat `local:true` as a delivery fact
  rather than a second call, or last-heard double-counts every locally
  repeated stream.

## D-18 — Local repeat lives in the HBP adapter; IPSC has none

An HBP master must repeat traffic among its own connected endpoints —
that is the client/server obligation — and it does so **inside the
adapter**, never via the core. The core still sees every stream exactly
once, because ingress is unconditional (ROUTING.md §2). Per-repeater
delivery filtering follows D-03.

**IPSC is peer-to-peer**: each speaker replicates its own traffic to all
peers and the master is only the peer-list bootstrap. The IPSC adapter
therefore has no local-repeat duty in any mode. It replicates its own
core-egress traffic like any speaking peer, and tracks slot occupancy
from observed peer traffic so it never transmits into a busy slot.

## D-19 — Same `(src, dst)` is not loop evidence; loop detection is a
## pattern alarm, never a datapath guess

Hams share one radio ID across all of their radios — current practice
actively encourages it — so two radios with one ID on two systems (a
couple in conversation, say) legitimately produce same-source, same-TG
streams in overlap or close succession. By header metadata alone, an echo
returning within the hang window is **indistinguishable** from such a
reply.

So the datapath never guesses. An overlapping same-source stream is
ordinary contention: arbitration drops the newcomer as it drops any
contender, and the `stream_contention` event notes `same_src: true` —
honest data, no inference.

Loops are caught by the one signature no human conversation produces:
**cadence**. A circulating loop re-captures a bridge within network RTT
of each stream end, repeatedly. The core counts consecutive same-source
hang-window re-captures whose turnaround is sub-human (`loop_gap`,
default 0.5 s) and raises `loop_suspected` at `loop_count` (default 4) —
an alarm-class event the dashboard must surface prominently.

Structural prevention stands independently: never forward a frame to its
origin system, and the OBP adapter drops re-received copies of a stream
it is already handling (per-peer stream-id dedupe), which catches the
common reflection case at the edge. Loops themselves are
misconfigurations; the fix is operator action, and a configurable circuit
breaker that temporarily disables an offending member is available later
as policy (D-21).

## D-20 — Scope: state / regional / national-group; ~100 systems;
## federate beyond

OmniLink is a tool for the state, regional, or national-group level, with
a design ceiling of about **100 configured systems per instance**. It is
explicitly not a platform for running a worldwide network, and a single
install will never carry several statewide networks' full loads.

Past the ceiling, deployments **federate** multiple instances over OBP
(D-06) — not for performance, but for resilience. Too many eggs in one
basket is an operational failure mode regardless of CPU headroom.

This scope is what makes D-08 correct rather than merely acceptable.
ARCHITECTURE.md §1 does the sizing, and the number that matters there is
egress fan-out, not ingress.

## D-21 — Policy in the core, mechanism at the edge

The core decides *whether and where* a stream goes: bridge rules,
dynamic-rule timers and triggers, ACL admission, arbitration, the unit
route cache, loop observation. Single-copy policy is what makes one rules
file, one dashboard, and one truth possible.

Adapters decide *how* it goes: TS/TGID rewrite execution, slot
arbitration, hang, cadence, pacing, protocol state, local repeat.

The same line explains why an adapter that drops on slot contention
merely events it (slot scarcity is mechanism, and the core has nothing
useful to say about it), and why adapters never hold rule fragments (rule
distribution would be policy at the edge).

## D-22 — Resolve hostnames at startup only

Both ancestors resolve hostnames *inside* the connect path (`cc2obp/net.c`,
`ipsc2hbpc/src/net.c`), reached from reconnect timers. At one or two
links that is invisible. At ~100 systems, many outbound by hostname (HBP
peer, XLX, OBP), **one unresponsive resolver blocks the entire daemon**
for the resolver timeout and drops every call on every system — and
systemd's watchdog does not fire, because the process is alive, just
deaf.

Therefore config load turns every hostname into a `sockaddr` once and
stores it; the datapath and reconnect timers use the stored address and
never call `getaddrinfo`. An address that changes needs a restart, which
is consistent with D-09 anyway. If re-resolution is ever wanted it
belongs on a slow timer that tolerates failure — never in a connect path.

This is the single-thread model's real hazard. It is not a spinning
protocol bug; the watchdog covers those.

## D-23 — No allocation on the datapath; reload allocates on the control
## plane

All tables, stream slots, and per-peer pools are sized from config and
allocated once at startup. Steady state runs `malloc`-free. Frames move
by `const` pointer through a synchronous chain.

**A rules reload is a control-plane event and may allocate** (D-09): it
builds a new table arena, validates it, swaps a pointer, and releases the
old arena once no in-flight stream references it. This does not weaken
the datapath rule, and the distinction must stay explicit in the code so
it is not "cleaned up" by someone reading only the headline.

Every fixed pool in a no-malloc daemon needs a stated behavior at its
limit, or an implementer invents one. In all cases: refuse the *new*
arrival, never evict a live one, emit a rate-limited event, keep running.
Sustained exhaustion is a misconfiguration or an attack, not a traffic
condition, so those events are alarm-worthy.

## D-24 — Existing production stays untouched until gates pass

ipsc2hbpc keeps the production IPSC bridge; hblink3, dmrlink3, and
hblink4 keep their networks. OmniLink earns each role by passing the
PLAN.md gate for it. Nothing double-binds a production port during
testing.

## D-25 — Frame extensibility policy

Normative in FRAME.md §6. In summary: the core routes on the fixed
header alone and **never parses payloads**; a `pfmt` field decouples
payload encoding from frame type; unknown in-stream traffic types forward
opaquely on an already-active stream; reserved space is zero-on-write; a
4-bit `ver` field carries version discipline; IDs live in DMR's 24-bit
space (D-02); packed fields use shift/mask accessors, never C struct
bitfields.

The frame is 64 bytes — one cache line.

## D-26 — Unit (private) calls route from a target route cache

hblink4's `user_cache.py` is the reference — the most robust unit
handling in the family — promoted from per-server to core-wide. Fed
passively from every ingress traffic frame using metadata the frame
already carries; a hit forwards to one system, a miss drops with a
`unit_no_route` event. Never flooded. Bridges are not consulted, and
per-bridge arbitration does not apply. Full semantics in ROUTING.md §6;
built in phase 6.

**A federated design cannot consume identifiers from a namespace it does
not administer**, and that is a standing constraint rather than a scope
cut. TGID space is operator-scoped: an operator hands out talkgroups
inside their own network and bounds their reach with explicit bridge
membership. Radio-ID space is globally administered and unit routing is
unbounded by construction. Any future in-daemon virtual endpoint —
recorder, announcer — must therefore be reachable by talkgroup, never by
radio ID. BrandMeister does the opposite and it works for them because
they are a single administered network that is the sole authority over
its own namespace; OmniLink assumes an internet of independent routers
whose operators coordinate only when they choose to. The precedent does
not transfer.

*Talkback/echo is deliberately not an OmniLink feature.* It exists as a
finished standalone product (`dmr-talkback`) that connects as an ordinary
HBP peer, and building it a second time inside OmniLink would duplicate
shipped, working software for no gain.

## D-27 — Data calls ride opaquely, later

Voice is the first-class payload. Data cannot be reduced without loss, so
it rides as an opaque tagged burst (`DATA_BURST`), deliverable only
between burst-native adapters, and it is built after the parity gate.
Because the core never parses payloads (D-25), specific ETSI data kinds
can be promoted to typed payloads later without the core changing at all.

No data-call reassembly, ever, in the daemon.

## D-28 — Colour code is RF-local: never read, never rewrite, 1 when
## invented

Colour code is an RF air-interface interference-mitigation mechanism: it
lets a receiver ignore a co-channel repeater it is not meant to hear.
That is the whole of its purpose. It has no meaning between repeaters,
and each repeater applies its own before transmitting.

Three rules, deliberately not one rule:

- **Never read it.** Not a routing key, not an ACL input, not a match
  condition, not a filter. Nothing in the daemon branches on a colour
  code.
- **Never rewrite it.** A passed-through burst keeps whatever the origin
  stamped, so a stream may legitimately carry a foreign colour code end
  to end. Normalizing would re-encode the slot type and both EMB halves
  on every burst on the dominant path, forfeiting D-05, to change a value
  nobody downstream reads.
- **Use 1 when you must invent one.** IPSC and CC-CC ingress construct
  bursts and the XLX module link builds one from nothing; all use
  `dmr/`'s precomputed CC-1 tables.

Colour code is not timeslot. A repeater must be *told* which slot a call
belongs in, because it has two and cannot infer the choice — so slot is a
routing key (D-04) and an explicit delivery parameter. A repeater is
never told its colour code; it already knows it. Slot is information the
network owes the repeater; colour code is information the repeater owes
itself.

Consequences:

- No `colorcode` key exists for burst construction. The value appears
  only in `[system.announce]`, as HBP login self-description
  (CONFIG.md §2.3), and a master parses the same field from connecting
  repeaters for the dashboard.
- `dmr/` is complete as vendored. No Golay(20,8) slot-type encoder is
  needed, and hblink3's `xlx_slot_type()` is not ported.
