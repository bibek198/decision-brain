# Phase 1: Data acquisition and freeze (Thu night fetch → Fri 9:00–10:30 AM) ★

**Status:** Not started · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [ ] 1. Shallow-clone `kubernetes/enhancements` and fetch `kubernetes/community/sigs.yaml` (no API cost).
- [ ] 2. `github_api.py` for the chosen SIGs: tracking issues and their comments, KEP PRs, PR review comments and reviews. Paginate, save each page, resume on failure, sleep on rate limit. Run it overnight.
- [ ] 3. Pick the 5–6 SIGs by KEP count and the share with rejected, withdrawn or replaced KEPs. Likely node, api-machinery, storage, network, scheduling, apps.
- [ ] 4. Write `MANIFEST.json` (SHA-256 per file, fetch time, counts), and reserve a **live feed**: the KEPs and comments updated in the last ~30 days.
- [ ] 5. Read 15 KEP threads yourself and hand-label `eval/gold_decisions.csv`: outcome, a one-line reason, the evidence URL, and the replacement. At least 4 should be rejected, withdrawn or replaced.
- **Done when:** raw data is frozen, the manifest is committed, and the 15 gold rows exist.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->

## Open questions
