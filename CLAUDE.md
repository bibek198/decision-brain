# CLAUDE.md

Decision Brain is a Company Brain agent built on FalkorDB for Graph Hacks (WeMakeDevs × FalkorDB), Track 3. The "company" is the Kubernetes project. Its parts are:
- SIGs as teams
- KEPs as design docs and decisions
- tracking issues as tickets
- PR threads as chats
- OWNERS files as code ownership

The agent answers who owns or knows what, why something was decided, what it affects and what changed. Every answer shows the graph path, the Cypher it ran and its sources. The graph also updates in real time as the company changes.

**Deadline: Sun 18 Oct 2026, 11:59 PM IST. Target submission: 7:00 PM.**

## Read these first
- `docs/project/PLAN.md`: the approved plan and the source of truth for scope, schema, phases and the real-time design.
- `docs/project/STATUS.md`: which phase is active.
- The active phase file in `docs/project/phases/`: its checklist and work log.
- `docs/project/DECISIONS.md`: decisions already made (D-NNN). Don't re-argue them; add a new entry if one changes.

## Working rules (required)
1. **Record every step in `docs/project/BUILD_LOG.md`**, in order, as the work happens. Each step is a `### Step N.M · <day time IST> · <title>` section with:
   - the exact commands
   - the expected output as `# →` lines
   - the **full code of every new file** (and the change for edited files)
   - one line on why, when it isn't obvious

   The user must be able to rebuild everything by hand from this file. Code blocks can be generated from a commit with `git show <sha>:<path>` so they match the repo exactly.
2. **Keep the tracking docs current.** At the end of each phase:
   - tick the phase file's checklist and add a dated work-log entry (what, problems, fixes, commit SHA)
   - update `STATUS.md`
   - add `DECISIONS.md` entries
   - add a row to `AI_USAGE.md` (what Claude did vs what the user did)
   - update `RISKS.md` when a risk fires
3. **Commit at every meaningful step** with a clear message ending in the attribution line from the system prompt. **Do not push.** The remote uses the SSH alias `github.com-personal`, whose key isn't loaded in Claude's shell, so ask the user to run `git push`.
4. **Never write secrets** into code, docs, logs or commits. `.env` is gitignored; `.env.example` lists the variable names.
5. **Don't do the user's learning tasks for them.** The gold labels in `eval/gold_decisions.csv` are hand-written by the user and must never be generated from the graph or the LLM. They are the evaluation's independent truth.
6. Follow the plan's cut order when behind. Never cut the items PLAN.md marks as never-cut.

## Hackathon rules that constrain the code
- FalkorDB must do real work (rule 3): every agent tool is a parameterized Cypher query, and the LLM never writes Cypher.
- Use only public data (rule 6). Drop names and emails from `sigs.yaml`; keep login and company only.
- The user must be able to explain everything (rules 12–13): keep code simple and readable, and put a short docstring on each module explaining its role.
- AI use is disclosed in `docs/project/AI_USAGE.md` (rule 11).

## Commands
```bash
docker compose up -d                      # FalkorDB on 127.0.0.1:6379, Browser on http://localhost:3000
.venv/bin/python scripts/smoke.py         # → [['Ada', 'Linus']] + vector neighbours
.venv/bin/pytest -q                       # all tests (pytest.ini sets pythonpath=.)

# data pipeline (each step is safe to rerun)
.venv/bin/python -m src.fetch.git_sources         # clone/update kubernetes/enhancements + community (full clones)
.venv/bin/python -m src.fetch.github_api all      # issues/PRs/comments/reviews for 6 SIGs; resumable
.venv/bin/python -m src.fetch.owners              # OWNERS + OWNERS_ALIASES from raw.githubusercontent
.venv/bin/python -m src.normalize.keps            # → data/seed/keps.jsonl
.venv/bin/python -m src.normalize.sigs            # → data/seed/teams.jsonl, people.jsonl, code_areas.jsonl
.venv/bin/python -m src.normalize.history         # → data/seed/history_events.jsonl
```
Use `.venv/bin/python` and run modules with `-m` from the repo root.

## Layout
- `src/brain/`: config (env), db (FalkorDB connection; all queries go through it so they can be traced)
- `src/fetch/`: raw data acquisition into `data/raw/` (gitignored, never committed)
- `src/normalize/`: raw → clean JSONL in `data/seed/` (committed, small)
- `src/load/`: MERGE-based loader built from small upsert functions, **reused by the real-time handlers** (one write path)
- `src/realtime/`: `events.py` holds the shared `Event` format (deterministic `event_id`); stream, poller, git diff, webhook, worker
- `src/tools/` (Cypher tools), `src/agent/` (Claude tool loop, citation guard, replay), `src/extract/` (LLM rationale extraction)
- `web/` (FastAPI and one static page with Cytoscape), `eval/` (questions, gold labels, harness), `tests/`

## Conventions
- IDs: `KEP-<number>`; teams by folder name (`sig-node`); people by lowercase GitHub login; code areas as `org/repo:path`.
- Times are stored as ISO 8601 UTC strings plus epoch `*_ts` integers for range filters. Facts that change carry `valid_from`, `valid_to` (null means current) and `observed_at`.
- Loading always uses `MERGE` on keys and `UNWIND` batches of about 500 with query parameters, never string formatting and never `CREATE`. Every loader and handler must be idempotent: running it twice changes nothing.
- Every tool returns `{data, notes, sources[], path{nodes,edges}, cypher_trace[], as_of, stale_flags[]}`.
- LLM: Claude Haiku 5.5 (`LLM_MODEL`, default `claude-haiku-5-5`) through the official `anthropic` SDK. The budget is about $5, so use the Batch API for bulk extraction and log spend in `STATUS.md`. With no key (or `DEMO_REPLAY=1`) the app runs in replay mode.
- Embeddings: fastembed `BAAI/bge-small-en-v1.5`, 384 dimensions, cosine vector index in FalkorDB.

## Known gotchas (verified)
- The FalkorDB image keeps data in `/var/lib/falkordb/data`. Its default query timeout is 1,000 ms, so compose sets `TIMEOUT 30000`. AOF is on (`--appendonly yes`).
- `db.idx.vector.queryNodes(label, attr, k, vecf32($v)) YIELD node, score` returns `score` as a **distance** (lower means closer).
- Check Cypher features against https://docs.falkordb.com (`/cypher/known-limitations`). FalkorDB is not Neo4j. Algorithms are `algo.pageRank(label, relType)`, `algo.betweenness({...})`, `algo.labelPropagation`, `algo.SPpaths`, `algo.WCC`. Full-text uses `db.idx.fulltext.createNodeIndex` / `queryNodes`.
- `kep.yaml` is messy: there are status typos (`imlpemented`), inline comments, invalid dates and `None` lists. Always parse through `src/normalize/common.py` (`load_yaml`, `clean_status`, `handles`).
- Git history needs **full** clones (blobless clones make `git log -p` take about 6 s per file).
- GitHub API: 5,000 requests per hour with the token. The fetcher sleeps on rate limits and resumes from saved pages (`_complete` markers).
