# OmniLink — Unified DMR Call Router

> **Status: design complete, implementation starting.** The engineering
> design in [`docs/`](docs/) is normative and code is written to it, but
> it is not frozen — see [`docs/DECISIONS.md`](docs/DECISIONS.md) for
> what is settled and why. Implementation proceeds in gated phases
> ([`PLAN.md`](PLAN.md)); nothing ships until it passes live on-air
> verification.

*Written for operators of DMR networks: people who have run c-Bridges,
HBlink or DMRlink instances, or built out their corner of Brandmeister
with clusters and bridges.*

## What it is

OmniLink is a DMR call router: **one daemon that terminates every major
DMR transport and routes calls between them through a single set of
bridge rules, with one dashboard and one log.**

| It speaks | So it connects to |
|---|---|
| HBP (server and client) | MMDVM repeaters and hotspots, HBlink servers |
| OpenBridge | Brandmeister, HBlink3/4, IPSC2, anything OBP |
| XLX | XLX reflector modules |
| IPSC (peer and master) | MOTOTRBO repeater systems |
| CC-CC | c-Bridges |

Today, bridging a MOTOTRBO system to a c-Bridge talkgroup and an MMDVM
network means chaining daemons — a DMRlink here, an HBlink there, a
translator between them — each with its own rules file, its own
dashboard, its own quirks, and a config change in three places every time
a talkgroup moves. OmniLink collapses that chain: every network plugs
into one router, and one bridge entry does what used to take coordinated
edits across several systems.

## It is HBlink3, in C, with the rest of the world plugged in

That is the shortest honest description, and it drives everything else.

**HBlink3's routing semantics are the specification.** Bridges are keyed
on `(system, timeslot, talkgroup)` and one transmission can feed several
bridges, exactly as it does today. Dynamic rules — `ON`/`OFF` triggers,
timeouts, resets — behave identically, down to the asymmetry where
activation fires on call start and deactivation on call end. The four
ACLs keep their grammar, their layering, and their fail-closed behavior.
**Your existing `rules.py` means the same thing after conversion**, and a
migration tool ships with the gate that proves it.

**MMDVM/HBP is the dominant traffic source, so it is the shape of the
core.** Voice rides through as the 33-byte on-air DMR burst — the same
bytes HBP and OpenBridge already put on the wire. HBP→HBP, HBP→OBP, and
HBP→XLX therefore re-encode *nothing*. IPSC and CC-CC translate at their
own edges, where the cost belongs, because they are the legacy inbound
cases and they are shrinking.

**OpenBridge is the trunk.** Instance-to-instance federation is OBP, not
a bespoke protocol — it is the most efficient hop available and both ends
of an OmniLink-to-OmniLink link are ours, so it can be extended where
that helps.

**c-Bridge conduits, XLX modules, and OpenBridge links stop being special
cases.** All three carry their bridge identity in the *connection* rather
than in the frame, and all three now use one member syntax where you
write what is meaningful and omit what is not:

```toml
[[bridge]]
name = "STATEWIDE"
members = [
  { system = "KS-DMR",    slot = 1, tgid = 3120 },  # HBP repeaters
  { system = "MOTO-EAST", slot = 2, tgid = 2 },     # IPSC / MOTOTRBO
  { system = "BM-3102",   tgid = 3120 },            # OpenBridge
  { system = "CC-KSDMR",  tgid = 3120 },            # c-Bridge conduit
  { system = "XLX307-D" },                          # XLX module D
]
```

No second table, no per-protocol grammar, nothing extra to remember. The
protocol of the named system decides what is required, optional, or an
error — and when you get it wrong, the validator tells you which line and
what to do about it.

## Rules you can edit without fear

Writing rules in native Python is unforgiving: one typo and the daemon
does not come up. That is the part being fixed properly.

- **Two files.** `omnilink.toml` holds systems, ports, and credentials
  and needs a restart. `rules.toml` holds bridges and ACLs and **reloads
  live**, on `SIGHUP` or a command.
- **Validate, then swap.** A rules file that fails validation changes
  *nothing* — the daemon keeps routing on the rules it has and tells you
  exactly what is wrong and how to fix it.
- **`omnilink --check-config`** runs that same validator against files on
  disk, so a bad rules file gets caught in your pre-commit hook or deploy
  script instead of on a live network.
- **Calls in flight finish on the rules they started under.** Reloading
  during a conversation does not cut it off.
- A **web rules editor** comes later ([`PLAN.md`](PLAN.md) phase 7), and
  it edits the same file — no database, no second source of truth, so
  your rules stay git-tracked and reviewable.

## How routing behaves

**Bridges work the way you expect.** A bridge is a named conference —
think c-Bridge bridge group, or an hblink3 conference bridge — whose
members are `(system, slot, talkgroup)`. Talkgroup numbers translate
naturally across members. One talker holds a bridge at a time.

**Hang time belongs to a repeater's channel, not to a bridge.** It keeps
a physical timeslot assigned to a talkgroup so another talkgroup cannot
seize it between transmissions, so it is configured per system and
applied per timeslot. A hang timer on a *bridge* would refuse every
contender, because every contender on a bridge is by definition on the
same talkgroup — a round-table net would lose every second station.

**Repeater service is built in.** An OmniLink HBP server repeats traffic
among its own repeaters and hotspots, filtered by per-repeater
subscriptions. Subscriptions *select from what the system already
carries*; they never create bridging. Routing stays in your rules.

**One dashboard, because there is one event stream.** Every adapter and
the router emit into a single ordered feed that drives one unified log
and a live dashboard: per-system views like the HBlink and DMRlink
dashboards you have seen, plus the view none of them could ever show —
the bridge table itself, live, with every bridge, its members, their
TS/TGID mappings, and who is talking.

## What it is not

- **Not a worldwide-network platform.** OmniLink targets the state,
  regional, or national-group level — roughly up to a hundred connected
  systems per instance. Bigger footprints federate multiple instances
  over OpenBridge, which keeps failure domains small. It is a tool for
  building *your* network, not another Brandmeister.
- **Not a multi-mode gateway.** OmniLink is DMR, end to end. No
  transcoding, no other digital modes, anywhere.
- **Not a stream doctor.** OmniLink forwards what it receives: late
  entries stay late entries, dropped bursts stay dropped, nothing is
  concealed or re-timed. DMR radios were engineered for exactly these
  conditions, and a router that "improves" the stream only masks
  problems. The one exception is transmit pacing toward MOTOTRBO
  repeaters, which genuinely need it.

## The shape of the software

One small C binary — single-threaded, no external dependencies, a direct
descendant of the field-proven
[ipsc2hbpc](https://github.com/n0mjs710/ipsc2hbpc) and cc2obp bridges,
sharing their DSP and protocol code. Two TOML files. The dashboard is a
separate Python web application fed by a local socket; the daemon runs
fine without it.

Existing networks do not migrate until they are ready: OmniLink connects
to today's HBlink, DMRlink, c-Bridge, and Brandmeister worlds as a peer
from day one, and each protocol's support is proven against live traffic
— with clean audio, verified byte-exact against captured reference calls
— before it is called done.

## The design documents

For implementers and the deeply curious, in reading order:

1. [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — scope, process model, modules, memory
2. [`docs/FRAME.md`](docs/FRAME.md) — the byte-exact internal frame
3. [`docs/ROUTING.md`](docs/ROUTING.md) — ACLs, bridges, dynamic rules, arbitration, hang, loops
4. [`docs/CONFIG.md`](docs/CONFIG.md) — the two files, validation, live reload, migration
5. [`docs/ADAPTERS.md`](docs/ADAPTERS.md) — per-protocol adapter specifications
6. [`docs/EVENTS.md`](docs/EVENTS.md) — event bus, control socket, logging, dashboard
7. [`docs/STYLE.md`](docs/STYLE.md) — implementation conventions (binding)
8. [`docs/DECISIONS.md`](docs/DECISIONS.md) — design rationale, numbered anchors
9. [`PLAN.md`](PLAN.md) — phased build plan with acceptance gates
10. [`docs/DEVIATIONS.md`](docs/DEVIATIONS.md) — where implementation met the docs and something gave

Questions and, once code exists, bug reports are welcome. The decision
records document why things are the way they are — including the things
that were tried on paper and rejected.

## Heritage

OmniLink is the convergence point of
[hblink3](https://github.com/n0mjs710/hblink3), hblink4,
[dmrlink](https://github.com/n0mjs710/dmrlink),
[ipsc2hbpc](https://github.com/n0mjs710/ipsc2hbpc), and cc2obp. It takes
over the routing-core role that hblink3 and dmrlink3 fill today. It does
**not** replace HBlink4, which remains the device-facing edge access
server and interconnects over OpenBridge (see
[`docs/DECISIONS.md`](docs/DECISIONS.md) D-03).

Copyright (C) 2026 Cortney T. Buffington, N0MJS — GNU GPLv3 (see
[LICENSE](LICENSE)).
