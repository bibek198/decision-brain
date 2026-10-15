# Phase 4: Cypher tools (Fri 4:00 – 8:00 PM) ★

**Status:** Not started · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- One `run(query, params)` wrapper records the Cypher, params, milliseconds and rows into the trace. A path collector returns nodes and edges for the UI.
- Common output shape: `{data, notes, sources[], path{nodes,edges}, cypher_trace[], as_of, stale_flags[]}`. Every read tool takes an optional `as_of`; the default is current, meaning `valid_to IS NULL`.
- **Read tools** (each is 1–3 parameterized queries):
  - `resolve`: full-text plus vector to candidate entities, with an "unknown" result when nothing matches.
  - `who_knows`, `who_owns`.
  - `trace_decision`: status, rationale, evidence, supersession chain, honest "not found".
  - `impact_of`: SEE_ALSO, SUPERSEDES and GATED_BY→AFFECTS up to 3 hops.
  - `timeline_as_of(kep|topic, release|date)`, `what_changed(since)`.
  - `connect(a, b)` via `algo.SPpaths`.
  - `influence` (PageRank and betweenness), `communities` (CDLP vs SIG overlap).
- pytest: one test per gold KEP; rejected never returns accepted; every tool returns at least one source.
- `docs/tools.md` holds every query; it becomes the README section rule 10 requires.
- **Done when:** all tools pass tests with no LLM involved.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->

## Open questions
