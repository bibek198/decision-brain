# Phase 2: Schema on paper → code (Fri 10:30 AM – 12:00 PM) ★

**Status:** Done · **Actual:** Fri 01:45 – 02:20 (pulled forward) · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [x] 1. Write the 10 demo questions as paths in words. Each must use 4 edges or fewer.
- [x] 2. Test node vs property and the process-label stoplist (`lgtm`, `approved`, `size/*`, `kind/*` stay properties).
- [x] 3. `schema.py` creates range indexes on keys, full-text on `KEP(title)`, `Comment(body)` and `Chunk(text)`, and a vector index on `Chunk(embedding)` (384-d, cosine). It is safe to run twice.
- [x] 4. Build a toy graph and run every tool query on it. Write `docs/schema.md` with a diagram.
- **Done when:** every demo question is answered on the toy graph.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->
- 2026-10-16 02:20 IST · schema.py (38 indexes, idempotent), docs/schema.md, toy graph answering all 10 demo questions in pytest. Problems: FalkorDB shortestPath is directed-only and limited to WITH/RETURN, so connection questions use algo.SPpaths with relDirection both. Intervals stored as epoch seconds. Process labels stay properties. Commit 0fe6f2c.

## Open questions
