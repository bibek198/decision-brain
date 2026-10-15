# Phase 5: LLM rationale extraction (Fri 8:00 – 11:30 PM, the batch runs while you build Phase 6)

**Status:** Not started · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [ ] 1. Contract: the model never decides the outcome, which comes from `kep.yaml`. It returns a summary, rationale, alternatives, and evidence as `{comment_id, quote_start}`, plus any supersession claims.
- [ ] 2. Pilot on the 15 gold KEPs and tune at most 3 times. Targets: at least 95% of evidence passes the quote check and at least 80% of rationales are right or partly right.
- [ ] 3. Submit the full scope through the **Batch API** (Haiku 5.5), caching one JSON file per KEP.
- [ ] 4. Validate (comment ID exists, and the quote is found after whitespace collapse) and write `Decision`, `DECIDED` and `EVIDENCED_BY`.
- [ ] 5. Read every LLM-proposed SUPERSEDES edge yourself, then audit 20 random decisions and record the counts.
- **Done when:** decisions are loaded, the audit counts are in `eval/extraction_report.md`, and `trace_decision` is upgraded.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->

## Open questions
