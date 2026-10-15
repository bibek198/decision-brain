# Phase 6: Agent, actions, MCP (Sat 9:00 AM – 1:30 PM) ★ for the agent

**Status:** Not started · **Planned window:** see title · **Actual:** —

## Plan (from PLAN.md)
- [ ] 1. Agent loop: Anthropic SDK, manual tool loop of at most 6 rounds, Haiku 5.5 by default (`LLM_MODEL` env), effort low or medium, `strict` tool schemas. The model sees only `data`, `notes` and `sources`, and the server keeps path and trace.
- [ ] 2. System prompt rules: answer only from tools; say "not in the graph"; never present rejected or withdrawn as adopted; name the superseded-by first; cite with `[n]`.
- [ ] 3. Citation guard: every URL in the answer must come from tool sources. Strip and retry once otherwise.
- [ ] 4. Action tools: `recommend_reviewers`, `draft_decision_brief` (a `Draft` node plus markdown), `route_question` (a `Question` node, i.e. memory). Writes go only to the current graph; nothing reaches GitHub.
- [ ] 5. MCP server (`mcp` FastMCP) exposes the same registry. Test it in the MCP Inspector and in Claude Desktop, which makes a nice 10-second video clip.
- [ ] 6. Run the 30 dev questions and fix tool descriptions. Hold out 10.
- **Done when:** at least 80% correct tool choice, the lure answers are honest, the guard test passes, and the Inspector lists the tools.

## Work log
<!-- newest first: YYYY-MM-DD HH:MM IST · what was done · problems · fixes · commit SHA -->

## Open questions
