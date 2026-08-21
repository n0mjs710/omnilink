# ADAPTERS.md — Protocol Adapter Specifications

An adapter is a self-contained protocol engine registered on the daemon's
single event loop — the same shape as ipsc2hbpc and cc2obp.

Five adapters: **HBP** (the dominant transport), **OBP** (the trunk),
**XLX** (HBP plus a module link), **IPSC** and **CC-CC** (the legacy
edges). This file defines the common contract, then each adapter.

## 1. The adapter contract (`src/adapters/adapter.h`)

Each adapter type exports an ops table; each configured system gets one
instance.

```c
/* Delivery parameters for one member. Filled by the core from the bridge
   member, or from the unit route cache for unit calls (ROUTING §6). */
typedef struct {
    uint32_t tgid;      /* destination TGID; equals dst_id for unit calls */
    uint8_t  slot;      /* 1 | 2; 0 = unslotted egress (OBP, CC-CC) */
    uint32_t endpoint;  /* deliver to this endpoint; 0 = adapter's choice */
} nx_egress_target;

typedef struct {
    /* Open sockets, register fds/timers on the shared loop. */
    void *(*init)(const nx_system_cfg *cfg, ev_loop *loop, uint16_t sys_idx);
    /* Core hands a packet to deliver on this system. Synchronous. */
    void  (*egress)(void *inst, uint32_t tag,
                    const uint8_t *dmrd, int len,
                    const nx_egress_target *t);
    /* Protocol goodbyes on daemon shutdown. */
    void  (*shutdown)(void *inst);
} nx_adapter_ops;

/* The whole core-facing API adapters may call: */
void     nx_core_ingress(uint16_t sys, uint32_t tag,
                         const uint8_t *dmrd, int len);
uint32_t nx_stream_tag_next(void);            /* FRAME §3.1 */
void nx_core_system_state(uint16_t sys, uint32_t endpoint, bool up);
void nx_event_emit(/* subsys, sys, ev, lvl, fields... */);
bool nx_acl_permit_reg(uint16_t sys, uint32_t endpoint_id);  /* §1.3 */
bool nx_acl_permit_traffic(uint16_t sys, uint32_t src, uint32_t dst,
                           uint8_t slot);                   /* §1.3 */
```

Everything is a direct synchronous call on one thread — no queues, no
rings, no handoffs (ARCHITECTURE §2).

**Each system is a fully isolated instance.** The adapter *type* is
shared code only. Every configured system gets its own state object —
sockets, endpoints, streams, counters, the moral equivalent of one
hblink3 SYSTEM — and nothing is shared between systems beyond the loop.
Inter-system traffic *always* transits the core, even between two systems
of the same protocol; shortcuts would make the accounting and the bridge
semantics lie.

### 1.1 Universal ingress duties (wire → core)

- All protocol machinery: auth, registration, keepalives, endpoint state.
- **Present a well-formed DMRD packet** and allocate its stream tag
  (FRAME §3.1). HBP, OBP, and XLX hand over what arrived; IPSC re-packs;
  CC-CC assembles. The three tiers and the burst-construction rules are
  FRAME §4.
- **The `bits` byte must be right, including on unslotted transports.**
  Slot is part of the routing key (D-04), so OBP, CC-CC, and XLX stamp
  the nominal slot D-07 assigns them. Frame type and dtype/vseq must
  reflect what the burst actually is: an adapter that constructs a burst
  already chose sync-at-A versus EMB-at-B–F to build it, so it knows the
  position and must record it. DMRD has no encoding for "position
  unknown".
- **Preserve call structure** (D-14). Nothing is synthesized; a
  headerless late-entry stream stays headerless end to end; **no timeout
  terminators ever enter the core**.
- **Forward every stream to the core unconditionally**, including streams
  local repeat also delivered in-system and streams expected to match no
  bridge (ROUTING §2).

### 1.2 Source peer: accept on ingress, decide on egress

The DMRD repeater ID (bytes 11–14) is the infrastructure device a call
entered the network through. Three rules:

- **Ingress accepts what arrives.** Whatever the wire carries *is* the
  source peer as far as we can know. An adapter never second-guesses it
  or substitutes something it likes better.
- **The core carries it unchanged** and never rewrites it. It feeds
  last-heard, the unit route cache, and the dashboard.
- **Egress decides what to put on its own wire**, per that system's
  config and its protocol's convention.

Egress is where the protocols differ, and one of them has no choice:

| adapter | egress repeater ID |
|---|---|
| HBP | **the original, preserved.** Substituting the local system's ID is neither required nor useful — an HBP client does not act on the field, and preserving it carries the true origin to the far dashboard. hblink3 does this: `bridge.py` copies `_data[11:15]` unchanged. |
| OBP | `network_id` by OBP convention; `preserve_source_peer` retains the original instead |
| IPSC | **must** be the local peer's radio ID — an IPSC mesh will not forward a call that did not come from one of its own peers, so a foreign source peer is simply not repeated. A protocol requirement, not a policy choice; dmrlink3 rewrites it unconditionally |
| CC-CC | the conduit's own identity; the originating peer rides in the B-on |

This is entirely an adapter concern. The core neither knows nor cares
which choice a system makes.

### 1.3 ACLs are enforced here, and TGID ACLs are HBP-only

**The core has no ACL layer** (D-31). Bridge rules are already the
routing whitelist: a TGID that is no member's TGID on that
`(system, slot)` matches nothing and goes nowhere. The only thing an ACL
adds is control over traffic that never reaches the core at all — **local
in-system repeat** — so enforcement belongs where local repeat happens.

The purpose is concrete: without it, a user on one repeater can key an
invented TGID and have it repeated to every other repeater on that
system. Bridge rules cannot stop that, because local repeat is not
bridge-driven (D-18).

| ACL | applies to | where | notes |
|---|---|---|---|
| `reg_acl` | endpoint registration (login) | HBP server, IPSC master | **always enforced**; `use_acl` cannot disable it |
| `sub_acl` | source radio ID of a call | every adapter, at ingress | keeps a banned radio off the network wherever it enters |
| `tgid_ts1_acl` / `tgid_ts2_acl` | destination TGID, by slot | **HBP only** | limits local repeat; a config error on any other protocol |

**Why TGID ACLs are HBP-only:**

- **IPSC cannot enforce it.** Peers hear each other directly and the
  router is not in that path (D-18), so there is nothing to filter.
- **OBP** has no need: its permitted TGIDs are exactly its bridge
  memberships, and an unmapped TGID is already fail-closed (D-07).
- **CC-CC and XLX** carry one talkgroup by construction.

**Grammar.** One string, one action: `PERMIT:` or `DENY:` followed by IDs
and/or inclusive ranges, or `ALL`. Exactly one action per ACL — mixing is
not expressible and is a config error. An ACL defines the action for the
IDs it lists **and the opposite action for every ID it does not**. That
second half is the part operators get wrong, and the validator's error
text says so.

**Order**, first denial wins, all must pass: global `sub_acl` → global
`tgid_ts*_acl` (HBP, selected by slot) → system `sub_acl` → system
`tgid_ts*_acl`. Registration is separately gated: global **and** system
`reg_acl` must both permit, or the login is refused.

**Fail closed.** A malformed ACL is a startup or reload failure, never a
silently-ignored ACL.

**Adapters query, never hold.** ACLs live in the reloadable rules arena,
so the adapter calls `nx_acl_*` rather than caching a copy — a reload
therefore takes effect on the next packet with no adapter involvement.

Denials emit rate-limited `acl_denied` events naming the ACL that fired.
An ACL that silently eats traffic is the hardest misconfiguration to
diagnose.

### 1.4 Universal egress duties (core → wire)

**Every value the adapter writes is handed to it.** An adapter does not
know the rules and never consults them: the core resolves the bridge
member (or the unit route cache) and passes the result as
`nx_egress_target` — `tgid`, `slot`, and optionally a specific
`endpoint`. The adapter is *told* where the traffic goes; it looks
nothing up. That is D-21, and it is also the only thing that could work,
since unit calls route from a cache no adapter can see.

An adapter holding a private replica of bridge membership would be rule
distribution to the edge, which is precisely what D-21 forbids.

- **Copy once, rewrite everything in that pass.** The core hands a
  `const` packet plus the target. The adapter copies it and writes
  `t->slot` into the `bits` byte, and — where `t->tgid` differs from the
  packet's destination — `t->tgid` into both the header and the LC
  together (FRAME §4.1, via `dmrlc.c`). Repeater ID and wire `stream_id`
  pass through. CC-CC additionally reduces the burst to AMBE
  (`dmr_ambe_72_to_49` ×3); IPSC unpacks to its own field layout and
  mints its own identifiers.
- Slotted protocols call **`channel.c`** for per-`(system, slot)`
  arbitration, hang time, and the `stream_to` contention horizon
  (ROUTING §5.2), with local-repeat traffic in the same arbitration
  state. This module is shared precisely because it is the easiest thing
  in the project to get subtly different in two places.
- **Preserve `dtype`/`vseq` from the incoming packet** — `t->slot` is the
  only part of the `bits` byte that changes (FRAME §1.1). For a
  headerless stream, build a minimal LC from the packet's own src/dst and
  emit **no** wire header (FRAME §3.1).
- **Event every drop** — `slot_busy`, `payload_unsupported`, `endpoint_down` —
  so the log and dashboard can join the core's forwarded-to *intent* with
  the edge's *outcome* (D-17). Nothing is reported back to the core; it
  keeps forwarding.
- **Forward on arrival. No pacing, no loss concealment** (D-15). Every
  endpoint stack already owns jitter buffering and gap handling. Gaps
  present as real gaps. The sole exception in the entire system is the
  IPSC egress clock (§5.1).

### 1.5 Always

Emit events (endpoint up/down, call boundaries seen at the edge, drops,
malformed packets) and 10 s `stats` counter events; report liveness via
`nx_core_system_state`. Never block — every socket is non-blocking on the
shared `ev_loop`, and hostnames are resolved at startup only (D-22).

---

## 2. HBP — HomeBrew Protocol

The dominant transport (D-05), and the only adapter split across two
files (ARCHITECTURE §3.1): `hbp_proto.c` for the wire, `hbp_service.c`
for the repeater service.

**HBP is client/server, the distinction is important.** A server accepts
registrations from repeaters and hotspots and repeats among them; a
client dials out and logs in. It is never "master and peer" — that is
IPSC's vocabulary, borrowed by tools that arrived after it, and it
misdescribes the architecture. IPSC keeps `master`/`peer` because IPSC
genuinely is a peer-to-peer mesh whose master only holds the bootstrap
list (§5.2). Both are correct; neither is the other.

- **Modes:** `server` — serve MMDVM repeaters and hotspots, the primary
  mode; and `client` — dial out to an existing server, useful for
  migration and for hanging OmniLink off a legacy HBlink3 site.
- **Devices below an HBP server are clients.** "Endpoint" is the neutral
  cross-protocol term used where the core and the event schema must span
  protocols (ROUTING §1, EVENTS §4.1); "client" is the word used when
  speaking about HBP.
- **Reuse:** the login/auth state machine and DMRD codec conventions from
  hblink3's `hblink.py` and hblink4's `protocol.py` as semantic
  reference; the C peer-mode engine from `ipsc2hbpc/src/hbp.c`.
- **Ingress (pure passthrough):** the packet already carries everything,
  including the `bits` byte, so ingress is a pointer hand-off plus the
  stream-tag lookup (FRAME §3.1), pooled per client and slot. **No
  decoding of any kind** — not the burst, not the LC.
- **Egress (passthrough):** write the packet back out, touched only
  where routing requires — the target slot in the `bits` byte, and the
  destination in header and LC if the member's TGID differs
  (FRAME §4.1). Repeater ID and wire `stream_id` pass through unchanged,
  as hblink3 does. Arbitration via `channel.c`. No pacing.
- **Repeater ID passes through unchanged** in both directions (§1.2).

### 2.1 Local repeat — the client/server obligation (D-18)

A server must repeat traffic among its own connected clients; that is
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
parsing radio IDs — and carry it in `endpoint_connected` events.
Dashboard color only; no behavior attaches (D-03).

One HBP system with no bridge members *is* a small standalone server:
local repeat plus subscriptions is most of what a small network needs, at
zero additional code. A supported deployment shape, though if there are no
other systems or routing, or only limited OBP connections, HBlink4 is a
better option.

---

## 3. OBP — OpenBridge

The trunk (D-06), and the HBlink4 interconnect (D-03).

- **Reuse:** `cc2obp/openbridge/obp_link.c` (HMAC-SHA1, DMRD-compatible
  framing) plus hblink4's `openbridge` work as the most complete semantic
  reference for multi-stream tracking and dedupe. Both ends of an
  OmniLink↔OmniLink link are ours, so extensions are available there and
  nowhere else.
- **Ingress:** HMAC verify, then hand the packet over — OBP carries
  DMRD, so there is nothing to convert. Slot comes from the wire (1 by
  OBP convention; the real slot when `both_slots` is set). Many
  concurrent streams from a per-peer pool, keyed on the wire `stream_id`
  (FRAME §3). **Per-stream dedupe** of retransmit and reflection cases —
  the cheapest and most effective loop control in the system
  (ROUTING §8).
- **Egress:** wire slot fixed at 1 per OBP convention, no arbitration
  (trunk semantics), no pacing, one wire `stream_id` per core stream.
- **Bridge mapping is ordinary membership.** An OBP system's permitted
  TGIDs are exactly the TGIDs it holds bridge memberships for — one
  config concept instead of two. hblink3 expands its `OBP_BRIDGES` rows
  into synthetic bridge members internally, so this is the same model
  stated directly (D-07, CONFIG §8).
- **`preserve_source_peer`** (§1.2) chooses what goes in the DMRD
  RptrId on this link: OBP convention is our `network_id`, and `true`
  forwards the originating peer instead so it propagates end to end.
  The reference implementation does not validate the field —
  authentication is the HMAC plus the source socket — so both are safe;
  it is per-system config because it is only *useful* when the far end
  preserves it too. This was added after it was discovered that while
  Brandmeister published the requirement subsequent to work with IPSC2,
  they do not bother to check/enforce the requirement, which allow us
  to pass the original source-peer as well.

---

## 4. XLX — reflector module link

**XLX's DMR interface is the homebrew protocol, unmodified.** DMRGateway
uses the same `CDMRNetwork` class for XLX and BrandMeister and its
network layer contains no XLX conditionals. So this adapter is HBP
outbound plus one thing: an in-band private call at connect that selects
the module.

It is a distinct adapter type for config clarity, event attribution, and
its own conformance vectors — not because the protocol differs.

- **Reuse:** the HBP client-mode engine in full (§2) — login, keepalives,
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

Build the LC from the same src/dst as the DMRD header
(FRAME §4 rule 1). xlxd reads the destination from header bytes 8–10
and never decodes the LC — its BPTC decode is commented out at
`:657-660` — so a contradictory LC *would* work, but is could be changed
in the future. It costs little to generate a proper LC, so we will
include that behavior here as a hedge against the future.

**This is a control protocol, not stream frame synthesis.** The link
commands never become a routed stream and never enters the core; the 
adapter emits it directly to the socket, exactly like a login or a 
keepalive.

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

**IPSC is not a bare-AMBE transport.** Its wire carries `VOICE_HEAD` and
`VOICE_TERM` burst types, a DMR LC, the reassembled embedded LC on burst
E, superframe position, and timeslot — the same material HBP carries,
arranged differently and without the over-the-air FEC and interleave. The
adapter **re-packs real elements**; it does not invent a call out of AMBE.

What IPSC does not carry is the assembled burst's structural wrapping:
BPTC, sync, slot type, and EMB. Colour code lives in EMB and slot type,
so it is absent here in both directions (D-28).

- **Reuse:** `ipsc2hbpc/src/ipsc.c` (auth/HMAC, registration, keepalive,
  burst parse — live-proven in production) is the engine; dmrlink3 is the
  semantic reference for master mode and multi-peer bookkeeping.
- **Modes:** `peer` — join an existing mesh, the primary mode; `master` —
  bootstrap master, as dmrlink does.
- **Ingress:** burst type → call boundaries; AMBE extracted as 3 × 49-bit
  and re-packed into a burst (`dmr/`: 49→72, BPTC, slot type, sync, EMB)
  per FRAME §4. **The LC comes from IPSC's own LC**, never invented, and
  must agree with the DMRD header. Burst position is real — burst E is
  explicitly identifiable (`GV_BE_FLAG`). Repeater ID from the IPSC
  RptrId field; keepalive state → `nx_core_system_state` plus events.
- **Egress:** unpack to AMBE and IPSC's own fields, slot from the egress
  target, IPSC sequence and RTP fields owned here — the ipsc2hbpc
  HBP→IPSC path, already solved. **The IPSC peer ID is always rewritten
  to this system's own radio ID** (§1.2).
- **Retarget rewrites the IPSC LC in place**, because IPSC has the same
  LC-in-payload duplication HBP does. dmrlink3 does exactly this
  (`bridge.py`, "Rewrite DST GROUP + IPSC SRC in DMR LC"), and it is why
  the IPSC loopback vector asserts on the LC as well as the AMBE bits
  (FRAME §6).

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

Neither ever enters the core; both are IPSC wire machinery.

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
  the core never routes a unit call to one.
- **Ingress:** B-on → a real voice-header packet. A B-on *is* a genuine
  call-start signal, so building the header from its metadata is
  translation of a real event, not synthesis. Media → AMBE
  triplets built up into bursts per FRAME §4, with the LC agreeing with
  the DMRD header and structure from `dmr/`'s fixed CC-1 tables (D-28).
  B-off → a terminator packet.

  **Stamp a real position.** The adapter chooses sync-at-A versus
  EMB-fragment-at-B–F in order to construct each burst, so it knows the
  position: run a per-stream A–F counter from the B-on and record it in
  the `bits` byte. Miscounting hands the far egress a fragment/position
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
(FRAME §6) — CC↔burst vectors validated against known-good HBP vectors
of the same audio — so the hard-won fix can never silently regress.

`cc2obp/translate.c` already implements this exact boundary
(`extract_ambe()` → `cccc_ambe_pack21()` and its inverse) against HBP
burst form, so it ports as a lift rather than a rewrite.

---

## 7. What is not an adapter

- **Talkback / echo.** `dmr-talkback` already does this as a standalone
  HBP client and works (D-26). Building it a second time inside OmniLink
  would duplicate shipped software for no gain.
- **Any non-DMR mode.** OmniLink is DMR end to end (D-02). A proposal
  requiring a second air interface is out of scope, not deferred.
- **A trunk other than OpenBridge** (D-06).
