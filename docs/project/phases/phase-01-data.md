# Phase 1: Data acquisition and freeze (Thu night fetch → Fri 9:00–10:30 AM) ★

**Status:** In progress (API fetch finishing; gold labels pending) · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [x] 1. Shallow-clone `kubernetes/enhancements` and fetch `kubernetes/community/sigs.yaml` (no API cost).
- [ ] 2. `github_api.py` for the chosen SIGs: tracking issues and their comments, KEP PRs, PR review comments and reviews. Paginate, save each page, resume on failure, sleep on rate limit. Run it overnight.
- [x] 3. Pick the 5–6 SIGs by KEP count and the share with rejected, withdrawn or replaced KEPs. Likely node, api-machinery, storage, network, scheduling, apps.
- [ ] 4. Write `MANIFEST.json` (SHA-256 per file, fetch time, counts), and reserve a **live feed**: the KEPs and comments updated in the last ~30 days.
- [ ] 5. Read 15 KEP threads yourself and hand-label `eval/gold_decisions.csv`: outcome, a one-line reason, the evidence URL, and the replacement. At least 4 should be rejected, withdrawn or replaced.
- **Done when:** raw data is frozen, the manifest is committed, and the 15 gold rows exist.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->
- 2026-10-16 01:40 IST · Normalizers done (pulled forward from Phase 3): 644 KEPs, 37 teams, 241 subprojects, 502 code areas from OWNERS (aliases expanded), 5,584 git-history events 2017–2026. Problems: YAML date parsing crashed on 2 files (fixed with a string-date loader); most `replaces` values point to old design docs, not KEPs (kept as doc links); date-style KEP names resolved by slug similarity. Gold candidate list written. Commit 1ac6da0.
- 2026-10-15 23:55 IST · Git sources cloned; API fetch running (issues/PRs for 6 SIGs, 3,725 comment threads, reviews for 2,783 PRs).

## How to hand-label (step 5, your task)
1. Open `eval/gold_candidates.csv`. Pick 15: at least 4 withdrawn/replaced/rejected, 3 with supersession, the rest from the most-debated implemented ones.
2. For each, read the tracking issue (`issue_url`) and skim the README design doc.
3. Add one row to `eval/gold_decisions.csv`:
   - `outcome`: accepted, rejected, withdrawn, replaced or deferred
   - `reason_one_line`: why, in your own words
   - `evidence_url`: link to the comment that states the reason (click a comment's timestamp to get its URL)
   - `replaced_by`: KEP id if any
   - `labelled_by`: your name
4. Budget about 3 minutes per KEP. These rows grade the LLM extraction and the agent, so never copy them from the graph.

## Open questions
