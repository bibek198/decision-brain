# Phase 2: Schema on paper → code (Fri 10:30 AM – 12:00 PM) ★

**Status:** Not started · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [ ] 1. Write the 10 demo questions as paths in words. Each must use 4 edges or fewer.
- [ ] 2. Test node vs property and the process-label stoplist (`lgtm`, `approved`, `size/*`, `kind/*` stay properties).
- [ ] 3. `schema.py` creates range indexes on keys, full-text on `KEP(title)`, `Comment(body)` and `Chunk(text)`, and a vector index on `Chunk(embedding)` (384-d, cosine). It is safe to run twice.
- [ ] 4. Build a toy graph and run every tool query on it. Write `docs/schema.md` with a diagram.
- **Done when:** every demo question is answered on the toy graph.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->

## Open questions
