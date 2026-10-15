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

### Step 1.7 · Fri 16 Oct, 00:50 · Check the fetch; survey the raw data
```bash
tail -6 data/raw/logs/fetch_api.log        # → comments done (3725 threads), reviews in progress
.venv/bin/python - <<'EOF'
import yaml, glob, collections
keys, types = collections.Counter(), collections.defaultdict(collections.Counter)
for f in glob.glob('data/raw/repos/enhancements/keps/sig-*/*/kep.yaml'):
    try: d = yaml.safe_load(open(f)) or {}
    except Exception as e: print('bad', f, str(e)[:60]); continue
    for k, v in d.items(): keys[k] += 1; types[k][type(v).__name__] += 1
for k, n in keys.most_common(25): print(f"{k:24}{n:4} {dict(types[k])}")
EOF
# → 2 files fail with "month must be in 1..12" (YAML turns dates into date objects)
# → many fields are sometimes None (reviewers, participating-sigs, replaces, ...)
```
Decision: parse YAML with dates kept as strings, and normalize `None` to empty values.

### Step 1.8 · 00:55 · Shared event format and parsing helpers
The event format is used by the history backfill now and by the real-time engine later. `event_id` is a hash of source, kind, entity, time and payload, so the same change seen twice is a no-op.

Create `src/realtime/events.py`:
```python
"""Change events: the one format every source produces and every handler consumes.

The git-history backfill, the GitHub poller, the git diff job and the sandbox
webhook all emit these. event_id is deterministic, so the same change seen twice
(a replay, or a poller and a webhook overlapping) has the same id and is a no-op.
"""
import hashlib
import json
from dataclasses import asdict, dataclass, field
from datetime import datetime, timezone


@dataclass
class Event:
    source: str          # git | github | webhook | inject
    kind: str            # kep_created, kep_status_changed, sig_lead_added, comment_added, ...
    entity_key: str      # e.g. "KEP-753", "sig-node", "comment:123456"
    occurred_at: str     # ISO 8601 UTC: when it happened in the company
    payload: dict = field(default_factory=dict)
    observed_at: str = ""  # ISO 8601 UTC: when the brain learned about it
    event_id: str = ""

    def __post_init__(self):
        if not self.observed_at:
            self.observed_at = now_iso()
        if not self.event_id:
            self.event_id = make_event_id(self.source, self.kind, self.entity_key,
                                          self.occurred_at, self.payload)

    def to_json(self) -> str:
        return json.dumps(asdict(self), sort_keys=True, default=str)

    @classmethod
    def from_dict(cls, d: dict) -> "Event":
        return cls(**d)


def make_event_id(source: str, kind: str, entity_key: str, occurred_at: str, payload: dict) -> str:
    raw = json.dumps([source, kind, entity_key, occurred_at, payload], sort_keys=True, default=str)
    return hashlib.sha256(raw.encode()).hexdigest()[:20]


def now_iso() -> str:
    return datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")


def to_utc_iso(value: str) -> str:
    """Normalize any ISO timestamp (with offset or Z) to UTC 'YYYY-MM-DDTHH:MM:SSZ'."""
    dt = datetime.fromisoformat(value.replace("Z", "+00:00"))
    if dt.tzinfo is None:
        dt = dt.replace(tzinfo=timezone.utc)
    return dt.astimezone(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")


def iso_to_ts(value: str) -> int:
    """ISO timestamp to epoch seconds (for fast range filters in Cypher)."""
    return int(datetime.fromisoformat(value.replace("Z", "+00:00")).timestamp())
```
Create `src/normalize/common.py` (YAML loader that keeps dates as strings; handle, status and KEP-reference cleanup):
```python
"""Shared parsing helpers: a YAML loader that keeps dates as strings, handle and status cleanup."""
import re

import yaml


class _StrDateLoader(yaml.SafeLoader):
    """SafeLoader that leaves timestamps as strings (two kep.yaml files have invalid dates)."""


_StrDateLoader.yaml_implicit_resolvers = {
    ch: [(tag, rx) for tag, rx in resolvers if tag != "tag:yaml.org,2002:timestamp"]
    for ch, resolvers in yaml.SafeLoader.yaml_implicit_resolvers.items()
}

try:  # the C loader is ~10x faster; used for the large sigs.yaml history
    class _CStrDateLoader(yaml.CSafeLoader):
        pass
    _CStrDateLoader.yaml_implicit_resolvers = _StrDateLoader.yaml_implicit_resolvers
except AttributeError:  # pragma: no cover
    _CStrDateLoader = _StrDateLoader


def load_yaml(text: str):
    return yaml.load(text, Loader=_CStrDateLoader)


BOTS = {"k8s-ci-robot", "k8s-triage-robot", "fejta-bot", "k8s-merge-robot", "dependabot",
        "github-actions", "codecov", "linux-foundation-easycla", "k8s-release-robot"}
_HANDLE = re.compile(r"@?([A-Za-z0-9](?:[A-Za-z0-9-]{0,38}))")


def clean_handle(value) -> str | None:
    """'@SergeyKanzhelev' / '"@foo"  # role' / 'Name (@foo)' -> 'sergeykanzhelev'. None if no handle."""
    if value is None:
        return None
    s = str(value).split("#")[0].strip().strip('"').strip("'")
    m = re.search(r"@([A-Za-z0-9-]+)", s) or _HANDLE.fullmatch(s.strip())
    if not m:
        return None
    login = m.group(1).lower().strip("-")
    if not login or is_bot(login):
        return None
    return login


def is_bot(login: str) -> bool:
    login = login.lower()
    return login in BOTS or login.endswith("[bot]") or login.endswith("-bot") or login.endswith("-robot")


def handles(values) -> list[str]:
    out = []
    for v in values or []:
        h = clean_handle(v)
        if h and h not in out:
            out.append(h)
    return out


# KEP status values seen in the wild, including typos and inline comments
_STATUS_MAP = {
    "provisional": "provisional", "implementable": "implementable", "implementeable": "implementable",
    "implemented": "implemented", "imlpemented": "implemented", "deferred": "deferred",
    "rejected": "rejected", "withdrawn": "withdrawn", "replaced": "replaced",
    "superseded": "replaced", "removed": "withdrawn",
}
STATUSES = sorted(set(_STATUS_MAP.values()))


def clean_status(value) -> str | None:
    if not value:
        return None
    s = str(value).split("#")[0].strip().strip('"').lower()
    s = s.split("(")[0].strip()
    if "|" in s:  # template placeholder left in place
        return None
    return _STATUS_MAP.get(s)


_KEP_PATH = re.compile(r"keps/(?:[\w-]+/)?(\d+)-")


def kep_id(number) -> str:
    return f"KEP-{int(number)}"


def kep_id_from_ref(ref) -> str | None:
    """'/keps/sig-node/4438-sidecar-restart-termination' or 'KEP-4438' or 4438 -> 'KEP-4438'."""
    if ref is None:
        return None
    s = str(ref)
    m = _KEP_PATH.search(s) or re.search(r"(?:KEP-?|#)(\d+)", s, re.I) or re.fullmatch(r"\s*(\d+)\s*", s)
    return kep_id(m.group(1)) if m else None
```

### Step 1.9 · 01:00 · KEP parser → `data/seed/keps.jsonl`
First version linked only 13 KEP→KEP supersessions. Inspecting the raw `replaces` / `superseded-by` values showed most point to old design docs (design-proposals archive, community PRs, Google Docs), and some use pre-numbering date-style names like `20191004-out-of-tree-credential-providers.md`. The final parser keeps the doc links as `replaces_docs` / `see_also_docs` and resolves date-style names to current KEPs by name similarity (difflib, cutoff 0.85).

Create `src/normalize/keps.py`:
```python
"""kep.yaml files -> data/seed/keps.jsonl (one clean record per KEP).

Usage: python -m src.normalize.keps
"""
import difflib
import json
import re
from pathlib import Path

from src.normalize.common import clean_status, handles, kep_id, kep_id_from_ref, load_yaml

ROOT = Path(__file__).resolve().parents[2]
ENH = ROOT / "data" / "raw" / "repos" / "enhancements"
SEED = ROOT / "data" / "seed"
GH = "https://github.com/kubernetes/enhancements"


def _str(v) -> str | None:
    if v is None:
        return None
    s = str(v).split("#")[0].strip().strip('"')
    return s or None


def _list(v) -> list:
    if v is None:
        return []
    return v if isinstance(v, list) else [v]


def _sig(v) -> str | None:
    s = _str(v)
    return s.lower() if s else None


_NOT_A_REF = {"n/a", "na", "none", "nil", "-", ""}
_DATE_STYLE = re.compile(r"(?:^|/)(?:\d{8})-([a-z0-9-]+?)(?:\.md)?/?$")


def _ref_values(v) -> list[str]:
    out = []
    for x in _list(v):
        s = str(x).strip().strip("()").strip()
        if s.lower() not in _NOT_A_REF:
            out.append(s)
    return out


def _doc_url(s: str) -> str | None:
    """Non-KEP references (design-proposals archive, community PRs, Google Docs) as URLs."""
    if s.startswith("http"):
        return s
    if "design-proposals" in s:
        tail = s.split("design-proposals", 1)[1].lstrip("/").removeprefix("blob/main/").removeprefix("blob/master/")
        return f"https://github.com/kubernetes/design-proposals-archive/blob/main/{tail}"
    return None


def parse_kep(data: dict, rel_path: str) -> dict | None:
    """One kep.yaml dict -> normalized record. rel_path is like keps/sig-node/753-sidecar-containers."""
    if not data or not data.get("kep-number"):
        return None
    try:
        number = int(str(data["kep-number"]).strip())
    except ValueError:
        return None
    milestones = {k: _str(v) for k, v in (data.get("milestone") or {}).items() if _str(v)}
    gates = []
    for g in _list(data.get("feature-gates")):
        if isinstance(g, dict) and g.get("name"):
            gates.append({"name": str(g["name"]).strip(),
                          "components": [str(c).strip() for c in _list(g.get("components")) if c]})
    me = kep_id(number)
    def refs(key):
        """KEP references stay as ids (date-style ones resolved later); others become doc URLs."""
        ids, docs = [], []
        for s in _ref_values(data.get(key)):
            m = _DATE_STYLE.search(s.lower())
            rid = f"slug:{m.group(1)}" if m else kep_id_from_ref(s)
            if rid and rid != me and rid not in ids:
                ids.append(rid)
            elif not rid and _doc_url(s):
                docs.append(_doc_url(s))
        return ids, docs

    see_also, see_also_docs = refs("see-also")
    replaces, replaces_docs = refs("replaces")
    superseded_by, _ = refs("superseded-by")
    return {
        "id": me,
        "number": number,
        "title": _str(data.get("title")),
        "owning_sig": _sig(data.get("owning-sig")),
        "participating_sigs": [s for s in (_sig(x) for x in _list(data.get("participating-sigs"))) if s],
        "status": clean_status(data.get("status")),
        "status_raw": _str(data.get("status")),
        "stage": _str(data.get("stage")),
        "latest_milestone": _str(data.get("latest-milestone")),
        "milestones": milestones,
        "authors": handles(data.get("authors")),
        "reviewers": handles(data.get("reviewers")),
        "approvers": handles(data.get("approvers")),
        "editor": (handles([data.get("editor")]) or [None])[0],
        "creation_date": _str(data.get("creation-date")),
        "last_updated": _str(data.get("last-updated")),
        "see_also": see_also,
        "see_also_docs": see_also_docs,
        "replaces": replaces,
        "replaces_docs": replaces_docs,
        "superseded_by": superseded_by,
        "feature_gates": gates,
        "path": rel_path,
        "url": f"{GH}/tree/master/{rel_path}",
        "readme_url": f"{GH}/blob/master/{rel_path}/README.md",
        "issue_url": f"{GH}/issues/{number}",
    }


def resolve_refs(records: dict) -> int:
    """Map 'slug:<name>' refs (pre-numbering KEP files) to current KEP ids by name similarity.
    Unresolvable or unknown refs are dropped. Returns the number of dropped refs."""
    slugs = {r["path"].rsplit("/", 1)[-1].split("-", 1)[-1]: r["id"] for r in records.values()}
    dropped = 0
    for r in records.values():
        for key in ("see_also", "replaces", "superseded_by"):
            out = []
            for ref in r[key]:
                if ref.startswith("slug:"):
                    m = difflib.get_close_matches(ref[5:], slugs, n=1, cutoff=0.85)
                    ref = slugs[m[0]] if m else None
                if ref and ref in records and ref != r["id"] and ref not in out:
                    out.append(ref)
                else:
                    dropped += 1
            r[key] = out
    return dropped


def main() -> None:
    SEED.mkdir(parents=True, exist_ok=True)
    records, skipped = {}, []
    for f in sorted((ENH / "keps").glob("*/*/kep.yaml")):
        rel = str(f.parent.relative_to(ENH))
        try:
            rec = parse_kep(load_yaml(f.read_text()), rel)
        except Exception as e:  # noqa: BLE001 - log and keep going
            skipped.append((rel, str(e)[:80]))
            continue
        if rec is None:
            skipped.append((rel, "no kep-number"))
            continue
        if rec["id"] in records:
            skipped.append((rel, f"duplicate of {records[rec['id']]['path']}"))
            continue
        records[rec["id"]] = rec
    dropped = resolve_refs(records)
    out = SEED / "keps.jsonl"
    with out.open("w") as fh:
        for rec in sorted(records.values(), key=lambda r: r["number"]):
            fh.write(json.dumps(rec, sort_keys=True) + "\n")
    print(f"{len(records)} KEPs -> {out.relative_to(ROOT)}; skipped {len(skipped)}; dropped {dropped} unresolvable refs")
    for s in skipped:
        print("  skipped", *s)


if __name__ == "__main__":
    main()
```
```bash
.venv/bin/python -m src.normalize.keps
# → 644 KEPs -> data/seed/keps.jsonl; skipped 4; dropped 29 unresolvable refs
# status counts → implemented 293, implementable 277, provisional 51, withdrawn 12, replaced 6, rejected 3, deferred 1
# links → 9 KEP→KEP supersessions, 17 replaced design docs, 324 see-also KEP links, 94 see-also docs; 699 people
```

### Step 1.10 · 01:10 · OWNERS files (who owns which code)
`sigs.yaml` subprojects point to OWNERS files on raw.githubusercontent.com (no API quota).

Create `src/fetch/owners.py`:
```python
"""Download the OWNERS files that sigs.yaml subprojects point to (raw.githubusercontent.com, no API quota),
plus each repo's OWNERS_ALIASES so alias names can be expanded to people.

Saved as data/raw/owners/<org>/<repo>/<path>/OWNERS and data/raw/owners/<org>/<repo>/OWNERS_ALIASES.
Usage: python -m src.fetch.owners
"""
from concurrent.futures import ThreadPoolExecutor
from pathlib import Path

import httpx

from src.normalize.common import load_yaml

ROOT = Path(__file__).resolve().parents[2]
SIGS = ROOT / "data" / "raw" / "repos" / "community" / "sigs.yaml"
OUT = ROOT / "data" / "raw" / "owners"


def owners_urls() -> list[str]:
    data = load_yaml(SIGS.read_text())
    urls = []
    for kind in ("sigs", "workinggroups", "committees"):
        for group in data.get(kind) or []:
            for sp in group.get("subprojects") or []:
                urls.extend(u for u in sp.get("owners") or [] if u.startswith("https://raw.githubusercontent.com/"))
    return sorted(set(urls))


def local_path(url: str) -> Path:
    # https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path...>/OWNERS
    org, repo, _branch, *rest = url.split("/")[3:]
    return OUT / org / repo / "/".join(rest)


def alias_url(url: str) -> str:
    org, repo, branch = url.split("/")[3:6]
    return f"https://raw.githubusercontent.com/{org}/{repo}/{branch}/OWNERS_ALIASES"


def download(client: httpx.Client, url: str, dest: Path) -> str:
    if dest.exists():
        return "cached"
    r = client.get(url)
    if r.status_code != 200:
        return f"http {r.status_code}"
    dest.parent.mkdir(parents=True, exist_ok=True)
    dest.write_text(r.text)
    return "ok"


def main() -> None:
    urls = owners_urls()
    jobs = [(u, local_path(u)) for u in urls]
    jobs += [(a, OUT / a.split("/")[3] / a.split("/")[4] / "OWNERS_ALIASES") for a in sorted({alias_url(u) for u in urls})]
    with httpx.Client(timeout=20, follow_redirects=True) as client, ThreadPoolExecutor(16) as pool:
        results = list(pool.map(lambda j: (j[0], download(client, *j)), jobs))
    bad = [(u, s) for u, s in results if s not in ("ok", "cached")]
    print(f"{len(jobs)} files ({len(urls)} OWNERS + aliases); failed {len(bad)}")
    for u, s in bad[:15]:
        print("  ", s, u)


if __name__ == "__main__":
    main()
```
```bash
.venv/bin/python -m src.fetch.owners
# → 848 files (518 OWNERS + aliases); failed 237   (mostly repos without OWNERS_ALIASES, plus stale paths)
find data/raw/owners -name OWNERS | wc -l            # → 502
find data/raw/owners -name OWNERS_ALIASES | wc -l    # → 107
```

### Step 1.11 · 01:15 · SIG registry + OWNERS → teams, people, code areas
OWNERS files often name aliases (`sig-node-approvers`), expanded through each repo's `OWNERS_ALIASES`. `emeritus_approvers` are kept as former owners. Names and emails from `sigs.yaml` are dropped (rule 6); only login and company are kept. One alias file was not a YAML mapping and crashed the first run, hence the `isinstance` guard.

Create `src/normalize/sigs.py`:
```python
"""sigs.yaml + OWNERS files -> data/seed/teams.jsonl, people.jsonl, code_areas.jsonl.

- teams.jsonl: one record per SIG / working group / committee, with leads and subprojects.
- people.jsonl: login + company for SIG leaders (names and emails are dropped, rule 6).
- code_areas.jsonl: one record per OWNERS file: the code path a subproject owns and who approves
  and reviews it (aliases expanded; emeritus approvers kept as former owners).

Usage: python -m src.normalize.sigs
"""
import json
from pathlib import Path

from src.fetch.owners import OUT as OWNERS_DIR
from src.fetch.owners import local_path
from src.normalize.common import clean_handle, handles, load_yaml

ROOT = Path(__file__).resolve().parents[2]
SIGS = ROOT / "data" / "raw" / "repos" / "community" / "sigs.yaml"
SEED = ROOT / "data" / "seed"
KINDS = {"sigs": "sig", "workinggroups": "wg", "committees": "committee"}


def _aliases(org: str, repo: str) -> dict[str, list[str]]:
    f = OWNERS_DIR / org / repo / "OWNERS_ALIASES"
    if not f.exists():
        return {}
    try:
        data = load_yaml(f.read_text()) or {}
    except Exception:  # noqa: BLE001 - malformed alias file: treat as no aliases
        return {}
    aliases = data.get("aliases") if isinstance(data, dict) else None
    if not isinstance(aliases, dict):
        return {}
    return {str(k).lower(): handles(v) for k, v in aliases.items()}


def _expand(values, aliases: dict) -> list[str]:
    out = []
    for v in values or []:
        key = str(v).split("#")[0].strip().lower()
        for login in aliases.get(key) or handles([v]):
            if login not in out:
                out.append(login)
    return out


def parse_owners(url: str, alias_cache: dict) -> dict | None:
    f = local_path(url)
    if not f.exists():
        return None
    org, repo, branch, *rest = url.split("/")[3:]
    path = "/".join(rest[:-1]) or "."
    if (org, repo) not in alias_cache:
        alias_cache[(org, repo)] = _aliases(org, repo)
    aliases = alias_cache[(org, repo)]
    try:
        data = load_yaml(f.read_text()) or {}
    except Exception:  # noqa: BLE001 - a malformed OWNERS file is skipped, not fatal
        return None
    if not isinstance(data, dict):
        return None
    return {
        "id": f"{org}/{repo}:{path}",
        "repo": f"{org}/{repo}",
        "path": path,
        "url": f"https://github.com/{org}/{repo}/tree/{branch}/{path}",
        "owners_url": url,
        "approvers": _expand(data.get("approvers"), aliases),
        "reviewers": _expand(data.get("reviewers"), aliases),
        "emeritus_approvers": _expand(data.get("emeritus_approvers"), aliases),
        "labels": [str(x) for x in data.get("labels") or []],
    }


def main() -> None:
    SEED.mkdir(parents=True, exist_ok=True)
    data = load_yaml(SIGS.read_text())
    teams, people, areas, alias_cache = [], {}, {}, {}
    for key, kind in KINDS.items():
        for g in data.get(key) or []:
            leads = []
            for role in ("chairs", "tech_leads", "emeritus_leads"):
                for p in (g.get("leadership") or {}).get(role) or []:
                    login = clean_handle(p.get("github"))
                    if not login:
                        continue
                    leads.append({"login": login, "role": role.rstrip("s")})
                    if p.get("company"):
                        people.setdefault(login, {"login": login, "company": str(p["company"]).strip()})
            subprojects = []
            for sp in g.get("subprojects") or []:
                area_ids = []
                for url in sp.get("owners") or []:
                    area = parse_owners(url, alias_cache)
                    if area:
                        areas.setdefault(area["id"], area)
                        area_ids.append(area["id"])
                subprojects.append({"name": sp["name"], "description": (sp.get("description") or "").strip(),
                                    "code_areas": area_ids})
            teams.append({
                "id": g["dir"], "kind": kind, "name": g["name"],
                "mission": " ".join(str(g.get("mission_statement") or "").split()),
                "label": g.get("label"),
                "stakeholder_sigs": [s.lower() for s in g.get("stakeholder_sigs") or []],
                "leads": leads, "subprojects": subprojects,
                "url": f"https://github.com/kubernetes/community/tree/master/{g['dir']}",
            })
    for name, rows in (("teams", teams), ("people", list(people.values())), ("code_areas", list(areas.values()))):
        with (SEED / f"{name}.jsonl").open("w") as fh:
            for r in rows:
                fh.write(json.dumps(r, sort_keys=True) + "\n")
    n_sp = sum(len(t["subprojects"]) for t in teams)
    print(f"{len(teams)} teams, {n_sp} subprojects, {len(areas)} code areas, {len(people)} leaders with company")


if __name__ == "__main__":
    main()
```
```bash
.venv/bin/python -m src.normalize.sigs
# → 37 teams, 241 subprojects, 502 code areas, 132 leaders with company
# kubernetes/kubernetes:pkg/kubelet → 9 approvers (alias expanded), 2 emeritus approvers
```

### Step 1.12 · 01:25 · Git history → dated change events
```bash
cd data/raw/repos/enhancements
git log --first-parent -m --format='C %H %cI %s' --name-status -- 'keps/*/*/kep.yaml' \
  | awk '/^C /{c++} /^[AMDR]/{f++} END{print "commits",c,"file-changes",f}'     # → commits 1970 file-changes 2674
cd ../community && git log --first-parent --format=%H -- sigs.yaml | wc -l     # → 1484
cd ../../../..
```
Each kep.yaml version is parsed and compared with the previous version of the same KEP (keyed by number, so folder moves are followed). Each event is dated by the merge commit and linked to its PR. One `git cat-file --batch` process reads all file versions.

Create `src/normalize/history.py`:
```python
"""Git history -> data/seed/history_events.jsonl (dated change events, oldest first).

Walks the first-parent history of master, so each change is dated by when it landed
(the merge commit) and linked to its PR. Each kep.yaml version is parsed and compared
with the previous version of the same KEP (keyed by kep-number, so moves between SIG
folders are followed). Each sigs.yaml version is compared for leadership and team changes.

These are the same Event records the live engine produces, so the history backfill and
the replay eval exercise the real handlers.

Usage: python -m src.normalize.history
"""
import json
import re
import subprocess
from pathlib import Path

from src.normalize.common import clean_handle, load_yaml
from src.normalize.keps import parse_kep
from src.realtime.events import Event, to_utc_iso

ROOT = Path(__file__).resolve().parents[2]
REPOS = ROOT / "data" / "raw" / "repos"
SEED = ROOT / "data" / "seed"
_PR = re.compile(r"Merge pull request #(\d+)")


class BlobReader:
    """Reads file contents at given commits through one long-running `git cat-file --batch`."""

    def __init__(self, repo: Path):
        self.p = subprocess.Popen(["git", "cat-file", "--batch"], cwd=repo,
                                  stdin=subprocess.PIPE, stdout=subprocess.PIPE)

    def read(self, rev: str, path: str) -> str | None:
        self.p.stdin.write(f"{rev}:{path}\n".encode())
        self.p.stdin.flush()
        header = self.p.stdout.readline().decode().split()
        if len(header) < 3 or header[1] == "missing":
            return None
        data = self.p.stdout.read(int(header[2]))
        self.p.stdout.read(1)  # trailing newline
        return data.decode("utf-8", "replace")

    def close(self):
        self.p.stdin.close()
        self.p.wait()


def first_parent_commits(repo: Path, pathspec: str):
    """Yield (sha, iso_date, pr_number, [(status, path, old_path)]) oldest first."""
    out = subprocess.run(
        ["git", "log", "--reverse", "--first-parent", "-m", "--format=C %H %cI %s", "--name-status",
         "-M", "--", pathspec], cwd=repo, check=True, capture_output=True, text=True).stdout
    commit = None
    for line in out.splitlines():
        if line.startswith("C "):
            if commit:
                yield commit
            _, sha, date, *subject = line.split(" ")
            m = _PR.search(" ".join(subject))
            commit = (sha, to_utc_iso(date), int(m.group(1)) if m else None, [])
        elif line.strip() and commit:
            parts = line.split("\t")
            status = parts[0][0]
            commit[3].append((status, parts[-1], parts[1] if status == "R" else None))
    if commit:
        yield commit


TRACKED = ("status", "stage", "latest_milestone", "owning_sig", "title")
SETS = ("approvers", "superseded_by", "replaces")


def kep_events() -> list[Event]:
    repo = REPOS / "enhancements"
    reader = BlobReader(repo)
    state: dict[str, dict] = {}       # KEP id -> last parsed record
    path_to_id: dict[str, str] = {}
    events = []
    for sha, when, pr, changes in first_parent_commits(repo, "keps/*/*/kep.yaml"):
        src = {"commit": sha, "pr": pr,
               "url": f"https://github.com/kubernetes/enhancements/pull/{pr}" if pr else
                      f"https://github.com/kubernetes/enhancements/commit/{sha}"}
        for status, path, old_path in changes:
            if status == "D":
                kid = path_to_id.pop(path, None)
                if kid and kid in state:
                    events.append(Event("git", "kep_file_removed", kid, when, {**src, "path": path}, when))
                continue
            text = reader.read(sha, path)
            try:
                rec = parse_kep(load_yaml(text), path.rsplit("/", 1)[0]) if text else None
            except Exception:  # noqa: BLE001 - unparseable version: skip it, keep the previous state
                rec = None
            if not rec:
                continue
            kid = rec["id"]
            path_to_id[path] = kid
            if old_path:
                path_to_id.pop(old_path, None)
            prev = state.get(kid)
            if prev is None:
                snapshot = {k: rec[k] for k in TRACKED + SETS + ("milestones", "authors", "path")}
                events.append(Event("git", "kep_created", kid, when, {**src, **snapshot}, when))
            else:
                for field in TRACKED:
                    if rec[field] != prev[field] and rec[field] is not None:
                        events.append(Event("git", f"kep_{field}_changed", kid, when,
                                            {**src, "from": prev[field], "to": rec[field]}, when))
                for stage in sorted(set(rec["milestones"]) | set(prev["milestones"])):
                    a, b = prev["milestones"].get(stage), rec["milestones"].get(stage)
                    if a != b:
                        events.append(Event("git", "kep_milestone_changed", kid, when,
                                            {**src, "stage": stage, "from": a, "to": b}, when))
                for field in SETS:
                    added = [x for x in rec[field] if x not in prev[field]]
                    removed = [x for x in prev[field] if x not in rec[field]]
                    if added or removed:
                        events.append(Event("git", f"kep_{field}_changed", kid, when,
                                            {**src, "added": added, "removed": removed}, when))
            state[kid] = rec
    reader.close()
    return events


def _leads(group: dict) -> set[tuple[str, str]]:
    out = set()
    for role in ("chairs", "tech_leads"):
        for p in (group.get("leadership") or {}).get(role) or []:
            login = clean_handle((p or {}).get("github"))
            if login:
                out.add((login, role.rstrip("s")))
    return out


def _groups(text: str) -> dict[str, dict]:
    data = load_yaml(text) or {}
    return {g["dir"]: g for kind in ("sigs", "workinggroups", "committees")
            for g in (data.get(kind) or []) if isinstance(g, dict) and g.get("dir")}


def sig_events() -> list[Event]:
    repo = REPOS / "community"
    reader = BlobReader(repo)
    prev: dict[str, dict] | None = None
    events = []
    for sha, when, pr, _ in first_parent_commits(repo, "sigs.yaml"):
        text = reader.read(sha, "sigs.yaml")
        try:
            cur = _groups(text) if text else None
        except Exception:  # noqa: BLE001
            cur = None
        if cur is None:
            continue
        src = {"commit": sha, "pr": pr,
               "url": f"https://github.com/kubernetes/community/pull/{pr}" if pr else
                      f"https://github.com/kubernetes/community/commit/{sha}"}
        if prev is not None:
            for team in sorted(set(cur) | set(prev)):
                if team not in prev:
                    events.append(Event("git", "team_created", team, when, {**src, "name": cur[team].get("name")}, when))
                elif team not in cur:
                    events.append(Event("git", "team_retired", team, when, src, when))
                    continue
                before = _leads(prev.get(team, {}))
                after = _leads(cur[team])
                for login, role in sorted(after - before):
                    events.append(Event("git", "team_lead_added", team, when, {**src, "login": login, "role": role}, when))
                for login, role in sorted(before - after):
                    events.append(Event("git", "team_lead_removed", team, when, {**src, "login": login, "role": role}, when))
                sp_before = {s["name"] for s in prev.get(team, {}).get("subprojects") or [] if isinstance(s, dict)}
                sp_after = {s["name"] for s in cur[team].get("subprojects") or [] if isinstance(s, dict)}
                for name in sorted(sp_after - sp_before):
                    events.append(Event("git", "subproject_added", team, when, {**src, "subproject": name}, when))
                for name in sorted(sp_before - sp_after):
                    events.append(Event("git", "subproject_removed", team, when, {**src, "subproject": name}, when))
        else:  # first version: everything that exists is "created" at that date
            for team, g in sorted(cur.items()):
                events.append(Event("git", "team_created", team, when, {**src, "name": g.get("name")}, when))
                for login, role in sorted(_leads(g)):
                    events.append(Event("git", "team_lead_added", team, when, {**src, "login": login, "role": role}, when))
        prev = cur
    reader.close()
    return events


def main() -> None:
    SEED.mkdir(parents=True, exist_ok=True)
    events = sorted(kep_events() + sig_events(), key=lambda e: (e.occurred_at, e.entity_key, e.kind))
    out = SEED / "history_events.jsonl"
    with out.open("w") as fh:
        for e in events:
            fh.write(e.to_json() + "\n")
    kinds: dict[str, int] = {}
    for e in events:
        kinds[e.kind] = kinds.get(e.kind, 0) + 1
    print(f"{len(events)} events -> {out.relative_to(ROOT)} ({events[0].occurred_at[:10]} .. {events[-1].occurred_at[:10]})")
    for k, n in sorted(kinds.items(), key=lambda x: -x[1]):
        print(f"  {k:28}{n}")


if __name__ == "__main__":
    main()
```
```bash
time .venv/bin/python -m src.normalize.history
# → 5584 events -> data/seed/history_events.jsonl (2017-05-16 .. 2026-10-14), ~17 s
# → milestone 1107, latest_milestone 1117, stage 694, kep_created 692, team_lead_added 522,
#   subproject_added 400, kep_status_changed 351, team_lead_removed 245, ...
```
Check: KEP-753 goes provisional (2020-12-17, PR #2193) → implementable/alpha (2023-02-09) → beta (2023-10-05, PR #4255) → implemented/stable (2025-02-04, PR #5081). Running the script twice gives a byte-identical file, and there are no duplicate event ids.

### Step 1.13 · 01:35 · Parser tests
Create `tests/test_normalize.py`:
```python
from src.normalize.common import clean_handle, clean_status, kep_id_from_ref, load_yaml
from src.normalize.keps import parse_kep
from src.realtime.events import Event


def test_clean_status_handles_typos_and_comments():
    assert clean_status("imlpemented") == "implemented"
    assert clean_status("implementable # provisional|implementable|implemented") == "implementable"
    assert clean_status("implemented (alpha)") == "implemented"
    assert clean_status("superseded") == "replaced"
    assert clean_status("provisional|implementable|implemented|deferred|rejected|withdrawn|replaced") is None


def test_clean_handle_variants_and_bots():
    assert clean_handle('"@SergeyKanzhelev"') == "sergeykanzhelev"
    assert clean_handle("@mrunalp      # overall") == "mrunalp"
    assert clean_handle("Jane Doe (@jdoe)") == "jdoe"
    assert clean_handle("k8s-ci-robot") is None
    assert clean_handle(None) is None


def test_kep_refs():
    assert kep_id_from_ref("/keps/sig-node/4438-sidecar-restart-termination") == "KEP-4438"
    assert kep_id_from_ref("KEP-753") == "KEP-753"
    assert kep_id_from_ref("n/a") is None


def test_yaml_keeps_dates_as_strings():
    assert load_yaml("creation-date: 2018-05-14")["creation-date"] == "2018-05-14"
    assert load_yaml("creation-date: 2024-13-01")["creation-date"] == "2024-13-01"  # invalid month, no crash


def test_parse_kep_minimal():
    rec = parse_kep({"kep-number": 753, "title": "Sidecar Containers", "owning-sig": "sig-node",
                     "status": "implemented", "authors": ["@A"], "approvers": ["@B"],
                     "see-also": ["/keps/sig-node/4438-sidecar-restart-termination"],
                     "replaces": ["n/a"], "milestone": {"alpha": "v1.28", "beta": None},
                     "feature-gates": [{"name": "SidecarContainers", "components": ["kubelet"]}]},
                    "keps/sig-node/753-sidecar-containers")
    assert rec["id"] == "KEP-753" and rec["status"] == "implemented"
    assert rec["authors"] == ["a"] and rec["approvers"] == ["b"]
    assert rec["see_also"] == ["KEP-4438"] and rec["replaces"] == []
    assert rec["milestones"] == {"alpha": "v1.28"}
    assert rec["feature_gates"][0]["components"] == ["kubelet"]


def test_event_id_is_deterministic_and_ignores_observed_at():
    a = Event("git", "kep_status_changed", "KEP-1", "2024-01-01T00:00:00Z", {"to": "implemented"}, "2024-01-02T00:00:00Z")
    b = Event("git", "kep_status_changed", "KEP-1", "2024-01-01T00:00:00Z", {"to": "implemented"}, "2026-10-16T00:00:00Z")
    assert a.event_id == b.event_id
```
```bash
.venv/bin/pytest -q      # → 7 passed
```

### Step 1.14 · 01:40 · Gold-label candidates for hand labelling
Lists every scoped-SIG KEP that ended withdrawn, replaced, rejected or deferred, or that has a supersession link, plus the 15 most-discussed implemented KEPs (comment counts from the fetched tracking issues). Written to `eval/gold_candidates.csv`.
```bash
.venv/bin/python - <<'EOF'
import json, glob, csv
SC = {"sig-node","sig-api-machinery","sig-storage","sig-scheduling","sig-auth","sig-cli"}
K = {json.loads(l)['id']: json.loads(l) for l in open('data/seed/keps.jsonl')}
iss = {}
for f in glob.glob('data/raw/api/issues/*/page_*.json'):
    for it in json.load(open(f)): iss[it['number']] = it
E = [json.loads(l) for l in open('data/seed/history_events.jsonl')]
ended = {e['entity_key'] for e in E if e['kind'] == 'kep_status_changed' and e['payload']['to'] in ('withdrawn','replaced','rejected')}
rows = []
for k in K.values():
    if k['owning_sig'] not in SC: continue
    why = []
    if k['status'] in ('withdrawn','replaced','rejected','deferred'): why.append('status:' + k['status'])
    if k['id'] in ended and not why: why.append('ended-in-history')
    if k['replaces'] or k['superseded_by']: why.append('supersession')
    rows.append(dict(kep=k['id'], title=k['title'], sig=k['owning_sig'], status=k['status'],
                     issue_comments=iss.get(k['number'], {}).get('comments', 0), why_candidate=';'.join(why),
                     issue_url=k['issue_url'], readme_url=k['readme_url'], _w=bool(why)))
neg = [r for r in rows if r['_w']]
debated = sorted([r for r in rows if not r['_w'] and r['status'] == 'implemented'], key=lambda r: -r['issue_comments'])[:15]
for r in debated: r['why_candidate'] = 'most-debated-implemented'
out = sorted(neg, key=lambda r: -r['issue_comments']) + debated
with open('eval/gold_candidates.csv', 'w', newline='') as fh:
    w = csv.DictWriter(fh, fieldnames=[k for k in out[0] if k != '_w']); w.writeheader()
    for r in out: w.writerow({k: v for k, v in r.items() if k != '_w'})
EOF
# → 21 withdrawn/replaced/rejected/deferred/supersession + 15 debated
echo "kep,outcome,reason_one_line,evidence_url,replaced_by,labelled_by,notes" > eval/gold_decisions.csv
git add -A && git commit -m "Phase 1: KEP, SIG/OWNERS and git-history normalizers; seed data; gold candidates"
# → commit 1ac6da0
```

### Step 1.15 · Fri 16 Oct, 02:00 · CLAUDE.md (instructions for the coding assistant)
Create `CLAUDE.md` at the repo root. Claude Code reads it at the start of every session, so it holds the project summary, the working rules (record every step here, keep the tracking docs current, commit but let the user push, no secrets, gold labels are hand-written), the hackathon rules that constrain the code, the commands, the layout, the conventions and the verified gotchas. Copy it from the repo at the commit below.
```bash
git add CLAUDE.md docs/project/BUILD_LOG.md
git commit -m "Add CLAUDE.md with project rules, commands, conventions and gotchas"
```
