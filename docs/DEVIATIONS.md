# DEVIATIONS.md — where implementation met the docs and something gave

Append-only. One entry per discovery, newest last. Referenced by
`CLAUDE.md` and `STYLE.md`: when reality contradicts a normative doc,
implement the smallest reasonable interpretation, record it here, and keep
going. **Do not edit the normative doc yourself** — this file is how the
change gets proposed to the project owner.

Entry format:

```
## YYYY-MM-DD — short title
**Doc:** which file and section
**Found:** what the doc says, and what reality is
**Did:** the smallest reasonable interpretation, as implemented
**Proposed fix:** what the doc should say (owner decides)
```

---

## 2026-08-16 — "No frame synthesis anywhere" vs the XLX module link

**Doc:** CLAUDE.md ("No frame synthesis anywhere"), D-16 ("Nothing in the
system synthesizes frames"), ADAPTERS.md §6.1

**Found:** XLX selects its module with an in-band private call — ten DMRD
packets the adapter must construct, since xlxd offers no other channel for
it. Read literally, the no-synthesis rule forbids exactly this, and an
implementer reaching §6.1 could reasonably stop and ask.

**Did:** Built the link in the XLX adapter and emitted it straight to the
socket. The rule is unaffected: it governs `nx_frame` — fabricating call
content into the routing path, which is what D-16 is defending against
(headerless streams staying headerless, no timeout terminators, no
invented CALL_START). The link never becomes a frame and never reaches
`nx_core_ingress`. It is protocol machinery in the same class as a login
packet or a keepalive, which ADAPTERS.md §1 already assigns to adapters.

**Proposed fix:** none to the rule — the distinction it rests on is real
and worth keeping sharp. ADAPTERS.md §6.1 now states it explicitly at the
point of confusion, which should be sufficient. Recorded here because the
question will recur for any future adapter whose protocol carries control
information in call-shaped packets.

---

## 2026-08-16 — Independent design review: four defects fixed pre-code

An outside review of the corpus (post-D-02-revision) found four defects.
All four are fixed in the normative docs by the owner; recorded here
because each was a rule an implementer would otherwise have built to.

**1. Per-bridge hang time was a floor lockout.** ROUTING.md §4 granted a
bridge only on `now >= hang_until OR src_id == hang_src`, and cited
hblink4. hblink4 actually admits a *different* user on the *same*
talkgroup during hang (`hblink4/hblink.py:1757-1762`, comment: "Same TG,
different user — allow"); hblink3 is the same shape. Since a bridge is
one talkgroup, every contender on it is that case — the rule would have
refused every second talker for 4 s after every transmission. **Fixed by
deleting per-bridge hang entirely**: hang holds a radio access channel,
and a bridge has no channel, so it now lives only at the adapter on
`(system, slot)` where D-15 already put slot truth. `hang_time` moved
from `[[bridge]]` to `[[system]]`, and `stream_to` (0.36 s) was added as
the separate short contention horizon hblink3 has and OmniLink lacked.

**2. CALL_START/CALL_END carried a decoded 9-byte LC.** A pre-D-02
residue: it forced HBP and OBP — the near-passthrough adapters — to BPTC
decode and re-encode the header and terminator of every call on the path
D-02 exists to keep free, replaced the origin's colour code with ours,
and made FRAME.md §7's loopback identity false as written. **Fixed** by
carrying the burst at `pfmt 0`, keeping the 9-byte LC at `pfmt 1` for
adapters with no burst (CC). Same review found the related residue that
CC ingress was specified to emit `vseq = 7` while also constructing a
burst — impossible, since building one requires choosing the position.

**3. The adapter `egress` contract could not express unit calls.** It
took only `(inst, frame)`, while ROUTING.md §6 required delivery to a
cached peer and slot held in the core, and FRAME.md §1.1 explicitly
forbids carrying slot to egress. There was no channel; phase 5 was
unbuildable. **Fixed** by adding `const nx_egress_target *t`
`{tgid, slot, peer}`, filled by the core from the bridge member or the
route cache. This also removes the per-adapter replica of bridge config
that ROUTING.md §5 previously implied, which D-23 forbids.

**4. D-16/D-17 were contradicted by the code phase 4 lifts.**
`ipsc2hbpc/src/translate.c` emits a terminator on stream timeout and
comfort silence into empty playout slots — both forbidden as written.
Neither is a defect: IPSC has no representation for absence. **Fixed** by
carving the exception explicitly, scoped to what an IPSC *egress* writes
to its own wire, never to frames or the core.

Also corrected from the same review: ARCHITECTURE.md §1 sized the daemon
on ingress frames when the real load is egress fan-out (two orders
larger); the LOC estimate was ~40% light and is now a table measured
against the ancestors; blocking `getaddrinfo` in the connect path is
called out as the concrete single-thread hazard the watchdog does not
catch; pool-exhaustion behavior is specified; `dmr/` is noted to lack the
Golay(20,8) slot-type encoder that FRAME.md §4.1 rule 2 requires; and
ROUTING.md §8's migration advice ("model it as two systems") is corrected,
since an HBP master serving real repeaters cannot be split.
