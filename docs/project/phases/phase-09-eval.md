# Phase 9: Evaluation (Sat 10:00 PM – 1:00 AM; write the harness on Friday night while the batch runs) ★ for lure, the table and the replay eval

**Status:** Not started · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [ ] 0. **Time-travel replay eval** (`eval/replay_eval.py`), the headline real-time result.
   - Start `brain_replay` at checkpoint T0 = 2 years ago, then stream `history_events.jsonl` in order through the **real worker**.
   - At ~8 checkpoints (release dates), ask ~10 status, supersession and owner questions. Truth comes from `kep.yaml` at the git commit for that date (`git show <sha>:path`), never from the graph.
   - Compare against a baseline whose vector index is (a) rebuilt only at T0, the stale index, and (b) incrementally appended, where old and new chunks coexist.
   - Report accuracy per checkpoint, how many answers state the old status as current, and the median event-to-correct-answer lag.
- [ ] 1. Baseline: vector-only top-k over `Chunk` in FalkorDB, with the same LLM and prompt style. It has no Decision nodes and no traversal.
- [ ] 2. 50 questions, about 7 per category:
   - A who-knows
   - B why
   - C impact
   - D lure (rejected or withdrawn KEPs phrased as adopted, plus invented features)
   - E supersession and freshness
   - F ownership and routing
   - G as-of release

   Expected answers come from GitHub pages and hand reading, **not from graph queries**.
- [ ] 3. `run_eval.py` saves raw runs. A Haiku judge applies the rubric, and you hand-check every D and E answer plus 30% of the rest. Record citation validity, latency and cost per question.
- [ ] 4. Stretch: a distractor curve that adds other SIGs' KEPs at N = 0, 100, 300.
- [ ] 5. `eval/results.md` gives honest findings, including where the baseline wins.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->

## Open questions
