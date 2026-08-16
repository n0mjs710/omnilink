# ADAPTERS.md — Protocol Adapter Specifications

An adapter is a self-contained protocol engine registered on the daemon's
single event loop — the same shape as ipsc2hbpc/cc2obp, which is the
point: we know how to build, test, and live-debug exactly this thing.
This file defines the common contract, then each adapter.

## 1. The adapter contract (`src/adapters/adapter.h`)

Each adapter type exports an ops table; each configured system gets one
instance:

```c
typedef struct {
    /* Open sockets, register fds/timers on the shared loop. */
    void *(*init)(const nx_system_cfg *cfg, ev_loop *loop, uint16_t sys_idx);
    /* Core hands an egress frame for this system. Synchronous. */
    void  (*egress)(void *inst, const nx_frame *f);
    /* Protocol goodbyes on daemon shutdown. */
    void  (*shutdown)(void *inst);
} nx_adapter_ops;

/* What adapters call (the whole core-facing API): */
void nx_core_ingress(const nx_frame *f);
void nx_core_system_state(uint16_t sys, uint32_t peer, bool up);
void nx_event_emit(/* subsys, sys, ev, lvl, fields... */);
```

Everything is a direct synchronous call on one thread — no queues, no
rings, no handoffs (ARCHITECTURE.md §2).

**Wire I/O is an adapter concern, not the core's.** An adapter need not
have a wire at all: a *virtual* adapter (§7) opens no sockets, implements
`egress` as a capture/sink, and originates streams via `nx_core_ingress` —
the same contract with a null transport. Everything below in these
universal duties that names sockets/protocol machinery simply does not
apply to such an adapter.

**Each system is a fully isolated instance.** The adapter *type* is
shared code only: every configured system gets its own state object
(sockets, peers, streams, ACLs, counters — the moral equivalent of one
hblink3 SYSTEM), and nothing is shared between systems beyond the loop.
Inter-system traffic *always* goes through the core, even between two
systems of the same protocol — no shortcuts, or the accounting and
bridge semantics lie.

Universal duties:

**Ingress (wire → core):**
- All protocol machinery: auth, registration, keepalives, peer state.
- Present voice as the 33-byte DMR burst (FRAME.md §4.1), extract/decode
  LC, assign `stream_tag`, stamp `origin_system/peer/slot`. For
  burst-native protocols (HBP, OBP, IPSC's burst-carrying paths) this is
  a copy; for bare-AMBE protocols (CC, IPSC voice) the adapter builds the
  burst with `dmr/`, with the LC agreeing with the frame header and
  synthesized fields taken from configured values.
- Preserve call structure (D-16): real headers/terminators
  cross as real CALL_START/CALL_END; nothing is ever synthesized — a
  headerless (late-entry) stream begins with VOICE and stays headerless
  end-to-end. **No timeout terminators anywhere** — when a stream just
  stops, the core emits the event and everyone's state expires on its
  own timers (ROUTING.md §3). Assign `vseq` once here: read from wire
  superframe position (free — position is classified anyway to harvest
  embedded LC), or 7 = unknown for structureless origins (FRAME.md
  §1.1).
- Forward every stream to the core unconditionally — including streams
  local repeat also delivers in-system and streams expected to match no
  bridge (ROUTING.md §2). The core must see all edge activity.
- Per-system ingress ACLs (subscriber/TGID allow/deny, lifted semantics
  from hblink3/dmrlink3 ACLS.md) — enforcement at the edge keeps garbage
  out of the core.

**Egress (core → wire):**
- Emit the payload burst in the protocol's framing. Burst-native
  adapters write it out and re-touch it only where routing requires
  (retarget LC splice, FRAME.md §4.1.1). Bare-AMBE adapters reduce it
  (`dmr_ambe_72_to_49` ×3) and wrap in their own framing. Protocol packet
  wrap and protocol stream IDs in both cases.
- Slotted protocols: per-(system, slot) arbitration (ROUTING.md §5),
  with local-repeat traffic in the same arbitration state; follow the
  frame's `vseq` faithfully, local A–F counter only on `vseq = 7`
  (FRAME.md §1.1). For a headerless stream, build minimal LC from the
  frame's src/dst and emit no wire header (FRAME.md §4.2).
- **Event every drop** — `slot_busy`, `pfmt_unsupported`, `peer_down` —
  so the dashboard/log can join the core's forwarded-to (intent) with
  the edge's outcome (D-15; ROUTING.md §5). No reporting
  back to the core; it keeps forwarding, and that's fine.
- **Forward on arrival — no pacing, no loss concealment (D-17).** Every
  endpoint protocol stack already owns jitter buffering and gap handling;
  we are a routing core and stay out of that business. Gaps present as
  real gaps (the proven cc2obp philosophy). The sole exception in the
  entire system is the IPSC egress clock (§3).

**Always:** emit events (peer up/down, call boundaries seen at the edge,
drops, malformed packets) and 10 s `stats` counter events; report
peer/system liveness via `nx_core_system_state`. Never block: all
sockets non-blocking via `ev_loop`.

## 2. HBP — HomeBrew Protocol

- **Modes:** `master` (serve MMDVM repeaters/hotspots — the primary mode)
  and `peer` (dial out to an existing master; useful for migration and for
  hanging OmniLink off legacy HBlink3 sites).
- **Reuse:** login/auth state machine and DMRD codec conventions from
  hblink3 `hblink.py` + hblink4 `protocol.py` (reference semantics), C
  peer-mode engine from `ipsc2hbpc/src/hbp.c`. Honor `PRESERVE_SOURCE_PEER`
  semantics learned in the OBP work: `origin_peer` = RptrId from DMRD on
  ingress; on egress DMRD RptrId = the local system's id (standard) —
  preservation across the core happens in the frame header, which is
  cleaner than the hblink3 flag hack.
- **Ingress (near-passthrough since D-02):** DMRD → frame with the
  33-byte burst copied verbatim into the payload. Classify voice
  sync/EMB to set `vseq`; BPTC-decode the VHEAD LC → CALL_START; VTERM →
  CALL_END. **No AMBE conversion** — the burst is already the core's
  representation. HBP stream_id → stream_tag map (pool, per
  repeater+slot).
- **Egress (near-passthrough):** write the payload burst back out as
  DMRD. The burst is re-touched only where routing requires it: if the
  member's TGID differs from the frame's, splice per-stream precomputed
  LCs per FRAME.md §4.1.1. New HBP stream_id per stream, slot from
  member config, arbitration per (system, slot). No pacing (D-17):
  forward on arrival, as hblink3/4 always have — MMDVM endpoints own
  their jitter buffers.
- **Local repeat — the client/server obligation (D-18):** a master must
  repeat traffic among its own connected repeaters; that is what being
  the server means, and it happens *inside the adapter*, without
  transiting the core (which still sees the stream once, via
  unconditional ingress). Delivery per repeater is subscription-filtered
  per **D-03**: the subscription (phase 2 = static per-repeater TG/slot
  ACL in config; follow-on = repeater-supplied HBP `options` at login,
  hblink4-style) filters **everything delivered to that repeater** —
  local repeat and bridge egress alike — and is a **limiter only**: it
  can decline traffic its system carries, it can never cause bridging.
  Bridge rules stay static and operator-owned; CONFIGURING must state
  the footgun plainly (subscribing to TG 4400 buys delivery *if the
  system carries 4400*, not a bridge to it). Local deliveries occupy
  slots in the same arbitration state as core egress and are evented
  with `local:true`.
- **Endpoint classification (dashboard color only, D-03):** flag
  hotspot vs repeater the way HBlink4 does — from the login
  software/package ID fields already parsed at RPTC time, never by
  parsing radio IDs — carried in `peer_connected` events. No behavior
  attaches to the flag.
- **Standalone shape (emergent, D-03):** one HBP system + no bridges
  *is* a small standalone server — local repeat + subscriptions ≈ 80%
  of HBlink4 — fine for very small networks at zero additional code.
  A supported deployment shape, not a feature.

## 3. IPSC — Motorola

- **Reuse:** `ipsc2hbpc/src/ipsc.c` (auth/HMAC, registration, keepalive,
  burst parse — live-proven in production) is the engine; dmrlink3 remains
  the semantic reference for master mode and multi-peer bookkeeping.
- **Modes:** `peer` (join an existing IPSC network — primary) and `master`
  (bootstrap master, as dmrlink does).
- **Ingress:** IPSC group/private voice → burst type classification, AMBE
  extracted as 49-bit and then **built into a 33-byte burst** (`dmr/`:
  49→72, BPTC LC, slot type, sync) per FRAME.md §4.1 — IPSC is one of the
  two adapters that constructs rather than copies, so the LC must agree
  with the frame header and colour code comes from config. Call boundaries
  from IPSC burst types (call start / terminator), `origin_peer` from IPSC
  RptrId field, per-network keepalive state → `nx_core_system_state` +
  events.
- **Egress:** IPSC voice packet synthesis (the ipsc2hbpc HBP→IPSC path,
  already solved), slot from member config, IPSC sequence/rtp fields owned
  here. The ipsc2hbpc jitter-buffer egress clock is retained as the
  **sole pacing exception in the system** (D-17): MOTOTRBO repeaters
  carry only a ~2-packet buffer, so timing must arrive correct, and
  correcting timing requires a small buffer to correct it *in*. The clock
  is position-preserving: playout is scheduled by burst `seq`
  (stream-start + n×60 ms), so a missing burst leaves its gap at the
  right time instead of shifting later bursts early — timing correction,
  never loss concealment. Per-system configurable (`pace = true`
  default) so live testing can settle how necessary it really is.
- **No local repeat — IPSC is peer-to-peer, not client/server (D-18).**
  Each IPSC peer replicates its own traffic to all other peers directly;
  the master role is only the peer-list bootstrap. Peers hear each other
  without us. The adapter's duties are the inverse of HBP's: replicate
  its own core-egress traffic to all peers like any speaking peer, and
  **track slot occupancy from observed peer traffic** so it never
  transmits into a busy slot (that observed traffic transits the core as
  normal ingress; the occupancy state stays here, per D-15).
- **Standalone shape (emergent, D-03):** an IPSC system with no bridge
  members is a **bootstrap-only master + monitor/logger** — it serves
  the peer list (a legitimate role in a CG-NAT world, where peers need
  a reachable master even though traffic flows peer-to-peer) and its
  unconditional ingress feeds the dashboard/log without routing
  anything. Zero additional code; a supported deployment shape.
- The production KS-DMR bridge (ipsc2hbpc) keeps running untouched until
  the IPSC adapter passes live gates (PLAN.md phase 4); never double-bind
  UDP 50000 on the prod host.

## 4. OBP — OpenBridge

- **Reuse:** `cc2obp/openbridge/obp_link.c` (C engine: HMAC-SHA1, DMRD-
  compatible framing) + the hblink4 `openbridge` branch as the most complete
  semantic reference (multi-stream tracking, dedupe). The per-OBP
  TGID↔bridge mapping table concept from
  `~/development/openbridge-hblink3-hblink4-design.md` maps 1:1 onto bridge
  membership here: an OBP system's allowed TGIDs are exactly the TGIDs it
  has bridge memberships for — no separate table needed, one config
  concept instead of two.
- **Ingress:** HMAC verify, DMRD parse, burst copied verbatim (OBP carries
  HBP-style bursts, so like HBP this is a copy, not a conversion), wire slot ignored
  (`origin_slot = 0`, unslotted), many concurrent streams (stream pool per peer),
  per-stream dedupe of the retransmit/reflection cases hblink3 handles.
- **Egress:** slot field fixed to 1 per OBP convention, no arbitration
  (trunk semantics), no pacing (immediate forward), one wire stream_id per
  core stream.
- This adapter is the HBlink4 interconnect (D-03): HBlink4 keeps its edge
  role and peers with OmniLink exactly as it peers with HBlink3 today on
  the `openbridge` branch — already live-verified in both directions.

## 5. CC — c-Bridge CC-CC

- **Reuse:** `cc2obp/cccc/cccc_link.c` (TCP 42421 control + UDP media,
  B-on/B-off call signaling) and `CC-CC_Link_Protocol_Specification.md`.
- **A conduit is a discrete endpoint, not a trunk (D-27).** The protocol
  spec is explicit: a conduit is identified by a Link ID and carries *one
  talkgroup's* traffic, and "TGID is local — it is not carried
  end-to-end; each end maps the conduit to its own configured TGID."
  The connection *is* the bridge identity and nothing on the wire
  disambiguates, so a conduit joins **exactly one bridge**. No new config
  syntax: it is an ordinary bridge member, with a validation rule.

  ```toml
  [[bridge]]
  name   = "STATEWIDE"
  member = [
    { system = "CC-KSDMR", tgid = 3120 },   # conduit: no slot, at most one bridge
  ]
  ```

  The declared TGID is the conduit's **core-side identity**, not a bridge
  selector — it is precisely what the CC-CC protocol expects each end to
  configure independently, and a wrong value is visible rather than
  silently fatal (traffic appears under a consistent but unexpected
  talkgroup), which is why it is exposed rather than injected.

  **No multiplexing.** cc2obp collapses N conduits onto one OBP trunk
  because OBP is its only egress; that is an artifact of that tool's
  purpose, not a property of CC-CC. Here each conduit is a first-class
  system, so cc2obp's `(openbridge_system, tgid)` uniqueness machinery
  has no analogue and is not ported.

  **No unit calls.** The CC-CC spec puts private calls out of scope
  entirely, so config load rejects a CC system as a unit-call target and
  the core never routes `NXF_UNIT` to one (ROUTING.md §6, §8).
- **Ingress:** B-on → CALL_START (a B-on is a *real* call-start signal;
  the 9-byte LC is constructed from its metadata — translation of a real
  event, not synthesis), media → AMBE triplets built up into bursts
  (`vseq = 7`, FRAME.md §1.1), B-off → CALL_END. Burst construction
  follows FRAME.md §4.1: LC agreeing with the frame header, colour code
  and sync from configured values.
- **Egress:** reduce the payload burst back to AMBE triplets, then
  B-on/media/B-off synthesis, immediate forward (matches cc2obp's
  no-jitter-buffer design).
- **AMBE representation — solved, port carefully (D-11):** the c-Bridge
  represents AMBE internally in a form unlike anything else we have seen,
  including the representations in the ETSI documents; the working
  translation is already in `cc2obp/cccc/cccc_ambe.c` and CC audio is
  clean today. Port that code verbatim and lock it in with conformance
  vectors (FRAME.md §7) — CC↔burst vectors validated against known-good
  HBP vectors of the same audio — so the hard-won fix can never silently
  regress. Note that `cc2obp/translate.c` already implements this exact
  boundary (`extract_ambe()` → `cccc_ambe_pack21()` and its inverse)
  against HBP burst form, so it ports as a lift rather than a rewrite.

## 6. *(withdrawn)* PORT — OmniLink native trunk

Removed with D-07. Federation between OmniLink instances is **OBP**: with
the burst as the frame payload (D-02) an OBP hop is near-passthrough, so
PORT's zero-translation-loss argument no longer distinguishes it. What
OBP does not carry the far end re-derives — `origin_system` from the
receiving peer, `vseq` from burst structure, flags from the LC.

Two things PORT also provided, and where they went:

- **External integration** (recorder, analytics feed) — the event bus
  (D-09, DASHBOARD.md), which is where truth already lives. A consumer
  needing traffic rather than events attaches as an OBP peer.
- **No-radio test injection** — the replay harness feeds captures into
  adapters directly, which is a better test anyway: it exercises the
  adapter under test rather than a second adapter written to be easy.

The frame layout remains trunk-capable (FRAME.md §6.1), so PORT can
return without redesign if OBP ever limits federation. Both ends of an
OmniLink↔OmniLink link are ours, so extending OBP is also available and
has precedent (`PRESERVE_SOURCE_PEER`).

## 7. Playback — virtual talkback adapter (no transport) (D-26)

A **virtual adapter**: no wire, no socket, no peer. It implements the §1
contract with a null transport and exists only to echo a caller's own audio
back so they can hear how they sound. Optional and load-if-needed (D-23):
absent from config, it does not exist. It is also the reference shape for
any future no-wire endpoint (recorder, announcer) — including D-26's
addressing-authority constraint, which binds those too.

- **Group calls only (D-26).** Playback is never addressed by radio ID. A
  federated design cannot consume identifiers from a namespace it does not
  administer: TGID space is operator-scoped and bounded by bridge membership,
  radio-ID space is globally administered and unit routing is unbounded, so
  reachability-by-ID would demand a registered unique ID for every instance
  ever deployed. See D-26 for the full argument.
- **Identity:** owns one 24-bit radio ID from config, used solely as the
  `src` of the streams it originates. It is **not** registered as a reachable
  subscriber and nothing routes to it; one ID an operator already owns serves
  every Playback instance they run.
- **Capture (`egress` = sink):** as a bridge member for its configured
  TGID(s) the core hands it egress frames; it buffers one stream's
  frames (CALL_START…VOICE…CALL_END, `vseq` as received). No wire encode — it
  keeps the frames verbatim.
- **Replay (`nx_core_ingress`):** after CALL_END (plus a configurable delay)
  it re-emits the buffered stream as a **new** stream — fresh `stream_tag`,
  `src` = its own radio ID, `dst` = the TGID it was captured from. The
  talkgroup hears it back. Ordinary bridge membership delivers it; the core
  learns nothing about playback.
- **Clocking:** position-preserving clocked egress from the buffer (the D-17
  egress-clock discipline), on the loop, only while a playback is live —
  never on the per-frame routing hot path. malloc-free: the capture buffer is
  sized at `init` from a bounded max-capture-length config.
- **Config:** radio ID; the TGID(s) or bridge it answers on; replay delay;
  max capture seconds.
- **Conformance (D-11):** the loopback identity vector — replayed frames are
  bit-identical to captured frames, since no re-encode occurs — is its test.

## 8. HBlink4's place (decision D-03, restated)

HBlink4 is **not** folded in and is **not** a special adapter in phase 1–5.
The agreed hblink3⇄hblink4 doc already settled the shape: edge access
server (auth, per-device options, user cache) peering over OBP with the
core router. OmniLink replaces the *hblink3/dmrlink3 core-router role*;
HBlink4 continues unchanged, now peering with OmniLink. If we later want
HBlink4 tighter-coupled, extending OBP between them is the path — both
ends are ours (D-07) — and nothing designed here has to change.

Note that D-03/D-18 deliberately narrow HBlink4's uniqueness:
options-driven subscriptions (filtering local repeat *and* bridge
egress) plus the target route cache preserve roughly **90% of HBlink4's
uniqueness** at the per-system level, so plain repeater/hotspot service
stops requiring HBlink4. What HBlink4 retains as its distinct value:
per-device authentication/access-policy as the primary organizing idea —
in OmniLink, login matching drops devices into protocol-primitive
systems, full stop.

## 9. Foreign digital modes (D-24)

OmniLink is a DMR application. If an adapter for another digital mode
(e.g. a Fusion adapter) ever exists, it is **that adapter's job to speak
DMR semantics to the core** — 24-bit IDs, DMR LC, A–F burst position (or
`vseq = 7`), AMBE framing or an agreed `pfmt`. The core learns nothing
about the foreign mode. Such adapters are expected to be rare,
**infrastructure-to-infrastructure only, and never repeater-facing**:
the overwhelming majority of traffic is DMR, and keeping the core's
language pure-DMR is what keeps every other adapter simple.
