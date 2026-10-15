# Phase 10: Package and deploy (Sun 9:00 AM – 12:30 PM) ★ for packaging and replay

**Status:** Not started · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [ ] 1. Tag `freeze`. Dockerfile (slim, fastembed model pre-downloaded). One `docker compose up` brings up FalkorDB and the app. `bootstrap` loads the seed, decisions and embeddings and is idempotent.
- [ ] 2. **Replay mode**: cached answers for the example questions, used by default with no key, with a clear banner.
- [ ] 3. Run the fresh-clone test twice (Path A all-Docker, Path B venv) and keep it under 10 minutes. Scan the history for secrets.
- [ ] 4. **Deploy**: AWS EC2 t3.medium on your credits.
   - Set a budget alert first.
   - Run Docker Compose (`falkordb`, `app`, `worker`, `caddy`) with Caddy, DuckDNS and HTTPS. FalkorDB stays bound to localhost.
   - Register the sandbox repo webhook to `https://<name>.duckdns.org/api/webhook/github` with a secret in `.env`, and run the poller against `brain_live`.
   - Replay is the default for questions, with a per-IP limit for live questions and a hard spend cap. The live feed and graph are always real.
- [ ] 5. `docs/explain.md`: five design answers in your own words (rules 12 and 13).

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->

## Open questions
