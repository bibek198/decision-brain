# Phase 7: Real-time change engine (Sat 1:30 – 6:00 PM) ★

**Status:** Not started · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [ ] 1. **Event log** (`events.py`, `stream.py`): event schema, deterministic `event_id`, `XADD brain:events`, consumer group `brain-workers`, `XACK` after the handler succeeds, and a dead-letter list after 3 failures.
- [ ] 2. **Sources:**
   - `poller.py`: every 60 s, issues, PRs and comments `since=cursor` with `If-None-Match`; the cursor and ETag live in `Meta`.
   - `gitdiff.py`: every 5 min, `git fetch` and diff `kep.yaml` and `sigs.yaml` from the last SHA to the new SHA, producing typed events.
   - `webhook.py`: `POST /api/webhook/github`, HMAC SHA-256 verified, for the **sandbox repo you own**.
- [ ] 3. **Handlers** (`handlers.py`): one function per event kind, all built on the Phase 3 upsert functions.
   - Close the old interval and open the new one.
   - Re-chunk and re-embed changed text.
   - Mark the dependent `Decision` stale and enqueue re-extraction (debounced 30 s, skipped if the thread hash is unchanged).
   - Run the supersession and contradiction check.
   - Link `Event-[:TOUCHED]->entity`.
   - Flag `Question` nodes that used a touched entity (`OUTDATED_BY`).
   - Fan out to `Watch` subscribers.
- [ ] 4. **Periodic jobs:** recompute `REVIEWED_FOR` and PageRank every 10 min or every 50 events, and run a nightly reconcile for deleted comments.
- [ ] 5. **Worker** (`worker.py`): a separate compose service (`app`, `worker`, `falkordb`) running an asyncio loop for the poller, git diff and consumer. It writes `Meta.lag_ms`.
- [ ] 6. **Push to UI** (`notify.py`): `GET /api/stream` (SSE) emits events and stale-answer notices.
- [ ] 7. **Agent awareness:** answers state `as_of` and data age. `what_changed(since | since_my_last_question)` reads `Event` nodes. The system prompt hedges on `stale` or unverified facts.
- [ ] 8. **Demo kit:** a sandbox repo `decision-brain-sandbox` (yours, public) with seeded issues. On camera you comment "We are replacing X with Y, decided" and the UI feed updates in about 5 s, so the same question now gives the new answer and the old answer shows "outdated". `reset_demo.sh` resets `brain_demo` in under 1 minute.
- [ ] 9. Tests:
   - Replaying the same events twice gives identical counts.
   - Out-of-order events still give the right final interval (handlers compare `occurred_at`).
   - A status change closes exactly one interval.
   - An edited comment that loses its quote drops the evidence.
- **Done when:** an injected `kep.yaml` status change is queryable within 10 s; a webhook comment shows in the UI feed and flips the answer; a second replay changes nothing; the worker survives a FalkorDB restart, because the stream is persistent.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->

## Open questions
