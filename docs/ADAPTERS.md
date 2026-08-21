# ADAPTERS.md — Protocol Adapter Specifications

An adapter is a self-contained protocol engine registered on the daemon's
single event loop — the same shape as ipsc2hbpc and cc2obp, which is the
point: we already know how to build, test, and live-debug exactly this
thing.

Five adapters: **HBP** (the dominant transport), **OBP** (the trunk),
**XLX** (HBP plus a module link), **IPSC** and **CC-CC** (the legacy
edges). This file defines the common contract, then each adapter.

## 1. The adapter contract (`src/adapters/adapter.h`)

Each adapter type exports an ops table; each configured system gets one
instance.

```c
/* Where this frame is going on this system. The core fills it: from the
   bridge member for group calls, from the unit route cache for unit calls
   (ROUTING.md §6). It is the only channel by which delivery parameters
   reach an adapter. */
typedef struct {
    uint32_t tgid;      /* destination TGID; equals frame dst_id for unit calls */
    uint8_t  slot;      /* 1 | 2; 0 = unslotted egress (OBP, CC) */
    uint32_t endpoint;  /* specific endpoint to deliver to; 0 = adapter's choice */
} nx_egress_target;

typedef struct {
    /* Open sockets, register fds/timers on the shared loop. */
    void *(*init)(const nx_system_cfg *cfg, ev_loop *loop, uint16_t sys_idx);
    /* Core hands an egress frame for this system. Synchronous. */
    void  (*egress)(void *inst, const nx_frame *f, const nx_egress_target *t);
    /* Protocol goodbyes on daemon shutdown. */
    void  (*shutdown)(void *inst);
} nx_adapter_ops;

/* The whole core-facing API adapters may call: */
void nx_core_ingress(const nx_frame *f);
void nx_core_system_state(uint16_t sys, uint32_t endpoint, bool up);
void nx_event_emit(/* subsys, sys, ev, lvl, fields... */);
bool nx_acl_check_reg(uint16_t sys, uint32_t peer_id);   /* §1.2 */
```

Everything is a direct synchronous call on one thread — no queues, no
rings, no handoffs (ARCHITECTURE §2).

**Why the target is a parameter and not config the adapter holds.** An
adapter must not carry a private replica of bridge membership: rule
distribution to the edge is policy at the edge, which D-21 forbids. It
also would not work — unit calls route from the core's cache, which no
adapter can see, and `origin_slot` is deliberately never copied to egress
(FRAME §1.1), so without this parameter there is no channel by which the
core could name an endpoint or a slot. One struct closes both holes and
keeps every routing decision in the core.

**Each system is a fully isolated instance.** The adapter *type* is
shared code only. Every configured system gets its own state object —
sockets, endpoints, streams, counters, the moral equivalent of one
hblink3 SYSTEM — and nothing is shared between systems beyond the loop.
Inter-system traffic *always* transits the core, even between two systems
of the same protocol; shortcuts would make the accounting and the bridge
semantics lie.

### 1.1 Universal ingress duties (wire → core)

- All protocol machinery: auth, registration, keepalives, endpoint state.
- Present voice as the 33-byte DMR burst (FRAME §4.1); assign
  `stream_tag`; stamp `origin_system`, `origin_peer`, and `origin_slot`.
  For burst-native protocols (HBP, OBP, XLX) this is a copy. For
  bare-AMBE protocols (IPSC, CC) the adapter *builds* the burst with
  `dmr/`, with the LC agreeing with the frame header and every
  synthesized field taken from configured values.
- **Stamp `origin_slot` always**, including on unslotted transports,
  using the nominal slot D-07 assigns. It is part of the routing key now
  (D-04); a frame carrying 0 cannot be routed.
- **Assign `vseq` exactly once, here.** Read it from wire superframe
  position where the protocol has one; an adapter that constructs bursts
  already knows the position because it had to choose sync-at-A versus
  EMB-at-B–F to build the burst at all, and must record what it chose.
  `vseq = 7` is for structureless origins only and is close to
  unreachable in practice (FRAME §1.1).
- **Preserve call structure** (D-14). Real headers and terminators cross
  as real CALL_START and CALL_END; nothing is synthesized; a headerless
  late-entry stream begins with VOICE and stays headerless end to end.
  **No timeout terminators ever enter the core** — when a stream stops,
  the core emits the event and everyone's state expires on its own
  timers.
- **Forward every stream to the core unconditionally** — including
  streams local repeat also delivered in-system, and streams expected to
  match no bridge (ROUTING §2). The core must see all edge activity or
  the dashboard, the accounting, and the route cache are all lying.

### 1.2 Where ACLs are enforced

Split by what the core can see (D-21):

- **Traffic ACLs** (`sub`, `tgid_ts1`, `tgid_ts2`) are **core policy**,
  enforced in `acl.c` on ingress, before bridge lookup (ROUTING §2.1).
  Adapters do not hold ACL state and do not filter traffic. This is what
  keeps ACLs in the live-reloadable rules file with one implementation
  and one enforcement order.
- **Registration ACLs** (`reg`) are enforced by the adapter, at login —
  the core never sees a login as a frame. The adapter *queries*
  (`nx_acl_check_reg`) rather than holding a copy, so a reload takes
  effect on the next registration attempt without adapter involvement.

### 1.3 Universal egress duties (core → wire)

- Emit the payload burst in the protocol's framing. Burst-native adapters
  write it out and re-touch it only where routing requires it (retarget
  LC splice, FRAME §4.1.1, via `dmrlc.c`). Bare-AMBE adapters reduce it
  (`dmr_ambe_72_to_49` ×3) and wrap it in their own framing. Protocol
  packet wrap and protocol stream IDs in both cases.
- Slotted protocols call **`channel.c`** for per-`(system, slot)`
  arbitration, hang time, and the `stream_to` contention horizon
  (ROUTING §5.2), with local-repeat traffic in the same arbitration
  state. This module is shared precisely because it is the easiest thing
  in the project to get subtly different in two places.
- Follow the frame's `vseq` faithfully; use a local A–F counter only on
  `vseq = 7`. For a headerless stream, build a minimal LC from the
  frame's src/dst and emit **no** wire header (FRAME §4.2).
- **Event every drop** — `slot_busy`, `pfmt_unsupported`, `peer_down` —
  so the log and dashboard can join the core's forwarded-to *intent* with
  the edge's *outcome* (D-17). Nothing is reported back to the core; it
  keeps forwarding, and that is fine.
- **Forward on arrival. No pacing, no loss concealment** (D-15). Every
  endpoint stack already owns jitter buffering and gap handling. Gaps
  present as real gaps. The sole exception in the entire system is the
  IPSC egress clock (§4).

### 1.4 Always

Emit events (endpoint up/down, call boundaries seen at the edge, drops,
malformed packets) and 10 s `stats` counter events; report liveness via
`nx_core_system_state`. Never block — every socket is non-blocking on the
shared `ev_loop`, and hostnames are resolved at startup only (D-22).

---

## 2. HBP — HomeBrew Protocol

The dominant transport (D-05), and the only adapter split across two
files (ARCHITECTURE §3.1): `hbp_proto.c` for the wire, `hbp_service.c`
for the repeater service.

- **Modes:** `master` — serve MMDVM repeaters and hotspots, the primary
  mode; and `peer` — dial out to an existing master, useful for migration
  and for hanging OmniLink off a legacy HBlink3 site.
- **Reuse:** the login/auth state machine and DMRD codec conventions from
  hblink3's `hblink.py` and hblink4's `protocol.py` as semantic
  reference; the C peer-mode engine from `ipsc2hbpc/src/hbp.c`.
- **Ingress (passthrough):** DMRD → frame with the 33-byte burst copied
  verbatim. Classify voice sync/EMB to set `vseq`; BPTC-decode the VHEAD
  LC → CALL_START; VTERM → CALL_END. **No AMBE conversion at all** — the
  burst is already the core's representation. HBP `stream_id` →
  `stream_tag` map, pooled per endpoint and slot.
- **Egress (passthrough):** write the payload burst back out as DMRD,
  re-touched only where routing requires it — if the member's TGID
  differs from the frame's, splice the per-stream precomputed LCs
  (FRAME §4.1.1). New HBP `stream_id` per stream, slot from the egress
  target, arbitration via `channel.c`. No pacing: forward on arrival, as
  hblink3 and hblink4 always have.
- **`origin_peer` handling.** On ingress, `origin_peer` = the DMRD
  RptrId. On egress, the DMRD RptrId is the local system's ID, which is
  standard. Preservation across the core happens in the frame header,
  which is cleaner than hblink3's `PRESERVE_SOURCE_PEER` flag hack — that
  flag exists only where OBP must carry the fact across a wire (§5).

### 2.1 Local repeat — the client/server obligation (D-18)

A master must repeat traffic among its own connected endpoints; that is
what being the server means, and it happens **inside `hbp_service.c`**,
without transiting the core (which still sees the stream exactly once,
via unconditional ingress).

Delivery per endpoint is subscription-filtered per D-03. The subscription
— a static per-endpoint TG/slot ACL in config first, repeater-supplied
HBP `options` at login later, hblink4-style — filters **everything
delivered to that endpoint**, local repeat and bridge egress alike. It is
a **limiter only**: it can decline traffic its system already carries; it
can never cause bridging.

Documentation must state the footgun plainly: subscribing to TG 4400 buys
delivery *if the system carries 4400*, not a bridge to it.

Local deliveries occupy slots in the same `channel.c` arbitration state
as core egress, and are evented with `local:true` (D-17).

### 2.2 Endpoint classification, and the standalone shape

Flag hotspot versus repeater the way HBlink4 does — from the login
software and package ID fields already parsed at RPTC time, **never** by
parsing radio IDs — and carry it in `repeater_connected` events.
Dashboard color only; no behavior attaches (D-03).

One HBP system with no bridge members *is* a small standalone server:
local repeat plus subscriptions is most of what a small network needs, at
zero additional code. A supported deployment shape, not a feature.

---

## 3. OBP — OpenBridge

The trunk (D-06), and the HBlink4 interconnect (D-03).

- **Reuse:** `cc2obp/openbridge/obp_link.c` (HMAC-SHA1, DMRD-compatible
  framing) plus hblink4's `openbridge` work as the most complete semantic
  reference for multi-stream tracking and dedupe. Both ends of an
  OmniLink↔OmniLink link are ours, so extensions are available there and
  nowhere else.
- **Ingress:** HMAC verify, DMRD parse, burst copied verbatim — OBP
  carries HBP-style bursts, so this is a copy, not a conversion. Stamp
  `origin_slot` from the wire slot (1 by convention; the real slot when
  `both_slots` is set). Many concurrent streams, from a per-peer stream
  pool. **Per-stream dedupe** of retransmit and reflection cases, which
  is the cheapest and most effective loop control in the system
  (ROUTING §8).
- **Egress:** wire slot fixed at 1 per OBP convention, no arbitration
  (trunk semantics), no pacing, one wire `stream_id` per core stream.
- **Bridge mapping is ordinary membership.** An OBP system's permitted
  TGIDs are exactly the TGIDs it holds bridge memberships for — one
  config concept instead of two. hblink3 expands its `OBP_BRIDGES` rows
  into synthetic bridge members internally, so this is the same model
  stated directly (D-07, CONFIG §8).
- **`preserve_source_peer`** carries `origin_peer` and `src_id` intact
  across a federated hop, so the loop observations in ROUTING §8 hold
  across instances. It is per-system config because it is only safe
  between consenting endpoints.

---

## 4. XLX — reflector module link

**XLX's DMR interface is the homebrew protocol, unmodified.** DMRGateway
uses the same `CDMRNetwork` class for XLX and BrandMeister and its
network layer contains no XLX conditionals. So this adapter is HBP
outbound plus one thing: an in-band private call at connect that selects
the module.

It is a distinct adapter type for config clarity, event attribution, and
its own conformance vectors — not because the protocol differs.

- **Reuse:** the HBP peer-mode engine in full (§2) — login, keepalives,
  DMRD codec, stream mapping.
- **Reference implementation:** hblink3's `send_xlx_link()` /
  `xlx_link_module()` and `tests/test_xlx_link.py`, which encode xlxd's
  acceptance gates as assertions. **Port the tests with the code.**

### 4.1 The module link

Two calls at connect — **unlink (4000) first, then the module** — five
DMRD frames each, sharing one stream ID: 3 × VOICE_LC_HEADER then
2 × TERMINATOR_WITH_LC.

**The unlink is not defensive.** While a client already holds a module,
xlxd discards a link naming a different one and rewrites the header to
the module already held (`cdmrmmdvmprotocol.cpp:297-325`). On any
reconnect where the prior binding survives, link-only silently leaves us
in the old room.

It fires on **every** entry to the connected state — the binding is
per-session state on xlxd and is lost on any re-login.

xlxd validates each frame and **drops silently on any failure** — no NAK,
no response of any kind (`cdmrmmdvmprotocol.cpp:636-655`):

| # | gate | value |
|---|---|---|
| 1 | packet size | **exactly 55 bytes** — 20 header + 33 burst + 2 trailing pad |
| 2 | `data[15] & 0x30 >> 4` | `2` (DATA_SYNC) |
| 3 | `data[15] & 0x80` | set — TS2 |
| 4 | `data[15] & 0x0F` | `1` (VOICE_LC_HEADER) |
| 5 | sync at `data[33..39]` | BS or MS data sync |

Gate 1 means the burst must be padded to 55; a bare 53-byte DMRD is
rejected. Gate 4 means **only the three header frames carry the command**
— the terminators are slot type 2 and are ignored as commands, but are
sent to frame the call correctly.

Build the LC honestly from the same src/dst as the DMRD header
(FRAME §4.1 rule 1). xlxd reads the destination from header bytes 8–10
and never decodes the LC — its BPTC decode is commented out at
`:657-660` — so a contradictory LC *would* work, and must still not be
shipped: it is invisible to every test that does not decode audio.

**This is protocol machinery, not frame synthesis.** The link never
becomes an `nx_frame` and never enters the core; the adapter emits it
directly to its socket, exactly like a login or a keepalive. The
"nothing synthesizes frames" rule (D-14) is about fabricating call
content into the routing path and is not weakened here.

**Never lift a canned payload blob** from prior art. The link burst's LC
is built from this system's own radio ID and the target module TGID every
time; its structure comes from `dmr/`'s fixed CC-1 tables.

**Do not port hblink3's `xlx_slot_type()`.** The link burst is built at
colour code 1 from `dmr/`'s precomputed tables (D-28).

### 4.2 Binding and config

- **Static module binding only.** No runtime module selection, no
  reflector switching, no end-user control — that breaks the nailed-up
  systems-and-bridges model. DMRGateway is the correct tool for
  user-selectable modules at a repeater or hotspot. Different, not worse.
- **One module per connection; exactly one bridge** (D-07). Every module
  presents as TS2/TG9 and no module identifier exists in any frame, so
  the connection *is* the bridge identity. To bridge two modules,
  configure two systems.
- **TS2/TG9 is injected at rules load, never read from config**, because
  a wrong value is *silently* fatal. The member entry omits `tgid` and
  `slot` entirely and supplying either is a config error naming the
  remedy (CONFIG §4).
- `module` validates as a single letter A–Z. Module *numbers* (4004 and
  the like) are rejected outright with the remedy in the message —
  hblink3's predecessor took numbers, so a migrating operator will try
  it.
- **Never a unit-call target.** Module selection *is* a private call to
  4001–4026, so a unit call reaching an XLX system could silently move
  the reflector into a different room for every user connected to it.

### 4.3 Operational truth

There is no acknowledgement of the link, and nothing on the wire states
which module a connection is in. The `xlx_link_sent` event (module,
target TGID, unlink-then-link) is therefore the **only** local evidence
the link was attempted, and the dashboard must present it as an attempt
rather than as a confirmed state. Verification is the reflector's own
dashboard.

Note also that a stock xlxd is built with `NB_OF_MODULES 10` (modules
A–J); `NB_MODULES_MAX 26` exists but is commented out. A–Z is accepted
because the protocol allows it and operators can rebuild, but a module
the reflector does not have produces no link and no error — one more
reason this section exists.

---

## 5. IPSC — Motorola

- **Reuse:** `ipsc2hbpc/src/ipsc.c` (auth/HMAC, registration, keepalive,
  burst parse — live-proven in production) is the engine; dmrlink3
  remains the semantic reference for master mode and multi-peer
  bookkeeping.
- **Modes:** `peer` — join an existing IPSC network, the primary mode;
  and `master` — bootstrap master, as dmrlink does.
- **Ingress:** IPSC group/private voice → burst type classification, AMBE
  extracted as 49-bit and then **built into a 33-byte burst** (`dmr/`:
  49→72, BPTC LC, slot type, sync) per FRAME §4.1. IPSC is one of the two
  adapters that constructs rather than copies, so the LC must agree with
  the frame header. Structure (slot type, EMB, sync) comes from `dmr/`'s
  fixed CC-1 tables; IPSC carries no colour code on the wire in either
  direction (D-28). Call boundaries from IPSC burst types; `origin_peer`
  from the IPSC RptrId field;
  per-network keepalive state → `nx_core_system_state` plus events.
- **Egress:** IPSC voice packet synthesis (the ipsc2hbpc HBP→IPSC path,
  already solved), slot from the egress target, IPSC sequence and RTP
  fields owned here.

### 5.1 The egress clock — the system's sole pacing exception (D-15)

The ipsc2hbpc jitter-buffer egress clock is retained. MOTOTRBO repeaters
carry only a ~2-packet buffer, so timing must arrive correct, and
correcting timing requires a small buffer to correct it *in*.

The clock is **position-preserving**: playout is scheduled by burst `seq`
(stream start + n × 60 ms), so a missing burst leaves its gap at the
right instant instead of shifting later bursts early. Timing correction,
never loss concealment. Per-system configurable so live testing can
settle how necessary it really is.

Two further IPSC-wire-only behaviors, both carved deliberately (D-14,
D-15) and permitted **nowhere else in the system**:

- A drained or vanished stream is closed with a **terminator**
  (`translate.c`, `on_stream_timeout` → `emit_term`). IPSC has no
  representation for "nothing," and a MOTOTRBO repeater left without a
  terminator stays keyed.
- An empty playout slot is filled with **comfort silence**
  (`tr->ambe_silence`), because IPSC's wire cannot express "nothing" at a
  scheduled instant — the choice is silence or a timing lie. The
  synthesis count is logged and evented, so nothing is hidden.

Neither ever becomes an `nx_frame` and neither re-enters the core.

### 5.2 No local repeat — IPSC is peer-to-peer (D-18)

Each IPSC peer replicates its own traffic to all other peers directly;
the master role is only the peer-list bootstrap. Peers hear each other
without us.

The adapter's duties are therefore the inverse of HBP's: replicate its
own core-egress traffic to all peers like any speaking peer, and **track
slot occupancy from observed peer traffic** so it never transmits into a
busy slot. That observed traffic transits the core as ordinary ingress;
the occupancy state stays here (D-17).

An IPSC system with no bridge members is a **bootstrap-only master plus
monitor** — it serves the peer list, a legitimate role in a CG-NAT world
where peers need a reachable master even though traffic flows
peer-to-peer, and its unconditional ingress feeds the dashboard and log
without routing anything. Zero additional code; a supported shape.

**The production KS-DMR bridge (ipsc2hbpc) keeps running untouched**
until this adapter passes its live gate (D-24). Never double-bind UDP
50000 on the production host.

---

## 6. CC — c-Bridge CC-CC

- **Reuse:** `cc2obp/cccc/cccc_link.c` (TCP control plus UDP media,
  B-on/B-off call signalling) and `CC-CC_Link_Protocol_Specification.md`.
- **A conduit is a discrete endpoint, not a trunk** (D-07). The
  specification is explicit: a conduit is identified by a Link ID,
  carries *one talkgroup's* traffic, and "TGID is local — it is not
  carried end-to-end; each end maps the conduit to its own configured
  TGID." The connection *is* the bridge identity and nothing on the wire
  disambiguates, so a conduit joins **exactly one bridge**.

  The declared TGID is the conduit's **core-side identity**, not a bridge
  selector. It is exposed rather than injected because a wrong value is
  *visible* — traffic appears under a consistent but unexpected talkgroup
  — which is the opposite of XLX's silent failure (D-07, CONFIG §4).

  The nominal slot is injected, because the routing key needs one (D-04)
  and CC has no slot concept of its own. Note the specification's own
  slot trace: IPSC LIDs are assigned per timeslot (system *n*, slot 1 =
  `2n−1`, slot 2 = `2n`), so the slot information a conduit carries lives
  in the Link ID numbering at the c-Bridge, not in the media.

- **No multiplexing.** cc2obp collapses *n* conduits onto one OBP trunk
  because OBP is its only egress; that is an artifact of that tool's
  purpose, not a property of CC-CC. Here each conduit is a first-class
  system, so cc2obp's `(openbridge_system, tgid)` uniqueness machinery
  has no analogue and is not ported.
- **No unit calls.** The specification puts private calls out of scope
  entirely, so rules load rejects a CC system as a unit-call target and
  the core never routes `NXF_UNIT` to one.
- **Ingress:** B-on → CALL_START — a B-on is a *real* call-start signal,
  so the 9-byte LC (`pfmt 1`) is constructed from its metadata, which is
  translation of a real event rather than synthesis. Media → AMBE
  triplets built up into bursts per FRAME §4.1, with the LC agreeing with
  the frame header and structure from `dmr/`'s fixed CC-1 tables (D-28).
  B-off → CALL_END.

  **Assign a real `vseq`.** The adapter is choosing sync-at-A versus
  EMB-fragment-at-B–F in order to construct each burst, so it knows the
  position: run a per-stream A–F counter from the B-on and stamp what it
  built. `vseq = 7` here would hand the far egress a fragment/position
  mismatch that only an audio decode would catch (FRAME §1.1).
- **Egress:** reduce the payload burst back to AMBE triplets, then
  B-on / media / B-off synthesis, immediate forward — matching cc2obp's
  no-jitter-buffer design.

### 6.1 The AMBE representation — solved, port carefully (D-11)

The c-Bridge represents AMBE internally as a **3-lane block interleave of
the ETSI d-bit order**, unlike anything in the ETSI documents. The
working translation is already in `cc2obp/cccc/cccc_ambe.c` and CC audio
is clean today.

**Port that code verbatim** and lock it in with conformance vectors
(FRAME §7) — CC↔burst vectors validated against known-good HBP vectors
of the same audio — so the hard-won fix can never silently regress.

`cc2obp/translate.c` already implements this exact boundary
(`extract_ambe()` → `cccc_ambe_pack21()` and its inverse) against HBP
burst form, so it ports as a lift rather than a rewrite.

---

## 7. What is not an adapter

- **Talkback / echo.** `dmr-talkback` already does this as a standalone
  HBP peer and works (D-26). Building it a second time inside OmniLink
  would duplicate shipped software for no gain.
- **Any non-DMR mode.** OmniLink is DMR end to end (D-02). A proposal
  requiring a second air interface is out of scope, not deferred.
- **A trunk other than OpenBridge** (D-06).
