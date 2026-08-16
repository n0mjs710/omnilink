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
