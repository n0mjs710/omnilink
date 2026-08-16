# DECISIONS.md — Design Rationale

Each record states a settled design decision and why it is what it is.
The numbering is stable and referenced throughout the docs. These are
**settled**: the docs elsewhere are written to them, and changing one is
a design change, not a discussion prompt.

## D-01 — Name

The project is **OmniLink**, in the hblink/dmrlink family tradition.

## D-02 — The frame payload is the 33-byte DMR burst; HBP is the core's
## native representation

*(Revised 2026-08-16. Supersedes the original D-02, which made the
payload a 21-byte triplet of 49-bit AMBE with FEC and interleave
stripped. The reasoning that replaced it is recorded here because it is
the kind of decision that gets re-litigated.)*

Voice rides the core as the **33-byte on-air DMR burst**, unmodified —
the same representation HBP and OBP carry. IPSC and CC adapters
translate to and from it at the edge.

The original choice was defensible: 49-bit AMBE is the least common
denominator already in service, and two of four adapters (IPSC, CC)
carry it natively. But it made the core a **neutral intermediate
representation**, and that is the architectural shape of a
general-purpose media router, not of a DMR call router. It is also the
shape whose costs land in the wrong place.

**The cost is not created or destroyed by this choice, only located.**
Converting between the two representations is a full burst teardown and
rebuild — FEC strip/re-encode plus EMB, sync, slot type, and LC
re-synthesis — and each design pays it on paths where the core's
representation is not the endpoints' :

| path | canonical 49-bit | burst-native |
|---|---|---|
| HBP→HBP, HBP→OBP, OBP→OBP | 2 conversions | **0** |
| IPSC→HBP, CC→HBP | 1 | 1 *(tie)* |
| IPSC→IPSC, CC→CC, IPSC→CC | **0** | 2 |

Mixed paths tie. The decision is therefore only ever about which
*same-protocol* path carries the traffic — and HBP+OBP carries every
MMDVM repeater, every hotspot, and (per the revised D-07) all
instance-to-instance federation. IPSC and CC systems are bridged *into*
that network, which is the tie row. Bridging two legacy systems to each
other through OmniLink is the c-Bridge use case and it is shrinking, so
this decision gets more right over time, not less.

Second reason, independent of cycles: **the hard adapters already exist
in C, written to this exact boundary.** `ipsc2hbpc` is in production and
`cc2obp` carries the hard-won CC AMBE de-interleave fix. Against a
burst-native core those are lifts; against an invented representation
they are rewrites against a form that has never been on the air.

Accepted costs, knowingly:

- Legacy↔legacy paths pay two conversions where they would have paid
  none. Lossless (the 72-bit FEC is derived from the 49 bits, so
  49→72→49 recovers exactly) — wasted cycles, not degraded audio, at
  µs scale under the D-22 ceiling.
- **The core is no longer representation-neutral.** HBP's LC-in-payload
  duplication — src/dst appearing in both the header and the burst LC —
  becomes a permanent core concern rather than something an adapter
  encapsulates, and keeping the two in sync on every retarget is a real
  and recurring bug class. HBlink3 solves it with per-stream precomputed
  LCs; that approach is the reference.
- A legacy→legacy path synthesizes burst fields (colour code, sync, EMB,
  LC) that no downstream consumer reads. Bounded, but it is invented
  data, so it is built from real configured values, never canned.

Unchanged by this revision: the frame abstraction itself. The core still
needs `origin_system`, a core-assigned `stream_tag`, and flags that a
DMRD packet has nowhere to put. What was dropped is the neutral payload
*representation*, not the frame.

## D-03 — HBlink4 stays separate; subscriptions are a delivery limiter,
never a router

HBlink4 remains the device-facing edge access server and interconnects
over OBP; OmniLink fills the core-router role (the hblink3/dmrlink3
role). Within OmniLink's HBP systems, a repeater's subscription (static
ACL, later HBP `options`) filters **everything delivered to that
repeater** — local repeat and bridge egress alike — and can only
*decline* traffic its system carries; it can never cause bridging.
Bridge rules stay static, operator-owned, the sole routing authority.
Documentation must state the footgun plainly: subscribing to a TGID buys
delivery *if the system carries it*, not a bridge to it. Two riders:
endpoint classification (hotspot/repeater) comes from HBP login
software/package fields only — dashboard color, no behavior; and the
standalone shapes are emergent at zero code (an HBP system with no
bridges is a small standalone server; an IPSC system with no bridges is
a bootstrap-only master + monitor — legitimate under CG-NAT).

## D-04 — Single-threaded

One thread, one event loop, direct function calls; no libpthread, no
atomics, no locks, no queues. At D-22 scope there is no performance or
scaling argument for concurrency, and the one real benefit threads would
buy (stall isolation) is already bounded by a systemd watchdog plus
instance federation. The adapter contract is a clean seam if the
calculus ever changes; that is not a supported build target.

## D-05 — Core routes on TGID alone; slot is edge config; one bridge per
(system, TGID)

Timeslot is an air-interface constraint local to slotted edges; within
the network it is meaningless. The core keys streams on TGID; egress
slot comes from bridge member config; slot arbitration lives in the
adapter. Deliberately unsupported: routing `(S, TG, TS1)` and
`(S, TG, TS2)` to different bridges — same TG on both slots meaning two
conferences is a configuration smell, and supporting it would put slot
back into the core's key. Model it as two systems if truly needed.

## D-06 — Voice-only routing first; data carried opaquely later

The payload model is "any payload the ETSI DMR specification defines,"
typed by `type` + `pfmt` + `payload_len` (FRAME.md §2.1); voice is the
first first-class payload. Data cannot be reduced without loss, so it
rides as an opaque tagged burst (DATA_BURST, phase 6), deliverable only
between burst-native adapters; specific ETSI data kinds can be promoted
to typed payloads later without touching the core, which never parses
payloads (D-14).

## D-07 — OBP is the trunk; PORT is deferred, not forbidden

*(Revised 2026-08-16. Supersedes the original D-07, which specified PORT
— the canonical frame over UDP — as the native core↔core trunk.)*

The D-22 federation mechanism is **OpenBridge**. With the burst as the
frame payload (D-02), an OBP hop is near-passthrough, so PORT's original
selling point — zero translation loss — is no longer a differentiator.
What an OBP hop does not carry (`origin_system`, `vseq`, flags) the far
end re-derives: `origin_system` from which OBP peer the traffic arrived
on, `vseq` from burst structure, flags from the LC. Nothing is lost;
some things are recomputed.

Deleting PORT removes an adapter, a wire format, and an auth scheme
(D-19, also deleted) from a project whose scope is deliberately modest.

**The door is left open deliberately.** The 64-byte frame remains
trunk-capable — it is a `memcpy` plus HMAC away from being a wire unit —
so if OBP ever limits federation, PORT can return without redesign. That
is a low risk for a reason worth recording: repeater-facing OBP must
interoperate with BrandMeister and DMR+, but **both ends of an
OmniLink↔OmniLink link are ours**, so OBP can be extended there without
anyone's cooperation. That has already been done once, in hblink3 and
cc2obp, with `PRESERVE_SOURCE_PEER`.

Consequences: no PORT adapter, no PORT test-injection harness (the
replay harness reads captures into adapters directly), and external
tools integrate over the event bus (D-09) rather than a traffic port.

## D-08 — License GPLv3, N0MJS copyright

Matches every ancestor repo, and required by the GPL'd lifted modules.

## D-09 — The event contract is normative; the dashboard is not

Binding: event schema, vocabulary, ordering, and the Unix JSON-lines
socket (daemon listens, consumers connect; DASHBOARD.md). Deliberately
arbitrary: the dashboard application itself — gen-1 is Python, but any
consumer that speaks the socket is legitimate. Name/ID resolution is
consumer-side, never in the daemon.

## D-10 — No config reload

Config tables are immutable after startup; restart on change. Immutable
config is part of what keeps the daemon simple and correct, and restart
is operationally identical to today's practice.

## D-11 — Conformance vectors are mandatory per adapter

Bit-representation mismatches between protocols are the dominant
cross-protocol failure mode and are invisible until a human decodes
audio. Every adapter ships ingress vectors, egress vectors, and the
loopback identity (FRAME.md §7) before it is "done." The CC adapter's
AMBE translation (`cccc_ambe.c` — the c-Bridge uses a 3-lane block
interleave of the d-bit order, unlike anything in the ETSI documents)
ports verbatim from cc2obp and is locked in by vectors.

## D-12 — Hosting: GitHub `n0mjs710/omnilink`, public before first code

All code is tracked on GitHub from its first commit.

## D-13 — Existing production stays untouched until phase gates pass

ipsc2hbpc keeps the production IPSC bridge; hblink3/dmrlink3/hblink4
keep their networks. OmniLink earns each role via PLAN.md gates. Nothing
double-binds production ports during testing.

## D-14 — Frame extensibility policy

Normative in FRAME.md §6: the core routes on the 28-byte header only and
never parses payloads; `pfmt` decouples payload encoding from frame type;
unknown in-stream traffic types forward opaquely with active streams;
reserved space is zero-on-write; 4-bit `ver` discipline; IDs are DMR's
24-bit space (D-24); 36-byte payload ceiling with NXF_FRAG reserved.
Expansion reserves are layered cheapest-first (FRAME.md §6.1). Packed
fields use shift/mask accessors, never C struct bitfields. The frame is
64 bytes — one cache line.

Unaffected by the D-02 revision: the 36-byte payload area was already
sized for a tagged 33-byte burst, so carrying the burst needs no layout
change. The policy now matters mainly in-process rather than on a wire
(D-07), but it is what keeps PORT reintroducible without redesign.

## D-15 — Slot truth lives at the adapter; delivery truth via events

Slot state cannot live in the core: local in-system traffic occupies
slots without ever transiting it, so core-side slot state would
sometimes lie. The adapter owns all slot truth and arbitration. Operator
visibility comes from the event stream: the core's `call_start` states
intent (members forwarded to); adapters event their drops (`slot_busy`
etc.) as outcome; the dashboard and log join the two. The core does not
track per-member outcomes and keeps forwarding — wasted frames cost
nothing at our scale, and slot scarcity is mechanism, not policy (D-23).

## D-16 — Preserve call structure; full pass-through; core owns timeout

Forward what was received; rebuild nothing. Real headers/terminators
cross as real CALL_START/CALL_END; a headerless (late-entry) stream
flows headerless end-to-end (egress builds minimal LC from frame fields
and emits no wire header); `vseq` is read once at ingress and followed
at egress — relabeling burst positions after loss would be editing the
stream; missing bursts stay missing. On stream timeout the core frees
state, releases hang, emits the event, and sends nothing downstream:
loss of signal is the RF world's steady state and every far edge handles
it natively. Nothing in the system synthesizes frames. Addressing vs
structure: routing rewrites addresses everywhere they appear (TGID,
slot, LC dst24); structure and timing pass through untouched.

## D-17 — No pacing, no loss concealment; IPSC egress clock is the sole
exception

Every endpoint stack owns its jitter buffering; forward on arrival, gaps
present as real gaps. The exception exists because MOTOTRBO repeaters
carry only a ~2-packet buffer: timing must arrive correct, and
correcting timing requires a small buffer to correct it in. That buffer
is *position-preserving* — playout scheduled by `seq` (stream-start +
n×60 ms) so a missing burst leaves its gap at the right time.
Per-system configurable. A correctly-placed gap is timing correction,
not concealment.

## D-18 — In-system local repeat lives in the adapter; IPSC has none

An HBP master must repeat traffic among its own connected endpoints —
the client/server obligation — inside the adapter, never via the core;
the core still sees every stream once (unconditional ingress). Delivery
filtering per repeater follows D-03. **IPSC is peer-to-peer**: each
speaker replicates its own traffic to all peers, the master is only the
peer-list bootstrap, so the IPSC adapter has no local-repeat duty in any
mode — it replicates its own core-egress traffic like any speaking peer
and tracks slot occupancy from observed peer traffic so it never
transmits into a busy slot.

## D-19 — *(withdrawn 2026-08-16)* PORT peer identification

Specified HMAC-SHA1 authentication for the PORT trunk. Withdrawn with
PORT itself (D-07). Federation authenticates as OBP already does — the
same HMAC-SHA1 routine, reached through the OBP adapter rather than a
second scheme. `crypto.c` ships regardless.

## D-20 — One routing core with N protocol edges, not federated
per-protocol daemons

The alternative — separate C daemons per protocol joined by a
purpose-built trunk — does not avoid the hard design: such a trunk *is*
the frame, so the design work is identical and the federation
merely duplicates the routing core around it while preserving the known
operational pains (rules multiplication across daemons, dashboard
divergence, boundary metadata hacks, per-boundary bit-convention risk,
no cross-network unit calls). The unified build is also the smaller
build. Costs accepted knowingly: one failure domain (bounded per D-04),
and no on-air value until the core plus two adapters exist.

## D-21 — The RF-equivalence test (standing design filter)

DMR's air interface already survives late entry, gaps, vanished signals,
and missing headers at the receiver — every consumer downstream was
built for weird RF. Therefore a weird stream through OmniLink should
degrade exactly like a weird signal on RF. We intervene only where an IP
failure mode has **no RF analog**: loops, cross-network floor control
(arbitration/hang), identity/admission, and operator truth. Every
feature proposal gets this test first; re-solving ETSI's receiver-side
problems in the middle of the network is tail-chasing by design.

## D-22 — Scope: state/regional/national-group tool; ~100 systems;
federate beyond

OmniLink is not a platform for worldwide networks, and a single install
will never carry several statewide networks' full loads. Design ceiling
~100 configured systems per instance; past that, federate instances over
OBP (D-07) — for resilience (too many eggs in one basket), not performance.
This scope is what makes D-04 correct rather than merely acceptable.

## D-23 — Policy lives in the core; mechanism lives at the edge

The core decides *whether and where* a stream goes: bridge rules,
timers, arbitration, hang, unit route cache, loop observation.
Single-copy policy is what makes one rules file, one dashboard, and one
truth possible. Adapters decide *how* it goes: TS/TGID rewrite
execution, slot arbitration, cadence, pacing, protocol state. The same
line explains why an adapter dumps on slot contention and merely events
it (slot scarcity is mechanism), and why adapters never hold rule
fragments (rule distribution would be policy at the edge).

## D-24 — OmniLink is a DMR application

Not digital-mode-agnostic: the core's native language is DMR — 24-bit
IDs, DMR LC, A–F burst positions, ETSI payload kinds. If a foreign-mode
adapter (e.g. Fusion) ever exists, it is that adapter's job to translate
to DMR semantics; such adapters are expected to be rare,
infrastructure-to-infrastructure only, and never repeater-facing
(ADAPTERS.md §9).

## D-25 — Same (src, dst) is not loop evidence; loop detection is a
pattern alarm, never a datapath guess

Hams share one radio ID across all their radios, so two radios with one
ID on two systems legitimately produce same-src, same-TG streams in
overlap or close succession — and by header metadata alone, an echo
returning in the hang window is indistinguishable from such a reply. So
the datapath never guesses: overlapping same-src streams are ordinary
contention (arbitration drops the newcomer; the event notes
`same_src`), and loops are detected by the one signature no human
conversation produces — sustained sub-human re-capture cadence
(ROUTING.md §7 rule 5) — raising the alarm-class `loop_suspected` event.
Loops are misconfigurations; the fix is operator action. Structural
prevention (never-to-origin, unique membership) and OBP edge
stream-dedupe stand; an optional phase-6 circuit breaker is available as
configurable policy.

## D-26 — Playback (talkback) is a virtual, transport-less adapter, and it
answers group calls only

A talkback — replay a caller's own audio so they hear how they sound — is an
**optional adapter, not a core feature**. The core stays unaware of it
(D-23); with no playback system configured, no playback code or config
exists. It is the first **virtual adapter**: no wire protocol, no socket, no
peer. It implements the ordinary adapter contract (ADAPTERS.md §1) with a
null transport — `init` opens no sockets (timers only), `egress` *captures*
the stream instead of transmitting it, and it originates the reply through
`nx_core_ingress` like any adapter forwarding traffic. This is the proof
that the contract's seam is core-facing frame I/O, not wire I/O, and the
template for other virtual endpoints later (recorder, announcer).

**Group calls only. Playback is never addressed by radio ID.** As a bridge
member for its configured TGID(s) it captures a stream off that TGID and
replays it re-keyed onto the same TGID, sourced from its own configured ID,
so the talkgroup hears it back. That is the whole mechanism. A private call
placed *to* a playback ID is not a supported way to reach it.

The reason is addressing authority, and it is a standing constraint on this
project, not a scope cut:

> **A federated design cannot consume identifiers from a namespace it does
> not administer.**

TGID space is operator-scoped. An operator hands out talkback talkgroups
inside their own network and bounds their reach with explicit bridge
membership; no coordination with anyone else is required, and nothing
escapes the rules. Radio-ID space is globally administered (radioid.net) and
unit routing is unbounded by construction — a private call to an
unknown destination floods until the target is located, and the resulting
location map is global. For Playback to be reachable by ID, **every instance
ever deployed** would have to hold a globally unique registered radio ID,
issued to a licensed operator, for something that is neither a radio nor a
repeater. Two instances sharing an ID anywhere in a connected mesh means
private calls chase whichever one transmitted last.

BrandMeister does exactly this (private call to 9990, per-master `MCC`997)
and it works for them, because BrandMeister is a single administered network
that is the sole authority over its own namespace. OmniLink assumes the
opposite: an internet of independent routers, run by people who coordinate
with each other only when they choose to. The precedent does not transfer.

Playback still owns a **24-bit radio ID** (config) as the source of its
replies — it appears in last-heard and in the LC of every stream it
originates — but that ID is not required to be reachable, and one ID an
operator already owns serves every Playback instance they run. Nothing
routes *to* it.

Replay is position-preserving clocked egress from a bounded, init-sized
buffer (the D-17 egress-clock discipline), scheduled after CALL_END and
running only while a playback is live — off the routing hot path, which is
why this had to be a bolt-on standalone process in the Python era and does
not here. It does no FEC or wire work — it buffers and replays frames as
received — so capture→replay is bit-identical and its conformance test is
the loopback identity (FRAME.md §7). Per D-21 it adds no receiver-side behavior; it is an
endpoint that happens to echo.

The addressing-authority constraint above applies to every virtual endpoint
this adapter is the template for. A recorder or an announcer reachable by
radio ID has the same problem; reachable by talkgroup, it does not.

*Revised 2026-08-15: unit-call support removed. The standalone C
implementation (github.com/n0mjs710/dmr-talkback) reached the same
conclusion during its build and is group-only for the same reason.*

## D-27 — Multi-bridge membership requires a wire discriminator;
## endpoints join exactly one bridge

*(Added 2026-08-16.)*

A system may be a member of **more than one bridge only if its wire
format carries a discriminator that identifies which bridge a given
stream belongs to.** In practice that discriminator is the TGID.

- **OBP** carries a TGID per stream, so one peer on many bridges is
  unambiguous in both directions. Config is nested:
  `{system: {bridge: tgid}}`.
- **XLX modules and CC-CC conduits *are* their bridge identity.** One
  connection carries one room or one talkgroup, and nothing on the wire
  distinguishes it — XLX presents every module as TS2/TG9 with no module
  identifier in any frame, and the CC-CC specification states plainly
  that TGID "is not carried end-to-end; each end maps the conduit to its
  own configured TGID." These join **exactly one bridge**. Config is
  flat: `{system: bridge}` for XLX, `{conduit: (bridge, tgid)}` for CC.

Two independent failures follow from allowing otherwise, and the second
is the load-bearing one:

1. **Ingress is ambiguous.** A stream arrives on the connection with
   nothing to say which bridge it belongs to, so the core would have to
   fan it to both — one stream becomes two. This is the same ingress
   fork already prohibited for OBP when one TGID maps to two bridges.
2. **Coherent egress would require U-turn replication.** Traffic
   arriving at the endpoint *from* bridge A belongs, if the endpoint is
   also on bridge B, on B as well. Delivering it would mean replicating
   and reversing direction inside the core. Decline that and the
   topology is incoherent in a way no operator would predict: A and B
   both reach the far system but never each other. Accept it and the
   endpoint has become a bridge-joiner — which is what a bridge already
   is. Out of scope, permanently.

Operators wanting two bridges joined join them. They do not do it
implicitly through a shared single-talkgroup endpoint.

Note the CC conduit still declares a local TGID. That value is the
conduit's **core-side identity**, not a bridge selector — it is what the
CC-CC protocol expects each end to configure independently. Unlike the
XLX case, a wrong value here is benign and visible: traffic appears
under a consistent but unexpected talkgroup rather than vanishing. That
is why CC exposes it and XLX does not (TS2/TG9 is a protocol constant
and a wrong value there is silently fatal).
