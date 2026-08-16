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
streams is ~3,400 frames/s of *ingress*. The work is **egress fan-out**,
and that is the number to size against: a 20-member bridge turns each
ingress frame into 19 sends, and an HBP master locally repeating to 100
connected repeaters turns each frame into up to 100. Realistic worst case
is on the order of 10^5 `sendto` calls per second, not 10^3 — two orders
above the ingress figure, and still comfortably inside one core with
microsecond-scale per-burst DSP (Golay/BPTC via `dmr/`) and no per-frame
allocation. **No performance or scaling argument for concurrency exists
at this scope** — which is why the process model below is the simplest
one that works. Quote the fan-out number, not the ingress number; the
ingress number is not the one that matters.

## 2. Process model: one thread, one loop

```
                 ┌────────────────────────────────────────┐
                 │            one ev_loop                  │
                 │                                         │
   UDP sockets ──┤  adapter instances        core          │
   TCP (CC) ─────┤  (hbp, ipsc, obp,   ◀──▶ (routing,     │
   timers ───────┤   cc, xlx)                 streams,     │
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
  - core egress → `ops->egress(inst, const nx_frame *f, const nx_egress_target *t)`
  - anyone → `nx_event_emit(...)`, adapters → `nx_core_system_state(...)`
- The hot path is a single synchronous chain with **zero queues, zero
  copies, zero locks**: `recvfrom → parse/reduce → nx_core_ingress →
  bridge lookup → egress(member) → encode → sendto`, all on one stack.
  The sole internal buffer in the entire daemon is the IPSC playout
  clock's small jitter buffer (ADAPTERS.md §3), which exists for the
  listener, not for the architecture.

### The real single-thread hazard is blocking name resolution

Not a spinning protocol bug — the watchdog covers that. Both ancestors
resolve hostnames *inside* the connect path (`cc2obp/net.c`,
`ipsc2hbpc/src/net.c`), which is reached from reconnect timers. At one or
two links that is invisible. At ~100 systems, many outbound by hostname
(HBP peer, XLX, OBP), **one unresponsive resolver blocks the entire
daemon** for the resolver timeout and drops every call on every system —
and systemd's watchdog does not fire, because the process is alive, just
deaf.

Therefore: **resolve at startup only.** Config load turns every hostname
into a `sockaddr` once and stores it; the datapath and reconnect timers
use the stored address and never call `getaddrinfo`. An address that
changes needs a restart, which is consistent with D-10 anyway. If
re-resolution is ever wanted it belongs on a slow timer that tolerates
failure, never in a connect path.

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
  arbitration (no hang — that is a channel property, ROUTING.md §4),
  loop observation (pattern alarm only, D-25),
  unit route cache (phase 5),
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
- Frames move **by `const` pointer** through the synchronous chain. Two
  components retain a frame past the call that delivered it, and both
  copy: the IPSC playout buffer, and the Playback adapter's capture
  buffer (ADAPTERS.md §7). 64 B each, both bounded and sized at init.
- The core does make **one stack copy per egress member**, since it
  rewrites `dst_id` to the member's TGID from a single `const` source
  frame. "Zero copies" below means no heap traffic and no queues, not
  literally zero — one 64-byte copy per delivery is the cost of routing.
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
- **Housekeeping:** core timer at 500 ms (stream timeouts → free stream +
  release bridge holder + `call_end` event; nothing synthesized
  downstream — D-16);
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
│       ├── hbp.c  ipsc.c  obp.c  cc.c  xlx.c  playback.c
├── dashboard/                  # Python (DASHBOARD.md)
├── docs/
└── tests/                      # unit tests + golden vectors + replay
```

Estimated new C, excluding lifted modules and measured against the
ancestors rather than guessed:

| | evidence | estimate |
|---|---|---|
| core (`core`,`route`,`event`,`config`,`frame`,`main`,`subscriber`) | `bridge.py` is 1,205 dense lines; `cc2obp/config.c` is 360 for a far smaller validation surface; the plane-complete snapshot serializer (DASHBOARD.md §6) has no ancestor | **3.0–3.8 k** |
| HBP | `ipsc2hbpc/src/hbp.c` is 312 and is **peer mode only**; the master side has no C ancestor | **1.1–1.5 k** |
| IPSC | `ipsc.c` 683 (peer only) + the clocked egress now in `translate.c` | **1.3–1.7 k** |
| CC | `cccc_link.c` 759 + `cccc_ambe.c` 48 | **0.9–1.1 k** |
| OBP | `obp_link.c` 178 | **0.3–0.45 k** |
| XLX | HBP peer + link + validation | **0.15–0.25 k** |
| tests | `ipsc2hbpc` ships 298 lines, `cc2obp` 930; D-11 mandates three vector classes × six adapters | **1.5–2.5 k** |

**≈7–9 kLOC plus ≈2 kLOC of tests.** That is larger than a first pass
suggests, and still inside the complexity class of ipsc2hbpc + cc2obp
combined, which is the claim that matters. Two line items are easy to
miss: `config.c` for six protocols with named validations and remedy text
is 700–900 lines by itself, and the snapshot serializer is a real 300–500
line module. Phase 1 is correspondingly the biggest phase, as PLAN.md
already says.

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
