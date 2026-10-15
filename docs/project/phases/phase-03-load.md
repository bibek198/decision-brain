# Phase 3: Normalize and load (Fri 12:00 – 3:30 PM) ★

**Status:** Not started · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [ ] 1. Parse `kep.yaml` defensively, since status values are messy (for example "imlpemented" and inline comments). Lowercase logins and drop bots.
- [ ] 2. Parse references: `#NNNN`, KEP paths, `see-also`, `replaces`, `superseded-by`. Classify them as closes, builds_on, mentions or supersedes.
- [ ] 3. Loader: UNWIND in batches of 500, parameters only, MERGE on keys. Load nodes first, then edges, then derived `REVIEWED_FOR`. **Write it as small upsert functions** (`upsert_kep`, `upsert_comment`, `set_status(kep, status, at)` …). The real-time handlers call these same functions in Phase 7, so there is one write path.
- [ ] 3b. **History backfill:** `git log -p --follow` over each `kep.yaml` and `sigs.yaml` produces dated status, milestone and leadership changes. Write them as `HAD_STATUS` and `LEADS` intervals with `valid_from` and `valid_to`, and save them as `data/seed/history_events.jsonl`, which the replay eval reuses.
- [ ] 4. Chunk and embed (fastembed `BAAI/bge-small-en-v1.5`, 384-d) KEP READMEs and comments into `Chunk` nodes.
- [ ] 5. Verify counts against GitHub for 5 KEPs, run the **idempotence test**, and take a Browser screenshot.
- **Done when:** the second load changes nothing and counts match.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->

## Open questions
