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
- Reduce voice to canonical frames (FRAME.md): strip FEC/interleave to
  49-bit AMBE (`dmr/` module), extract/decode LC, assign `stream_tag`,
  stamp `origin_system/peer/slot`.
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
- Reconstruct full protocol framing: FEC encode (AMBE 49→72, BPTC LC
  header/terminator, EMB + embedded LC fragments, sync patterns — all in
  `dmr/`), protocol packet wrap, protocol stream IDs.
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
  preservation across the network happens via the canonical frame, which is
  cleaner than the hblink3 flag hack.
- **Ingress:** DMRD → 33-byte burst; voice sync/EMB classification; VHEAD
  LC (BPTC decode) → CALL_START; embedded-LC accumulation for late entry;
  `dmr_ambe_72_to_49` ×3 → VOICE; VTERM → CALL_END. HBP stream_id →
  stream_tag map (pool, per repeater+slot).
- **Egress:** burst encoding from what the frames carry (VHEAD only for a
  real CALL_START, voice bursts at their `vseq` positions with
  EMB/embedded LC from the 9-byte LC, VTERM), new HBP stream_id per
  stream, slot from member config, arbitration per (system, slot). No
  pacing (D-17): forward on arrival, as hblink3/4 always have — MMDVM
  endpoints own their jitter buffers.
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
  extraction to 49-bit, call boundaries from IPSC burst types (call start /
  terminator), `origin_peer` from IPSC RptrId field, per-network keepalive
  state → `nx_core_system_state` + events.
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
- **Ingress:** HMAC verify, DMRD parse, burst → canonical (OBP carries HBP-
  style bursts; same 72→49 path as HBP), wire slot ignored
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
- **Ingress:** B-on → CALL_START (a B-on is a *real* call-start signal;
  the 9-byte LC is constructed from its metadata — translation of a real
  event, not synthesis), media → AMBE triplets (`vseq = 7`, FRAME.md
  §1.1), B-off → CALL_END. One conduit = one TGID by construction, so a
  CC system's bridge membership is a single member entry per conduit.
- **Egress:** B-on/media/B-off synthesis, immediate forward (matches
  cc2obp's no-jitter-buffer design).
- **AMBE representation — solved, port carefully (D-11):** the c-Bridge
  represents AMBE internally in a form unlike anything else we have seen,
  including the representations in the ETSI documents; the working
  translation is already in `cc2obp/cccc/cccc_ambe.c` and CC audio is
  clean today. Port that code verbatim and lock it in with conformance
  vectors (FRAME.md §7) — CC↔canonical vectors validated against
  known-good HBP vectors of the same audio — so the hard-won fix can
  never silently regress.

## 6. PORT — OmniLink native trunk (Trunk v2 successor)

- **What:** `nx_frame` itself over UDP: tiny header (magic, LE u32 seq),
  keepalive ping/pong (carrying frame version, FRAME.md §6 rule 4), dead-peer
  detection. Peer identification is per-link config (D-19): `auth = "hmac"`
  — HMAC-SHA1 with a shared secret, the exact routine and tag format OBP
  already uses from `crypto.c`, so zero new crypto code; sender
  authentication, **not** secrecy — or `auth = "none"` (source address is
  the identity; right for VPNs and trusted paths). The direct
  descendant of dmrlink3 `trunk.py`: unslotted, no contention, forward
  everything the bridge table allows.
- **Why:** core↔core interconnect with zero translation loss (no FEC
  re-encode, no LC synthesis — the canonical frame goes through
  untouched, `origin_peer`/`src` intact for loop observation, ROUTING.md §7);
  and the external integration point: a recorder, a parrot, an analytics
  feed, or one day HBlink4-as-a-port speaks this instead of a DMR protocol.
- **Bridge membership:** like OBP, membership TGIDs are the allow list.
- Simplest adapter (~400 LOC); built in phase 3 partly as the test harness
  for everything else (a Python `port` speaker in `tests/` can inject and
  capture canonical frames without any radio hardware).

## 7. HBlink4's place (decision D-03, restated)

HBlink4 is **not** folded in and is **not** a special adapter in phase 1–5.
The agreed hblink3⇄hblink4 doc already settled the shape: edge access
server (auth, per-device options, user cache) peering over OBP with the
core router. OmniLink replaces the *hblink3/dmrlink3 core-router role*;
HBlink4 continues unchanged, now peering with OmniLink. If we later want
HBlink4 tighter-coupled, the PORT protocol is the path (a Python nx_frame
codec is trivial), and nothing designed here has to change.

Note that D-03/D-18 deliberately narrow HBlink4's uniqueness:
options-driven subscriptions (filtering local repeat *and* bridge
egress) plus the target route cache preserve roughly **90% of HBlink4's
uniqueness** at the per-system level, so plain repeater/hotspot service
stops requiring HBlink4. What HBlink4 retains as its distinct value:
per-device authentication/access-policy as the primary organizing idea —
in OmniLink, login matching drops devices into protocol-primitive
systems, full stop.

## 8. Foreign digital modes (D-24)

OmniLink is a DMR application. If an adapter for another digital mode
(e.g. a Fusion adapter) ever exists, it is **that adapter's job to speak
DMR semantics to the core** — 24-bit IDs, DMR LC, A–F burst position (or
`vseq = 7`), AMBE framing or an agreed `pfmt`. The core learns nothing
about the foreign mode. Such adapters are expected to be rare,
**infrastructure-to-infrastructure only, and never repeater-facing**:
the overwhelming majority of traffic is DMR, and keeping the core's
language pure-DMR is what keeps every other adapter simple.
