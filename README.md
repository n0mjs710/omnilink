# OmniLink — Unified DMR Call Router

> **Status: design complete, implementation starting.** The engineering
> design in [`docs/`](docs/) is normative and code is written to it, but
> it is not frozen — see [`docs/DECISIONS.md`](docs/DECISIONS.md) for
> what is settled and why. Implementation proceeds in gated phases
> ([`PLAN.md`](PLAN.md)).

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

That is the short description.

**HBlink3's routing semantics are the specification.** Bridges are keyed
on `(system, timeslot, talkgroup)` and one transmission can feed several
bridges, exactly as it does today. Dynamic rules — `ON`/`OFF` triggers,
timeouts, resets — behave identically, down to the asymmetry where
activation fires on call start and deactivation on call end. The four
ACLs keep their grammar, their layering, and their fail-closed behavior.
**Your existing `rules.py` means the same thing after conversion**, and a
migration tool ships with the gate that proves it.

**MMDVM/HBP is the dominant traffic source, so it is the shape of the
core.** What moves between adapters is the DMRD packet itself, so
HBP→HBP, HBP→OBP and HBP→XLX re-encode *nothing* — there is no internal
format to convert to. IPSC carries the same DMR material unpacked and
re-packs it at its own edge. CC-CC — the only transport carrying bare
AMBE with no burst or superframe data — is the one place anything is
synthesized, and it is the shrinking case.

**OpenBridge is the trunk.** Instance-to-instance federation is OBP, not
a bespoke protocol — it is the most efficient hop available and both ends
of an OmniLink-to-OmniLink link are ours, so it can be extended where
that helps.

**c-Bridge conduits, XLX modules, and OpenBridge links stop being special
cases.** All three carry their bridge identity in the *connection* rather
than in the packet, and all three now use one member syntax where you
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

**Triggers and timers are per-member fields on the same line**, keeping
their `rules.py` names and their units. Omit them and the member is
simply always up, which is why the bridge above has none:

```toml
[[bridge]]
name = "TAC-1"
members = [
  # Key up TG 8951 to connect this bridge; it drops after 10 idle
  # minutes, or immediately if someone keys 4000.
  { system = "KS-DMR", slot = 2, tgid = 8951, active = false,
    to_type = "ON", timeout = 10, on = [8951], off = [4000], reset = [] },

  { system = "KS-WEST", slot = 2, tgid = 8951, active = false,
    to_type = "ON", timeout = 10, on = [8951], off = [4000] },
]
```

`active`, `to_type`, `timeout`, `on`, `off`, and `reset` are hblink3's
fields, unchanged — including `timeout` being in **minutes**, because
that is what your existing rules mean and silently reinterpreting it as
seconds would be the worst kind of migration bug. Full semantics in
[`docs/CONFIG.md`](docs/CONFIG.md) §4 and
[`docs/ROUTING.md`](docs/ROUTING.md) §4.

## Rules files without python syntax

Writing rules in native Python can be unforgiving: one typo or  and the 
errant space and the daemon does not come up.

- **Two files.** `omnilink.toml` holds systems, ports, and credentials
  and needs a restart. `rules.toml` holds bridges and ACLs and **reloads
  live**, on `SIGHUP` or a command.
- **Validate, then swap.** A rules file that fails validation changes
  *nothing* — the daemon keeps routing on the rules it has and tells you
  what is wrong and how to fix it.
- **`omnilink --check-config`** runs that same validator against files on
  disk, so a bad rules file gets caught you restart your live network.
- **Calls in flight finish on the rules they started under.** Reloading
  during a conversation does not cut it off.
- A **web rules editor** comes later ([`PLAN.md`](PLAN.md) phase 7), and
  it edits the same file — no database, no second source of truth, so
  your rules stay git-tracked and reviewable.

## How routing behaves

**Bridges work the way you expect.** A bridge is a named conference —
no change from hblink3 or dmrlink3 — whose members are `(system, slot, 
talkgroup)`. Talkgroup numbers translate naturally across members. 
One talker holds a bridge at a time.

**Hang time belongs to a repeater's channel, not to a bridge.** It keeps
a physical timeslot assigned to a talkgroup so another talkgroup cannot
seize it between transmissions, so it is configured per system and
applied per timeslot.

**Repeater service is built in.** An OmniLink HBP system repeats traffic
among its own repeaters and hotspots, filtered by per-repeater
subscriptions. Subscriptions *select from what the system already
carries*; they never create bridging rules. This adds the best of
HBlink4 with the structured transit routing capabilities of HBlink3.

**One dashboard, because there is one event stream.** Every adapter and
the router emit into a single ordered feed that drives one unified log
and a live dashboard: per-system views like the HBlink and DMRlink
dashboards you have seen.

## What it is not

- **Not a worldwide-network platform.** OmniLink targets the state,
  regional, or national-group level — roughly up to a hundred connected
  devices per instance. Bigger footprints federate multiple instances
  over OpenBridge, which keeps failure domains small. It is a tool for
  building a club, statewide, regional or smaller national, network. It
  is not for building another Brandmeister.
- **Not a multi-mode gateway.** OmniLink is DMR, end to end. No
  transcoding, no other digital modes, it is not DV switch.
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

## The design documents

In reading order:

1. [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — scope, process model, modules, memory
2. [`docs/FRAME.md`](docs/FRAME.md) — the DMRD packet, stream identity, the burst
3. [`docs/ROUTING.md`](docs/ROUTING.md) — ACLs, bridges, dynamic rules, arbitration, hang, loops
4. [`docs/CONFIG.md`](docs/CONFIG.md) — the two files, validation, live reload, migration
5. [`docs/ADAPTERS.md`](docs/ADAPTERS.md) — per-protocol adapter specifications
6. [`docs/EVENTS.md`](docs/EVENTS.md) — event bus, control socket, logging, dashboard
7. [`docs/STYLE.md`](docs/STYLE.md) — implementation conventions (binding)
8. [`docs/DECISIONS.md`](docs/DECISIONS.md) — design rationale, numbered anchors
9. [`PLAN.md`](PLAN.md) — phased build plan with acceptance gates
10. [`docs/DEVIATIONS.md`](docs/DEVIATIONS.md) — where implementation met the docs and something gave

Questions and, once code exists, (genuine) bug reports are welcome. The decision records document 
why things are the way they are — including the things that were tried on paper and rejected. 
Configuration and operational assistance are not provided.

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
