# PLAN.md — Phased Build Plan

Ordering principle: get real RF audio flowing through the canonical core as
early as possible (phase 2), because the canonical-frame reduction is the
one genuinely novel risk; everything after that is porting things we've
already built once. Each phase has an **acceptance gate**; a phase isn't
done until its gate passes, and production migration only follows gates
(D-13).

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
This gate proves the entire thesis (burst → 49-bit AMBE → burst).

## Phase 3 — PORT adapter + replay harness + dashboard v1

- PORT adapter (ADAPTERS.md §6) + Python nx_frame codec in `tests/replay/`
  → capture/inject canonical calls without radios; retro-fit phase-2
  vectors into automated replay tests.
- Dashboard v1 (DASHBOARD.md): backend model + WebSocket, front-end ported
  from hblink4 dashboard, bridge-matrix view, last-heard, peer tables.

**Gate:** two OmniLink instances bridged over PORT pass replayed and live
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
  position-preserving replay via `nx_core_ingress` after CALL_END; its own
  configured 24-bit radio ID registered in the subscriber registry (phase 5)
  so unit calls route to it. Group and unit modes fall out of mirroring the
  inbound call type onto the reply's `dst`.
- Conformance (D-11): the loopback-identity vector — replayed frames
  bit-identical to captured frames, since nothing re-encodes.

**Gate:** on a live system, a caller hears their own audio back, sourced from
the playback ID, with correct timing — in **both** group mode (re-keyed onto
the TGID) and unit mode (private reply to the caller). Depends on phase-5
unit routing + subscriber registry.

---

## Standing risks

| Risk | Carried where |
|------|---------------|
| 49-bit reduction breaks some receiver's expectations (EMB/LC nuance) | Phase 2 gate is early *because* of this; `dmr/` is the proven reference |
| CC's exotic AMBE representation regresses in the port | D-11 solved it in cc2obp; vectors lock it in; CC ships last anyway |
| A stalled adapter freezes the whole (single-threaded) daemon | Accepted at D-22 scope: systemd watchdog restart + instance federation bound the blast radius (D-04) |
| Slot-arbitration edge cases at busy sites | slot logic confined to adapters; `slot_busy` events make it observable |
| Design/implementation drift under a different model | STYLE.md is binding; DEVIATIONS.md; docs normative |
