# Build log

Every step taken to build this project, in order, with the exact commands, their important output, and the full code of every file created. Following it from top to bottom rebuilds the project by hand.

**Conventions**
- All commands run from the repo root (`decision-brain-graph-hack/`) unless a `cd` is shown. Times are IST.
- "Create file" blocks show the complete file content at the time it was created. If a file is changed later, a later step shows the change.
- Lines starting with `# →` show the output you should see.
- Prose documents in `docs/project/` (plan, decisions, risks) are not repeated here. Copy them from the repo at the commit named in the step.
- Secrets never appear here. `.env` is created by you and never committed.

**Prerequisites:** macOS or Linux, Docker Desktop, Python 3.10 or newer, git, a GitHub account.

---

## Phase 0: Foundation

### Step 0.1 · Thu 15 Oct, 23:10 · Create the project tracking docs
```bash
mkdir -p docs/project/phases
```
`docs/project/PLAN.md` is the approved plan (copy it from the repo at commit `ce792de`). Each phase file was generated from it with this one-off script, run from the repo root:
```python
# generate docs/project/phases/phase-NN-*.md from the "### Phase N:" sections of PLAN.md
import os, re
plan = open('docs/project/PLAN.md').read()
secs = re.split(r'(?m)^### (Phase \d+:.*)$', plan)
names = {0:'foundation',1:'data',2:'schema',3:'load',4:'tools',5:'extraction',
         6:'agent',7:'realtime',8:'ui',9:'eval',10:'deploy',11:'submit'}
for i in range(1, len(secs), 2):
    title = secs[i]
    body = secs[i+1].split('\n### Cut order')[0].split('\n---')[0].strip()
    n = int(re.match(r'Phase (\d+)', title).group(1))
    body = re.sub(r'(?m)^(\s*)(\d+[b]?\.)\s', r'\1- [ ] \2 ', body)   # numbered steps → checkboxes
    open(f'docs/project/phases/phase-{n:02d}-{names[n]}.md', 'w').write(
        f"# {title}\n\n**Status:** Not started · **Planned window:** see title · **Actual:** —\n\n"
        f"## Plan (from PLAN.md)\n{body}\n\n## Work log\n"
        "<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->\n\n"
        "## Open questions\n")
print(sorted(os.listdir('docs/project/phases')))
# → ['phase-00-foundation.md', 'phase-01-data.md', ... 'phase-11-submit.md']
```
Then create `docs/project/README.md`, `STATUS.md`, `DECISIONS.md`, `RISKS.md` and `AI_USAGE.md` (copy them from commit `ce792de`).

### Step 0.2 · 23:21 · Scaffold folders and config files
```bash
mkdir -p src/brain src/fetch src/normalize src/load src/extract src/tools src/agent src/realtime \
         web/static eval tests scripts data/seed data/raw docs
for d in src src/brain src/fetch src/normalize src/load src/extract src/tools src/agent src/realtime; do
  touch $d/__init__.py
done
touch data/seed/.gitkeep data/raw/.gitkeep
```
Create `.gitignore`:
```text
# secrets
.env
.env.*
!.env.example

# python
.venv/
__pycache__/
*.pyc
.pytest_cache/
*.egg-info/

# data: raw API dumps are not republished; seed is committed
data/raw/*
!data/raw/.gitkeep
data/cache/

# local
.DS_Store
.idea/
.vscode/
*.log
```
Create `.env.example`:
```bash
# GitHub fine-grained token, read-only access to public repositories
GITHUB_TOKEN=
# Anthropic API key (leave blank to run in replay mode)
ANTHROPIC_API_KEY=
LLM_MODEL=claude-haiku-5-5

FALKORDB_HOST=localhost
FALKORDB_PORT=6379
FALKORDB_PASSWORD=
GRAPH_NAME=brain_eval

# 1 = serve recorded answers only (no LLM calls)
DEMO_REPLAY=0

# Real-time engine
POLL_INTERVAL_S=60
GIT_DIFF_INTERVAL_S=300
WEBHOOK_SECRET=
SANDBOX_REPO=
```
Create `requirements.txt` (versions pinned to what was current on 15 Oct 2026):
```text
falkordb==1.8.0
redis>=5.0
fastembed==0.9.0
anthropic==1.14.0
fastapi==0.143.2
uvicorn[standard]>=0.30
mcp==2.3.0
httpx>=0.27
pyyaml>=6.0
python-dotenv>=1.0
pytest>=8.0
```
Create `LICENSE` with the standard MIT text (copyright line: `Copyright (c) 2026 Bibek`).

### Step 0.3 · 23:23 · Start Docker and check the FalkorDB image
```bash
open -a Docker                     # macOS: start Docker Desktop; wait until `docker info` works
docker pull falkordb/falkordb:latest
docker image inspect falkordb/falkordb:latest --format '{{json .Config.Env}}'
# → [... "FALKORDB_DATA_PATH=/var/lib/falkordb/data", ...
#     "FALKORDB_ARGS=MAX_QUEUED_QUERIES 25 TIMEOUT 1000 RESULTSET_SIZE 10000"]
```
Two findings: the data directory is `/var/lib/falkordb/data` (the docs mention both `/data` and this path), and the default query timeout is 1,000 ms, too short for bulk loads and PageRank.

### Step 0.4 · 23:25 · Compose file and start FalkorDB
Create `docker-compose.yml`:
```yaml
services:
  falkordb:
    image: falkordb/falkordb:latest
    container_name: decision-brain-falkordb
    ports:
      - "127.0.0.1:6379:6379"   # Redis protocol; bound to localhost only
      - "127.0.0.1:3000:3000"   # FalkorDB Browser
    environment:
      # AOF persistence so the graph and the event stream survive restarts
      REDIS_ARGS: "--appendonly yes"
      # Image default TIMEOUT is 1000 ms, too short for bulk loads and PageRank
      FALKORDB_ARGS: "MAX_QUEUED_QUERIES 25 TIMEOUT 30000 RESULTSET_SIZE 50000"
    volumes:
      # Data path verified from the image env: FALKORDB_DATA_PATH=/var/lib/falkordb/data
      - falkordb_data:/var/lib/falkordb/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 20

volumes:
  falkordb_data:
```
```bash
docker compose up -d
docker inspect -f '{{.State.Health.Status}}' decision-brain-falkordb     # → healthy
docker exec decision-brain-falkordb redis-cli CONFIG GET dir             # → /var/lib/falkordb/data
docker exec decision-brain-falkordb redis-cli CONFIG GET appendonly      # → yes
docker exec decision-brain-falkordb redis-cli GRAPH.CONFIG GET TIMEOUT   # → 30000
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000           # → 200 (FalkorDB Browser)
```

### Step 0.5 · 23:27 · Python environment
```bash
python3 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install -r requirements.txt
.venv/bin/python -c "import falkordb, fastembed, anthropic, mcp, fastapi; print('ok')"   # → ok
```

### Step 0.6 · 23:29 · Config, DB helper and smoke test
Create `src/brain/config.py`:
```python
"""Settings read from the environment (.env is loaded if present)."""
import os

from dotenv import load_dotenv

load_dotenv()

FALKORDB_HOST = os.getenv("FALKORDB_HOST", "localhost")
FALKORDB_PORT = int(os.getenv("FALKORDB_PORT", "6379"))
FALKORDB_PASSWORD = os.getenv("FALKORDB_PASSWORD") or None
GRAPH_NAME = os.getenv("GRAPH_NAME", "brain_eval")

GITHUB_TOKEN = os.getenv("GITHUB_TOKEN") or None
ANTHROPIC_API_KEY = os.getenv("ANTHROPIC_API_KEY") or None
LLM_MODEL = os.getenv("LLM_MODEL", "claude-haiku-5-5")
DEMO_REPLAY = os.getenv("DEMO_REPLAY", "0") == "1" or not ANTHROPIC_API_KEY

EMBED_MODEL = "BAAI/bge-small-en-v1.5"
EMBED_DIM = 384
```
Create `src/brain/db.py`:
```python
"""FalkorDB connection helpers. Every query goes through run() so it can be traced."""
from falkordb import FalkorDB

from src.brain import config

_db = None


def client() -> FalkorDB:
    global _db
    if _db is None:
        _db = FalkorDB(host=config.FALKORDB_HOST, port=config.FALKORDB_PORT,
                       password=config.FALKORDB_PASSWORD)
    return _db


def graph(name: str | None = None):
    return client().select_graph(name or config.GRAPH_NAME)
```
Create `scripts/smoke.py`:
```python
"""Phase 0 smoke test: basic Cypher round trip and a vector index query on a throwaway graph."""
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from src.brain.db import client  # noqa: E402

g = client().select_graph("smoke_test")
try:
    g.query("CREATE (:Person {name:$a})-[:KNOWS]->(:Person {name:$b})", {"a": "Ada", "b": "Linus"})
    print(g.query("MATCH (a:Person)-[:KNOWS]->(b:Person) RETURN a.name, b.name").result_set)

    g.query("CREATE VECTOR INDEX FOR (d:Doc) ON (d.emb) OPTIONS {dimension:3, similarityFunction:'cosine'}")
    for name, vec in [("north", [0.0, 1.0, 0.0]), ("east", [1.0, 0.0, 0.0]), ("north-east", [0.7, 0.7, 0.0])]:
        g.query("CREATE (:Doc {name:$n, emb: vecf32($v)})", {"n": name, "v": vec})
    rows = g.query(
        "CALL db.idx.vector.queryNodes('Doc', 'emb', 2, vecf32($q)) YIELD node, score "
        "RETURN node.name, score ORDER BY score",
        {"q": [0.1, 0.9, 0.0]},
    ).result_set
    print("vector:", rows)
finally:
    g.delete()
```
```bash
.venv/bin/python scripts/smoke.py
# → [['Ada', 'Linus']]
# → vector: [['north', 0.0061...], ['north-east', 0.219...]]
```
Note: the vector `score` is a **distance** (lower means closer), not a similarity.

### Step 0.7 · 23:31 · Persistence test (graph and Redis Stream survive a restart)
```bash
docker exec decision-brain-falkordb redis-cli GRAPH.QUERY persist_test "CREATE (:Probe {v:42})"
docker exec decision-brain-falkordb redis-cli XADD probe:stream '*' k v
docker compose restart falkordb
# wait until healthy, then:
docker exec decision-brain-falkordb redis-cli GRAPH.QUERY persist_test "MATCH (p:Probe) RETURN p.v"   # → 42
docker exec decision-brain-falkordb redis-cli XLEN probe:stream                                        # → 1
docker exec decision-brain-falkordb redis-cli GRAPH.DELETE persist_test
docker exec decision-brain-falkordb redis-cli DEL probe:stream
```

### Step 0.8 · 23:33 · README skeleton, pytest, first commit
Create `README.md`:
```markdown
# Decision Brain

A Company Brain agent on FalkorDB: it connects the people, teams, design documents, tickets, chats and decisions of the Kubernetes project in one graph. It answers who owns or knows what, why something was decided, what a change affects, and what changed, and every answer shows the graph path and its sources. It keeps up as the project changes, in real time.

> Graph Hacks 2026 (WeMakeDevs × FalkorDB). Track 3: Company Brain. Work in progress; see [docs/project/STATUS.md](docs/project/STATUS.md).

## Overview
## Quick start
## Architecture
## Graph data model
## Cypher queries and graph algorithms
## Real-time updates
## Data
## How the LLM is used and checked
## Evaluation
## Limitations and future work
## AI usage disclosure
See [docs/project/AI_USAGE.md](docs/project/AI_USAGE.md).
## Track
Track 3: Company Brain.
## License
MIT. Data: Kubernetes enhancements and community repositories (Apache-2.0).
```
Create `tests/test_smoke.py`:
```python
from src.brain.db import client


def test_cypher_roundtrip():
    g = client().select_graph("pytest_smoke")
    try:
        g.query("CREATE (:Person {name:'Ada'})-[:KNOWS]->(:Person {name:'Linus'})")
        rows = g.query("MATCH (a:Person)-[:KNOWS]->(b:Person) RETURN a.name, b.name").result_set
        assert rows == [["Ada", "Linus"]]
    finally:
        g.delete()
```
Create `pytest.ini`:
```ini
[pytest]
testpaths = tests
pythonpath = .
```
```bash
.venv/bin/pytest -q                  # → 1 passed
git init && git branch -M main
git add -A
git status --short                   # check that .env and .venv/ are NOT listed
git commit -m "Phase 0: scaffold, FalkorDB compose, smoke tests, project tracking docs"
# → commit ce792de
```

## Phase 1: Data acquisition (started early)

### Step 1.1 · 23:40 · Clone the git sources
Create `src/fetch/git_sources.py`:
```python
"""Clone or update the git sources (no API rate limit).

- kubernetes/enhancements: every keps/**/kep.yaml plus its full history (status changes over time).
- kubernetes/community: sigs.yaml plus its history (leadership changes over time).

Blobless clones (--filter=blob:none) keep the download small; git fetches file
contents lazily, only for the paths we read.
"""
import subprocess
import sys
from pathlib import Path

ROOT = Path(__file__).resolve().parents[2]
REPOS_DIR = ROOT / "data" / "raw" / "repos"
REPOS = {
    "enhancements": "https://github.com/kubernetes/enhancements.git",
    "community": "https://github.com/kubernetes/community.git",
}


def run(cmd: list[str], cwd: Path | None = None) -> str:
    return subprocess.run(cmd, cwd=cwd, check=True, capture_output=True, text=True).stdout.strip()


def sync(name: str) -> str:
    """Clone if missing, otherwise fetch and fast-forward. Returns the HEAD sha."""
    path = REPOS_DIR / name
    if not path.exists():
        REPOS_DIR.mkdir(parents=True, exist_ok=True)
        run(["git", "clone", "--filter=blob:none", "--quiet", REPOS[name], str(path)])
    else:
        run(["git", "fetch", "--quiet", "origin"], cwd=path)
        run(["git", "reset", "--quiet", "--hard", "origin/HEAD"], cwd=path)
    return run(["git", "rev-parse", "HEAD"], cwd=path)


if __name__ == "__main__":
    for name in sys.argv[1:] or REPOS:
        print(name, sync(name))
```
```bash
.venv/bin/python -m src.fetch.git_sources
# → enhancements 6ebe0a9...
# → community   53c338e...
```

### Step 1.2 · 23:45 · Make the clones full (history parsing was too slow)
With a blobless clone, `git log -p` downloads each old file version on demand, which took about 6 s per file. Converting to full clones fixes it:
```bash
for r in enhancements community; do
  (cd data/raw/repos/$r && git config --unset remote.origin.partialclonefilter;
   git config remote.origin.promisor false; git fetch --refetch --quiet origin)
done
# → enhancements 166M, community 212M
cd data/raw/repos/enhancements
git log -p --format='COMMIT %H %cI' -- 'keps/sig-node/*/kep.yaml' | grep -cE '^[+-]status:'   # → 365, in 0.2 s
cd ../community && git log --format=%H -- sigs.yaml | wc -l                                    # → 2244
cd ../../../..
```
Example of what the history shows for KEP-753 (sidecar containers):
```bash
git -C data/raw/repos/enhancements log -p --format='COMMIT %H %cI' \
  -- keps/sig-node/753-sidecar-containers/kep.yaml | grep -E '^(COMMIT|[+-]status:|[+-]  (alpha|beta|stable):)'
# → COMMIT 4a64601... 2025-02-03   -status: implementable   +status: implemented
# → COMMIT f494a1b... 2023-06-09   -alpha: "v1.27" ... +alpha: "v1.28" ...
```

### Step 1.3 · 23:50 · Choose the SIGs to scope the API fetch
```bash
.venv/bin/python - <<'EOF'
import yaml, glob, collections
c, odd, sup, bad = collections.Counter(), collections.Counter(), collections.Counter(), 0
for f in glob.glob('data/raw/repos/enhancements/keps/sig-*/*/kep.yaml'):
    sig = f.split('/')[5]
    try: d = yaml.safe_load(open(f))
    except Exception: bad += 1; continue
    c[sig] += 1
    st = str(d.get('status', '')).split()[0].strip('"').lower() if d.get('status') else ''
    if st not in ('implemented', 'implementable', 'provisional'): odd[sig] += 1
    if d.get('replaces') or d.get('superseded-by'): sup[sig] += 1
print('parse failures', bad)
for s, n in c.most_common(): print(f"{s:28} keps={n:3} odd_status={odd[s]:2} supersession={sup[s]:2}")
EOF
# → parse failures 2
# → sig-node keps=139 odd=4 sup=13 · sig-api-machinery 84/5/7 · sig-storage 69/2/5
# → sig-network 62/3/1 · sig-scheduling 59/0/3 · sig-auth 36/3/5 · sig-cli 35/2/6 ...
```
Chosen: node, api-machinery, storage, scheduling, auth, cli (decision D-011).

### Step 1.4 · 23:55 · GitHub API fetcher
Create `src/fetch/github_api.py`:
```python
"""GitHub REST fetcher for the scoped SIGs.

Every response page is saved under data/raw/api/<key>/, so a crashed run resumes
where it stopped. The same GitHub client (ETag + rate-limit handling) is reused
by the real-time poller.

Usage:
    python -m src.fetch.github_api issues      # issues + PRs labelled with each scoped SIG
    python -m src.fetch.github_api comments    # conversation comments for every item with comments
    python -m src.fetch.github_api reviews     # reviews + inline review comments for PRs
    python -m src.fetch.github_api all
"""
import json
import sys
import time
from pathlib import Path

import httpx

from src.brain import config

ROOT = Path(__file__).resolve().parents[2]
API_DIR = ROOT / "data" / "raw" / "api"
REPO = "kubernetes/enhancements"
SCOPED_SIGS = ["node", "api-machinery", "storage", "scheduling", "auth", "cli"]


class GitHub:
    """Small GitHub REST client: auth, pagination, rate-limit sleep, ETag support."""

    def __init__(self, token: str | None = config.GITHUB_TOKEN):
        headers = {"Accept": "application/vnd.github+json", "X-GitHub-Api-Version": "2022-11-28"}
        if token:
            headers["Authorization"] = f"Bearer {token}"
        self.http = httpx.Client(base_url="https://api.github.com", headers=headers, timeout=30)

    def get(self, url: str, params: dict | None = None, etag: str | None = None) -> httpx.Response:
        headers = {"If-None-Match": etag} if etag else {}
        for attempt in range(5):
            r = self.http.get(url, params=params, headers=headers)
            if r.status_code in (403, 429) and r.headers.get("X-RateLimit-Remaining") == "0":
                wait = int(r.headers.get("X-RateLimit-Reset", time.time() + 60)) - time.time() + 2
                print(f"  rate limited, sleeping {wait:.0f}s", flush=True)
                time.sleep(max(wait, 1))
                continue
            if r.status_code >= 500 or r.status_code in (403, 429):
                time.sleep(2 ** attempt)
                continue
            return r
        r.raise_for_status()
        return r

    def pages(self, url: str, params: dict | None = None):
        """Yield (page_number, items) following Link rel="next"."""
        page, next_url, next_params = 1, url, dict(params or {}, per_page=100)
        while next_url:
            r = self.get(next_url, params=next_params)
            r.raise_for_status()
            yield page, r.json()
            next_url = r.links.get("next", {}).get("url")
            next_params, page = None, page + 1


def fetch_all_pages(gh: GitHub, key: str, url: str, params: dict | None = None) -> list:
    """Fetch every page into API_DIR/key/, skipping the work if a complete marker exists."""
    out = API_DIR / key
    done = out / "_complete"
    if done.exists():
        return load_pages(key)
    out.mkdir(parents=True, exist_ok=True)
    for page, items in gh.pages(url, params):
        (out / f"page_{page:04d}.json").write_text(json.dumps(items))
    done.write_text(time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()))
    return load_pages(key)


def load_pages(key: str) -> list:
    items = []
    for f in sorted((API_DIR / key).glob("page_*.json")):
        items.extend(json.loads(f.read_text()))
    return items


def fetch_issues(gh: GitHub) -> list:
    """Issues and PRs labelled sig/<name>. KEP tracking issues share the KEP number."""
    items = {}
    for sig in SCOPED_SIGS:
        rows = fetch_all_pages(gh, f"issues/sig-{sig}", f"/repos/{REPO}/issues",
                               {"state": "all", "labels": f"sig/{sig}"})
        print(f"sig/{sig}: {len(rows)} issues+PRs", flush=True)
        for it in rows:
            items[it["number"]] = it
    return list(items.values())


def fetch_comments(gh: GitHub, items: list) -> None:
    todo = [it for it in items if it.get("comments", 0) > 0]
    print(f"comments: {len(todo)} threads", flush=True)
    for i, it in enumerate(todo, 1):
        fetch_all_pages(gh, f"comments/{it['number']}", f"/repos/{REPO}/issues/{it['number']}/comments")
        if i % 50 == 0:
            print(f"  {i}/{len(todo)}", flush=True)


def fetch_reviews(gh: GitHub, items: list) -> None:
    prs = [it for it in items if "pull_request" in it]
    print(f"reviews: {len(prs)} PRs", flush=True)
    for i, it in enumerate(prs, 1):
        n = it["number"]
        fetch_all_pages(gh, f"reviews/{n}", f"/repos/{REPO}/pulls/{n}/reviews")
        fetch_all_pages(gh, f"review_comments/{n}", f"/repos/{REPO}/pulls/{n}/comments")
        if i % 50 == 0:
            print(f"  {i}/{len(prs)}", flush=True)


def main(steps: list[str]) -> None:
    if not config.GITHUB_TOKEN:
        sys.exit("GITHUB_TOKEN is not set. Copy .env.example to .env and add a read-only token.")
    gh = GitHub()
    items = fetch_issues(gh)
    if "comments" in steps or "all" in steps:
        fetch_comments(gh, items)
    if "reviews" in steps or "all" in steps:
        fetch_reviews(gh, items)
    r = gh.get("/rate_limit").json()["rate"]
    print(f"done. rate limit remaining {r['remaining']}/{r['limit']}")


if __name__ == "__main__":
    main(sys.argv[1:] or ["all"])
```
Update the phase 0 work log and add D-011 to `DECISIONS.md` (see commit `ac67226`), then:
```bash
git add -A && git commit -m "Phase 1: git source fetcher, scoped GitHub API fetcher, phase 0 log"
# → commit ac67226
```

### Step 1.5 · Thu 15 Oct, late · Your part: GitHub repo and secrets
1. On github.com, create a **public, empty** repo `decision-brain` (no README or license).
2. Push:
   ```bash
   git remote add origin <your repo URL>      # here: git@github.com-personal:bibek198/decision-brain.git
   git push -u origin main
   ```
3. Create a fine-grained GitHub token (Settings → Developer settings → Fine-grained tokens) with **public repositories, read-only** access.
4. Create the secrets file:
   ```bash
   cp .env.example .env
   # edit .env: set GITHUB_TOKEN=... and ANTHROPIC_API_KEY=...
   ```

### Step 1.6 · Fri 16 Oct, 00:15 · Verify and start the overnight fetch
```bash
git status -sb | head -1          # → ## main...origin/main   (pushed, in sync)
git ls-files .env                 # → (nothing: .env is not tracked)
.venv/bin/python -c "from src.brain import config; import httpx; \
print(httpx.get('https://api.github.com/rate_limit', headers={'Authorization': f'Bearer {config.GITHUB_TOKEN}'}).json()['rate'])"
# → {'limit': 5000, 'remaining': 5000, ...}
mkdir -p data/raw/logs
.venv/bin/python -u -m src.fetch.github_api all 2>&1 | tee data/raw/logs/fetch_api.log
```
This runs for a while (about 1,500–2,500 API calls). If it stops, rerun the same command: finished pages are skipped.
