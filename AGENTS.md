# Agent guide

This repo gives you the **Reflex** capability: verifying that a deployed web
feature actually works for real users, by driving a real browser against a
live URL via the `mcp__reflex__*` MCP tools.

- The skill is at [`skills/reflex/SKILL.md`](skills/reflex/SKILL.md) — load it
  when the user wants to verify, test, or confirm a **deployed** feature works
  (not for local unit/integration tests).
- The MCP server is pre-configured in [`.mcp.json`](.mcp.json)
  (`https://reflex.mcp.getdoable.ai/mcp`). It needs the user's Doable API key
  in the `Authorization` header — see the README if a tool returns
  `missing_doable_key`.
- **Core rule:** the service is instruction-driven. After every tool call,
  read the `instruction` field in the response and do what it says, until the
  run reaches a terminal phase (`reported`, `failed`, or `canceled`). Do not
  hardcode the tool sequence.

Full human-readable guide: [`docs/usage.md`](docs/usage.md).
