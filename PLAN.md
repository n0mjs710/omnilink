# PLAN.md — Phased Build Plan

Ordering principle: get real RF audio flowing through the core as early as
possible (phase 2), because the routing core is the one genuinely novel
part; everything after that is porting things we've already built once.
Each phase has an **acceptance gate**; a phase isn't done until its gate
passes, and production migration only follows gates (D-13).

The D-02 revision strengthened this ordering. With the burst as the frame
payload, the first two adapters are near-passthrough rather than
translators, so phase 2 is smaller and first audio comes sooner — which
means IPSC and CC land against a core that has already been on the air
instead of against a design.

Implementer notes: docs are normative; deviations use the flag-then-fix
pattern (STYLE.md). Design questions that the docs don't answer go to
the project owner (N0MJS) — do not improvise policy.

---

## Phase 0 — Skeleton (small)

- Repo scaffolding: Makefile, lifted modules copied in (`dmr/`,
  `eventloop`, `toml`, `net`, `log`, `crypto`), `make test` runs the
  lifted modules' existing self-tests under this tree.

**Gate:** clean build + lifted-module tests green on the dev server.

## Phase 1 — Core (the biggest single phase)

- `frame.h` (layout + packed-field accessor macros, unit-tested
  byte-exact against FRAME.md).
- `config.c`: TOML → immutable system/bridge tables + full rejection
  suite (ROUTING.md §8 validations).
- `core.c`/`route.c`: stream table, bridge lookup, talker arbitration,
  hang time, timeouts, loop rules; unit-tested against scripted frame
  sequences (no sockets needed — call `nx_core_ingress` directly).
- `event.c`: event bus, unified log, Unix snapshot+stream socket.
- `main.c`: startup/shutdown/signals; a `null` test adapter (loops frames,
  emits events) proving the loop+event plumbing end-to-end.

**Gate:** `make test` green including a two-fake-adapter routing test:
scripted call on fake A appears on fake B with correct TGID rewrite,
arbitration, hang time, and a correct event trail on the socket.

## Phase 2 — HBP + OBP adapters (first real audio)

- HBP adapter (master mode first, peer mode second) per ADAPTERS.md §2,
  with conformance vectors cross-checked against dmr_utils3 output;
  includes local repeat with static per-repeater TG/slot ACLs (D-18,
  filtering local + bridge-egress delivery per D-03), endpoint
  classification from login fields (D-03), and drop-event reporting
  (`slot_busy` etc., D-15).
- OBP adapter per ADAPTERS.md §4 (lift obp_link.c).
- Live test on the dev server: a hotspot/repeater on OmniLink-HBP bridged
  to (a) another HBP system and (b) HBlink4 over OBP.

**Gate:** clean audio, correct metadata (src/dst/peer on the far
dashboard), correct hang-time behavior, in **both directions** on both
pairs — the live-verified standard used for the hblink4 openbridge work.

Since the D-02 revision these two adapters are near-passthrough, so this
phase is markedly smaller than originally planned and first audio arrives
correspondingly earlier. The gate proves the routing core, arbitration,
hang, and retarget LC splicing — not a representation round trip, which
no longer exists on this path.

### Phase 2 rider — XLX adapter

XLX is HBP outbound plus a module link (ADAPTERS.md §6), so it becomes
available as soon as HBP peer mode works and is built here rather than
waiting. Its gate is **separate and does not block the phase-2 audio
gate**, because passing it requires finding a live reflector with an
observable dashboard, which is a scheduling dependency rather than an
engineering one.

- Link construction with the five xlxd acceptance gates asserted as unit
  tests (port hblink3's `tests/test_xlx_link.py`, which already encodes
  them), including the reference vector that reproduces a known-good link
  burst byte-for-byte.
- Config validation: module letter A–Z, module numbers rejected with the
  remedy, TS2/TG9 injection, one-bridge and no-unit-target rules.

**Gate:** connect to a public XLX reflector, confirm on *that reflector's*
dashboard that the expected module was joined, and pass audio both ways to
a bridged HBP system. Note the local daemon cannot self-verify this — no
acknowledgement exists (ADAPTERS.md §6.3).

## Phase 3 — Replay harness + dashboard v1

*(PORT removed per D-07; federation is OBP, already built in phase 2.)*

- Replay harness in `tests/replay/`: feed captured wire packets directly
  into an adapter's ingress and capture its egress, no radios and no
  second adapter in the path. Retro-fit phase-2 vectors into automated
  replay tests.
- Dashboard v1 (DASHBOARD.md): backend model + WebSocket, front-end ported
  from hblink4 dashboard, bridge-matrix view, last-heard, peer tables.

**Gate:** two OmniLink instances bridged over OBP pass replayed and live
audio; dashboard tracks a live call end-to-end (call_start latency < 1 s,
survives daemon restart).

## Phase 4 — IPSC adapter

- Port ipsc2hbpc's IPSC engine + jitter-buffer egress clock; peer mode
  first (join existing KS-DMR IPSC network from a **test** instance),
  master mode second.
- Vectors from live IPSC captures (we have production access + dmrlink3 as
  reference decoder).

**Gate:** MOTOTRBO repeater ↔ HBP hotspot through OmniLink, clean audio
both ways, pacing verified against a real repeater. Only after soak
(≥ 1 week shadow running) does it become a candidate to succeed ipsc2hbpc
in production.

## Phase 5 — CC adapter + unit calls

- Lift cccc_link.c and the solved AMBE translation (cccc_ambe.c, D-11)
  verbatim; lock the fix in with ingress/egress conformance vectors
  validated against known-good HBP vectors of the same audio.
- Subscriber registry + unit-call routing (ROUTING.md §6).

**Gate:** CC: clean audio both ways against the production c-Bridge (test
conduit). Unit calls: hotspot↔hotspot private call across systems.

## Phase 6 — Hardening & the wanted extras (backlog, ordered on demand)

Dynamic bridge rules (hblink3 ACTIVE/ON/OFF/timeout/trigger semantics on
the existing `enabled` flag) · repeater-supplied subscriptions via HBP
`options` at login (D-03 limiter semantics) · DATA_BURST forwarding HBP↔IPSC · talker
alias / TALKER_META · ingress ACL polish · multi-core dashboard
aggregation · systemd units + INSTALL.md + CONFIGURING.md to the standard
of the existing repos · soak, fuzz malformed-packet paths, valgrind clean.

## Phase 7 — Playback (talkback) adapter

The optional virtual adapter last, because it is the only adapter that is
purely additive to operators and depends on machinery earlier phases build.

- Implement the transport-less adapter per ADAPTERS.md §7 / D-26: `egress`
  as a capture sink into a bounded, init-sized buffer; clocked
  position-preserving replay via `nx_core_ingress` after CALL_END, re-keyed
  onto the captured TGID and sourced from its own configured 24-bit radio ID.
- **Group calls only** (D-26, revised 2026-08-15). Playback is not registered
  as a reachable subscriber and is never addressed by radio ID — a federated
  design cannot demand a globally unique registered ID per instance. This
  also drops the phase-5 dependency: Playback no longer needs unit routing or
  the subscriber registry, so it could move earlier if ever wanted.
- Conformance (D-11): the loopback-identity vector — replayed frames
  bit-identical to captured frames, since nothing re-encodes.

**Gate:** on a live system, a caller hears their own audio back on the
talkgroup, sourced from the playback ID, with correct timing.

---

## Standing risks

| Risk | Carried where |
|------|---------------|
| Burst reconstruction for IPSC/CC breaks a receiver's expectations (EMB/LC nuance) | Narrowed by D-02: HBP/OBP no longer reconstruct at all, so this risk is now confined to the two legacy adapters, which ship last against a core already proven on air; `dmr/` is the reference |
| Header/LC disagreement in a constructed burst (FRAME.md §4.1 rule 1) | The cost of a burst-native core; invisible to any test that does not decode audio, so it is a conformance-vector requirement, not a review item |
| CC's exotic AMBE representation regresses in the port | D-11 solved it in cc2obp; vectors lock it in; CC ships last anyway |
| A stalled adapter freezes the whole (single-threaded) daemon | Accepted at D-22 scope: systemd watchdog restart + instance federation bound the blast radius (D-04) |
| Slot-arbitration edge cases at busy sites | slot logic confined to adapters; `slot_busy` events make it observable |
| Design/implementation drift under a different model | STYLE.md is binding; DEVIATIONS.md; docs normative |
