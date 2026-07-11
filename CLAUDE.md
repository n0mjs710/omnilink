# OmniLink — rules for coding agents

One single-threaded C11 daemon (see README.md). The design is complete
and frozen; you are implementing it, not designing it.

## Non-negotiable

- **`docs/` are normative.** Code is written to them. Never edit a doc
  to match your code. If reality contradicts a doc, append an entry to
  `docs/DEVIATIONS.md` (what, where, why, proposed fix), implement the
  smallest reasonable interpretation, and keep going — flag-then-fix.
  Doc changes themselves are made only by the project owner's session.
- **Design questions the docs don't answer go to the owner (N0MJS).**
  Do not improvise policy. Routing semantics, timers, protocol behavior
  choices — ask, don't guess.
- **Concurrency is banned.** `grep -E 'pthread_|sem_|_Atomic|atomic_'`
  over `src/` must return nothing. No threads, no locks, no atomics.
  If you feel you need one, stop — you've misread the design.
- **No malloc on the datapath**; allocation happens in `*_new()`/init
  at startup only. No frame synthesis anywhere. Full rules: STYLE.md.
- Read the referenced doc sections for your task before writing code;
  FRAME.md layouts are byte-exact and tested as such.

## Definition of done

- `make` warning-clean (`-Wall -Wextra -Werror`), `make test` green.
- Adapter work additionally requires conformance vectors per FRAME.md
  §7 — ingress, egress, loopback identity. No vectors, not done.
- Phase gates in PLAN.md are the acceptance criteria; do not declare a
  phase complete without its gate.

## Housekeeping

- Commit at each green step; push immediately after committing.
- GPLv3 header (N0MJS copyright) on new files, matching existing form.
- Lifted modules (`dmr/`, `eventloop`, `toml`, `net`, `log`, `crypto`)
  are vendored read-only; local changes need a `/* omnilink: */` comment
  and a DEVIATIONS entry.
