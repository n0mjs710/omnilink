# OmniLink — rules for coding agents

One single-threaded C11 daemon: **HBlink3's routing semantics, in C, with
IPSC, CC-CC, OpenBridge, and XLX plugged into the same core.** Read
README.md first, then the doc for your task.

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
  question, hblink3's `bridge.py` behavior is the answer (D-03). There is
  exactly one deliberate departure from it, D-16 (the per-bridge talker
  holder). Do not invent a second one.
- **Concurrency is banned.** `grep -E 'pthread_|sem_|_Atomic|atomic_'`
  over `src/` must return nothing. No threads, no locks, no atomics. If
  you feel you need one, stop — you've misread the design.
- **No malloc on the datapath.** Allocation happens in `*_new()`/init at
  startup. The single carve-out is a rules reload, which is control-plane
  work and is commented as such (D-23). No frame synthesis anywhere.
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
  That asymmetry is deliberate and load-bearing (ROUTING §4.1).
- **Same TGID from a different source is admitted during hang.** Getting
  this "right" locks out every round-table net (D-16).
- **A locally repeated call emits two `call_start` events.** Consumers
  fold them on `(origin_system, stream_tag)` (D-17).
- **`timeout` in rules config is minutes**, ×60 at load, because that is
  what existing `rules.py` files mean (CONFIG §4).
- **IPSC egress emits terminators and comfort silence.** Sole carve-outs,
  IPSC wire only, and they never become frames (D-14, D-15).

## Definition of done

- `make` warning-clean (`-Wall -Wextra -Werror`), `make test` green.
- Adapter work additionally requires conformance vectors per FRAME.md §7
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
  and a DEVIATIONS entry. The one planned exception is the Golay(20,8)
  slot-type encoder added to `dmr/` in phase 0 (FRAME.md §4.1) — do that
  deliberately and upstream it.
