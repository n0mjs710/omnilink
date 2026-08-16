# FRAME.md — Frame Specification (normative)

The frame `nx_frame` is the contract between every adapter and the core.
Get this file right and every adapter becomes independently testable;
every AMBE-garble class of bug becomes localized to exactly one
adapter's conformance vectors.

**The payload is the 33-byte on-air DMR burst (D-02).** The frame is a
carrier for routing metadata the burst has nowhere to put —
`origin_system`, a core-assigned `stream_tag`, flags — not a neutral
re-encoding of the voice. HBP and OBP adapters move the burst through
untouched; IPSC and CC adapters translate at the edge. The frame is
in-process only; federation is OBP (D-07), though the layout remains
trunk-capable should that change.

**The frame speaks DMR (D-24).** OmniLink is a DMR application, not a
digital-mode-agnostic one: IDs are DMR's 24-bit space, burst position is
DMR's A–F superframe, payload kinds are those the ETSI DMR specification
defines. If a foreign-mode adapter (say, Fusion) ever exists, it is that
adapter's job to speak DMR semantics to the core — such adapters are
expected to be rare, infrastructure-to-infrastructure, and never
repeater-facing.

All multi-byte scalars are **little-endian**. Total size is fixed:
**64 bytes — one cache line** — a 28-byte header + 36-byte payload area,
sized to the largest thing DMR can hand us, a tagged 33-byte burst. That
sizing predates the D-02 revision and is why carrying the burst needed
no layout change.

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
  drop. In-process this can only fire during a partial rebuild, but the
  check is kept: it is what makes the layout safe to put on a wire later
  (D-07).
- `pfmt` (4 bits): payload format per type, §2.1.
- `vseq` (3 bits): DMR voice-superframe burst position, 0–5 = A–F,
  7 = unknown. Assigned **exactly once, at ingress** — read from wire
  structure where the origin protocol has it (HBP, IPSC, OBP; ingress
  must classify position anyway to harvest embedded LC, so it is free),
  and 7 only where no position is knowable. Egress adapters follow it
  faithfully (sync at A, EMB + embedded-LC fragment at B–F); on 7 they
  fall back to a local per-stream A–F counter. Position is preserved, not
  re-counted, because that is what RF does: losing bursts C,D shows a
  receiver A,B,gap,E,F — relabeling would be editing the stream (D-16,
  D-21).

  **7 is now a narrow case, and emitting it wrongly is a defect.** Since
  D-02 the payload is a burst, so an adapter that *constructs* one (CC,
  IPSC) has already chosen sync-at-A versus EMB-fragment-at-B–F in order
  to build it at all — the position is known and must be recorded. An
  egress adapter told 7 falls back to its own counter and will splice
  fragment *n* into a burst built as position *m*, which no test that
  stops short of decoding audio will catch.
- `origin_slot` (2 bits): 0 = unslotted/unknown origin, 1, 2.
  **Metadata only** — the core never routes on it and never copies it to
  egress; egress slot reaches the adapter in the `nx_egress_target`
  (ADAPTERS.md §1), from bridge member config or the route cache. Its one
  functional consumer is feeding the phase-5 target route cache.

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
| 0x01      | CALL_START  | header burst, or 9-byte LC (§4.2) per `pfmt` — only when a real header was received |
| 0x02      | VOICE       | 33-byte DMR burst (§4.1) |
| 0x03      | CALL_END    | terminator burst or 9-byte LC, + LE u64 frame count (§4.3) — only on a real terminator |
| 0x04      | DATA_BURST  | tag + opaque 33-byte burst (§4.4, phase 6) |
| 0x05      | TALKER_META | reserved: talker alias / GPS (phase 6) |
| 0x06–0x0F | reserved    | in-stream auxiliary traffic (§6 rule 2) |
| ≥ 0x10    | reserved    | unused; there are no control frames — all control/diagnostics ride the event bus (DASHBOARD.md) |

The frame path carries **traffic only**; the event bus carries all
truth — adapters report peer/system state, statistics, and drops as
events (DASHBOARD.md §4), never as frames.

### 2.1 Payload formats (`pfmt`)

`pfmt` qualifies what is inside the payload, per type; `0x0` always
means "this type's default format as specified in §4".

Defined today:

| type | pfmt | payload |
|---|---|---|
| CALL_START | 0 | 33-byte voice LC header burst (§4.2) |
| CALL_START | 1 | 9-byte decoded LC — for adapters with no burst (CC) |
| VOICE | 0 | 33-byte burst (§4.1) |
| CALL_END | 0 | 33-byte terminator burst + u64 count (§4.3) |
| CALL_END | 1 | 9-byte decoded LC + u64 count |

The field also exists so payload kinds beyond voice —
the ETSI DMR data payload family (rate ½ / ¾ / rate 1 blocks, data
headers, CSBK) — can later be promoted from the opaque DATA_BURST to
typed payloads through an **unchanged core**: the core never interprets
`payload`/`pfmt`; an egress adapter that cannot encode a given
`(type, pfmt)` drops the frame with a `pfmt_unsupported` event. New
values are allocated in this file, per type, append-only. Per D-24,
payload kinds are DMR's: a foreign-mode adapter translates to these, or
carries its traffic opaquely between its own kind only.

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

### 4.1 VOICE — 33 bytes

The on-air DMR burst, verbatim: 264 bits as HBP and OBP carry them,
including FEC, interleave, EMB, sync or embedded-LC fragment, and slot
type. No re-encoding (D-02).

```
payload[0..32]  the 33-byte burst
payload_len = 33
```

For **HBP and OBP** this is a copy — ingress and egress touch the burst
only where routing requires it. For **IPSC and CC**, which carry bare
3×49-bit AMBE, the adapter builds the burst on ingress and reduces it on
egress; `dmr/` implements every primitive (`dmr_ambe_49_to_72` and its
inverse, BPTC, Golay/QR slot type, sync tables).

Two rules for adapters that must *construct* a burst:

1. **Header and payload must agree.** The burst's LC carries src and dst
   redundantly with the frame header. They are patched together or not
   at all. A burst whose LC contradicts its header is the single most
   expensive class of bug this project can ship — it survives every test
   that does not decode audio, and it has been shipped before in this
   ecosystem.
2. **Invented fields come from real configuration.** Colour code, sync
   flavour, and EMB are not present in IPSC or CC traffic and must be
   synthesized. They come from the system's configured values. Never
   from a captured constant.

   Note `dmr/` cannot do this as vendored: `DMR_SLOT_TYPE_VHEAD`/`VTERM`
   in `dmr_const.c` are precomputed for **colour code 1**, and `golay.c`
   carries `golay2312` (for AMBE) but no Golay(20,8) slot-type encoder.
   Building a burst at a configured colour code needs one — about twenty
   lines, and hblink3 had to add exactly this (`xlx_slot_type()`) for the
   XLX link. Add it to `dmr/` deliberately and upstream before phase 2;
   CLAUDE.md declares `dmr/` read-only vendored, so an implementer who
   discovers the gap mid-adapter will stop rather than patch it.

### 4.1.1 Retargeting

When a stream is delivered to a member whose TGID differs from the
frame's, the burst's LC and embedded-LC fragments carry the old dst and
must be rewritten. Generate the replacement LCs **once per stream per
target**, then splice per burst by position — do not rebuild per packet.
`hblink3`'s `gen_lcs()` / `embed_lc()` pair is the reference
implementation of exactly this and is known-correct in production.

### 4.2 CALL_START — 33 bytes (`pfmt 0`) or 9 bytes (`pfmt 1`)

**`pfmt 0` — the voice LC header burst, verbatim, 33 bytes.** Emitted by
every adapter whose wire carries a burst (HBP, OBP, IPSC). This is the
form that keeps D-02's promise: an HBP header crosses the core untouched
like every other burst, with no BPTC decode on ingress and no re-encode
on egress. It also preserves the origin's colour code and sync flavour,
which a rebuilt header would silently replace with ours.

**`pfmt 1` — the decoded 9-byte LC** (FLCO, FID, service options, dst24,
src24). For adapters whose wire has no burst to carry: CC-CC, which
signals call start with a B-on message, and any future protocol in the
same position. An egress adapter that needs a burst builds one per §4.1;
an egress adapter that needs fields reads them from the frame header,
which carries src and dst regardless of `pfmt`.

The core does not care which: it never parses payloads (§6 rule 1) and
reads src/dst from the header. `pfmt` exists exactly so this distinction
costs the core nothing.

**Exists only when the origin actually delivered a header.** A stream that begins with
voice (late entry — a normal, designed-for DMR condition) simply begins
with VOICE frames: the core opens streams on any traffic frame, and a
burst-emitting egress adapter builds its minimal LC from the frame's own
src/dst fields, emitting no wire header — late entry in, late entry out.

The dst/src fields *inside* a carried LC are addressing, not structure:
an egress member whose TGID differs patches dst24 to the frame's routed
`dst_id` before BPTC/embedded-LC encoding. Routing rewrites addresses
everywhere they appear; everything else in the LC (FLCO, FID, service
options) is preserved (D-16).

### 4.3 CALL_END — 41 bytes (`pfmt 0`) or 17 bytes (`pfmt 1`)

The terminator in the same two forms as §4.2 — the 33-byte terminator
burst verbatim, or the decoded 9-byte LC — then LE u64 count of VOICE
frames the ingress adapter forwarded. **Exists only on a real protocol terminator.**
When a stream just stops, the core's timeout frees state, releases the
bridge into hang time, and emits the `call_end` event — nothing is
synthesized downstream; every far edge (MMDVM, MOTOTRBO, HBP masters)
has native loss-of-signal handling, because loss of signal is the RF
world's steady state (D-21).

### 4.4 DATA_BURST — 34 bytes (phase 6; type reserved now)

Byte 0: burst tag (DMR data sync burst type), bytes 1–33: the raw
264-bit on-air burst.

Since the D-02 revision this differs from VOICE only by the leading tag
— both carry a burst. The types stay distinct because their *routability*
differs, not their encoding: voice is deliverable everywhere (adapters
that cannot carry bursts natively reduce it), while data cannot be
reduced without loss and is therefore deliverable only to burst-native
adapters (HBP, OBP, IPSC). Not routed in phases 1–5.

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
   ingress(egress(f)) reproduces the burst bits exactly — including
   CALL_START and CALL_END, which is why those carry the burst (§4.2).
   For IPSC and CC, whose native form is 3x49-bit AMBE, the identity is
   on the AMBE bits: synthesized burst fields are not round-tripped and
   must not be asserted on.

Vector sources: paired live captures, cross-checks against dmr_utils3
for HBP/IPSC framing, and `cccc_ambe.c`'s golden data for CC.
