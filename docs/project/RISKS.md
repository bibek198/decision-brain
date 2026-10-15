# Risk register

| Risk | Early warning sign | Fallback | Status |
|---|---|---|---|
| FalkorDB or Docker will not start | Smoke test failing after 30 min | FalkorDB Cloud (rule 4); ask in Discord | Open |
| Data lost on restart | Graph empty after `docker compose restart` | Fix volume path (`/data` vs `/var/lib/falkordb/data`), enable AOF | Open |
| GitHub fetch slow or rate-limited | Fetch not done by Fri 9:00 AM | Cut to 4 SIGs; resume from saved pages | Open |
| `kep.yaml` too messy to parse | Many parse failures | Normalize status with a mapping table; log and skip the rest | Open |
| Git history backfill too slow or noisy | `git log -p` over 686 files takes too long | Only status and `superseded-by` lines, only the chosen SIGs | Open |
| LLM extraction poor | Pilot below 80% rationale or 95% quote check after 3 edits | Shrink scope; show low-confidence labels | Open |
| LLM credits run out | Spend over $3 by Sat | Replay mode for demos; Gemini free-tier adapter | Open |
| Agent picks wrong tools | Under 80% by Sat noon | Shorter descriptions, routing examples, fewer tools | Open |
| Real-time engine unstable | Duplicates after replay, or lag over 10 s | Simplify to poller plus git diff; keep the event log and intervals | Open |
| Webhook cannot be reached | Sandbox deliveries fail on EC2 | Poller on the sandbox repo every 15 s; "Replay next change" button | Open |
| Eval shows no advantage | Lure and freshness equal for both systems | Report honestly; lead with traceability, as-of and lag | Open |
| Fresh clone fails for judges | Fresh-clone test fails | Replay mode, Path B only, clearer README | Open |
| Leaked secret | Scan finds a key | Rotate the key at the provider immediately | Open |
| Cannot explain AI-written code (rules 12–13) | Unsure what a file does | Read and explain every core file; `docs/explain.md` | Open |
