# DECISIONS.md — Design Rationale

Each record states a settled design decision and why it is what it is.
The numbering is stable and referenced throughout the docs. These are
**settled**: the docs elsewhere are written to them, and changing one is
a design change, not a discussion prompt.

## D-01 — Name

The project is **OmniLink**, in the hblink/dmrlink family tradition.

## D-02 — Canonical voice = 3×49-bit AMBE, FEC/interleave stripped

49-bit AMBE is the **least common denominator already in service**: IPSC
natively carries it, so the canonical form is IPSC's form and HBP/OBP's
72-bit FEC'd representation is the derived one. We normalize to the
simplest existing representation, not an invented one. Accepted
consequences: egress to burst protocols re-encodes FEC and re-synthesizes
EMB/sync (µs-scale; `dmr/` implements all of it); FEC's error-correction
benefit is consumed at ingress (loses nothing real on IP transport); data
calls can't be canonicalized (D-06). Wins: every adapter fully owns its
bit conventions — cross-protocol representation bugs become per-adapter
and vector-testable (D-11) — and PORT trunking is translation-lossless.

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

## D-07 — PORT: the canonical frame over UDP as the native trunk

Core↔core interconnect (the D-22 federation mechanism), external-tool
integration point, and the no-radios test-injection harness. Zero
translation loss — the canonical frame crosses untouched with origin
metadata intact. HMAC-SHA1 auth per D-19. The designated path for
anything that wants lossless canonical access.

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

## D-19 — PORT peer identification: HMAC-SHA1 default, `auth = "none"`
option

Identification, not secrecy: a bare UDP port feeding transmitters is
spoofable from off-path, so a keyed tag is the lightest thing that
actually identifies. HMAC-SHA1 is the exact routine OBP already uses
(`crypto.c` ships regardless); `none` is for VPN/trusted paths.

## D-20 — One routing core with N protocol edges, not federated
per-protocol daemons

The alternative — separate C daemons per protocol joined by a
purpose-built trunk — does not avoid the hard design: such a trunk *is*
the canonical frame, so the design work is identical and the federation
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
PORT — for resilience (too many eggs in one basket), not performance.
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
(ADAPTERS.md §8).

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
