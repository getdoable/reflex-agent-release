# Connecting an MCP client to Reflex

Reflex serves MCP over Streamable HTTP at `POST /mcp`. Point your client
at it with the Doable Bearer in the `Authorization` header.

## Claude Code (`.mcp.json` in the project root)

```json
{
  "mcpServers": {
    "reflex": {
      "type": "http",
      "url": "https://<your-reflex-host>/mcp",
      "headers": { "Authorization": "Bearer <doable-api-key>" }
    }
  }
}
```

- **Hosted:** set `url` to the Reflex endpoint your provider gave you.
- **Local dev:** `http://127.0.0.1:8000/mcp`.

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
