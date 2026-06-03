# Connecting an MCP client to Reflex

Reflex serves MCP over Streamable HTTP at `POST /mcp`. The server is a
**hosted endpoint** (a URL), so it isn't tied to any directory — you
register it once in the scope you want, then it's available wherever you
work. Whatever's in the `Authorization` header is the Doable Bearer.

## Registering the server (Claude Code)

Pick a scope. All of these make the same `reflex` server available; they
differ only in *where* it's available:

- **User scope (all your projects):**
  ```bash
  claude mcp add --transport http --scope user reflex \
    https://<your-reflex-host>/mcp \
    --header "Authorization: Bearer <doable-api-key>"
  ```
- **Project scope (this project, committable):** write the block below into
  the project's `.mcp.json`, or use `claude mcp add --scope project ...`.
- **Plugin:** installing the Reflex Claude Code plugin registers the server
  (and skill) automatically — no manual step.

`.mcp.json` block (project scope, or the bundled file in the release repo):

```json
{
  "mcpServers": {
    "reflex": {
      "type": "http",
      "url": "https://<your-reflex-host>/mcp",
      "timeout": 660000,
      "headers": { "Authorization": "Bearer <doable-api-key>" }
    }
  }
}
```

- **Hosted:** set `url` to the Reflex endpoint your provider gave you.
- **Local dev:** `http://127.0.0.1:8000/mcp`.

## Tool-call timeout (important)

The `start` tool blocks 1–5 minutes while Doable generates browser tests, so
the MCP client's per-tool timeout must be generous or that call (and a long
`wait_for_findings`) will be cut off client-side. In Claude Code, set the
**`timeout` field (milliseconds) on the server entry** in `.mcp.json` — as
shown above, `660000` (11 min) comfortably covers it. This is a hard
wall-clock limit per tool call. (`MCP_TIMEOUT` only controls server *startup*,
not tool execution, so it won't help here.) If you registered the server with
`claude mcp add` instead, add the same `timeout` field to its entry in your
config.

## Supplying the Doable key — pick one

**(a) Hardcode in `.mcp.json` (simplest).** Put the literal key in the
header. `.mcp.json` is typically gitignored, so it won't reach git. No
shell setup; works from any terminal. When the key rotates, update it
here.

**(b) `${QA_DOABLE_API_KEY}` placeholder.** Keep the header as
`"Bearer ${QA_DOABLE_API_KEY}"` and provide the value from the
environment. It's interpolated **at client launch**, so the variable must
be exported in the launch shell, every session:

```bash
export QA_DOABLE_API_KEY=<doable-api-key>   # or: set -a; source .env; set +a
claude                                       # launch from THIS shell
```

Env vars don't cross terminal sessions — a new tab needs the export
again. If it's unset, the Bearer resolves to empty and Doable-touching
tools fail with `missing_doable_key`.

**(c) Claude Code settings `env`.** Put
`"env": {"QA_DOABLE_API_KEY": "<key>"}` in `.claude/settings.local.json`
(gitignored). Keeps the key out of `.mcp.json` with no per-terminal step.

## Verify

```bash
claude mcp list
```

Expect `reflex: <url> (HTTP) - ✓ Connected` with **no** "Missing
environment variables" warning. If you see that warning, you're in
placeholder mode (b) without the var exported — hardcode the key (a) or
re-export it.

## Notes

- The Doable key is only needed by the **MCP client**. A self-hosted
  Reflex **server** does not need it (the Bearer flows in per request).
- Once connected, the client exposes ten tools namespaced
  `mcp__reflex__*`.
