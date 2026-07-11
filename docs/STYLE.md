# STYLE.md — Implementation Conventions (binding)

This project will be designed under one model and implemented under others.
These conventions are extracted from ipsc2hbpc and cc2obp — the two C
codebases whose style is the house style — and are **binding on
implementers**. When in doubt, open `ipsc2hbpc/src/` and match it.

## Language and build

- C11 (`-std=c11`), POSIX 2008 (`-D_POSIX_C_SOURCE=200809L`).
- `-Wall -Wextra -Werror -O2`; builds warning-clean or it doesn't merge.
- Dependencies: libc (librt if needed). **Nothing else — not even
  libpthread; the daemon is single-threaded by design (D-04).** No
  libevent, no OpenSSL (crypto.c carries its own SHA/HMAC), no cJSON
  (events are printf-formatted; we only *emit* JSON, never parse it in C).
- One `Makefile`, plain make, `make`/`make test`/`make clean`. Objects live
  beside sources (house habit), `.gitignore` covers them.

## Shape

- One module = `name.c` + `name.h`, header starts with a comment block
  saying what the module is and where it came from (see any existing file).
- Opaque structs (`typedef struct foo foo;` in header, definition in .c)
  for anything with behavior; plain visible structs for pure data
  (`nx_frame`).
- Module-prefixed snake_case: `nx_core_ingress`, `hbp_handle_dmrd`. Constants
  UPPER_SNAKE in `*_const.h` where the list is long (house pattern).
- Lifted modules (`eventloop`, `toml`, `net`, `log`, `crypto`, `dmr/`) are
  copied **unchanged** where possible; local changes to them must be
  commented `/* omnilink: ... */` so future re-syncs with the source repos
  stay tractable. `dmr/` in particular is bit-for-bit shared with
  ipsc2hbpc/cc2obp — treat it as read-only vendored code.

## Discipline

- No allocation on the datapath (ARCHITECTURE.md §4). `malloc` appears in
  `*_new()` at startup and nowhere else. If an implementer needs a dynamic
  structure mid-call, the design is wrong — stop and raise it.
- No concurrency, full stop: no threads, mutexes, semaphores, condvars,
  or atomics. `grep -E 'pthread_|sem_|_Atomic|atomic_'` over `src/` must
  return nothing. If an implementer feels the need for any of these, the
  design has been violated somewhere — stop and raise it.
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
- Unit level: frame pack/unpack (incl. the packed-field accessor macros),
  bridge-table lookup, arbitration/hang state machine, config parse
  (valid + a rejection suite).
- Adapter level: golden conformance vectors per FRAME.md §7 — an adapter
  PR without vectors is incomplete by definition.
- System level: `tests/replay/` Python harness speaking PORT to inject
  captured calls and assert what comes out of another adapter's socket.

## Process

- Git from day zero; commit style as in the existing repos; push
  immediately after committing once a remote exists.
- The docs in `docs/` are normative. Implementation discoveries that
  contradict a doc: **flag it in the commit message and a `docs/DEVIATIONS.md`
  entry, adjust the doc, and keep going** (the flag-then-fix pattern proven
  on cc2obp) — do not silently diverge and do not silently halt.
- GPLv3 headers on new files, N0MJS copyright, matching the house form.
