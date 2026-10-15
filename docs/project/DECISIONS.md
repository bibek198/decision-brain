# Decision log

Newest first. Format: decision, options considered, why, consequences.

## D-010 · 2026-10-15 · Drop emails from the SIG registry
- **Options:** keep all `sigs.yaml` fields; keep handle, name and company only.
- **Why:** rule 6 forbids personal data. Public GitHub handles and company affiliation are enough for ownership and affiliation edges.
- **Consequences:** the normalizer strips `email` before anything is written to `data/seed/`.

## D-009 · 2026-10-15 · A sandbox repo we own provides push webhooks
- **Options:** poll only; webhooks on Kubernetes repos (impossible, we are not admins); a sandbox repo with a webhook.
- **Why:** polling a repo we do not own gives near-real-time at best. A public repo we own can send webhooks to the deployed app, so a change made on camera reaches the graph in seconds.
- **Consequences:** needs a public HTTPS endpoint (EC2 plus Caddy) and HMAC verification. Sandbox data is our own, which rule 6 allows.

## D-008 · 2026-10-15 · fastembed for local embeddings
- **Options:** sentence-transformers (torch, large image); hosted embedding API; fastembed (ONNX).
- **Why:** free, no API key, small Docker image, fast on CPU.
- **Consequences:** `BAAI/bge-small-en-v1.5`, 384 dimensions, cosine vector index in FalkorDB.

## D-007 · 2026-10-15 · Deploy on AWS EC2 with replay as the default
- **Options:** local setup only; FalkorDB Cloud plus PaaS; one EC2 instance with Docker Compose.
- **Why:** organisers prefer a live deployment, the webhook needs a public endpoint, and AWS credits are available.
- **Consequences:** t3.medium, Caddy and DuckDNS for HTTPS, ports 6379 and 3000 never public, budget alert first, per-IP limit and spend cap on live questions.

## D-006 · 2026-10-15 · Bi-temporal facts (valid_from, valid_to, observed_at)
- **Options:** overwrite state in place; append snapshots; interval edges.
- **Why:** "keeping knowledge current" needs both the current view and history. Intervals answer "now", "as of v1.28" and "what changed since" from one graph.
- **Consequences:** read tools filter `valid_to IS NULL` by default and take an optional `as_of`; status changes close one interval and open another.

## D-005 · 2026-10-15 · Real-time change engine is a core feature
- **Options:** frozen snapshot plus one replayed feed; periodic full rebuild; event-driven incremental updates.
- **Why:** the user's requirement is that the brain adapts as the company changes. An event log with idempotent handlers gives continuous, safe updates.
- **Consequences:** Redis Stream `brain:events` inside FalkorDB's own Redis; sources are a GitHub poller (ETags), a git diff of `kep.yaml` and `sigs.yaml`, and the sandbox webhook. Loader upserts are reused as handlers, so there is one write path. A time-travel replay eval measures it.

## D-004 · 2026-10-15 · Hybrid retrieval inside FalkorDB; baseline in the same store
- **Options:** external vector DB for retrieval and baseline; FalkorDB for everything.
- **Why:** entity resolution by full-text and vector search, followed by Cypher traversal, keeps the graph load-bearing (rule 3). A vector-only baseline in the same store is a fair ablation.
- **Consequences:** one database to run; the baseline differs from the agent only by the graph.

## D-003 · 2026-10-15 · Claude Haiku 5.5 as the default LLM
- **Options:** Sonnet 5.5; free tiers (Gemini, Groq); Haiku 5.5.
- **Why:** Haiku 5.5 is $0.10 in and $0.50 out per million tokens, and the Batch API halves extraction cost. The whole project is estimated at about $1.50 of the $5 credit.
- **Consequences:** `LLM_MODEL` env var; Sonnet 5.5 is optional for the recorded demo; a Gemini adapter is the fallback if credits run out.

## D-002 · 2026-10-15 · Kubernetes KEPs as the "company"
- **Options:** an RFC repo (vuejs, reactjs, emberjs, rust-lang); Kubernetes enhancements plus the community SIG registry.
- **Why:** Track 3 asks for people, teams, projects, documents and decisions, ownership, and multiple sources. KEPs give SIGs as teams, owners and approvers, design docs, tracking issues, PR chats, structured status and supersession, and the repo is active (good for real-time). vuejs and reactjs RFCs have been inactive since 2024–2025.
- **Consequences:** all 686 `kep.yaml` files come from a git clone (no rate limit); API data (issues and comments) is scoped to 5–6 SIGs.

## D-001 · 2026-10-15 · Enter Track 3, and Track 1 if it fits honestly
- **Options:** Track 3 only; Tracks 3 and 1.
- **Why:** rule 5 allows multiple tracks. Graph algorithms as agent tools, MCP, and write actions also match the Track 1 focus.
- **Consequences:** name Track 1 at submission only if the algorithm and action tools are solid.
