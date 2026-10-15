# Status

All times IST. Deadline: Sun 18 Oct 2026, 11:59 PM. Target submission: Sun 7:00 PM.

| Phase | Window | Status | Done when | Commit |
|---|---|---|---|---|
| [0 Foundation](phases/phase-00-foundation.md) | Thu 11:00 PM – 12:30 AM | Done | Repo public, both smoke tests pass, data survives restart, fetch running | ac67226 |
| [1 Data](phases/phase-01-data.md) | Thu night → Fri 10:30 AM | In progress | Raw data frozen, manifest committed, 15 gold rows | |
| [2 Schema](phases/phase-02-schema.md) | Fri 10:30 AM – 12:00 PM | Done | Every demo question answered on the toy graph | 0fe6f2c |
| [3 Load](phases/phase-03-load.md) | Fri 12:00 – 3:30 PM | Not started | Second load changes nothing, counts match, history intervals loaded | |
| [4 Tools](phases/phase-04-tools.md) | Fri 4:00 – 8:00 PM | Not started | All read tools pass tests with no LLM | |
| [5 Extraction](phases/phase-05-extraction.md) | Fri 8:00 – 11:30 PM | Not started | Decisions loaded, audit recorded | |
| [6 Agent](phases/phase-06-agent.md) | Sat 9:00 AM – 1:30 PM | Not started | ≥80% tool choice, honest lures, guard test, MCP lists tools | |
| [7 Real-time](phases/phase-07-realtime.md) | Sat 1:30 – 6:00 PM | Not started | Change queryable ≤10 s, webhook flips answer, replay idempotent | |
| [8 UI](phases/phase-08-ui.md) | Sat 6:00 – 10:00 PM | Not started | Path, Cypher, compare, live feed work on camera | |
| [9 Eval](phases/phase-09-eval.md) | Sat 10:00 PM – 1:00 AM | Not started | Results table and replay eval filled | |
| [10 Deploy](phases/phase-10-deploy.md) | Sun 9:00 AM – 12:30 PM | Not started | Fresh clone <10 min, replay works, EC2 live with webhook | |
| [11 Submit](phases/phase-11-submit.md) | Sun 12:30 – 7:00 PM | Not started | Submitted, all links checked logged out | |

## Checkpoints
| When | Expected state | Met? |
|---|---|---|
| Fri 8:00 PM | Graph loaded and idempotent with history intervals; read tools pass tests | |
| Sat 1:30 PM | Decisions extracted, agent answers with sources and `as_of`, MCP works | |
| Sat 6:00 PM | Real-time engine end to end within 10 s | |
| Sat end of day | UI demo-ready, eval and replay eval filled | |
| Sun 7:00 PM | Submitted and deployed | |

## LLM spend
| Date | What | Cost (USD) | Running total |
|---|---|---|---|
