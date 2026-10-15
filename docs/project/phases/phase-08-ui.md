# Phase 8: UI for the camera (Sat 6:00 – 10:00 PM; start the static shell during the Phase 5 batch on Friday night) ★ for path, Cypher, compare and live feed

**Status:** Not started · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- Real-time UI pieces:
  - **Live activity feed** (SSE): each event as a row, the touched node flashing in the graph.
  - **Freshness badge**: "as of 12 s ago · lag 3.1 s".
  - **"Answer outdated"** toast with a re-ask button.
  - **Time slider**: release or date into `as_of`, so the graph and answer show the past state.
- FastAPI provides `/api/ask` (modes graph, baseline, both), `/api/health` (counts and last sync), `/api/examples`, `/api/sync`, `/api/act/*`.
- One static page:
  - Answer panel with `[n]` links.
  - Cytoscape panel: node colour by type, KEP colour by status, path animated, click to see details.
  - "How I got this" panel with the Cypher and milliseconds.
  - Compare toggle against the baseline.
  - Action cards: Draft brief and Suggested reviewers.
  - "Replay next change" button (demo fallback if no live event arrives on camera).
- Test 10 questions in the browser, check at 1366×768, and save screenshots.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->

## Open questions
