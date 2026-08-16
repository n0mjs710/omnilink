# OmniLink Architecture

C11, single binary, **single thread**, zero external library dependencies
(libc only — not even libpthread). This document defines the scope, the
process model, the module boundaries, and the memory discipline. The
frame it moves is in `FRAME.md`; what the core does with it is in
`ROUTING.md`.

## 1. Scope declaration (D-22) — read this first

OmniLink is a tool for the **state / regional / national-group** level.
Its design ceiling is **~100 configured systems per instance**. It is
explicitly *not* a platform for running a worldwide network, and a single
install will never carry several statewide networks' full traffic loads:
past the ceiling, deployments **should** federate multiple instances over
OBP — not for performance, but for resilience (too many eggs in one
basket is an operational failure mode regardless of CPU headroom).

Sizing at the ceiling: 100 systems carrying a generous 200 concurrent
streams is ~3,400 frames/s ≈ 220 KB/s of internal traffic, with
microsecond-scale per-burst DSP (Golay/BPTC via `dmr/`). That is
single-digit percent of one core. **No performance or scaling argument
for concurrency exists at this scope** — which is why the process model
below is the simplest one that works.

## 2. Process model: one thread, one loop

```
                 ┌────────────────────────────────────────┐
                 │            one ev_loop                  │
                 │                                         │
   UDP sockets ──┤  adapter instances        core          │
   TCP (CC) ─────┤  (hbp, ipsc, obp,   ◀──▶ (routing,     │
   timers ───────┤   cc)                      streams,     │
   unix socket ──┤                            bridges)     │
                 │            │                  │         │
                 │            └──▶ event module ◀┘         │
                 │                 (log + dashboard socket)│
                 └────────────────────────────────────────┘
```

- Everything runs on one `ev_loop` (`eventloop.c`, lifted from ipsc2hbpc
  unchanged): every socket non-blocking with a read callback, every timer
  a loop timer. This is exactly the proven ipsc2hbpc/cc2obp shape, scaled
  to more sockets.
- Modules interact by **direct function call**, with `const nx_frame *`
  as the only currency between adapters and core:
  - adapter ingress → `nx_core_ingress(const nx_frame *f)`
  - core egress → `ops->egress(system, const nx_frame *f)`
  - anyone → `nx_event_emit(...)`, adapters → `nx_core_system_state(...)`
- The hot path is a single synchronous chain with **zero queues, zero
  copies, zero locks**: `recvfrom → parse/reduce → nx_core_ingress →
  bridge lookup → egress(member) → encode → sendto`, all on one stack.
  The sole internal buffer in the entire daemon is the IPSC playout
  clock's small jitter buffer (ADAPTERS.md §3), which exists for the
  listener, not for the architecture.

### Why not threads (D-04)

Threads would buy exactly one real thing here: stall isolation (a
spinning protocol bug freezing only its own protocol). At D-22 scope
that trade loses — the machinery cost (rings or locks, atomics, wakeup
plumbing, cross-thread reasoning) outweighs an isolation benefit that a
systemd watchdog plus instance federation already bounds. A stalled
adapter stalls the daemon; the watchdog restarts it in ~1 s; calls in
flight are lost exactly as they would be on an RF site taking a power
hit. The adapter contract (§3) remains a clean seam: adapters could move
behind rings (threads) or sockets (processes) without core changes if
the calculus ever changes, but that is not a supported build target.

## 3. Module boundaries

- **Core** (`core.c`, `route.c`): bridge table, stream table, talker
  arbitration, hang time, loop suppression, unit route cache (phase 5),
  event sequencing. Policy lives here (D-23) — the core decides *whether
  and where* a stream goes.
- **Adapters** (`adapters/*.c`): all protocol machinery — auth,
  keepalives, peer state, FEC/framing codecs, TS/TGID rewrite execution,
  slot arbitration, local repeat, cadence, pacing. Mechanism lives here
  (D-23) — adapters decide *how* it goes.
- **Each configured system is a fully isolated instance** (its own
  sockets, peers, streams, ACLs, counters — one hblink3 SYSTEM's moral
  equivalent). The adapter *type* is just shared code. Inter-system
  traffic always transits the core, even between systems of the same
  protocol — no shortcuts, or accounting and bridge semantics lie.
- **Event module** (`event.c`): single serializer for the unified log
  and the dashboard socket (DASHBOARD.md). Single-threading makes the
  strict total order trivial: emission order *is* the order.

The adapter contract in full is `ADAPTERS.md` §1.

## 4. Memory discipline

- **No allocation on the datapath.** All tables, stream slots, and
  per-peer pools are sized from config and allocated once at startup;
  steady state runs `malloc`-free. (Config parse and event JSON
  formatting into a fixed 2 KiB buffer are the startup/edge exceptions.)
- Frames move **by `const` pointer** through the synchronous chain —
  nothing retains a frame past the call that delivered it, except the
  IPSC playout buffer, which copies the frames it holds (64 B each,
  FRAME.md).
- Fixed-capacity pools use freelists with plain indices; single thread
  means single owner for everything, no exceptions to reason about.
- Config strings are interned at startup; systems are referred to by
  `uint16_t` index everywhere (`origin_system` in the frame).
- After startup, config structures are immutable and `const`-enforced
  (no reload — D-10; restart on change).

## 5. Control flow

- **Startup:** parse TOML → build immutable system/bridge tables →
  adapters `init()` (open sockets, register fds/timers on the loop) →
  `ev_run`.
- **Housekeeping:** core timer at 500 ms (stream timeouts → free + hang
  release + `call_end` event; nothing synthesized downstream — D-16);
  adapters own their protocol timers (keepalives, peer expiry, playout
  ticks).
- **Shutdown:** SIGINT/SIGTERM → adapters send protocol goodbyes →
  flush log → exit. No joins, no drain choreography.
- **Log/dashboard writes** are non-blocking on the same loop; a slow
  consumer is disconnected, never waited on (DASHBOARD.md §4).

## 6. Source layout

```
omnilink/
├── Makefile                    # -std=c11 -Wall -Wextra -Werror -O2; libc only
├── omnilink.toml.sample
├── src/
│   ├── main.c                  # config, module init, signals, ev_run
│   ├── frame.h                 # nx_frame — normative, see FRAME.md
│   ├── core.c/.h               # ingress dispatch, stream table
│   ├── route.c/.h              # bridge table, arbitration (ROUTING.md)
│   ├── subscriber.c/.h         # target route cache (phase 5)
│   ├── event.c/.h              # event bus: serialize, log, dash socket
│   ├── eventloop.c/.h          # lifted from ipsc2hbpc unchanged
│   ├── toml.c/.h  net.c/.h  log.c/.h  crypto.c/.h   # lifted from cc2obp
│   ├── config.c/.h             # TOML → immutable tables
│   ├── dmr/                    # libdmrdsp, lifted from ipsc2hbpc unchanged
│   └── adapters/
│       ├── adapter.h           # the adapter contract (ADAPTERS.md §1)
│       ├── hbp.c  ipsc.c  obp.c  cc.c
├── dashboard/                  # Python (DASHBOARD.md)
├── docs/
└── tests/                      # unit tests + golden vectors + replay
```

Estimated new C (excluding lifted modules): core ≈ 2 kLOC, adapters ≈
0.8–1.5 kLOC each — comfortably inside the complexity class of
ipsc2hbpc + cc2obp combined, now without any concurrency machinery.

## 7. What is deliberately NOT here

- No threads, locks, atomics, rings, or wakeup plumbing — see "Why not
  threads" above (D-04).
- No transcoding, no data-call reassembly, no SQL, no ID-database
  lookups in the daemon (the dashboard resolves names in Python).
- No stream repair, loss concealment, or re-timing (sole exception:
  IPSC egress clock, D-17). The RF-equivalence test (D-21) is the
  standing filter for feature proposals: DMR already survives late
  entry, gaps, and vanished signals at the receiver; we intervene only
  where an IP failure mode has no RF analog (loops, cross-network floor
  control, identity, operator truth).
- No dynamic bridge control until phase 6 (`enabled` flag exists from
  day one).
- No in-daemon HTTP. External surfaces: protocol sockets, one Unix
  event socket, one log file.
