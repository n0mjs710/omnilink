# STYLE.md — Implementation Conventions (binding)

This project will be designed under one model and implemented under others.
These conventions are extracted from ipsc2hbpc and cc2obp — the two C
codebases whose style is the house style — and are **binding on
implementers**. When in doubt, open `ipsc2hbpc/src/` and match it.

## Language and build

- C11 (`-std=c11`), POSIX 2008 (`-D_POSIX_C_SOURCE=200809L`).
- `-Wall -Wextra -Werror -O2`; builds warning-clean or it doesn't merge.
- **Dependencies: the C11 standard library, POSIX 2008, and glibc, and
  nothing else** (D-32). The test is whether a thing can be counted on
  to still be there and still behave the same way years from now — not
  whether it is popular. libpthread is out for a different reason: the
  daemon is single-threaded by design (D-08).
- Not used, and each for a stated reason: no OpenSSL (`crypto.c` carries
  its own SHA/HMAC), no cJSON (events are `printf`-formatted; the daemon
  emits JSON and never parses it), no libevent (`eventloop.c` is lifted
  and proven). None of these is a quality judgement — they are simply
  unnecessary here.
- One `Makefile`, plain make, `make`/`make test`/`make clean`. Objects live
  beside sources (house habit), `.gitignore` covers them.

## Shape

- One module = `name.c` + `name.h`, header starts with a comment block
  saying what the module is and where it came from (see any existing file).
- Opaque structs (`typedef struct foo foo;` in header, definition in .c)
  for anything with behavior; plain visible structs for pure data
  (`nx_egress_target`).
- Module-prefixed snake_case: `nx_core_ingress`, `hbp_handle_dmrd`. Constants
  UPPER_SNAKE in `*_const.h` where the list is long (house pattern).
- **The `nx_` prefix is reserved for the adapter/core boundary contract**
  (from "nexus" — the meeting point): the core API adapters call
  (`nx_core_*`, `nx_event_emit`, `nx_acl_*`, `nx_stream_tag_next`) and the contract types (`nx_adapter_ops`,
  `nx_egress_target`, `nx_system_cfg`) — i.e., everything declared in
  `adapter.h`, and nothing else. DMRD accessors are `dmrd_*`. Seeing `nx_` means
  "core data or the contract that carries it across the boundary."
  Module internals — including the core's own — use their module prefix
  (`route_`, `event_`, `hbp_`, ...), never `nx_`.
- Lifted modules (`eventloop`, `toml`, `net`, `log`, `crypto`, `dmr/`) are
  copied **unchanged** where possible; local changes to them must be
  commented `/* omnilink: ... */` so future re-syncs with the source repos
  stay tractable. `dmr/` in particular is bit-for-bit shared with
  ipsc2hbpc/cc2obp — treat it as read-only vendored code.

## Discipline

- No allocation on the datapath (ARCHITECTURE.md §4). `malloc` appears in
  `*_new()` at startup and nowhere else. If an implementer needs a dynamic
  structure mid-call, the design is wrong — stop and raise it.
- **The one carve-out is a rules reload** (D-23, CONFIG.md §6.3), which
  builds and frees a table arena on the control plane. Those allocation
  sites carry a `/* control plane: D-23 */` comment so the distinction
  survives a reader who only remembers the headline rule.
- **The daemon is single-threaded as designed** (D-08), so `grep -E
  'pthread_|sem_|_Atomic|atomic_'` over `src/` returns nothing today.
  Treat a hit as a tripwire, not a crime: adding a thread is a design
  change to be raised and decided, never introduced mid-task. If one is
  ever added, what is avoided is locking rather than threading: lock-free
  is fine — atomics for state, an SPSC ring for packets — and locks are
  the last resort.
- Every socket non-blocking; every fd in an `ev_loop`. No `sleep`, no
  blocking `connect` (use the eventloop timer + retry pattern from
  cc2obp `net.c`).
- Check every syscall return; on config/startup errors, print clearly and
  exit non-zero; at runtime, log an event and continue (a routing daemon
  does not crash because one packet was malformed).
- All timestamps `CLOCK_MONOTONIC` internally; wall clock only at the
  event-bus boundary (core stamps `ts`).

## Comments

Match the density and register of ipsc2hbpc: a real header comment per
module explaining design intent; sparse inline comments only where the
*why* isn't visible (protocol quirks, bit-order gotchas, spec section
references like `/* ETSI TS 102 361-1 §9.3 */`). No narration of the
obvious, no changelog comments.

## Testing

- `tests/` mirrors the ipsc2hbpc/cc2obp harness style: small C test
  binaries + shell/Python drivers, run by `make test`, no framework deps.
- Unit level: DMRD field and `bits`-byte accessors, stream-tag maps,
  bridge-table lookup and fan-out, arbitration/hang state machine, the
  dynamic-rule engine, ACL grammar and layering, config parse (valid +
  a rejection suite).
- **Rejection tests assert on the message, not just the failure.** A
  validator that rejects the right file with the wrong explanation has
  failed its actual job (CONFIG.md §6.1).
- Adapter level: golden conformance vectors per FRAME.md §6 — an adapter
  PR without vectors is incomplete by definition.
- System level: `tests/replay/` Python harness feeding captured wire
  packets straight into one adapter's ingress and asserting what comes out
  of another adapter's socket.
- Parity level: `tests/parity/` drives identical traffic through hblink3
  and OmniLink from one converted config and asserts identical routing
  decisions (PLAN.md phase 3). An undocumented divergence is a bug.

## Process

- Git from day zero; commit style as in the existing repos; push
  immediately after committing once a remote exists.
- The docs in `docs/` are normative. Implementation discoveries that
  contradict a doc: **flag it in the commit message and a `docs/DEVIATIONS.md`
  entry, implement the smallest reasonable interpretation, and keep going**
  (the flag-then-fix pattern proven on cc2obp) — do not silently diverge and
  do not silently halt. **Do not edit the doc yourself**: normative doc
  changes are made only by the lead author (D-33). The DEVIATIONS entry
  is how the change gets proposed.
- GPLv3 headers on new files, N0MJS copyright, matching the house form.
