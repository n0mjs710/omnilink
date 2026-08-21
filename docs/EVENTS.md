# EVENTS.md — Event Bus, Control Socket, Logging, Dashboard

The requirement: every protocol module plus the core, logging **in
unison**, feeding **one** dashboard, and accepting operator commands
through **one** channel.

The mechanism: every component emits semantic events; the event module is
the single serialization point; everything human-facing — text log,
dashboard, alerting — is a *rendering* of the one ordered event stream.
The same socket carries commands inbound (D-10).

## 1. Information architecture: three planes

The dashboard is organized around what each part of the daemon actually
knows, which is the D-21 policy/mechanism line made visible.

**Systems plane** (source: adapters — mechanism). Per-system panels, the
direct descendants of the hblink4 and dmrlink3 dashboards: endpoint
tables with class (hotspot/repeater, D-03), connection state, per-slot
hang state, and live activity — TS pills for slotted systems, concurrent
stream pills for OBP. This is most of the dashboard, because most of what
happens, happens in systems.

**Bridge plane** (source: core — policy). The router's one table: each
bridge, its members, and per member the `(system, slot, tgid)` rewrite
parameters — this *is* the rewrite made visible — plus `active` state,
the dynamic-rule columns (`to_type`, `timeout`, remaining time, triggers),
and the current talker. The bridge matrix view (bridges × members, live
talker highlighted) renders this plane, and it is the one visualization
no per-protocol dashboard could ever show.

Hang time is a per-`(system, slot)` property and belongs to the systems
plane, not here (ROUTING §5).

**Global plane** (source: both). The combined last-heard, fed by call
events from every system, and the unified event/log window.

## 2. Derivation rule: the daemon emits semantics, never visuals

Adapters and core emit **what happened** — calls, endpoints, drops,
state. The dashboard backend **derives** every visual from those plus the
snapshot's static config.

A TS pill lights because a call event's system and slot say so (origin
side), or because the core's `call_start` listed that member and config
gives the member's slot (egress side). OBP stream pills are the set of
active streams touching that system. Bridge-row talkers come from
`call_start` and `call_end`.

There are no bespoke "UI events," and there never will be. New visuals
are backend work against the same vocabulary, which is what lets the C
and Python sides evolve independently.

**Events are keyed by `(origin_system, stream_tag)`.** A stream that is
both locally repeated and bridged legitimately produces a `local:true`
event from its adapter *and* one from the core, and the backend must fold
them (D-17). A backend that does not will double-count every locally
repeated call in last-heard.

## 3. Flow

```
adapters ──nx_event_emit()──▶ event module ──┬─▶ unified log (text render)
core     ──nx_event_emit()──▶ (in core)      └─▶ Unix control socket
                                                    │      ▲
                                             JSON-lines    │ commands (§6)
                                                    ▼      │
                                              dashboard (Python)
                                                    │
                                             browser WebSockets
```

- Adapters and core call `nx_event_emit()` directly — a synchronous
  format-and-fan-out into a fixed 2 KiB buffer. **The snapshot (§7) does
  not use this buffer**: a plane-complete snapshot of 100 systems and
  their endpoints is far larger than 2 KiB, so it is serialized
  incrementally straight to the socket from its own writer, never
  formatted whole in memory.
- Nobody but the event module touches the log file or the socket.
- The event module assigns each event a global **monotonic sequence
  number** and wall-clock time. On a single thread, emission order *is*
  the total order — logs interleave correctly by construction, and a
  snapshot is self-consistent for free, since nothing mutates while it is
  being serialized.

## 4. Event schema

One JSON object per event, one line:

```json
{"seq":18234,"ts":1755800000.123456,"sub":"hbp","sys":"KS-DMR",
 "ev":"call_end","lvl":"info", ...}
```

Envelope fields, always present: `seq`; `ts` (wall clock, float seconds);
`sub` (subsystem — `core`, `hbp`, `ipsc`, `obp`, `cc`, `xlx`); `sys`
(system name, or absent); `ev`; `lvl` (`debug|info|warn|error`).

**Append-only. Consumers must ignore unknown `ev` values and unknown
fields.** Plane: S = systems, B = bridge, G = global.

| ev | plane | fields | source |
|---|---|---|---|
| `startup`, `shutdown` | G | version, config summary | core |
| `system_up`, `system_down` | S | | core, via `nx_core_system_state` |
| `repeater_connected`, `repeater_lost` | S | id, callsign if known, address, endpoint class | adapters |
| `call_start` | B+S+G | bridge(s), or `local:true`; src; dst; origin system/peer/slot; stream key; unit and headerless flags; **members forwarded to** (intent) | core; adapters for local repeat |
| `call_end` | B+S+G | + `duration_s`, `frames`, `reason` (`term`\|`timeout`) | core; adapters for local |
| `stream_contention` | B | bridge, loser src/system, holder src/system, `same_src` flag (D-19) | core |
| `loop_suspected` | B | bridge, src, systems involved, capture count and gaps — **alarm class**, surface prominently (D-19) | core |
| `slot_busy`, `pfmt_unsupported`, `peer_down` | S | system, slot, dropped stream, holder (including local holders) — the **outcome** half; join with `call_start` intent (D-17) | adapters |
| `bridge_member_state` | B | bridge, member, `active`, cause (`trigger_on`\|`trigger_off`\|`timeout`\|`operator`\|`reload`) | core |
| `acl_denied` | G | acl that fired, system, src or tgid, slot — rate-limited (ROUTING §2.1) | core; adapters for `reg` |
| `reload_ok`, `reload_failed` | G | generation, findings (all of them, verbatim) | core |
| `xlx_link_sent` | S | module, target tgid, unlink-then-link — **attempt only**; XLX never acknowledges a link, so this must never render as confirmed state (ADAPTERS §4.3) | xlx |
| `unbridged`, `unit_no_route`, `malformed`, `stream_pool_full` | G | rate-limited diagnostics | both |
| `stats` | S+G | per-system counter snapshot, 10 s cadence | adapters + core; the event module caches the latest per system for snapshot replay |

## 5. Unified text log

Each event at or above the configured level renders to one classic log
line — the same feel as the hblink and dmrlink logs the operator already
greps:

```
2026-08-21 14:02:11.123 INFO  [hbp/KS-DMR] call_end src=3120123 tg=3120 dur=8.4s frames=140 reason=term
```

One file, one writer, buffered stdio with per-event flush at `warn` and
above and a 1 s flush timer otherwise. logrotate-friendly: reopen on
`SIGUSR1`.

## 6. The control socket

One Unix stream socket (path in `[core]`), JSON-lines, **bidirectional**
(D-10). The daemon listens; consumers connect. The C side never blocks
and never reconnects.

Writes are non-blocking; a consumer that stops reading is disconnected
once its kernel buffer fills. **A slow dashboard can never back-pressure
routing.**

### 6.1 Commands (inbound)

```json
{"cmd":"reload","id":7}
{"ok":true,"id":7,"generation":4}
```

Every command carries a caller-chosen `id` echoed in the reply. Every
reply carries `ok`, and on failure a `findings` array holding the
validator's messages verbatim — the same text `--check-config` prints,
because there is one validator (CONFIG §6.1).

| cmd | effect |
|---|---|
| `reload` | re-read and validate `rules.toml`; swap only on success (CONFIG §6.3) |
| `check` | validate the file on disk and report; swap nothing, ever |
| `enable` / `disable` | set a member's or a whole bridge's `active` flag (ROUTING §4.5) |
| `snapshot` | re-send the plane-complete snapshot (§7) |
| `stats` | immediate counter dump without waiting for the 10 s tick |

Commands that change state emit their ordinary events as well, so a
second consumer watching the stream sees an operator override exactly as
it sees a trigger firing.

### 6.2 Authority

The socket is filesystem-permissioned and nothing more: it is a local
administrative interface, in the same class as `systemctl`. There is no
in-daemon authentication, no HTTP, and no network listener (D-08). A
remote GUI reaches it the way remote administration always has — over
SSH, or through a local companion process.

## 7. Snapshot

On connect, and on the `snapshot` command, the daemon sends one snapshot
before the live stream. It must be **plane-complete**, so a consumer can
render everything without history:

- **Bridge plane:** the full bridge table — every bridge, and per member
  its system, slot, TGID, `active` state, dynamic-rule parameters, and
  remaining timer; plus current holders and the rules generation.
- **Systems plane:** the system inventory (name, protocol, mode,
  up/down), connected endpoints with class, active streams per system,
  and per-`(system, slot)` hang state.
- **Global plane:** the last-heard cache and the latest cached `stats`
  per system.

Single-threading makes this trivially consistent: it is serialized in one
synchronous pass with nothing mutating underneath.

## 8. Dashboard application

Python, in `dashboard/`, deliberately boring and separate from the daemon
— the event and command contracts above are normative, this application
is not (D-10).

- **Backend:** FastAPI plus uvicorn and `websockets`, with zero coupling
  to the C build. One asyncio task holds the Unix-socket connection
  (auto-reconnect, 10 s retry, so daemon restarts are invisible beyond a
  gap), maintains the in-memory model **structured as the three planes**,
  and rebroadcasts deltas to browser WebSocket clients.
- **Front-end:** systems-plane panels ported from the hblink4 and
  dmrlink3 dashboards — they are already good, and this is a re-skin fed
  by a richer unified feed. The bridge plane is the new work: bridge
  status table plus the bridge matrix.
- **ID databases:** subscriber, peer, and talkgroup name resolution
  happens here, from JSON files as in the existing dashboards. **Never in
  the daemon** (D-10).
- Ping-quality diagnostics carry over only where the protocol makes them
  meaningful. IPSC pings are firewall timers, not health signals, and
  rendering them as health is a known way to mislead an operator.
- **Write to the operator, not to the implementer.** Never render a bare
  "unknown"; drop a field entirely when a system is disconnected rather
  than showing an empty shell, and keep plumbing and internal vocabulary
  off the page.

## 9. Multi-instance and remote

The phase-1 dashboard serves one daemon. The schema namespaces by `sub`
and `sys`, and the backend keys internally on `(daemon, sys)`, so
aggregating several federated cores — *n* event sockets, or a future TCP
transport with HMAC as hblink4's EventEmitter does — is a backend-only
extension. It is the natural companion to D-20 federation and is built
when federation is actually deployed, not before.
