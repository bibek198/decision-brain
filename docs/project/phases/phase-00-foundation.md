# Phase 0: Foundation (Thu 11:00 PM – 12:30 AM) ★

**Status:** Not started · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [ ] 1. Scaffold the layout above in this folder; set up MIT, `.gitignore` (`.env`, `data/raw/`, `.venv/`), `.env.example` (`GITHUB_TOKEN`, `ANTHROPIC_API_KEY`, `FALKORDB_HOST/PORT`, `GRAPH_NAME`, `DEMO_REPLAY`, `LLM_MODEL`).
- [ ] 2. `git init`, create the public GitHub repo `decision-brain` (you, in the browser), and push the first commit.
- [ ] 3. `docker-compose.yml` with `falkordb/falkordb:latest` on 6379 and 3000, `REDIS_ARGS=--appendonly yes`, and a named volume. **Verify the data path with `docker inspect`** and do a restart test.
- [ ] 4. Create a Python 3.10 venv and install `falkordb fastembed anthropic fastapi uvicorn mcp pyyaml httpx pytest`.
- [ ] 5. `scripts/smoke.py` prints `[['Ada','Linus']]`, plus a vector-index smoke test (create, insert `vecf32`, query).
- [ ] 6. **Start the overnight fetch** (Phase 1 step 2) before sleeping.
- **Done when:** the repo is public, both smoke tests pass, data survives `docker compose restart`, and the fetch is running.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->

## Open questions
