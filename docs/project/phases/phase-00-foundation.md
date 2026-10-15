# Phase 0: Foundation (Thu 11:00 PM – 12:30 AM) ★

**Status:** In progress (waiting on GitHub repo + token) · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [x] 1. Scaffold the layout above in this folder; set up MIT, `.gitignore` (`.env`, `data/raw/`, `.venv/`), `.env.example` (`GITHUB_TOKEN`, `ANTHROPIC_API_KEY`, `FALKORDB_HOST/PORT`, `GRAPH_NAME`, `DEMO_REPLAY`, `LLM_MODEL`).
- [ ] 2. `git init`, create the public GitHub repo `decision-brain` (you, in the browser), and push the first commit.
- [x] 3. `docker-compose.yml` with `falkordb/falkordb:latest` on 6379 and 3000, `REDIS_ARGS=--appendonly yes`, and a named volume. **Verify the data path with `docker inspect`** and do a restart test.
- [x] 4. Create a Python 3.10 venv and install `falkordb fastembed anthropic fastapi uvicorn mcp pyyaml httpx pytest`.
- [x] 5. `scripts/smoke.py` prints `[['Ada','Linus']]`, plus a vector-index smoke test (create, insert `vecf32`, query).
- [ ] 6. **Start the overnight fetch** (Phase 1 step 2) before sleeping.
- **Done when:** the repo is public, both smoke tests pass, data survives `docker compose restart`, and the fetch is running.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->
- 2026-10-15 23:55 IST · Cloned kubernetes/enhancements (166 MB) and kubernetes/community (212 MB) as full clones into data/raw/repos. Blobless clones made `git log -p` take ~6 s per file because old versions were fetched lazily, so both were converted with `git fetch --refetch`; history parsing is now instant (365 status changes in sig-node, 2,244 commits to sigs.yaml). Wrote src/fetch/github_api.py (resumable pages, rate-limit sleep, ETag support), scoped to 6 SIGs (see D-011).
- 2026-10-15 23:35 IST · Scaffolded the repo, compose file, venv and smoke tests. Findings: the image stores data in `/var/lib/falkordb/data` (from `FALKORDB_DATA_PATH`), which settles the `/data` ambiguity in the docs. The image default `TIMEOUT 1000` ms would kill bulk loads, so compose sets 30000. Vector `score` from `db.idx.vector.queryNodes` is a distance (lower is closer). Graph and Redis Stream both survived `docker compose restart` with AOF on. pytest is green. Commit ce792de.

## Open questions
