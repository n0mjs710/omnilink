# OmniLink — rules for coding agents

One single-threaded C11 daemon: **HBlink3's routing semantics, in C, with
IPSC, CC-CC, OpenBridge, and XLX plugged into the same core.** Read
README.md first, then the doc for your task.

**What crosses the adapter/core boundary is the DMRD packet itself**, plus
a system index and a core-assigned stream tag as arguments (D-05,
FRAME.md). There is no internal frame struct. The core reads the DMRD
header and the `bits` byte and treats the burst as opaque. It is not
protocol-agnostic, by design: OmniLink is DMR only, and DMRD is what the
network runs.

**The design corpus was rebuilt from zero on 2026-08-21.** Everything
predating that commit is withdrawn. If you find a document, comment, or
issue describing a representation-neutral core, TGID-only routing, a
`PORT` trunk, a Playback/talkback adapter, foreign digital modes, or "no
config reload," it is a leftover — treat it as absent and tell the owner.

## Non-negotiable

- **`docs/` are normative.** Code is written to them. Never edit a doc to
  match your code. If reality contradicts a doc, append an entry to
  `docs/DEVIATIONS.md` (what, where, why, proposed fix), implement the
  smallest reasonable interpretation, and keep going — flag-then-fix.
  Doc changes themselves are made only by the project owner's session.
- **Design questions the docs don't answer go to the owner (N0MJS).** Do
  not improvise policy. Routing semantics, timers, protocol behavior
  choices — ask, don't guess.
- **hblink3 is the tie-breaker.** Where the docs are silent on a routing
  question, hblink3's `bridge.py` behavior is the answer (D-03). There
  are exactly three deliberate departures — D-16 (per-bridge talker
  holder), D-29 (delivery set resolved once, grow-only), D-30 (call end
  is call end, terminator or timeout). Each is marked **[departure]** at
  its site in ROUTING.md. Do not invent a fourth.
- **The daemon is single-threaded** (D-08) and
  `grep -E 'pthread_|sem_|_Atomic|atomic_'` over `src/` returns nothing.
  That is the current design, not a taboo — but adding a thread is a
  design change for the owner to make, so stop and raise it rather than
  introducing one. What is actually avoided is **locking**, not threads:
  lock-free is fine — atomics for state, an SPSC ring for packets.
- **No malloc on the datapath.** Allocation happens in `*_new()`/init at
  startup. The single carve-out is a rules reload, which is control-plane
  work and is commented as such (D-23). Synthesize nothing.
  Full rules: STYLE.md.
- **`getaddrinfo` appears in `config.c` and nowhere else** (D-22). One
  blocking resolve on a reconnect timer deafens the whole daemon.
- Read the referenced doc sections for your task before writing code.
  FRAME.md layouts are byte-exact and tested as such.

## Things that look like bugs and are not

Check here before "fixing" one of these:

- **One ingress stream feeding several bridges** is correct (D-04). The
  bridge index maps to a *list*.
- **Activation triggers fire on call start, deactivation on call end.**
  Read ROUTING §4.1 before changing either.
- **Same TGID from a different source is admitted during hang.** Getting
  this "right" locks out every round-table net (D-16).
- **A locally repeated call emits two `call_start` events.** Consumers
  fold them on the stream tag (D-17).
- **`timeout` in rules config is minutes**, ×60 at load, because that is
  what existing `rules.py` files mean (CONFIG §4).
- **IPSC egress emits terminators and comfort silence.** Sole carve-outs,
  IPSC wire only, and they never enter the core (D-14, D-15).
- **HBP is `server`/`client`; IPSC is `master`/`peer`.** Not a style
  inconsistency — HBP is strictly client/server, IPSC is a real
  peer-to-peer mesh whose master just holds the bootstrap list. Do not
  normalize them (CONFIG §2.5). `endpoint` is the neutral term where the
  core or event schema must span protocols.
- **The core enforces no ACLs** (D-31). Adapters do, at ingress, by
  querying `acl.c`. TGID ACLs are HBP-only — they limit local repeat,
  the one thing bridge rules cannot reach.
- **A burst can carry a "wrong" colour code end to end.** Never read it,
  never rewrite it, build at CC 1 (D-28). `dmr/` needs no Golay(20,8)
  encoder; `xlx_slot_type()` is not ported; `colorcode` exists only in
  a system's login self-description (CONFIG §2.3).

## Definition of done

- `make` warning-clean (`-Wall -Wextra -Werror`), `make test` green.
- Adapter work additionally requires conformance vectors per FRAME.md §6
  — ingress, egress, loopback identity. **No vectors, not done** (D-11).
- Validator work requires rejection tests that assert on the *message*,
  not just the failure (CONFIG §6.1).
- Phase gates in PLAN.md are the acceptance criteria; do not declare a
  phase complete without its gate.

## Housekeeping

- Commit at each green step; push immediately after committing.
- GPLv3 header (N0MJS copyright) on new files, matching existing form.
- Lifted modules (`dmr/`, `eventloop`, `toml`, `net`, `log`, `crypto`)
  are vendored read-only; local changes need a `/* omnilink: */` comment
  and a DEVIATIONS entry. There are **no** planned exceptions — `dmr/` in
  particular is complete as vendored (D-28).
