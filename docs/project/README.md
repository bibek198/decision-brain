# Project tracking

This folder records how Decision Brain was planned and built during Graph Hacks (Oct 15–18, 2026). It is kept current at the end of every phase.

| File | What it holds |
|---|---|
| [PLAN.md](PLAN.md) | The full plan and the source of truth for scope: research findings, architecture, real-time design, schema, phases, cut order |
| [STATUS.md](STATUS.md) | One row per phase: window, status, done-when, commit |
| [DECISIONS.md](DECISIONS.md) | Architecture decision log, newest first |
| [RISKS.md](RISKS.md) | Risk register: warning sign and fallback |
| [AI_USAGE.md](AI_USAGE.md) | Running disclosure of AI-assistant use (rule 11) |
| [phases/](phases/) | One file per phase: plan checklist, work log, open questions |

## Working rule
At the end of each phase:
1. Tick the steps in that phase file and append a dated work-log entry.
2. Update `STATUS.md`.
3. Add any new decision to `DECISIONS.md`.
4. Commit as `docs: phase N complete`.

If scope changes, edit `PLAN.md` and record why in `DECISIONS.md`.
