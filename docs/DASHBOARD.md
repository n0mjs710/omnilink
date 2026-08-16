# DASHBOARD.md — Unified Event Bus, Logging, Dashboard

The requirement: five protocol modules plus a core, logging **in unison**,
feeding **one** dashboard. The mechanism: every component emits semantic
events; the event module is the single serialization point; everything
human-facing (text log, dashboard, future alerting) is a *rendering* of
the one ordered event stream.

## 1. Information architecture: three planes

The dashboard is organized around what each part of the daemon actually
knows — which is the D-23 policy/mechanism line made visible:

**Systems plane** (source: adapters — mechanism). Per-system panels, the
direct descendants of today's hblink4/dmrlink3 dashboards: peer/repeater
tables (with endpoint class — hotspot/repeater — per D-03), connection
state, and live activity — TS pills for slotted systems, concurrent
stream pills for OBP. This is most of the dashboard, because most
of what happens, happens in systems.

**Bridge plane** (source: core — policy). The router's one table: each
bridge, its members, and per member the (system, TGID, slot) rewrite
parameters — this *is* the rewrite made visible — plus enabled/active
status, hang time, and the current talker/holder. Phase 6 adds the
dynamic-rule columns (timeout, timeout type, trigger, reset) on the same
rows. The bridge matrix view (bridges × members, live talker highlighted)
renders this plane — the one visualization no per-protocol dashboard
could ever show.

**Global plane** (source: both). The combined last-heard (fed by call
events from every system) and the unified event/log window.

## 2. Derivation rule: the daemon emits semantics, never visuals

Adapters and core emit **what happened** (calls, peers, drops, state);
the dashboard backend **derives** every visual from those plus the
snapshot's static config. A TS pill lights because a call event's
system+slot says so (origin side) or because the core's `call_start`
listed that member and config gives the member's slot (egress side); OBP
stream pills are the set of active streams touching that system;
bridge-row talkers come from `call_start`/`call_end`. There are no
bespoke "UI events," and there never will be — new visuals are backend
work against the same vocabulary, which is what keeps C and Python
evolving independently.

## 3. Event flow

```
adapters ──nx_event_emit()──▶ event module ──┬─▶ unified log (text render)
core     ──nx_event_emit()──▶ (in core)      └─▶ Unix event socket (JSON-lines)
                                                         │
                                                  dashboard (Python)
                                                         │
                                                  browser WebSockets
```

- Adapters and core call `nx_event_emit()` directly — a synchronous
  format-and-fan-out into a fixed 2 KiB buffer. Nobody but the event
  module touches the log file or socket.
- The event module assigns each event a global **monotonic sequence
  number** and wall-clock time. On a single thread, emission order *is*
  the total order — logs interleave correctly by construction, and a
  snapshot (§6) is self-consistent for free, since nothing mutates while
  it is being serialized.

## 4. Event schema

One JSON object per event, single line:

```json
{"seq":18234,"ts":1752200000.123456,"sub":"hbp","sys":"KS-DMR-HBP",
 "ev":"call_end","lvl":"info", ...event fields}
```

Envelope fields (always): `seq`, `ts` (wall, float s), `sub` (subsystem:
`core`,`hbp`,`ipsc`,`obp`,`cc`), `sys` (system name or absent),
`ev`, `lvl` (`debug|info|warn|error`).

Vocabulary (phase-1 set; append-only; consumers must ignore unknown `ev`
and unknown fields). Plane: S = systems, B = bridge, G = global.

| ev | plane | fields | source |
|----|-------|--------|--------|
| `startup`, `shutdown` | G | version, config summary | core |
| `system_up`, `system_down` | S | | core (via `nx_core_system_state`) |
| `peer_connected`, `peer_lost` | S | peer id, callsign if known, addr, endpoint class (hotspot/repeater, from HBP login software/package fields — D-03) | adapters |
| `call_start` | B+S+G | bridge (or `local:true`), src, dst(tgid), origin sys/peer/slot, stream key, unit flag, headerless flag, **members forwarded to** (intent) | core; adapters for local repeat |
| `call_end` | B+S+G | + duration_s, frames, reason(term/timeout) | core; adapters for local |
| `stream_contention` | B | bridge, loser src/sys, holder src/sys, `same_src` flag (shared-ID doubles, D-25) | core |
| `loop_suspected` | B | bridge, src, systems involved, capture count/gaps — **alarm-class**: surface prominently (D-25) | core |
| `slot_busy`, `pfmt_unsupported`, `peer_down` | S | system, slot, dropped stream, holder (incl. local holders) — the **outcome** half; join with `call_start` intent (D-15) | adapters |
| `bridge_member_state` | B | bridge, member, enabled, cause (reserved for phase-6 dynamic rules: timeout/trigger/reset) | core |
| `unbridged`, `unit_no_route`, `malformed` | G | rate-limited diagnostics | both |
| `stats` | S+G | per-system counter snapshot, 10 s cadence | adapters + core; event module caches latest per system for snapshot replay |

## 5. Unified text log

The event module renders each event ≥ configured level to one classic
log line — same feel as the hblink/dmrlink logs the operator already
greps:

```
2026-07-11 14:02:11.123 INFO  [hbp/KS-DMR-HBP] call_end src=3120123 tg=3120 dur=8.4s frames=140 reason=term
```

One file, one writer, buffered stdio with per-event flush at `warn`+ and
a 1 s flush timer otherwise. logrotate-friendly (reopen on SIGUSR1).

## 6. Event socket and snapshot

Unix stream socket (path in `[core]` config), JSON-lines. **The daemon
listens; consumers connect** (the C side never blocks or reconnects).
Non-blocking writes; a consumer that stops reading is disconnected after
its kernel buffer fills — a slow dashboard can never back-pressure
routing.

On connect, the daemon sends one `snapshot` event before the live
stream. The snapshot must be **plane-complete**, so a consumer can render
everything without history:

- **Bridge plane:** the full bridge table — every bridge, hang time, and
  per member: system, TGID, slot, enabled (and, phase 6, the dynamic-rule
  parameters); current holders/hang state.
- **Systems plane:** system inventory (name, protocol, mode, up/down),
  connected peers with endpoint class, active streams per system.
- **Global plane:** the last-heard cache and the latest cached `stats`
  per system.

Single-threading makes this trivially consistent: the snapshot is
serialized in one synchronous pass with nothing mutating underneath.

## 7. Dashboard application

Python, `dashboard/`, deliberately boring and separate from the daemon
(D-09: the event contract above is normative; this application is not —
any consumer that speaks the socket is equally legitimate):

- **Backend:** FastAPI + uvicorn (+ `websockets`), zero coupling to the C
  build. One asyncio task holds the Unix-socket connection
  (auto-reconnect, 10 s retry — daemon restarts are invisible beyond a
  gap), maintains the in-memory model **structured as the three planes**,
  rebroadcasts deltas to browser WebSocket clients.
- **Front-end:** systems-plane panels ported from the hblink4/dmrlink3
  dashboards (they are already world-class; this is a re-skin fed by a
  richer, unified feed); the bridge plane is the new work (bridge status
  table + bridge matrix); global plane is the combined last-heard +
  event window.
- **ID databases:** subscriber/peer/talkgroup name resolution happens
  here (JSON files as in dmrlink3/hblink4 dashboards), never in the
  daemon.
- Ping-quality style diagnostics carry over only where the protocol
  makes them meaningful (per the hblink/dmrlink dashboard-parity
  lesson: IPSC pings are firewall timers, not health signals).

## 8. Multi-instance / remote

Phase-1 dashboard serves one daemon. The schema namespaces by
`sub`/`sys` and the backend keys internally on `(daemon, sys)`, so
aggregating several federated OmniLink cores (N event sockets, or a
future TCP transport with HMAC like hblink4's EventEmitter TCP mode) is
a backend-only extension, reserved for phase 6 — and the natural
companion to D-22 federation.
