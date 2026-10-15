# Decision Brain v2: analysis and full phase plan (Graph Hacks, Track 3)

## Context

You are entering Graph Hacks (WeMakeDevs × FalkorDB, Oct 15–18 2026, deadline **Sun 18 Oct 11:59 PM IST**), Track 3 "Company Brain" (prize: M6 Mac mini). The earlier master plan (Claude Doc "Graph Hacks Master Plan") assumed a Thursday-noon start; it is now **Thu ~11 PM**, the project folder is empty, and the Phase 0 zip was never unpacked. You asked for a full re-analysis before rebuilding from the start, aimed at winning rather than just shipping.

All 1,390 lines of the old plan were read, along with the official overview, rules, schedule and resources pages, the FalkorDB docs (indexes, algorithms, MCP, Docker), the organisers' "agent that reasons over a graph" blog post, current LLM pricing, and the candidate data repos.

---

## Part 1: What the research changed

### 1. The old plan under-serves the Track 3 brief (the biggest gap)
The brief, verbatim: *"Build an agent that understands how a company works. Connect people, teams, projects, documents, and decisions in one graph, so the agent can answer questions and **get work done** across all of it."* Focus areas: knowledge from **docs, tickets, and chats** in one graph; **who owns what** and who knows what; tracing decisions to source; keeping knowledge current; answers with clear sources.

The old plan (RFC PRs only) covers decisions, sources and freshness. It has **no teams, no ownership, one source type, and no "get work done" actions**. A judge reading the brief will tick fewer boxes.

### 2. A better "company": Kubernetes enhancements (KEPs) plus the community SIG registry
Verified tonight (Apache-2.0, pushed yesterday, so it is a live source):
| Brief asks for | Kubernetes gives it, mostly structured (no LLM needed) |
|---|---|
| Teams | 24 SIGs in `kubernetes/community/sigs.yaml`, with chairs, tech leads, subprojects and missions |
| Who owns what | SIG → subprojects → OWNERS paths; KEP `owning-sig`, `participating-sigs`, `approvers` |
| People and roles | ~730 handles across KEP authors, reviewers and approvers; ~370 SIG leaders (drop emails, keep handle and company) |
| Documents | 686 KEPs (`kep.yaml` metadata plus README design docs) |
| Tickets | KEP tracking issues in `kubernetes/enhancements` (one per KEP, with release-cycle comments) |
| Chats | PR review threads and issue comments |
| Decisions | KEP `status` (implementable / implemented / rejected / withdrawn / replaced / deferred) and per-release milestones (alpha, beta, stable). The LLM adds only the rationale |
| Supersession ground truth | 63 `replaces` / `superseded-by` entries plus `see-also` links, so freshness answers have independent truth |
| "What does it affect" | `feature-gates` → `components` (kubelet, kube-apiserver…) |
| Time | releases v1.x with dates, so "as of release X" questions work |

KEPs are a real company's decision system, and the domain is widely recognised by judges. Scope the GitHub-API part (issues and comments) to **5–6 SIGs (~200–250 KEPs)** and load all 686 `kep.yaml` files from a git clone, which costs no rate limit.

### 3. "Get work done" means write actions, with a human in the loop
Add 3 action tools that write to the graph and produce artefacts, never posting to GitHub:
- `recommend_reviewers(area | kep | draft text)`: ranks people by ownership, approvals and PageRank, explaining each pick by its path.
- `draft_decision_brief(topic)`: a pre-filled KEP or ADR skeleton listing prior related decisions, superseded ones to avoid, owning SIG and approvers. It is saved as a `Draft` node and shown as markdown.
- `route_question(question)`: finds the owner team and best person, then logs a `Question` node linked to the entities used. This is also memory, which gives a Track 2 crossover.

### 4. Hybrid retrieval inside FalkorDB makes the graph more load-bearing
FalkorDB has an HNSW vector index (`CREATE VECTOR INDEX … OPTIONS {dimension, similarityFunction:'cosine'}`, `db.idx.vector.queryNodes`) and full-text search (`db.idx.fulltext.createNodeIndex` / `queryNodes`). Entity resolution becomes vector and full-text lookup in FalkorDB, then a Cypher traversal. The **baseline also lives in FalkorDB** as vector-only chunk retrieval with no traversal, which makes it a clean, fair ablation: same store, same LLM, the graph switched off.

### 5. Graph algorithms with a story (all confirmed in the docs)
`algo.pageRank(label, relType)` for influence; `algo.betweenness({nodeLabels, relationshipTypes})` for **people who bridge SIGs**; `algo.labelPropagation` (CDLP) for **informal teams vs the formal SIG chart**, a strong demo insight; `algo.SPpaths` / BFS for **"how is Alice connected to KEP X"**; WCC for data-quality checks.

### 6. LLM cost is a non-issue with Claude Haiku 5.5
Pricing (skill cache dated 2026-10-06): **Haiku 5.5 is $0.10 in / $0.50 out per million tokens**, Sonnet 5.5 is $2 / $10, and the Batch API is 50% off.
- Extracting ~250 threads at ~8k tokens in and 600 out comes to about **$0.30** at batch price.
- One agent question at ~12k in and 600 out costs about **$0.0015** on Haiku. 2 systems × 70 questions × 3 runs is about **$0.65**.
- The LLM judge on Haiku costs about $0.10.
- Total is about **$1.50 of your $5**. Keep Sonnet 5.5 for the recorded demo only (~$0.03 per question).

Embeddings are free and local through **fastembed** (ONNX, no torch), which also fixes the old plan's "huge Docker image" problem. If credits run out, a Gemini free-tier adapter is a 30-minute fallback, not a default.

### 7. Rules and judging facts that change priorities
- **A live deployment is preferred** (overview page). Deployment moves from "optional, cut first" to **planned**, in replay-default mode with rate limits.
- No scoring rubric is published. The organisers' blog says what they value: a fixed, validated tool catalogue (the model never writes Cypher); resolving entities against the DB first; returning the path; admitting "not in the graph"; the same question at two depths; memory stored as nodes.
- Multiple tracks are allowed (rule 5). Name **Track 3**, and Track 1 if the algorithm and action tools are solid. It is free to add and fits honestly.
- Blog side prize: AirPods 5. There are no submission instructions yet, so ask in the Discord.
- Docker persistence path is ambiguous in the docs (`/data` vs `/var/lib/falkordb/data`). Verify with `docker inspect`, and set `REDIS_ARGS=--appendonly yes`.
- Rule 6 (no personal data): drop emails from `sigs.yaml` and keep public handles and company only.

### 8. Kept from the old plan (it was strong)
Structure first and the LLM second; the quote-checked evidence contract; the citation guard; the lure (category D) test; the honest baseline comparison; replay mode; the fresh-clone test; secret scanning; and the cut-rules discipline.

### 9. Real-time adaptation must be a core feature, not a stretch goal (your requirement)
The earlier draft did **not** fully meet this. It used a frozen snapshot, one replayed "live feed" and optional polling, which proves the mechanism but does not adapt continuously. Part 2b below makes it a first-class subsystem with its own phase, demo and measured evaluation. "Real-time" here means each of the following:
1. **Continuous ingestion.** A worker polls `kubernetes/enhancements` (issues, PRs and comments updated since the cursor, using ETags so a 304 does not use rate limit) and runs `git fetch` to diff `kep.yaml` and `sigs.yaml`. A **sandbox repo you own** pushes real webhooks (HMAC-verified) to the EC2 app. That gives true push real-time on camera, which polling a repo you do not own cannot.
2. **Incremental, idempotent updates.** Each change becomes an event in a durable log, then a handler, then MERGE upserts. There are no rebuilds.
3. **Temporal truth.** Facts have `valid_from`, `valid_to` and `observed_at`. A status change *closes* the old interval instead of overwriting it, so "now", "as of v1.28" and "what changed since" are all answerable.
4. **Dependent knowledge reacts.**
   - Changed threads are re-embedded.
   - Affected Decisions are marked `stale` and re-extracted, debounced by thread hash.
   - Supersession and contradiction mark the old Decision `superseded` or `contested`.
   - Derived edges and PageRank are recomputed on a schedule.
5. **The agent knows what it knew.**
   - Every answer carries `as_of` and data age.
   - Past `Question` nodes linked to entities that changed are flagged "your earlier answer is now outdated".
   - Watch subscriptions push notifications to the UI over SSE.
6. **Measured.** A time-travel replay eval replays real `kep.yaml` git history in order and asks questions at checkpoints, with truth taken from git at that commit. It reports accuracy over time and the **event-to-correct-answer lag**, graph vs vector baseline.

### 10. Kept from earlier chats
- Use a public repo.
- Keep `brain_eval` (frozen) separate from `brain_live` (polled).
- Never open ports 6379 or 3000 publicly.
- Never commit `.env`.

---

## Part 2: Architecture

```
GitHub (git clone + REST)       kubernetes/community sigs.yaml
        │                                  │
        ▼                                  ▼
  fetch/ ──► data/raw (frozen, MANIFEST) ──► normalize/ ──► data/seed/*.jsonl (committed, small)
                                                   │
                     extract/ (Haiku 5.5, Batch, quote-checked) ──► data/seed/decisions.jsonl
                                                   │
                                            loader (MERGE, UNWIND)
                                                   ▼
                    FalkorDB: graph brain_eval (frozen) / brain_live (polled) / brain_demo
                    full-text + vector (fastembed 384-d) + range indexes
                                                   │
                         tools/ (fixed parameterized Cypher; validated inputs; trace + path)
                         ├─ read: resolve, who_knows, who_owns, trace_decision, impact_of,
                         │        timeline_as_of, what_changed, connect, influence, communities
                         └─ act:  recommend_reviewers, draft_decision_brief, route_question
                                 │                    │
                       agent/ (Claude tool loop,   mcp_server/ (same tools, FastMCP)
                       citation guard, replay)
                                 │
                    web/ FastAPI + one static page (Cytoscape.js): answer · path · Cypher · compare
                    eval/ graph agent vs vector-only baseline (same FalkorDB, same LLM)
```

### Schema v2 (final; checked against demo questions in Phase 2)
**Nodes**
- `Person{login}`, `Org{name}`
- `Team{id: sig-x, name, mission}`, `Subproject{name}`
- `KEP{number, title, status, stage, created_ts, updated_ts, url, path}`
- `Release{version, ts}`, `FeatureGate{name}`, `Component{name}`
- `Ticket{number, title, state, url}`, `PR{number, title, state, merged, url}`, `Comment{id, body, at_ts, url}`
- `Decision{id, outcome, rationale, alternatives, valid_from, valid_to, confidence, verified}`
- `Chunk{text, embedding}`
- Memory and actions: `Question`, `Draft`
- Real-time: `Status{name}`, `Event{event_id, kind, source, occurred_at, observed_at}`, `Watch{target, created_at}`, `Meta{key, value}`

**Edges**
- `LEADS{role}`, `MEMBER_OF`, `AFFILIATED_WITH`, `OWNS`, `PARTICIPATES_IN`, `HAS_SUBPROJECT`
- `AUTHORED`, `REVIEWS`, `APPROVES`
- `TRACKED_BY` (KEP→Ticket), `PROPOSED_IN` (KEP→PR), `WROTE`, `ON` (Comment→Ticket|PR)
- `SUPERSEDES{source, verified}`, `SEE_ALSO`, `GATED_BY`, `AFFECTS` (FeatureGate→Component), `TARGETS{stage}` (KEP→Release)
- `DECIDED` (Decision→KEP), `EVIDENCED_BY` (Decision→Comment), `MENTIONS` (Comment→KEP), `CHUNK_OF`
- `ASKED_ABOUT`, `REVIEWED_FOR{weight}` (derived)
- Temporal and real-time: `HAD_STATUS{valid_from, valid_to, observed_at}` (KEP→Status). `LEADS`, `OWNS` and `APPROVES` also carry `valid_from` and `valid_to`. `TOUCHED` (Event→any entity), `WATCHES` (Watch→entity), `OUTDATED_BY` (Question→Event)

`valid_from` and `valid_to` on Decision and status give **as-of** queries ("what was the plan for sidecars at v1.28?"), which is the strongest "keeping knowledge current" story.

## Part 2b: Real-time change engine (design)

```
 sources                       event log                    handlers (worker)                  consumers
 ─────────                     ─────────                    ─────────────────                  ─────────
 GitHub poller (60s, ETag) ─┐                           ┌─► upsert entities (MERGE)           agent tools read
 git fetch + yaml diff (5m)─┼─► Redis Stream ─► worker ─┼─► close/open temporal intervals ─►  current or as-of view
 Sandbox repo webhook ──────┘   `brain:events`  (XREAD,  ├─► re-chunk + re-embed changed text   SSE /api/stream ─► UI
   (HMAC, EC2, push)            in FalkorDB's     group, ├─► mark Decision stale → debounced    live feed, node flash,
                                own Redis)        ack)   │    re-extract (hash cache, Haiku)    "answer outdated" toast
                                                         ├─► supersession/contradiction check
                                                         ├─► flag dependent Question/Answers
                                                         ├─► notify Watch subscribers
                                                         └─► periodic: derived edges, PageRank
```
- **Event shape:** `{event_id, source, kind (kep_status_changed | comment_added | comment_edited | issue_closed | sig_lead_changed | …), entity_key, payload, occurred_at, observed_at}`. `event_id` is deterministic, as a hash of source, kind, key and occurred_at, so a replayed event is a no-op. The event is also written as an `Event` node linked to the entity it touched, so "what changed" is itself a graph query.
- **Why a Redis Stream:** FalkorDB *is* a Redis server, so the durable queue needs no extra service. Consumer groups give at-least-once delivery, and idempotent handlers make that safe.
- **Temporal model:** status, ownership and leadership are edges or nodes with `valid_from` and `valid_to` (null means current) plus `observed_at`. Examples: `(:KEP)-[:HAD_STATUS {status, valid_from, valid_to}]->(:Status)`, `(:Person)-[:LEADS {role, valid_from, valid_to}]->(:Team)`. Read tools filter `valid_to IS NULL` by default, or take `$as_of`.
- **History backfill:** `git log -p` over every `kep.yaml` and `sigs.yaml` yields the real past status and leadership changes as events. The **same pipeline** that runs live builds the history, so the replay eval exercises real code.
- **Freshness metadata:** `Meta{last_poll, last_event_at, lag_ms}`. `/api/health` exposes it, and every tool result carries `as_of`.
- **Staleness and contradiction rules:**
  - A new comment on a decided KEP marks the Decision `stale` until re-extraction finishes.
  - A `superseded-by` change or a supersession event sets the old Decision to `superseded` and creates `SUPERSEDES {source, verified:false}`.
  - The agent must hedge on unverified or stale facts ("reported as replaced by…, not yet confirmed").
- **Targets:** a structured change is queryable in **≤10 s** after the event is observed; LLM rationale refresh in **≤90 s**; the poll interval is 60 s (the GitHub limit of 5,000/h allows it easily with ETags).
- **Isolation:** `brain_eval` is frozen and used for every reported number. `brain_live` receives the poller and the webhook (EC2). `brain_replay` is rebuilt per eval run.

### Repo layout (fresh, in this folder)
```
decision-brain-graph-hack/
  README.md  LICENSE(MIT)  .gitignore  .env.example  docker-compose.yml  Dockerfile  pyproject.toml|requirements.txt
  src/brain/{config.py, db.py(trace wrapper), schema.py, ids.py}
  src/fetch/{git_keps.py, github_api.py, sigs.py}        src/normalize/{keps.py, threads.py, refs.py}
  src/load/{loader.py, derived.py, embed.py}             src/extract/{prompt.py, run_batch.py, validate.py}
  src/tools/{read_*.py, act_*.py, registry.py, paths.py} src/agent/{loop.py, guard.py, replay.py, cli.py}
  src/realtime/{events.py(schema+ids), stream.py(Redis Stream), poller.py, gitdiff.py, webhook.py,
                handlers.py, worker.py, notify.py(SSE)}       # handlers reuse src/load upsert functions
  src/mcp_server.py   src/bootstrap.py   src/baseline.py
  eval/replay_eval.py                                           # time-travel replay over git history
  web/{app.py, static/index.html, static/app.js}
  eval/{questions.jsonl, gold_decisions.csv, run_eval.py, judge.py, results.md}
  docs/{schema.md, tools.md, explain.md, demo_script.md, blog_notes.md}
  tests/  scripts/{smoke.py, reset_demo.sh}
  data/seed/ (committed)  data/raw/ (gitignored)
```

---

## Part 3: Phase plan (all times IST; about 73 hours remain)

Each phase has a done-when test. ★ means never cut. Commit at the end of every phase.

### Phase 0: Foundation (Thu 11:00 PM – 12:30 AM) ★
1. Scaffold the layout above in this folder; set up MIT, `.gitignore` (`.env`, `data/raw/`, `.venv/`), `.env.example` (`GITHUB_TOKEN`, `ANTHROPIC_API_KEY`, `FALKORDB_HOST/PORT`, `GRAPH_NAME`, `DEMO_REPLAY`, `LLM_MODEL`).
2. `git init`, create the public GitHub repo `decision-brain` (you, in the browser), and push the first commit.
3. `docker-compose.yml` with `falkordb/falkordb:latest` on 6379 and 3000, `REDIS_ARGS=--appendonly yes`, and a named volume. **Verify the data path with `docker inspect`** and do a restart test.
4. Create a Python 3.10 venv and install `falkordb fastembed anthropic fastapi uvicorn mcp pyyaml httpx pytest`.
5. `scripts/smoke.py` prints `[['Ada','Linus']]`, plus a vector-index smoke test (create, insert `vecf32`, query).
6. **Start the overnight fetch** (Phase 1 step 2) before sleeping.
- **Done when:** the repo is public, both smoke tests pass, data survives `docker compose restart`, and the fetch is running.

### Phase 1: Data acquisition and freeze (Thu night fetch → Fri 9:00–10:30 AM) ★
1. Shallow-clone `kubernetes/enhancements` and fetch `kubernetes/community/sigs.yaml` (no API cost).
2. `github_api.py` for the chosen SIGs: tracking issues and their comments, KEP PRs, PR review comments and reviews. Paginate, save each page, resume on failure, sleep on rate limit. Run it overnight.
3. Pick the 5–6 SIGs by KEP count and the share with rejected, withdrawn or replaced KEPs. Likely node, api-machinery, storage, network, scheduling, apps.
4. Write `MANIFEST.json` (SHA-256 per file, fetch time, counts), and reserve a **live feed**: the KEPs and comments updated in the last ~30 days.
5. Read 15 KEP threads yourself and hand-label `eval/gold_decisions.csv`: outcome, a one-line reason, the evidence URL, and the replacement. At least 4 should be rejected, withdrawn or replaced.
- **Done when:** raw data is frozen, the manifest is committed, and the 15 gold rows exist.

### Phase 2: Schema on paper → code (Fri 10:30 AM – 12:00 PM) ★
1. Write the 10 demo questions as paths in words. Each must use 4 edges or fewer.
2. Test node vs property and the process-label stoplist (`lgtm`, `approved`, `size/*`, `kind/*` stay properties).
3. `schema.py` creates range indexes on keys, full-text on `KEP(title)`, `Comment(body)` and `Chunk(text)`, and a vector index on `Chunk(embedding)` (384-d, cosine). It is safe to run twice.
4. Build a toy graph and run every tool query on it. Write `docs/schema.md` with a diagram.
- **Done when:** every demo question is answered on the toy graph.

### Phase 3: Normalize and load (Fri 12:00 – 3:30 PM) ★
1. Parse `kep.yaml` defensively, since status values are messy (for example "imlpemented" and inline comments). Lowercase logins and drop bots.
2. Parse references: `#NNNN`, KEP paths, `see-also`, `replaces`, `superseded-by`. Classify them as closes, builds_on, mentions or supersedes.
3. Loader: UNWIND in batches of 500, parameters only, MERGE on keys. Load nodes first, then edges, then derived `REVIEWED_FOR`. **Write it as small upsert functions** (`upsert_kep`, `upsert_comment`, `set_status(kep, status, at)` …). The real-time handlers call these same functions in Phase 7, so there is one write path.
3b. **History backfill:** `git log -p --follow` over each `kep.yaml` and `sigs.yaml` produces dated status, milestone and leadership changes. Write them as `HAD_STATUS` and `LEADS` intervals with `valid_from` and `valid_to`, and save them as `data/seed/history_events.jsonl`, which the replay eval reuses.
4. Chunk and embed (fastembed `BAAI/bge-small-en-v1.5`, 384-d) KEP READMEs and comments into `Chunk` nodes.
5. Verify counts against GitHub for 5 KEPs, run the **idempotence test**, and take a Browser screenshot.
- **Done when:** the second load changes nothing and counts match.

### Phase 4: Cypher tools (Fri 4:00 – 8:00 PM) ★
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

### Phase 5: LLM rationale extraction (Fri 8:00 – 11:30 PM, the batch runs while you build Phase 6)
1. Contract: the model never decides the outcome, which comes from `kep.yaml`. It returns a summary, rationale, alternatives, and evidence as `{comment_id, quote_start}`, plus any supersession claims.
2. Pilot on the 15 gold KEPs and tune at most 3 times. Targets: at least 95% of evidence passes the quote check and at least 80% of rationales are right or partly right.
3. Submit the full scope through the **Batch API** (Haiku 5.5), caching one JSON file per KEP.
4. Validate (comment ID exists, and the quote is found after whitespace collapse) and write `Decision`, `DECIDED` and `EVIDENCED_BY`.
5. Read every LLM-proposed SUPERSEDES edge yourself, then audit 20 random decisions and record the counts.
- **Done when:** decisions are loaded, the audit counts are in `eval/extraction_report.md`, and `trace_decision` is upgraded.

### Phase 6: Agent, actions, MCP (Sat 9:00 AM – 1:30 PM) ★ for the agent
1. Agent loop: Anthropic SDK, manual tool loop of at most 6 rounds, Haiku 5.5 by default (`LLM_MODEL` env), effort low or medium, `strict` tool schemas. The model sees only `data`, `notes` and `sources`, and the server keeps path and trace.
2. System prompt rules: answer only from tools; say "not in the graph"; never present rejected or withdrawn as adopted; name the superseded-by first; cite with `[n]`.
3. Citation guard: every URL in the answer must come from tool sources. Strip and retry once otherwise.
4. Action tools: `recommend_reviewers`, `draft_decision_brief` (a `Draft` node plus markdown), `route_question` (a `Question` node, i.e. memory). Writes go only to the current graph; nothing reaches GitHub.
5. MCP server (`mcp` FastMCP) exposes the same registry. Test it in the MCP Inspector and in Claude Desktop, which makes a nice 10-second video clip.
6. Run the 30 dev questions and fix tool descriptions. Hold out 10.
- **Done when:** at least 80% correct tool choice, the lure answers are honest, the guard test passes, and the Inspector lists the tools.

### Phase 7: Real-time change engine (Sat 1:30 – 6:00 PM) ★
1. **Event log** (`events.py`, `stream.py`): event schema, deterministic `event_id`, `XADD brain:events`, consumer group `brain-workers`, `XACK` after the handler succeeds, and a dead-letter list after 3 failures.
2. **Sources:**
   - `poller.py`: every 60 s, issues, PRs and comments `since=cursor` with `If-None-Match`; the cursor and ETag live in `Meta`.
   - `gitdiff.py`: every 5 min, `git fetch` and diff `kep.yaml` and `sigs.yaml` from the last SHA to the new SHA, producing typed events.
   - `webhook.py`: `POST /api/webhook/github`, HMAC SHA-256 verified, for the **sandbox repo you own**.
3. **Handlers** (`handlers.py`): one function per event kind, all built on the Phase 3 upsert functions.
   - Close the old interval and open the new one.
   - Re-chunk and re-embed changed text.
   - Mark the dependent `Decision` stale and enqueue re-extraction (debounced 30 s, skipped if the thread hash is unchanged).
   - Run the supersession and contradiction check.
   - Link `Event-[:TOUCHED]->entity`.
   - Flag `Question` nodes that used a touched entity (`OUTDATED_BY`).
   - Fan out to `Watch` subscribers.
4. **Periodic jobs:** recompute `REVIEWED_FOR` and PageRank every 10 min or every 50 events, and run a nightly reconcile for deleted comments.
5. **Worker** (`worker.py`): a separate compose service (`app`, `worker`, `falkordb`) running an asyncio loop for the poller, git diff and consumer. It writes `Meta.lag_ms`.
6. **Push to UI** (`notify.py`): `GET /api/stream` (SSE) emits events and stale-answer notices.
7. **Agent awareness:** answers state `as_of` and data age. `what_changed(since | since_my_last_question)` reads `Event` nodes. The system prompt hedges on `stale` or unverified facts.
8. **Demo kit:** a sandbox repo `decision-brain-sandbox` (yours, public) with seeded issues. On camera you comment "We are replacing X with Y, decided" and the UI feed updates in about 5 s, so the same question now gives the new answer and the old answer shows "outdated". `reset_demo.sh` resets `brain_demo` in under 1 minute.
9. Tests:
   - Replaying the same events twice gives identical counts.
   - Out-of-order events still give the right final interval (handlers compare `occurred_at`).
   - A status change closes exactly one interval.
   - An edited comment that loses its quote drops the evidence.
- **Done when:** an injected `kep.yaml` status change is queryable within 10 s; a webhook comment shows in the UI feed and flips the answer; a second replay changes nothing; the worker survives a FalkorDB restart, because the stream is persistent.

### Phase 8: UI for the camera (Sat 6:00 – 10:00 PM; start the static shell during the Phase 5 batch on Friday night) ★ for path, Cypher, compare and live feed
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

### Phase 9: Evaluation (Sat 10:00 PM – 1:00 AM; write the harness on Friday night while the batch runs) ★ for lure, the table and the replay eval
0. **Time-travel replay eval** (`eval/replay_eval.py`), the headline real-time result.
   - Start `brain_replay` at checkpoint T0 = 2 years ago, then stream `history_events.jsonl` in order through the **real worker**.
   - At ~8 checkpoints (release dates), ask ~10 status, supersession and owner questions. Truth comes from `kep.yaml` at the git commit for that date (`git show <sha>:path`), never from the graph.
   - Compare against a baseline whose vector index is (a) rebuilt only at T0, the stale index, and (b) incrementally appended, where old and new chunks coexist.
   - Report accuracy per checkpoint, how many answers state the old status as current, and the median event-to-correct-answer lag.
1. Baseline: vector-only top-k over `Chunk` in FalkorDB, with the same LLM and prompt style. It has no Decision nodes and no traversal.
2. 50 questions, about 7 per category:
   - A who-knows
   - B why
   - C impact
   - D lure (rejected or withdrawn KEPs phrased as adopted, plus invented features)
   - E supersession and freshness
   - F ownership and routing
   - G as-of release

   Expected answers come from GitHub pages and hand reading, **not from graph queries**.
3. `run_eval.py` saves raw runs. A Haiku judge applies the rubric, and you hand-check every D and E answer plus 30% of the rest. Record citation validity, latency and cost per question.
4. Stretch: a distractor curve that adds other SIGs' KEPs at N = 0, 100, 300.
5. `eval/results.md` gives honest findings, including where the baseline wins.

### Phase 10: Package and deploy (Sun 9:00 AM – 12:30 PM) ★ for packaging and replay
1. Tag `freeze`. Dockerfile (slim, fastembed model pre-downloaded). One `docker compose up` brings up FalkorDB and the app. `bootstrap` loads the seed, decisions and embeddings and is idempotent.
2. **Replay mode**: cached answers for the example questions, used by default with no key, with a clear banner.
3. Run the fresh-clone test twice (Path A all-Docker, Path B venv) and keep it under 10 minutes. Scan the history for secrets.
4. **Deploy**: AWS EC2 t3.medium on your credits.
   - Set a budget alert first.
   - Run Docker Compose (`falkordb`, `app`, `worker`, `caddy`) with Caddy, DuckDNS and HTTPS. FalkorDB stays bound to localhost.
   - Register the sandbox repo webhook to `https://<name>.duckdns.org/api/webhook/github` with a secret in `.env`, and run the poller against `brain_live`.
   - Replay is the default for questions, with a per-IP limit for live questions and a hard spend cap. The live feed and graph are always real.
5. `docs/explain.md`: five design answers in your own words (rules 12 and 13).

### Phase 11: Video, README, blog, submit (Sun 12:30 – 7:00 PM)
1. Video (≤3:00):
   1. Problem.
   2. Why question with path and Cypher.
   3. **Lure compare.**
   4. **Live change:** comment in the sandbox repo, the feed flashes, the same question gets a new answer, and the old answer is marked outdated.
   5. Time slider for as-of.
   6. Action: recommend reviewers and draft brief.
   7. MCP in Claude Desktop.
   8. Results table and replay-eval chart.
   9. Close.
2. README: all rule-10 items, the Mermaid architecture diagram, schema, queries, eval, AI disclosure, and "Tracks: 3 (and 1)".
3. Blog post on dev.to or Hashnode, 800–1,200 words. Ask in the Discord how to submit it.
4. Submit by 7:00 PM, check every link logged out, then make only safe edits until 11:30 PM.

### Cut order (when a checkpoint slips by more than 45 minutes)
1. Distractor curve
2. `communities` tool
3. Time slider in the UI (keep `as_of` in the tools)
4. MCP-in-Claude-Desktop clip
5. Watch subscriptions (keep the outdated-answer flag)
6. Action card UI (keep the action tools)
7. Path A Docker app image
8. Nightly reconcile job

**Never cut:**
- Honest not-found, the citation guard and the lure test.
- The event log, worker and temporal intervals, plus at least one live source (poller or webhook).
- The replay eval, the path, Cypher and live-feed panels.
- The README model and queries, AI disclosure, replay mode, and the submission.

Deployment on EC2 is now needed for the webhook. If it fails, fall back to the poller and the "replay next change" button locally.

### Daily checkpoints
| When | State |
|---|---|
| Fri 8:00 PM | Graph loaded and idempotent with history intervals; read tools (with `as_of`) pass tests |
| Sat 1:30 PM | Decisions extracted, agent answers with sources and `as_of`, MCP works |
| Sat 6:00 PM | Real-time engine: poller, git diff and webhook flow through the stream to the graph within 10 s |
| Sat end of day | UI with live feed is demo-ready; eval table and replay eval filled |
| Sun 7:00 PM | Submitted and deployed on EC2 with the webhook live |

---

## Verification (end to end)
- `python scripts/smoke.py` prints `[['Ada','Linus']]`; the vector smoke test returns nearest neighbours.
- `pytest -q` stays green: tools, guard, never-adopted-if-rejected, sync idempotence.
- `python -m src.agent.cli "Why was KEP-XXXX withdrawn?"` gives an answer with `[n]` sources, plus the tool and Cypher trace.
- `python -m src.load.loader` run twice gives identical counts per label and edge type.
- `python eval/run_eval.py --both` writes `eval/runs/<ts>/` and the results table.
- Fresh-clone test from an empty folder following only the README takes under 10 minutes, and replay works with no key.
- The deployed URL opens on a phone; ports 6379 and 3000 are unreachable from outside.
- Real-time:
  - `python -m src.realtime.inject --kep 753 --status withdrawn` makes the agent's answer change within 10 s and logs `Meta.lag_ms`.
  - A sandbox repo comment appears in the UI feed over SSE.
  - `pytest tests/test_realtime.py` covers replay twice (no change), out-of-order events, a single interval closed, and edited-quote evidence dropped.
  - `python eval/replay_eval.py` writes the accuracy-over-time table and lag stats.

## Decisions confirmed
- Data: **Kubernetes KEPs** (`kubernetes/enhancements` plus `kubernetes/community/sigs.yaml`), with the API data scoped to 5–6 SIGs.
- Deploy: **AWS EC2 on your credits** (Docker Compose, Caddy, HTTPS), replay by default.
- LLM: **Claude Haiku 5.5** for everything during development, eval and extraction (Batch API). Sonnet 5.5 is optional for the recorded demo. Estimated total is about $1.50 of the $5.

## Project tracking docs (created first in Phase 0, kept current every phase)
`docs/project/` is committed, so judges see the reasoning too (it backs rules 12 and 13):
- `README.md`: index and how to use these files.
- `PLAN.md`: this plan, the source of truth for scope. It covers context, research findings, architecture, the real-time design, schema, phases, cut order and checkpoints.
- `STATUS.md`: one table of phase · owner · planned window · status (Not started / In progress / Done / Cut) · done-when · commit link. Updated at the end of every phase.
- `DECISIONS.md`: an ADR-style log, newest first, as `D-NNN · date · decision · options considered · why · consequences`. It is seeded with the decisions already made:
  - D-001 Track 3 (+1)
  - D-002 Kubernetes KEPs as the company
  - D-003 Haiku 5.5 with Batch and the $5 budget
  - D-004 hybrid retrieval in FalkorDB, with the baseline in the same store
  - D-005 real-time event engine (Redis Stream, poller, git diff, webhook)
  - D-006 bi-temporal intervals
  - D-007 EC2 deploy with replay default
  - D-008 fastembed local embeddings
  - D-009 sandbox repo for push webhooks
  - D-010 drop emails (rule 6)
- `phases/phase-00-foundation.md` … `phase-11-submit.md`: one file per phase covering goal, steps (checkboxes), done-when, cut rule, and a dated **work log** (what was done, problems, fixes, commit SHAs) plus open questions.
- `RISKS.md`: a risk register (sign → fallback), updated as risks fire.
- `AI_USAGE.md`: a running disclosure of what the assistant wrote vs what you wrote or verified (feeds the README's rule-11 section).

The working rule: at the end of each phase, tick that phase file, append to its log, update `STATUS.md`, add any new `DECISIONS.md` entries, then commit as `docs: phase N complete`.

## First build step after approval
Phase 0:
1. Create `docs/project/` (all files above, phase files pre-filled from this plan).
2. Scaffold the repo.
3. Set up compose, run the smoke tests and the persistence check.
4. Start the overnight KEP fetch.
