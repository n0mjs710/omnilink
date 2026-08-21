# DEVIATIONS.md — Where Implementation Met the Docs and Something Gave

The docs in `docs/` are normative and code is written to them
(STYLE.md). When implementation discovers that a document is wrong,
ambiguous, or impossible, the response is **flag-then-fix**: record it
here, implement the smallest reasonable interpretation, and keep going.
Do not silently diverge, and do not silently halt.

**Implementers do not edit the normative docs.** An entry here is how a
change gets proposed; the project owner folds accepted entries back into
the doc and the decision record, then marks the entry resolved.

An entry is worth writing when a doc is contradicted, not when a doc is
merely silent. Silence is a question for the owner (CLAUDE.md).

## Format

```
## YYYY-MM-DD — short title

**Doc:** the file and section that is wrong or unclear
**Found:** what implementation actually required
**Did:** the smallest reasonable interpretation, as shipped
**Proposed fix:** what the doc should say
**Status:** open | accepted (doc updated <commit>) | rejected (why)
```

---

*No entries yet — the corpus was rebuilt on 2026-08-21 and no code has
been written against it.*

The previous corpus's entries were retired with it. Two of them had
already been folded into the current documents and are noted here so the
reasoning is not lost:

- **The XLX module link is protocol machinery, not frame synthesis.** The
  five-frame link burst never becomes an `nx_frame` and never enters the
  core, so it does not violate "nothing synthesizes frames" — that rule
  governs fabricating call content into the routing path. Now stated
  directly in ADAPTERS.md §4.1 and D-14.
- **`dmr/` cannot build a burst at a configured colour code as
  vendored.** `DMR_SLOT_TYPE_VHEAD`/`VTERM` are precomputed for colour
  code 1 and `golay.c` has no Golay(20,8) encoder. Now a named phase-0
  task rather than a discovery waiting to happen (FRAME.md §4.1,
  PLAN.md phase 0).
