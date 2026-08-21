# FRAME.md — The Interchange Unit and the DMR Burst

What crosses the adapter/core boundary is the **HBP DMRD packet itself,
unmodified**, plus two scalars the wire has no room for (D-05):

```c
void nx_core_ingress(uint16_t sys, uint32_t tag, const uint8_t *dmrd, int len);
```

- `sys` — which configured system the traffic arrived on. An argument at
  ingress and core stream-table state thereafter; never needed on egress
  (ADAPTERS §1).
- `tag` — the core-assigned stream identity (§3).

There is no internal frame format, deliberately. DMRD is already
specified, already carries everything the core routes on, and is already
exercised in production by four codebases in this family. A second
byte-exact representation would have to be specified, tested, and kept
correct, in exchange for a modularity claim the core does not need.

## 1. The DMRD packet

```
offset  size  field
------  ----  ------------------------------------------------------
 0       4    "DMRD"
 4       1    sequence number
 5       3    source ID (24-bit radio ID)
 8       3    destination ID (24-bit TGID, or radio ID for a unit call)
11       4    repeater ID — the originating infrastructure device
15       1    bits — §1.1
16       4    stream ID — the originating protocol's, not ours (§3)
20      33    the on-air DMR burst, verbatim
53       2    BER, RSSI — present on 55-byte packets, absent on 53-byte
------  ----
53 or 55 total
```

Both lengths exist in the wild; adapters carry `len` and neither truncate
nor pad on the core's behalf.

### 1.1 The `bits` byte

```
0x80  timeslot          set = TS2, clear = TS1
0x40  call type         set = unit (private)
0x30  frame type        >> 4: 0 = voice, 1 = voice sync, 2 = data sync
0x0F  dtype / vseq      data sync: 1 = voice header, 2 = terminator, 3 = CSBK
                        voice:     0 = burst A … 5 = burst F
```

hblink3's decode is the reference, including its CSBK special case
(`(bits & 0x23) == 0x23` → `vcsbk`).

Accessors are explicit shift and mask. **Never C struct bitfields for
wire data** — their layout is implementation-defined.

**Egress rewrites exactly one thing in this byte**: the timeslot, to the
target member's slot. Frame type, dtype/vseq, and call type pass through
untouched. Burst position is preserved rather than re-counted, because
that is what RF does — losing bursts C and D shows a receiver A, B, gap,
E, F, and relabeling would be editing the stream (D-13, D-14).

## 2. What the core reads, and what it does not

The core reads source, destination, repeater ID, and the `bits` byte. It
treats bytes 20–52 — the burst — as **opaque** (D-25).

Any feature that would require the core to look inside a burst is a
design violation: extend an adapter instead. Talker alias, GPS, and data
payloads all live in the burst, and all of them are an adapter's
business.

Forward compatibility falls out of this for free. An unrecognized
`frame_type`/`dtype_vseq` combination arriving on a stream the core
already has open is forwarded like any other traffic, so new in-stream
DMR features need no core change and egress adapters ignore what they do
not implement.

## 3. Stream identity — the core-assigned tag

Every adapter needs a stable per-stream identity, and every one already
maintains the state to produce it. What differs is where that identity
comes from:

| adapter | one stream per | new stream when | wire has an ID? |
|---|---|---|---|
| HBP | (client, slot) | wire `stream_id` changes, or a voice header arrives | yes |
| XLX | the connection (TS2 only) | wire `stream_id` changes | yes |
| OBP | (link, wire `stream_id`) — many concurrent | an unseen `stream_id` | yes |
| IPSC | (peer, slot) | `VOICE_HEAD`, or RTP-timestamp discontinuity | **no** |
| CC-CC | the conduit | B-on | **no** |

**The tag's one job is preventing two streams from sharing an identity.**

- For IPSC and CC-CC it adds no capability. Neither wire carries a usable
  per-call ID — IPSC's call-seq byte is re-minted every superframe by
  XPR8400 firmware when Talker Alias is interleaved, so `ipsc2hbpc`
  anchors on RTP continuity and mints its own — so those adapters invent
  an identifier either way.
- For HBP, OBP, and XLX the wire ID is real and stable within a call. The
  tag exists because that ID is chosen at random by the originating
  repeater, not by us.

**The failure it removes:** two clients on one system pick the same
`stream_id` concurrently, their streams merge into one core entry, and
the second one's audio is delivered on the first one's cached delivery
set (D-29) — wrong talkgroup, wrong members, until the overlap ends.
Bounded and self-healing, but wrong while it lasts, and unreproducible.

**It is nearly free.** The ingress map must exist to detect a new stream
at all, and the egress map must exist because one core stream fanned to
three HBP systems needs three distinct wire stream IDs. The tag is just
what those maps hold.

### 3.1 Mechanism

**The tag** is a global monotonic `u32` from `nx_stream_tag_next()` — one
increment, no core state, no locking (single thread). Global rather than
per-system means the tag alone identifies a stream, so no compound key.
At ten new streams per second it wraps in roughly thirteen years.

**Ingress map**, private to each adapter: protocol identity → tag. For
HBP and IPSC the client or peer struct already exists for authentication
and keepalives, so this is `ep->slot[ts].tag` — compare the stored wire
ID against the arriving one, equal on nearly every packet, so the common
path is two loads and a compare. On change, allocate a tag.

**Egress map**, also private: tag → this system's wire identity. Mint a
fresh `stream_id` on the first packet of an unseen tag.

Both maps expire on call end or their own inactivity timer.

**The originating wire `stream_id` must appear in the `call_start`
event.** It is how an operator correlates a call against a *far* system's
dashboard or logs. The adapter has it in its ingress map, so this is
free — but it has to be specified, or it is quietly lost.

**Headerless streams need no flag.** An egress adapter seeing its first
packet for an unseen tag knows: if it is not a voice header, the
stream is a late entry and it emits no wire header. Self-contained, no
core involvement (D-14).

## 4. The burst

Bytes 20–52 are the on-air DMR burst, verbatim: 264 bits including FEC,
interleave, EMB, sync or embedded-LC fragment, and slot type.

Adapters fall into three tiers, and the gap between the second and third
is much larger than the gap between the first and second:

| tier | adapters | what the wire has | ingress work |
|---|---|---|---|
| **assembled burst** | HBP, OBP, XLX | the DMRD packet itself | none |
| **unpacked elements** | IPSC | LC, embedded LC, headers, terminators, superframe position, timeslot — everything but the OTA FEC/interleave, EMB, and slot type | re-pack |
| **AMBE + call signalling** | CC-CC | three 49-bit AMBE frames, RTP, B-on/B-off | **synthesize** |

**CC-CC is the only adapter without the burst and superframe data already
baked in.** Everything else has the material and merely has to assemble
it. That is why the rules below have one adapter as their real audience,
why only CC-CC runs its own A–F position counter (ADAPTERS §6), and why
only CC-CC's conformance identity is asserted on the AMBE bits alone
(§6).

`dmr/` implements every primitive either tier needs (`dmr_ambe_49_to_72`
and its inverse, BPTC, embedded LC, RS(12,9), and the precomputed
slot-type, EMB, and sync tables).

Two rules bind any adapter that assembles a burst:

1. **Header and payload must agree.** The burst's LC carries source and
   destination redundantly with the DMRD header. They are patched
   together or not at all. A burst whose LC contradicts its header is the
   single most expensive class of bug this project can ship: it survives
   every test that does not decode audio, and it has been shipped before
   in this ecosystem. See ROUTING §7 for the standing rule.

2. **The LC must be built from real values, never from a captured
   constant.** FLCO, FID, service options, source, and destination come
   from the stream being carried. Lifting a known-good burst blob from
   prior art and patching a couple of fields is how contradictory bursts
   get shipped.

   Structural fields genuinely absent from the origin — sync flavour,
   EMB, slot type — come from `dmr/`'s tables at colour code 1, per §4.2.
   They are not configuration.

### 4.1 Retargeting

When a stream is delivered to a member whose TGID differs, the header
destination **and** the burst's LC and embedded-LC fragments all carry the
old destination and must be rewritten together.

**The core does not do this; the egress adapter does.** The core passes a
`const` pointer and the delivery parameters, and the adapter makes the one
copy it was always going to make — to write its own `stream_id` and
repeater ID — rewriting header destination and LC in the same pass. That
is the cheapest way to satisfy rule 1, because the two edits then cannot
physically drift apart.

Generate the replacement LCs **once per stream per target**, then splice
per burst by position. Do not rebuild per packet. hblink3's `gen_lcs()` /
`embed_lc()` pair is the reference implementation and is known-correct in
production.

**Known limitation: retargeting clobbers Talker Alias.** TA rides in the
embedded LC on bursts B–E, which is exactly the field a TGID rewrite
replaces (`embed_lc()` substitutes `_emb_lc[_dtype_vseq]`). A real
problem to be solved on its own merits — not created by any choice here,
and no internal representation would have avoided it.

### 4.2 Colour code is RF-local

Colour code is an RF air-interface interference-mitigation mechanism. It
has no meaning between repeaters; each applies its own before
transmitting. Three rules, deliberately not one rule (D-28):

- **Never read it.** Not a routing key, not an ACL input, not a match
  condition, not a filter.
- **Never rewrite it.** A passed-through burst keeps whatever the origin
  stamped, so a stream may carry a foreign colour code end to end.
- **Use 1 when you must invent one.** IPSC and CC-CC re-pack or
  synthesize, and the XLX module link builds a burst from nothing; all
  use `dmr/`'s precomputed CC-1 tables.

Retargeting never touches it: the colour code lives in the slot type and
in EMB, and §4.1 rewrites neither. `dmr/` is therefore complete as
vendored — no Golay(20,8) slot-type encoder is required.

## 5. Repeater ID and source peer

Bytes 11–14 identify the originating infrastructure device. Ingress
accepts what arrives, the core carries it unchanged, and egress decides
what to put on its own wire per that system's config and protocol
convention. The full rule and the per-adapter table are in ADAPTERS §1.2
— including IPSC, which has no choice.

## 6. Conformance vectors (binding on every adapter, D-11)

A lesson bought with the cc2obp garble: bit-representation mismatches
between protocols are the dominant cross-protocol failure mode, and they
are invisible until a human decodes audio. Every adapter ships, **before
it is declared done**:

1. **Ingress vectors** — captured real packets for at least one full call
   → the expected DMRD sequence, byte-exact, checked by `tests/`.
2. **Egress vectors** — a DMRD sequence → the expected wire packets,
   byte-exact.
3. **The loopback identity:**
   - **HBP, OBP, XLX** — `ingress(egress(pkt))` reproduces the packet bit
     for bit, headers and terminators included.
   - **IPSC** — asserted on the AMBE bits **and the LC**. IPSC carries a
     DMR LC and a retarget rewrites it in place (dmrlink3 does exactly
     this), so a vector ignoring the LC would miss the entire addressing
     bug class.
   - **CC-CC** — asserted on the AMBE bits only; its wire genuinely
     carries nothing else.

   In every case, purely structural fields the origin never carried —
   sync, slot type, EMB — are not round-tripped and must not be asserted
   on.

Vector sources: paired live captures, cross-checks against `dmr_utils3`
for HBP and IPSC framing, and `cccc_ambe.c`'s golden data for CC-CC.
