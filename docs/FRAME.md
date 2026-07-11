# FRAME.md — Canonical Frame Specification (normative)

The canonical frame `nx_frame` is the contract between every adapter and
the core, and (via the PORT adapter) the wire unit between cores. Get
this file right and every adapter becomes independently testable; every
AMBE-garble class of bug becomes localized to exactly one adapter's
conformance vectors.

**The frame speaks DMR (D-24).** OmniLink is a DMR application, not a
digital-mode-agnostic one: IDs are DMR's 24-bit space, burst position is
DMR's A–F superframe, payload kinds are those the ETSI DMR specification
defines. If a foreign-mode adapter (say, Fusion) ever exists, it is that
adapter's job to speak DMR semantics to the core — such adapters are
expected to be rare, infrastructure-to-infrastructure, and never
repeater-facing.

All multi-byte scalars are **little-endian** everywhere (in-process and
PORT wire). Total size is fixed: **64 bytes — one cache line** — a
28-byte header + 36-byte payload area, sized to the largest thing DMR
can hand us (a tagged 33-byte burst, §4.4).

## 1. Layout

```
offset  size  field
------  ----  ------------------------------------------------------
 0       1    ver:4 (hi) | pfmt:4 (lo)         §2.1
 1       1    type                              §2
 2       1    flags                             §3
 3       1    seq            per-stream frame counter, wraps mod 256
 4       4    stream_tag     §5 — unique per (origin_system, stream)
 8       4    src_id         24-bit DMR radio ID; top octet zero
12       4    dst_id         24-bit TGID (group) or radio ID (unit);
                             top octet zero
16       2    origin_system  index into the core's system table
18       1    vseq:3 (hi) | origin_slot:2 | rsvd:3   §1.1
19       1    payload_len    bytes of payload[] that are meaningful
20       4    origin_peer    24-bit radio ID of originating repeater/
                             peer (0 = unknown); top octet zero
24       4    reserved       zero u32 — primary expansion (§6.1)
28      36    payload        §4, zero-padded to 36
------  ----
64 total
```

Access rule (STYLE.md-binding): **never C struct bitfields for wire
data** — layout is implementation-defined. `frame.h` provides explicit
shift/mask accessor macros for the packed fields.

### 1.1 Packed-field semantics

- `ver` (4 bits): header-layout version, NX_VER = 1. Unknown `ver` →
  drop (PORT peers report version in keepalives, so a mismatch is one
  loud event, not silent loss).
- `pfmt` (4 bits): payload format per type, §2.1.
- `vseq` (3 bits): DMR voice-superframe burst position, 0–5 = A–F,
  7 = unknown. Assigned **exactly once, at ingress** — read from wire
  structure where the origin protocol has it (HBP, IPSC, OBP; ingress
  must classify position anyway to harvest embedded LC, so it is free),
  7 where it does not (CC, foreign PORT implementations). Egress
  adapters that emit burst structure follow it faithfully (sync at A,
  EMB + embedded-LC fragment at B–F); on 7 they fall back to a local
  per-stream A–F counter. Position is preserved, not re-counted,
  because that is what RF does: losing bursts C,D shows a receiver
  A,B,gap,E,F — relabeling would be editing the stream (D-16, D-21).
- `origin_slot` (2 bits): 0 = unslotted/unknown origin, 1, 2.
  **Metadata only** — the core never routes on it and never copies it
  to egress; egress slot comes from bridge member config (ROUTING.md
  §5). Its one functional consumer is the phase-5 target route cache.

`src_id`/`dst_id`/`origin_peer` top octets: a DMR ID is 24 bits, period
— no wider form ever appears over the air (the IPv4-like 4-byte ID form
exists only logically inside radios passing data). The fields are 4-byte
for alignment; the top octets are reserved-zero (§6.1 deep reserve), and
config validation rejects IDs above 0xFFFFFF.

`seq` increments per traffic frame within a stream and carries **gap
timing**: the IPSC playout clock schedules burst *n* at
stream-start + n×60 ms so a missing burst leaves its gap at the right
time (D-17).

## 2. Frame types

| value     | name        | payload |
|-----------|-------------|---------|
| 0x01      | CALL_START  | 9-byte LC (§4.2) — only when a real header was received |
| 0x02      | VOICE       | 21-byte AMBE triplet (§4.1) |
| 0x03      | CALL_END    | 9-byte LC + LE u64 frame count (§4.3) — only on a real terminator |
| 0x04      | DATA_BURST  | tag + opaque 33-byte burst (§4.4, phase 6) |
| 0x05      | TALKER_META | reserved: talker alias / GPS (phase 6) |
| 0x06–0x0F | reserved    | in-stream auxiliary traffic (§6 rule 2) |
| ≥ 0x10    | reserved    | unused; there are no control frames — all control/diagnostics ride the event bus (DASHBOARD.md) |

The frame path carries **traffic only**; the event bus carries all
truth — adapters report peer/system state, statistics, and drops as
events (DASHBOARD.md §4), never as frames.

### 2.1 Payload formats (`pfmt`)

`pfmt` qualifies what is inside the payload, per type; `0x0` always
means "this type's default format as specified in §4" and is the only
defined value today. The field exists so payload kinds beyond voice —
the ETSI DMR data payload family (rate ½ / ¾ / rate 1 blocks, data
headers, CSBK) — can later be promoted from the opaque DATA_BURST to
typed payloads through an **unchanged core**: the core never interprets
`payload`/`pfmt`; an egress adapter that cannot encode a given
`(type, pfmt)` drops the frame with a `pfmt_unsupported` event. New
values are allocated in this file, per type, append-only. Per D-24,
payload kinds are DMR's: a foreign-mode adapter translates to these, or
carries its traffic opaquely over PORT between its own kind only.

## 3. Flags

| bit | name          | meaning |
|-----|---------------|---------|
| 0   | NXF_UNIT      | 1 = unit (private) call; 0 = group call |
| 1   | NXF_EMERGENCY | emergency service flag from origin LC/protocol |
| 2   | NXF_FRAG      | reserved for a future chained-fragment scheme (§6 rule 6) |
| 3–7 | spare         | zero |

There is deliberately no "synthesized" flag: nothing in the system
synthesizes frames — headerless streams flow headerless, and timeouts
produce events, not frames (D-16, D-21).

## 4. Payloads

### 4.1 VOICE — 21 bytes

Three 49-bit AMBE+2 vocoder frames, FEC-stripped and de-interleaved
(`dmr_ambe_72_to_49` output), in burst order, each packed MSB-first into
7 bytes (low 7 bits of byte 6 zero). The bit order is *defined as*
whatever `dmr_bits_to_bytes(dmr_ambe_72_to_49(x))` produces — the
existing tested code in `dmr/` is the reference implementation, not
prose. 49-bit is the canonical form because it is the least common
denominator already in service: IPSC natively carries it (D-02).

```
payload[0..6]  frame 1   payload[7..13] frame 2   payload[14..20] frame 3
payload_len = 21
```

### 4.2 CALL_START — 9 bytes

The full 9-byte DMR Link Control (FLCO, FID, service options, dst24,
src24) exactly as received in a real voice LC header. **Exists only when
the origin actually delivered a header.** A stream that begins with
voice (late entry — a normal, designed-for DMR condition) simply begins
with VOICE frames: the core opens streams on any traffic frame, and a
burst-emitting egress adapter builds its minimal LC from the frame's own
src/dst fields, emitting no wire header — late entry in, late entry out.

The dst/src fields *inside* a carried LC are addressing, not structure:
an egress member whose TGID differs patches dst24 to the frame's routed
`dst_id` before BPTC/embedded-LC encoding. Routing rewrites addresses
everywhere they appear; everything else in the LC (FLCO, FID, service
options) is preserved (D-16).

### 4.3 CALL_END — 17 bytes

Same 9-byte LC (the terminator's), then LE u64 count of VOICE frames the
ingress adapter forwarded. **Exists only on a real protocol terminator.**
When a stream just stops, the core's timeout frees state, releases the
bridge into hang time, and emits the `call_end` event — nothing is
synthesized downstream; every far edge (MMDVM, MOTOTRBO, HBP masters)
has native loss-of-signal handling, because loss of signal is the RF
world's steady state (D-21).

### 4.4 DATA_BURST — 34 bytes (phase 6; type reserved now)

Byte 0: burst tag (DMR data sync burst type), bytes 1–33: the raw
264-bit on-air burst. Voice reduces to canonical AMBE; data cannot be
reduced without loss, so it rides opaque and is only deliverable to
burst-native adapters (HBP, IPSC). Not routed in phases 1–5.

## 5. Stream identity

A stream is identified core-wide by `(origin_system, stream_tag)`.

- `stream_tag` is assigned by the ingress adapter: a per-system
  monotonically increasing u32 (wrap is fine; uniqueness must only hold
  across the ~2 s stream-timeout horizon).
- Mapping protocol stream identity (HBP stream_id, IPSC RTP-ish stream,
  CC conduit, OBP stream_id) to `stream_tag` is the ingress adapter's
  private business; the reverse on egress is the egress adapter's.
  Protocol stream IDs never cross the core.
- Stream-state loss is benign by construction: a frame whose key is
  unknown opens a stream, and a stray late packet after timeout *is*
  late entry — the native path. Stream state never needs to be careful,
  only fast.

## 6. Extensibility policy (normative)

1. **The core routes on the 28-byte header only.** It reads `type`,
   `flags`, `seq`, `stream_tag`, `src_id`, `dst_id`, `origin_system`;
   it treats `payload`, `pfmt`, `vseq`, and `origin_slot` as opaque or
   metadata. Any feature that would require the core to parse a payload
   is a design violation — extend an adapter, or claim reserve space
   per §6.1.
2. **New traffic types.** 0x06–0x0F are reserved for in-stream auxiliary
   traffic (TALKER_META at 0x05 is the pattern). Forward-compatibility
   rule, active from phase 1: a traffic frame with an unknown type whose
   key matches an ACTIVE stream is forwarded to the same members as
   VOICE, opaquely; unknown types outside an active stream are dropped
   with a rate-limited event. New in-stream features therefore need zero
   core changes, and old egress adapters ignore what they don't
   implement.
3. **Reserved space is must-be-zero on write, ignored on read.** A new
   field claims reserved space without a `ver` bump only if zero is a
   correct value for old senders; anything else bumps `ver`.
4. **`ver` discipline.** 4 bits covers 15 layout revisions; a reader
   receiving an unknown `ver` drops the frame.
5. **ID space is DMR's 24 bits, fixed (D-24).** No wider ID form exists
   over the air to ever claim the top octets.
6. **Payloads never exceed 36 bytes**; DMR cannot hand us more. A future
   need beyond that defines a chained-fragment scheme under a new type
   using NXF_FRAG — the slot size never grows.
7. **Unit and group are equal citizens from day one.** NXF_UNIT,
   `dst_id`, `origin_peer`, and `origin_slot` ride in every frame so
   unit routing (ROUTING.md §6) needs no frame change when it lands.

### 6.1 Expansion reserves, layered cheapest-first

A future need claims, in order:

1. **~250 unused `type` values and 5 spare `flags` bits** — most "new
   field" needs are actually a new type or flag (rule 2), zero layout
   change.
2. **The reserved u32 at offset 24** — a full aligned word for a genuine
   new header field (e.g. data-session/fragment bookkeeping), claimable
   without a `ver` bump per rule 3.
3. **3 spare bits at offset 18** — small enums.
4. **The three reserved ID top octets** — the deep reserve, 8 bits each
   at the cost of `& 0xFFFFFF` masks on ID reads. Kept unclaimed
   deliberately; overlaying fields into ID words is the last resort.

## 7. Conformance vectors (binding on every adapter)

Lesson bought with the cc2obp garble (D-11): bit-representation
mismatches between protocols are the dominant cross-protocol failure
mode, and they are invisible until a human decodes audio. Every adapter
ships, before it is declared done:

1. **Ingress vectors:** captured real packets for at least one full call
   → expected `nx_frame` sequence, byte-exact, checked by `tests/`.
2. **Egress vectors:** `nx_frame` sequence → expected wire packets,
   byte-exact.
3. **The loopback identity:** for burst-native protocols,
   ingress(egress(f)) reproduces the canonical AMBE bits exactly.

Vector sources: paired live captures, cross-checks against dmr_utils3
for HBP/IPSC framing, and `cccc_ambe.c`'s golden data for CC.
