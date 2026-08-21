# FRAME.md — The Internal Frame (normative, byte-exact)

`nx_frame` is the only currency crossing the adapter/core boundary. It is
**64 bytes — one cache line**: a 28-byte header the core routes on, and a
36-byte payload the core never parses.

The payload carries the **33-byte on-air DMR burst, verbatim** (D-05).
HBP and OpenBridge therefore pass through with no re-encoding at all;
IPSC and CC-CC build the burst on ingress and reduce it on egress, at
their own edges.

**The frame is not the DMRD packet**, and that distinction is
load-bearing (D-05). DMRD has nowhere to put which configured system a
frame arrived on, the core's own stream tag, headerless/late-entry
status, or call-type flags that outlive one protocol's encoding — and
using it would force IPSC and CC to fabricate a repeater ID and a stream
ID just to say anything at all. So: explicit metadata, wrapping an
untouched burst.

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
20       4    origin_endpoint 24-bit radio ID of the originating
                             repeater, hotspot, or peer (0 = unknown);
                             top octet zero
24       4    reserved       zero u32 — primary expansion (§6.1)
28      36    payload        §4, zero-padded to 36
------  ----
64 total
```

Access rule (STYLE.md-binding): **never C struct bitfields for wire
data** — their layout is implementation-defined. `frame.h` provides
explicit shift/mask accessor macros for every packed field.

### 1.1 Packed-field semantics

- **`ver`** (4 bits): header-layout version, `NX_VER = 1`. Unknown `ver`
  → drop. In-process this can only fire during a partial rebuild, but
  the check is kept: it is what would make the layout safe to put on a
  wire later.

- **`pfmt`** (4 bits): payload format, per type — §2.1.

- **`vseq`** (3 bits): DMR voice-superframe burst position, 0–5 = A–F,
  7 = unknown. Assigned **exactly once, at ingress** — read from wire
  structure where the origin protocol has it (HBP, IPSC, OBP; ingress
  must classify position anyway to harvest embedded LC, so it is free) —
  and 7 only where no position is knowable. Egress adapters follow it
  faithfully (sync at A, EMB plus embedded-LC fragment at B–F); on 7 they
  fall back to a local per-stream A–F counter.

  Position is **preserved, not re-counted**, because that is what RF
  does: losing bursts C and D shows a receiver A, B, gap, E, F.
  Relabeling would be editing the stream (D-13, D-14).

  **7 is a narrow case, and emitting it wrongly is a defect.** An adapter
  that *constructs* a burst (CC, IPSC) has already chosen sync-at-A
  versus EMB-fragment-at-B–F in order to build it at all, so the position
  is known and must be recorded. An egress adapter told 7 will splice
  fragment *n* into a burst built as position *m* — which no test that
  stops short of decoding audio will catch.

- **`origin_slot`** (2 bits): 1 or 2. **Routing-significant** — it is one
  third of the bridge lookup key `(system, slot, tgid)` (D-04,
  ROUTING §2.2).

  Every ingress adapter must stamp it, including unslotted ones. For
  transports with no slot on the wire, the adapter stamps the **nominal**
  slot that D-07 assigns: OBP stamps the wire slot (1 by convention, or
  the real slot when `both_slots` is set), XLX stamps 2, CC stamps its
  injected nominal. The rules loader indexes bound-endpoint members under
  that same nominal value, so bridge lookup is uniform and has no
  per-protocol special case at all.

  0 is reserved for "unknown" and a traffic frame carrying it is a
  defect: it cannot be routed, and it is dropped with an event naming the
  adapter. This is worth an assertion in debug builds — a silently
  unroutable frame from one adapter looks exactly like a config mistake
  from the operator's side.

  `origin_slot` is the *origin's* slot. It is never copied to egress:
  the egress slot reaches the adapter in the `nx_egress_target`
  (ADAPTERS.md §1), from bridge member config or from the unit route
  cache.

`src_id` / `dst_id` / `origin_endpoint` top octets: a DMR ID is 24 bits,
period — no wider form ever appears over the air (the IPv4-like 4-byte ID
form exists only logically inside radios passing data). The fields are
4-byte for alignment; the top octets are reserved-zero (§6.1), and config
validation rejects IDs above `0xFFFFFF`.

`seq` increments per traffic frame within a stream and carries **gap
timing**: the IPSC playout clock schedules burst *n* at
stream-start + n × 60 ms, so a missing burst leaves its gap at the right
instant (D-15).

## 2. Frame types

| value | name | payload |
|---|---|---|
| 0x01 | CALL_START | header burst, or 9-byte LC (§4.2) per `pfmt` — only when a real header was received |
| 0x02 | VOICE | 33-byte DMR burst (§4.1) |
| 0x03 | CALL_END | terminator burst or 9-byte LC, plus LE u64 frame count (§4.3) — only on a real terminator |
| 0x04 | DATA_BURST | tag + opaque 33-byte burst (§4.4, D-27) |
| 0x05 | TALKER_META | reserved: talker alias / GPS |
| 0x06–0x0F | reserved | in-stream auxiliary traffic (§6 rule 2) |
| ≥ 0x10 | reserved | unused — there are no control frames |

**The frame path carries traffic only.** All control and diagnostics ride
the event bus (EVENTS.md): adapters report system state, statistics, and
drops as events, never as frames.

### 2.1 Payload formats (`pfmt`)

`pfmt` qualifies what is inside the payload, per type; `0x0` always means
"this type's default format as specified in §4."

| type | pfmt | payload |
|---|---|---|
| CALL_START | 0 | 33-byte voice LC header burst (§4.2) |
| CALL_START | 1 | 9-byte decoded LC — for adapters with no burst (CC) |
| VOICE | 0 | 33-byte burst (§4.1) |
| CALL_END | 0 | 33-byte terminator burst + u64 count (§4.3) |
| CALL_END | 1 | 9-byte decoded LC + u64 count |

The field also exists so payload kinds beyond voice — the ETSI DMR data
family (rate ½ / ¾ / rate 1 blocks, data headers, CSBK) — can later be
promoted from the opaque `DATA_BURST` to typed payloads through an
**unchanged core**: the core never interprets `payload` or `pfmt`, and an
egress adapter that cannot encode a given `(type, pfmt)` drops the frame
with a `pfmt_unsupported` event. New values are allocated in this file,
per type, append-only. Per D-02, payload kinds are DMR's.

## 3. Flags

| bit | name | meaning |
|---|---|---|
| 0 | `NXF_UNIT` | 1 = unit (private) call; 0 = group call |
| 1 | `NXF_EMERGENCY` | emergency service flag from origin LC/protocol |
| 2 | `NXF_FRAG` | reserved for a future chained-fragment scheme (§6.1) |
| 3–7 | spare | zero |

There is deliberately **no "synthesized" flag**, because nothing in the
system synthesizes frames: headerless streams flow headerless, and
timeouts produce events, not frames (D-14).

## 4. Payloads

### 4.1 VOICE — 33 bytes

The on-air DMR burst, verbatim: 264 bits as HBP and OBP carry them,
including FEC, interleave, EMB, sync or embedded-LC fragment, and slot
type.

```
payload[0..32]   the 33-byte burst
payload_len = 33
```

For **HBP and OBP** this is a copy — ingress and egress touch the burst
only where routing requires it (§4.1.1). For **IPSC and CC**, which carry
AMBE plus unpacked DMR elements rather than an assembled burst, the
adapter re-packs on ingress and unpacks on egress; `dmr/` implements
every primitive (`dmr_ambe_49_to_72` and
its inverse, BPTC, embedded LC, RS(12,9), and the precomputed slot-type,
EMB, and sync tables).

Two rules bind any adapter that must **construct** a burst:

1. **Header and payload must agree.** The burst's LC carries source and
   destination redundantly with the frame header. They are patched
   together or not at all. A burst whose LC contradicts its header is the
   single most expensive class of bug this project can ship: it survives
   every test that does not decode audio, and it has been shipped before
   in this ecosystem. See ROUTING §7 for the standing rule.

2. **The LC must be built from real values, never from a captured
   constant.** FLCO, FID, service options, source, and destination come
   from the stream being carried. Lifting a known-good burst blob from
   prior art and patching a couple of fields is how contradictory bursts
   get shipped.

   Structural fields that are genuinely absent from the origin — sync
   flavour, EMB, slot type — come from `dmr/`'s tables at colour code 1,
   per §4.1.2. They are not configuration.

#### 4.1.1 Retargeting

When a stream is delivered to a member whose TGID differs from the
frame's, the burst's LC and its embedded-LC fragments carry the old
destination and must be rewritten.

Generate the replacement LCs **once per stream per target**, then splice
per burst by position. Do not rebuild per packet. hblink3's `gen_lcs()` /
`embed_lc()` pair is the reference implementation and is known-correct in
production.

#### 4.1.2 Colour code is RF-local

Colour code is an RF air-interface interference-mitigation mechanism. It
has no meaning between repeaters; each applies its own before
transmitting. Three rules, deliberately not one rule (D-28):

- **Never read it.** Not a routing key, not an ACL input, not a match
  condition, not a filter.
- **Never rewrite it.** A passed-through burst keeps whatever the origin
  stamped, so a stream may carry a foreign colour code end to end.
  Rewriting would re-encode the slot type and both EMB halves on every
  burst on the dominant path, which is what D-05 exists to avoid.
- **Use 1 when you must invent one.** IPSC and CC-CC ingress and the XLX
  module link build from `dmr/`'s precomputed CC-1 tables.

Retargeting never touches it: the colour code lives in the slot type and
in EMB, and §4.1.1 rewrites neither. `dmr/` is therefore complete as
vendored — no Golay(20,8) slot-type encoder is required.

### 4.2 CALL_START — 33 bytes (`pfmt 0`) or 9 bytes (`pfmt 1`)

**`pfmt 0` — the voice LC header burst, verbatim.** Emitted by every
adapter whose wire carries a burst (HBP, OBP, IPSC). This is the form
that keeps D-05's promise: an HBP header crosses the core untouched like
every other burst, with no BPTC decode on ingress and no re-encode on
egress. The origin's colour code and sync flavour ride along untouched,
which is the correct handling for both (§4.1.2) and costs nothing.

**`pfmt 1` — the decoded 9-byte LC** (FLCO, FID, service options, dst24,
src24). For adapters whose wire has no burst to carry: CC-CC, which
signals call start with a B-on message. An egress adapter needing a burst
builds one per §4.1; one needing fields reads them from the frame header,
which carries source and destination regardless of `pfmt`.

The core does not care which. It never parses payloads (§6 rule 1) and
reads src/dst from the header — `pfmt` exists exactly so this distinction
costs the core nothing.

**CALL_START exists only when the origin actually delivered a header.** A
stream beginning with voice — late entry, a normal and designed-for DMR
condition — simply begins with VOICE frames. The core opens streams on
any traffic frame, and a burst-emitting egress adapter builds its minimal
LC from the frame's own src/dst, emitting no wire header. Late entry in,
late entry out (D-14).

The dst/src fields *inside* a carried LC are addressing, not structure:
an egress member whose TGID differs patches dst24 to the frame's routed
`dst_id` before BPTC and embedded-LC encoding. Routing rewrites addresses
everywhere they appear; everything else in the LC — FLCO, FID, service
options — is preserved.

### 4.3 CALL_END — 41 bytes (`pfmt 0`) or 17 bytes (`pfmt 1`)

The terminator in the same two forms as §4.2, followed by an LE u64 count
of VOICE frames the ingress adapter forwarded.

**Exists only on a real protocol terminator.** When a stream just stops,
the core's timeout frees state, releases arbitration, and emits the
`call_end` event — nothing is synthesized downstream. Every far edge
(MMDVM, MOTOTRBO, HBP servers) has native loss-of-signal handling,
because loss of signal is the RF world's steady state (D-13, D-14). The
one carve-out is what an IPSC *egress* puts on its own wire, which is
protocol machinery, not a frame (D-14).

### 4.4 DATA_BURST — 34 bytes (D-27)

Byte 0: burst tag (DMR data sync burst type). Bytes 1–33: the raw 264-bit
on-air burst.

This differs from VOICE only by the leading tag — both carry a burst. The
types stay distinct because their **routability** differs, not their
encoding: voice is deliverable everywhere (adapters that cannot carry
bursts natively reduce it), while data cannot be reduced without loss and
is therefore deliverable only to burst-native adapters (HBP, OBP, IPSC).
Not routed until after the parity gate.

## 5. Stream identity

A stream is identified core-wide by `(origin_system, stream_tag)`.

- `stream_tag` is assigned by the ingress adapter: a per-system
  monotonically increasing u32. Wrap is fine — uniqueness need only hold
  across the ~2 s stream-timeout horizon.
- Mapping protocol stream identity (HBP `stream_id`, IPSC stream, CC
  conduit, OBP `stream_id`) to `stream_tag` is the ingress adapter's
  private business, and the reverse on egress is the egress adapter's.
  **Protocol stream IDs never cross the core.**
- Stream-state loss is benign by construction: a frame whose key is
  unknown opens a stream, and a stray late packet after timeout *is* late
  entry — the native path. Stream state never needs to be careful, only
  fast.

## 6. Extensibility policy (normative, D-25)

1. **The core routes on the header only.** It reads `type`, `flags`,
   `seq`, `stream_tag`, `src_id`, `dst_id`, `origin_system`, and
   `origin_slot`; it treats `payload`, `pfmt`, and `vseq` as opaque. Any
   feature requiring the core to parse a payload is a design violation —
   extend an adapter, or claim reserve space per §6.1.
2. **New traffic types.** 0x06–0x0F are reserved for in-stream auxiliary
   traffic (TALKER_META at 0x05 is the pattern). Forward-compatibility
   rule, active from phase 1: a traffic frame with an unknown type whose
   key matches an ACTIVE stream is forwarded to the same members as
   VOICE, opaquely; unknown types outside an active stream are dropped
   with a rate-limited event. New in-stream features therefore need zero
   core changes, and old egress adapters ignore what they do not
   implement.
3. **Reserved space is must-be-zero on write, ignored on read.** A new
   field claims reserved space without a `ver` bump only if zero is a
   correct value for old senders; anything else bumps `ver`.
4. **`ver` discipline.** 4 bits covers 15 layout revisions; a reader
   receiving an unknown `ver` drops the frame.
5. **ID space is DMR's 24 bits, fixed** (D-02). No wider ID form exists
   over the air to ever claim the top octets.
6. **Payloads never exceed 36 bytes.** DMR cannot hand us more. A future
   need beyond that defines a chained-fragment scheme under a new type
   using `NXF_FRAG` — the slot size never grows.
7. **Unit and group are equal citizens from day one.** `NXF_UNIT`,
   `dst_id`, `origin_endpoint`, and `origin_slot` ride in every frame, so
   unit routing (ROUTING §6) needs no frame change when it lands.

### 6.1 Expansion reserves, layered cheapest-first

A future need claims, in order:

1. **~250 unused `type` values and 5 spare `flags` bits** — most "new
   field" needs are actually a new type or flag (rule 2), at zero layout
   change.
2. **The reserved u32 at offset 24** — a full aligned word for a genuine
   new header field, claimable without a `ver` bump per rule 3.
3. **3 spare bits at offset 18** — small enums.
4. **The three reserved ID top octets** — the deep reserve, 8 bits each
   at the cost of `& 0xFFFFFF` masks on ID reads. Kept unclaimed
   deliberately; overlaying fields into ID words is the last resort.

## 7. Conformance vectors (binding on every adapter, D-11)

A lesson bought with the cc2obp garble: bit-representation mismatches
between protocols are the dominant cross-protocol failure mode, and they
are invisible until a human decodes audio. Every adapter ships, **before
it is declared done**:

1. **Ingress vectors** — captured real packets for at least one full call
   → the expected `nx_frame` sequence, byte-exact, checked by `tests/`.
2. **Egress vectors** — an `nx_frame` sequence → the expected wire
   packets, byte-exact.
3. **The loopback identity** — for burst-native protocols,
   `ingress(egress(f))` reproduces the burst bits exactly, *including*
   CALL_START and CALL_END, which is why those carry the burst (§4.2).
   For **IPSC**, whose wire carries the LC, embedded LC, headers, and
   terminators in unpacked form, the identity is asserted on the AMBE
   bits **and the LC** — a retarget rewrites that LC in place (dmrlink3
   does exactly this), so a vector that ignores it would miss the whole
   addressing bug class. For **CC-CC**, whose wire really does carry
   bare AMBE plus call signalling, assert on the AMBE bits only.

   In both, purely structural fields the origin never carried — sync,
   slot type, EMB — are not round-tripped and must not be asserted on.

Vector sources: paired live captures, cross-checks against `dmr_utils3`
for HBP and IPSC framing, and `cccc_ambe.c`'s golden data for CC.
