# OmniLink — Unified DMR Call Router

> ## ⚠️ THERE IS NO CODE HERE YET
>
> **This is a project at its starting line.** What you are looking at is
> the complete, frozen engineering design — published first, on purpose,
> so that every line of code is tracked here from its first commit.
> Implementation proceeds in gated phases (see [`PLAN.md`](PLAN.md));
> code will land as it is written and nothing ships until it passes live
> on-air verification. The design documents in [`docs/`](docs/) are
> normative: the code is written *to them*.

*This README is written for operators of DMR networks: people who have
run c-Bridges, HBlink or DMRlink instances, or built out their corner of
Brandmeister with clusters and bridges.*

## What it is

OmniLink is a DMR call router: **one daemon that terminates every major
DMR transport and routes calls between them through a single set of
bridge rules, with one dashboard and one log.**

| It speaks | So it connects to |
|-----------|-------------------|
| HBP (master and peer) | MMDVM repeaters and hotspots, HBlink masters |
| IPSC (peer and master) | MOTOTRBO repeater systems |
| OpenBridge | Brandmeister, HBlink3/4, IPSC2, anything OBP |
| CC-CC | c-Bridges |

Today, bridging a MOTOTRBO system to a c-Bridge talkgroup and an MMDVM
network means chaining daemons — a DMRlink here, an HBlink there, a
translator between them — each with its own rules file, its own
dashboard, its own quirks, and a config change in three places every
time a talkgroup moves. OmniLink collapses that chain: every network
plugs into one router, and one bridge entry does what used to take
coordinated edits across several systems.

## How it works

**Voice moves as the DMR burst itself.** Internally a call is carried as
the same 33-byte on-air burst that HBP and OpenBridge already put on the
wire, plus routing metadata — source ID, talkgroup or destination ID,
originating system and repeater. HBP and OpenBridge traffic therefore
passes through without being taken apart and rebuilt; the two legacy
protocols that carry bare vocoder frames, IPSC and CC-CC, translate at
their own edge. All protocol knowledge lives at the edges; the router in
the middle is protocol-blind.

The alternative — inventing a neutral internal format everything
converts into — was the original design and was dropped before any code
was written. It made the common path (MMDVM-to-MMDVM, which is most
traffic) pay a full teardown and rebuild in order to make the rarest
path free. The reasoning is recorded in [D-02](docs/DECISIONS.md).

**Timeslot is not a routing concept.** If you have fought TS1/TS2
mapping across c-Bridge conduits or HBP bridges, this is the part to
notice: inside OmniLink, a stream is identified by talkgroup alone.
Timeslot is just a per-member delivery parameter — "on this IPSC
system, this bridge appears on TS2" — configured where it belongs and
invisible everywhere else. Slot contention is handled at the edge it
belongs to; trunked transports (OpenBridge) carry unlimited
concurrent talkgroups with no slot mapping at all.

**Bridges work the way you expect.** A bridge is a named conference —
think c-Bridge bridge group, or an hblink3 conference bridge — whose
members are (system, talkgroup, slot-where-applicable). Talkgroup
numbers translate naturally across members. One talker holds a bridge at
a time, with hang time so a conversation isn't trampled between
transmissions. Rules are static and operator-owned; dynamic on/off
timers and triggers in the hblink3 style are on the roadmap on the same
foundation.

**Repeater service is built in, HBlink4-style.** An OmniLink HBP master
repeats traffic among its own repeaters and hotspots, filtered by
per-repeater subscriptions — eventually supplied by the repeater itself
via its login options, as HBlink4 users know. Subscriptions *select from
what the system carries*; they never create bridging — routing stays in
the operator's rules. Unit-to-unit calls route on a last-heard cache of
where each radio was last active, network-wide.

**One dashboard, because there is one event stream.** Every adapter and
the router emit into a single ordered event feed, which drives one
unified log and a live dashboard: per-system views (repeater tables,
timeslot activity, OpenBridge stream activity) like the HBlink/DMRlink
dashboards you've seen, plus the view none of them could ever show — the
bridge table itself, live: every bridge, its members, their TS/TGID
mappings, and who is talking, all in one place.

## What it is not

- **Not a worldwide-network platform.** OmniLink is built for the
  state / regional / national-group level — roughly up to a hundred
  connected systems per instance. Bigger footprints federate multiple
  OmniLink instances over OpenBridge, which also keeps failure
  domains small. It is a tool for building *your* network, not another
  Brandmeister.
- **Not a multi-mode gateway.** OmniLink is DMR, end to end. No
  transcoding, no other digital modes in the core.
- **Not a stream doctor.** OmniLink forwards what it receives: late
  entries stay late entries, dropped bursts stay dropped, nothing is
  concealed or re-timed. DMR radios were engineered for exactly these
  conditions; a router that "improves" the stream only masks problems.
  (The one exception: transmit pacing toward MOTOTRBO repeaters, which
  genuinely need it.)

## The shape of the software

One small C binary — single-threaded, no external dependencies, a
direct descendant of the field-proven
[ipsc2hbpc](https://github.com/n0mjs710/ipsc2hbpc) and cc2obp bridges,
sharing their DSP/FEC code. One TOML config file: your systems, your
bridges. The dashboard is a separate Python web application fed by a
local socket; the daemon runs fine without it. Existing networks don't
migrate until they're ready: OmniLink connects to today's HBlink,
DMRlink, c-Bridge, and Brandmeister worlds as a peer from day one, and
each protocol's support is proven against live traffic — with clean
audio, verified byte-exact against captured reference calls — before it
is called done.

## The design documents

For implementers and the deeply curious, in reading order:

1. [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — scope, single-thread model, memory discipline
2. [`docs/FRAME.md`](docs/FRAME.md) — the byte-exact frame (64 bytes)
3. [`docs/ROUTING.md`](docs/ROUTING.md) — bridges, stream arbitration, hang time, loops, slots
4. [`docs/ADAPTERS.md`](docs/ADAPTERS.md) — per-protocol adapter specifications
5. [`docs/DASHBOARD.md`](docs/DASHBOARD.md) — unified event bus, logging, dashboard
6. [`docs/STYLE.md`](docs/STYLE.md) — implementation conventions (binding)
7. [`docs/DECISIONS.md`](docs/DECISIONS.md) — design rationale (settled; numbered anchors)
8. [`PLAN.md`](PLAN.md) — phased build plan with acceptance gates

The design is settled. Questions and (once code exists) bug reports are
welcome; the decision records in `docs/DECISIONS.md` document why things
are the way they are.

## Heritage

OmniLink is the convergence point of
[hblink3](https://github.com/n0mjs710/hblink3), hblink4,
[dmrlink](https://github.com/n0mjs710/dmrlink), ipsc2hbp(c), and cc2obp.
It fills hblink3's and dmrlink3's routing role (the conference-bridge
core). It does **not** replace HBlink4, which remains the device-facing
edge access server and interconnects over OpenBridge (see
`docs/DECISIONS.md` D-03).

Copyright (C) 2026 Cortney T. Buffington, N0MJS — GNU GPLv3 (see
[LICENSE](LICENSE)).
