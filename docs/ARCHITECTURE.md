# ARCHITECTURE.md — Process Model, Modules, Memory

C11, single binary, **single thread**, zero external library dependencies
(libc only — not even libpthread). This document defines the scope, the
process model, the module boundaries, and the memory discipline. What crosses the
adapter/core boundary is in FRAME.md; what the core does with it is in
ROUTING.md; how it is configured is in CONFIG.md.

## 1. Scope and sizing (D-20)

OmniLink is a tool for the **state / regional / national-group** level,
with a design ceiling of about **100 configured systems per instance**.
Past that, deployments federate over OpenBridge (D-06) — for resilience,
not performance.

**Size against egress fan-out, not ingress.** One hundred systems
carrying a generous 200 concurrent streams is roughly 3,400 frames/s
arriving. That is not the work. The work is delivery: a 20-member bridge
turns each ingress packet into 19 sends, and an HBP server locally
repeating to 100 connected clients turns each packet into up to 100.
Realistic worst case is on the order of 10⁵ `sendto` calls per second —
two orders of magnitude above the ingress figure, and still comfortably
inside one core given no per-packet allocation.

Size against the fan-out number, not the ingress number.

## 2. Process model: one thread, one loop

```
                ┌─────────────────────────────────────────────┐
                │                 one ev_loop                  │
                │                                              │
  UDP sockets ──┤   adapters                    core           │
  TCP (CC) ─────┤   hbp · ipsc · obp   ◀────▶   route · rules  │
  timers ───────┤   cc  · xlx                   acl   · streams│
  control sock ─┤        │                          │          │
                │        └────▶  event module  ◀────┘          │
                │            (log + control socket)            │
                └─────────────────────────────────────────────┘
```

- Everything runs on one `ev_loop` (`eventloop.c`, lifted from ipsc2hbpc
  unchanged): every socket non-blocking with a read callback, every timer
  a loop timer. This is exactly the proven ipsc2hbpc/cc2obp shape, scaled
  to more sockets.
- Modules interact by **direct function call**. The currency across the
  adapter/core boundary is the DMRD packet itself, `const`, plus the
  system index and stream tag alongside it (D-05, FRAME.md):
  - adapter ingress → `nx_core_ingress(sys, tag, dmrd, len)`
  - core egress → `ops->egress(inst, tag, dmrd, len, target)`
  - anyone → `nx_event_emit(...)`; adapters → `nx_core_system_state(...)`
- The hot path is one synchronous chain with **no queues and no locks**:
  `recvfrom → parse → acl → nx_core_ingress → bridge lookup → arbitrate
  → egress(member) → encode → sendto`, all on one stack. The sole
  internal buffer in the daemon is the IPSC playout clock's small jitter
  buffer (D-15), which exists for the listener, not for the
  architecture.

### The real single-thread hazard is blocking name resolution (D-22)

Not a spinning protocol bug — the watchdog covers that. Both ancestors
resolve hostnames *inside* the connect path, reached from reconnect
timers. At one or two links that is invisible. At a hundred systems, many
outbound by hostname, one unresponsive resolver blocks the entire daemon
for the resolver timeout and drops every call on every system — and
systemd's watchdog does not fire, because the process is alive, just
deaf.

**Resolve at startup only.** Config load turns every hostname into a
`sockaddr` once; the datapath and reconnect timers use the stored address
forever after.

### Why not threads (D-08)

Threads would buy exactly one real thing here: stall isolation. At D-20
scope that trade loses — the machinery cost (rings or locks, atomics,
wakeup plumbing, cross-thread reasoning) outweighs an isolation benefit
that a systemd watchdog plus instance federation already bounds. A
stalled adapter stalls the daemon; the watchdog restarts it in about a
second; calls in flight are lost exactly as they would be at an RF site
taking a power hit.

## 3. Module boundaries

**Core** — the bridge table, the ACL layer, the stream table, per-bridge
arbitration, the dynamic-rule engine, the unit route cache, loop
observation, event sequencing. Policy lives here (D-21): the core decides
*whether and where* a stream goes.

**Adapters** — all protocol machinery: auth, keepalives, endpoint state,
FEC and framing codecs, TS/TGID rewrite execution, slot arbitration,
hang, local repeat, cadence, pacing. Mechanism lives here (D-21):
adapters decide *how* it goes.

**Each configured system is a fully isolated instance** — its own
sockets, endpoints, streams, ACLs, and counters, the moral equivalent of
one hblink3 SYSTEM. The adapter *type* is just shared code.
Inter-system traffic always transits the core, even between two systems
of the same protocol. No shortcuts, or accounting and bridge semantics
lie.

**Event module** — the single serializer for the unified log and the
control socket (EVENTS.md). Single-threading makes strict total order
trivial: emission order *is* the order.

### 3.1 Splitting the HBP adapter, and the shared channel module

HBP carries far more responsibility than any other adapter — it is the
dominant transport (D-05), the only one with a local-repeat obligation
(D-18), and the only one with endpoint-level state. Left as one file it
would be half the daemon. It splits in two, along the seam that D-03
already draws:

- **`hbp_proto.c`** — the wire: DMRD/RPTL/RPTK framing, the login and
  authentication state machine, keepalives and ping-loss tracking, client
  timeout. Knows sockets and bytes; knows nothing about bridges.
- **`hbp_service.c`** — the repeater service: the endpoint table, local
  repeat, per-endpoint delivery filtering and subscriptions,
  hotspot/repeater classification. This is the "HBlink4 role" piece, and
  keeping it in its own file is what makes D-03's boundary legible in the
  code rather than only in prose.

Two further factorings keep protocols homogeneous even though the core
does not require it:

- **`channel.c`** — per-`(system, slot)` outbound stream tracking, hang
  time, and the `stream_to` contention horizon (ROUTING §5.2). This
  logic is *identical* for HBP and IPSC and is the single easiest thing
  in the project to get subtly different in two places, so it exists
  once and both adapters call it.
- **`dmrlc.c`** — Link Control construction, splicing, and the
  header/LC coherence rule (ROUTING §7). Every burst-native adapter
  needs it and it is a standing bug class (D-05).

The result is that `hbp`, `ipsc`, `obp`, `cc`, and `xlx` each reduce to
their genuinely protocol-specific parts, which was the point.

## 4. Memory discipline (D-23)

- **No allocation on the datapath.** All tables, stream slots, and
  per-endpoint pools are sized from config and allocated once at
  startup; steady state runs `malloc`-free. Config parse and event JSON
  formatting into a fixed buffer are the startup and edge exceptions.
- **A rules reload allocates, on the control plane.** It builds a new
  table arena, validates it, swaps a pointer, and frees the old arena
  when no in-flight stream still references it (CONFIG §6.3). The
  allocation sites carry a comment saying which rule they are under, so
  the distinction is not "cleaned up" by someone reading only the
  headline.
- Packets move **by `const` pointer** through the synchronous chain. The
  one component that retains one past the call delivering it is the IPSC
  playout buffer, and it copies. Bounded, sized at init.
- **The core copies nothing.** It never modifies a packet: it passes a
  `const` pointer plus delivery parameters, and the egress adapter makes
  the one copy it was always going to make — to write its own stream ID
  and repeater ID — rewriting the destination in the header and in the
  LC in the same pass (FRAME.md §4.1). One copy per delivery, in the
  place that has to touch the bytes anyway.
- Fixed-capacity pools use freelists with plain indices. Single thread
  means single owner for everything; there are no exceptions to reason
  about.
- Config strings are interned at startup; systems are referred to by
  `uint16_t` index everywhere.
- The **system** table is immutable and `const`-enforced after startup.
  The **rules** table is immutable within a generation.

## 5. Control flow

- **Startup:** parse `omnilink.toml` → build the immutable system table
  → resolve every hostname (D-22) → parse and validate `rules.toml` into
  generation 1 → adapters `init()` (open sockets, register fds and timers)
  → `ev_run`. Any validation failure is fatal and loud (CONFIG §6.4).
- **Housekeeping:**
  - core stream sweep, 500 ms — timeouts free the stream, release
    arbitration, emit `call_end`, send nothing downstream (D-14);
  - dynamic-rule sweep, 10 s — ROUTING §4.4;
  - unit-cache sweep, on the same 10 s tick;
  - adapters own their own protocol timers (keepalives, endpoint expiry,
    playout ticks).
- **Reload:** `SIGHUP` or control command → CONFIG §6.3.
- **Shutdown:** SIGINT/SIGTERM → adapters send protocol goodbyes → flush
  log → exit. No joins, no drain choreography.
- **Log and control writes** are non-blocking on the same loop; a slow
  consumer is disconnected, never waited on (EVENTS.md).

## 6. Source layout

```
omnilink/
├── Makefile                  # -std=c11 -Wall -Wextra -Werror -O2; libc only
├── omnilink.toml.sample
├── rules.toml.sample
├── src/
│   ├── main.c                # config, module init, signals, ev_run
│   ├── dmrd.h                # DMRD accessors + bits byte, see FRAME.md
│   ├── config.c/.h           # TOML → immutable system table
│   ├── rules.c/.h            # TOML → generation-tracked rules arena
│   ├── validate.c/.h         # the one validator (CONFIG §6.1)
│   ├── acl.c/.h              # ACL grammar, layering, admission
│   ├── core.c/.h             # ingress dispatch, stream table
│   ├── route.c/.h            # bridge index, arbitration, dynamic rules
│   ├── unitcache.c/.h        # target route cache (phase 6)
│   ├── channel.c/.h          # per-(system,slot) hang + contention
│   ├── dmrlc.c/.h            # LC build/splice/coherence
│   ├── event.c/.h            # event bus, log, control socket
│   ├── eventloop.c/.h        # lifted from ipsc2hbpc unchanged
│   ├── toml.c  net.c  log.c  crypto.c        # lifted from cc2obp
│   ├── dmr/                  # libdmrdsp, lifted from ipsc2hbpc unchanged
│   └── adapters/
│       ├── adapter.h         # the adapter contract (ADAPTERS.md §1)
│       ├── hbp_proto.c  hbp_service.c
│       └── ipsc.c  obp.c  cc.c  xlx.c
├── tools/
│   └── rules_migrate.py      # rules.py → rules.toml (CONFIG §8)
├── dashboard/                # Python (EVENTS.md)
├── docs/
└── tests/                    # unit tests + golden vectors + replay
```

## 7. Size estimate

Measured against the ancestors rather than guessed, excluding lifted
modules.

| | evidence | estimate |
|---|---|---|
| core (`core`,`route`,`rules`,`config`,`validate`,`dmrd`,`main`) | `bridge.py` is 1,205 dense lines and the validator has no C ancestor | **3.0–3.8 k** |
| `acl` | hblink3's ACL parse + four-layer enforcement, plus range grammar | **0.3–0.45 k** |
| `channel` + `dmrlc` | factored out of what would otherwise be duplicated | **0.4–0.6 k** |
| `event` (bus, log, bidirectional control socket, snapshot serializer) | no ancestor for the snapshot serializer or the command side | **0.6–0.9 k** |
| HBP (`hbp_proto` + `hbp_service`) | `ipsc2hbpc/src/hbp.c` is 312 and is **peer mode only**; master side and repeater service have no C ancestor | **1.3–1.8 k** |
| IPSC | `ipsc.c` 683 (peer only) + clocked egress from `translate.c` | **1.3–1.7 k** |
| CC | `cccc_link.c` 759 + `cccc_ambe.c` 48 | **0.9–1.1 k** |
| OBP | `obp_link.c` 178 | **0.3–0.45 k** |
| XLX | HBP client + module link + validation | **0.15–0.25 k** |
| tests | ipsc2hbpc ships 298 lines, cc2obp 930; D-11 mandates three vector classes × five adapters, plus the parity suite | **2.0–3.0 k** |

**≈8.5–11 kLOC plus ≈2–3 kLOC of tests** — inside the complexity class of
ipsc2hbpc + cc2obp combined. The two items easiest to underestimate are
the validator (specific messages with remedies for five protocols is
700–900 lines by itself) and the dynamic-rule engine, which looks like a
flag and is a state machine with asymmetric triggers.

## 8. What is deliberately not here

- No threads, locks, atomics, rings, or wakeup plumbing (D-08).
- No transcoding, no second air interface (D-02).
- No data-call reassembly, no SQL, no ID-database lookups in the daemon
  — the dashboard resolves names in Python (D-10).
- No stream repair, loss concealment, or re-timing. Sole exception: the
  IPSC egress clock (D-15). The RF-equivalence test (D-13) is the
  standing filter for feature proposals.
- No in-daemon HTTP. External surfaces are the protocol sockets, one Unix
  control socket, and one log file.
- No talkback/echo adapter — `dmr-talkback` already does that as a
  standalone HBP client (D-26).
