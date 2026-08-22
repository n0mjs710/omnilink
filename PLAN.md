# PLAN.md — Phased Build Plan

**Ordering principle:** get real RF audio through the core as early as
possible, then make it a credible hblink3 replacement, then add the
legacy edges. The routing core is the only novel part;
everything after it is porting things we have already built once.

Each phase has an **acceptance gate**. A phase is not done until its gate
passes.

Two consequences of D-12 shape this plan:

- **ACLs and the dynamic-rules engine are phase 1**, not backlog. A
  network cannot cut over to a router that will not express its current
  rules or admit its current repeaters.
- **Phase 3 exists at all** because "replaces hblink3" is a claim that
  has to be demonstrated against a real hblink3 config, not asserted.

---

## Phase 0 — Skeleton

- Repo scaffolding: Makefile (`-std=c11 -Wall -Wextra -Werror -O2`,
  no dependencies beyond D-32), directory layout per ARCHITECTURE §6.
- Lifted modules copied in: `dmr/`, `eventloop`, `toml`, `net`, `log`,
  `crypto`. `make test` runs their existing self-tests under this tree.
- **`dmr/` is complete as vendored and needs no additions** (D-28).
  Constructed bursts are built at colour code 1, so the precomputed
  `DMR_SLOT_TYPE_VHEAD`/`VTERM` and `DMR_EMB` tables are exactly right.
  Do not add a Golay(20,8) slot-type encoder.

**Gate:** clean build, lifted-module tests green.

## Phase 1 — Core (the biggest single phase)

- **`dmrd.h`** — DMRD field and `bits`-byte accessors (explicit
  shift/mask, no bitfields), unit-tested against FRAME §1, plus the
  stream-tag allocator and the ingress/egress map helpers (FRAME §3.1).
- **`config.c`** — `omnilink.toml` → the immutable system table;
  hostname resolution at startup (D-22).
- **`rules.c` + `validate.c`** — `rules.toml` → a generation-tracked
  rules arena, with the full rejection suite of CONFIG §6.1. Every
  rejection names its line and its remedy; the tests assert on the
  *messages*, not just on the failure.
- **`omnilink --check-config`** — same validator, exits non-zero
  (CONFIG §6.2).
- **`acl.c`** — the two-part grammar, four ACL types, global-then-system
  layering, first-denial-wins, fail-closed (ADAPTERS §1.3). Enforced by
  adapters, not the core (D-31).
- **`core.c` / `route.c`** — stream table; the `(system, slot, tgid)`
  bridge index with multi-bridge fan-out; per-bridge arbitration; the
  dynamic-rules engine with its asymmetric triggers and 10 s sweep;
  timeouts; loop cadence detection. Unit-tested against scripted packet
  sequences — no sockets needed, call `nx_core_ingress` directly.
- **`channel.c`, `dmrlc.c`** — shared hang/contention and LC handling,
  unit-tested standalone.
- **`event.c`** — event bus, unified log, bidirectional control socket,
  snapshot serializer (EVENTS §3–§7).
- **Live reload** — validate-then-swap, generation-tracked arenas,
  in-flight streams pinned (CONFIG §6.3).
- **`main.c`** — startup, shutdown, signals; a `null` test adapter that
  loops packets and emits events, proving the loop and event plumbing end
  to end.

**Gate:** `make test` green, including —

1. A two-fake-adapter routing test: a scripted call on fake A appears on
   fake B with correct TGID rewrite, arbitration, hang behavior, and a
   correct event trail on the socket.
2. A **fan-out** test: one ingress stream matching three bridges is
   delivered to all three, with per-bridge arbitration independent.
3. A **dynamic-rules** test covering activation on call start,
   deactivation on call end, timer reset from own-TGID traffic, sweep
   timeout, and **timeout deferral while a call is in progress** — the
   most user-visible thing to get wrong (ROUTING §4.4).
4. An **ACL** test asserting enforcement order, that an unlisted ID gets
   the opposite action, and that a `tgid_ts*_acl` on a non-HBP system is
   rejected.
5. A **reload** test: a bad rules file changes nothing and reports its
   findings; a good one swaps; a call in flight across the swap completes
   on its original rules.

## Phase 2 — HBP + OBP + XLX (first real audio)

- **HBP adapter**, split per ARCHITECTURE §3.1: `hbp_proto.c` (server
  mode first, client mode second) and `hbp_service.c` (endpoint table,
  local repeat, static per-endpoint delivery filtering, endpoint
  classification). Conformance vectors cross-checked against
  `dmr_utils3` output.
- **OBP adapter** — lift `obp_link.c`; per-peer stream pools and stream
  dedupe.
- **XLX adapter** — HBP client plus the module link. Port hblink3's
  `tests/test_xlx_link.py`, which already encodes xlxd's five acceptance
  gates as assertions, including the reference vector that reproduces a
  known-good link burst byte for byte.
- Live test: a hotspot or repeater on OmniLink-HBP bridged to (a) another
  HBP system and (b) HBlink4 over OBP.

**Gate:** clean audio and correct metadata (src, dst, endpoint on the far
dashboard), correct hang behavior, in **both directions** on both pairs —
the live-verified standard used for the hblink4 OpenBridge work.

**XLX rider gate, separate and non-blocking:** connect to a public XLX
reflector, confirm on *that reflector's* dashboard that the expected
module was joined, and pass audio both ways to a bridged HBP system. The
local daemon cannot self-verify this — no acknowledgement exists
(ADAPTERS §4.3) — so this gate depends on scheduling, not engineering,
and must not hold up the phase-2 audio gate.

## Phase 3 — hblink3 parity (the cutover gate)

This is the phase that makes "replaces hblink3" true rather than
asserted.

- **`tools/rules_migrate.py`** — convert `rules.py` + `hblink.cfg` to
  `rules.toml` + `omnilink.toml`, reporting anything it cannot represent
  rather than guessing, and running its own output through
  `--check-config` (CONFIG §8).
- **The parity suite** — take a real `rules.py`, convert it,
  and drive identical scripted traffic through both hblink3 and OmniLink,
  asserting identical routing decisions: same members reached, same
  rewrites, same arbitration outcomes, same dynamic-rule state
  transitions. Differences are either bugs or documented departures
  (there is exactly one, D-16), and any undocumented difference blocks
  the gate.
- **Shadow deployment** — OmniLink alongside a live hblink3, on separate
  ports, fed real traffic, with its decisions logged and compared.
- **Dashboard v1** (EVENTS §8): backend model, WebSocket, front-end
  ported from hblink4, bridge-matrix view, last-heard, endpoint tables.

**Gate:** the parity suite passes on a real config; one week of shadow
running with no unexplained routing divergence; the dashboard tracks a
live call end to end (`call_start` within 1 s, survives daemon restart);
a rules reload is demonstrated on the shadow instance with a call in
flight.

**Only after this gate** is OmniLink a candidate to succeed hblink3 on a
live network. What an operator does with that is theirs; this plan builds
the thing and proves it, and stops there.

## Phase 4 — IPSC

- Port ipsc2hbpc's IPSC engine plus the jitter-buffer egress clock. Peer
  mode first — join an existing IPSC network from a **test** instance —
  master mode second.
- Vectors from live IPSC captures, with dmrlink3 as the reference
  decoder.

**Gate:** MOTOTRBO repeater ↔ HBP hotspot through OmniLink, clean audio
both ways, pacing verified against a real repeater, plus at least one
week of shadow running without divergence.

## Phase 5 — CC-CC

- Lift `cccc_link.c` and the solved AMBE translation (`cccc_ambe.c`)
  **verbatim**; lock the fix in with ingress and egress conformance
  vectors validated against known-good HBP vectors of the same audio
  (D-11).
- Bound-endpoint validation: one bridge, injected nominal slot, exposed
  TGID, never a unit target (D-07).

**Gate:** clean audio both ways against a c-Bridge over a test conduit,
with the AMBE vectors green.

## Phase 6 — Unit calls, data, hardening

- Unit-call routing and the target route cache (ROUTING §6).
- `DATA_BURST` forwarding between burst-native adapters (D-27).
- Repeater-supplied subscriptions via HBP `options` at login, with D-03
  limiter semantics.
- Talker alias / `TALKER_META`.
- Loop circuit breaker as configurable policy (D-19).
- Multi-instance dashboard aggregation (EVENTS §9).
- systemd units, `INSTALL.md`, `CONFIGURING.md` to the standard of the
  existing repos.
- Soak, fuzz the malformed-packet paths, valgrind clean.

**Gate:** hotspot ↔ hotspot private call across systems; a fuzzed
malformed-packet corpus produces no crash and no leak.

## Phase 7 — Web rules editor

Last, because it is purely additive and blocks nothing (D-09, CONFIG §7).

- Reads and writes `rules.toml`; no database, no second source of truth.
- Validates by shelling `omnilink --check-config`.
- Applies via `reload` on the control socket, surfacing findings verbatim
  when a reload is refused.
- Preserves comments and formatting where practical.

**Gate:** an operator edits a bridge in the browser, the change lands in
git-diffable TOML, and the daemon picks it up with a call in flight and
no audio interruption.

---

## Standing risks

| risk | carried where |
|---|---|
| Header/LC disagreement in a constructed or retargeted burst | Invisible to any test that does not decode audio, so it is a conformance-vector requirement (D-11), not a review item. `dmrlc.c` keeps it in one place. |
| Synthesized burst structure breaks a receiver's EMB/LC expectations | Confined to **CC-CC**, the only adapter without burst and superframe data on its wire. HBP, OBP, and XLX carry the assembled burst; IPSC carries every element unpacked and re-packs them. CC-CC also ships last, against a core already proven on air. `dmr/` is the reference. |
| CC's exotic AMBE representation regresses in the port | Solved once in cc2obp; vectors lock it in; CC ships last anyway (D-11). |
| Parity suite finds hblink3 behavior the docs did not capture | Expected, and the reason phase 3 exists. Each finding is either a bug fixed or a DEVIATIONS entry plus a DECISIONS update — never a silent difference. |
| Reload swaps rules under a live call | Directly addressed by generation-pinned streams (CONFIG §6.3), and explicitly tested in the phase-1 gate. |
| A stalled adapter freezes the single-threaded daemon | Accepted at D-20 scope: systemd watchdog restart plus instance federation bound the blast radius (D-08). |
| Blocking name resolution deafens the daemon at scale | Structurally prevented: resolve at startup only (D-22). Grep for `getaddrinfo` outside `config.c` is a review check. |
| Slot-arbitration edge cases at busy sites | Slot logic confined to `channel.c`; `slot_busy` events make it observable (D-17). |
| Design/implementation drift under a different model | STYLE.md is binding; DEVIATIONS.md is the pressure valve; docs are normative. |
