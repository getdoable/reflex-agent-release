# Reflex — Agent Skill & MCP (Doable in the Loop)

Verify that your **deployed** web feature actually works for real users —
before you call it done. Reflex drives a real browser against your live URL,
checks it against your PRD, and tells your AI coding agent — in plain,
machine-followable instructions — what's broken and what to fix.

Reflex is delivered as a hosted **MCP server**. This repo is an *optional*
enhancement on top of it — an **Agent Skill** (`skills/reflex/`, in the
[Agent Skills](https://agentskills.io/) format) plus an `.mcp.json.example`
config — that makes setup easier for any skills-compatible agent (Claude Code,
Cursor, Copilot, and [others](https://agentskills.io/clients)). You can also
skip the repo entirely and point your agent straight at the MCP server.

---

## Install

Reflex is an MCP service. The one thing you always need is the **MCP server**
registered with your Doable API key (a hosted endpoint —
`https://reflex.mcp.getdoable.ai/mcp` — that works from any project once
registered). The **skill** in this repo is *optional*: it gives your agent
extra guidance, but the service is instruction-driven, so the tools work on
their own without it.

Pick **one** of the three self-contained options below.

### Option A — Claude Code plugin (skill + server)

1. Install the skill (available in every project):
   ```bash
   claude plugin marketplace add getdoable/reflex-agent-release
   claude plugin install reflex@reflex-agent-release
   ```
2. Register the MCP server with your key:
   ```bash
   claude mcp add --transport http --scope user reflex \
     https://reflex.mcp.getdoable.ai/mcp \
     --header "Authorization: Bearer <your-doable-api-key>"
   ```
3. Reload your agent, then [verify](#verify).

### Option B — npx skills (skill + server)

1. Install the skill (`--global` = every project; omit it for the current project only):
   ```bash
   npx skills add getdoable/reflex-agent-release --global
   ```
2. Register the MCP server with your key:
   ```bash
   claude mcp add --transport http --scope user reflex \
     https://reflex.mcp.getdoable.ai/mcp \
     --header "Authorization: Bearer <your-doable-api-key>"
   ```
3. Reload your agent, then [verify](#verify).

### Option C — MCP server only (no skill, nothing from this repo)

The skill is optional — to use Reflex with **nothing from this repo**, just
register the hosted MCP server. The agent then follows each response's
`instruction` field on its own.

1. Register the MCP server with your key:
   ```bash
   claude mcp add --transport http --scope user reflex \
     https://reflex.mcp.getdoable.ai/mcp \
     --header "Authorization: Bearer <your-doable-api-key>"
   ```
2. Reload your agent, then [verify](#verify). That's the whole install.

> **Scope:** `--scope user` makes the server available in every project (stored
> in `~/.claude.json`). Swap in `--scope project` to write it into the current
> project's `.mcp.json` instead (committable, shared with your team).
>
> **The key** is forwarded to Doable per request and never stored by Reflex.
> Don't have one? Get it from your Doable account.

### Verify

```bash
claude mcp list
# reflex: https://reflex.mcp.getdoable.ai/mcp (HTTP) - ✓ Connected
```

## Use it

From **your own project**, launch your agent and state the goal — the service
is instruction-driven and the skill knows the flow, so you don't recite steps:

> I just deployed a feature and want to confirm it works for real users.
> Use the Reflex tools to verify my deployment.
> - Deployed URL: `https://my-app.example.com`
> - What it should do: `<paste your PRD, or point to a PRD file>`
>
> Walk me through whatever the service tells you to do next, until you reach
> a final verdict. If something's broken and I fix + redeploy, I'll give you
> the new URL. When done, show me the report.

The agent runs `init → start → … → finalize → report`, following each
response's `instruction` field. `start` blocks 1–5 minutes while Doable
generates and runs browser tests — that's expected.

---

## What's in here

```
reflex-agent-release/
├── .mcp.json.example              # MCP server template — copy to .mcp.json (gitignored), set your key
├── .claude-plugin/
│   └── marketplace.json           # Claude Code plugin manifest
├── .env.example                   # The one key you need
├── .gitignore                     # Protects .env / local secrets
├── AGENTS.md                      # Cross-agent pointer to the skill
├── LICENSE                        # MIT
├── README.md                      # This file
├── skills/
│   └── reflex/
│       ├── SKILL.md               # The skill: principles, tools, when-to-use
│       └── references/            # workflow, troubleshooting, connect
└── docs/
    └── usage.md                   # Full human-readable usage guide
```

## How it works

```
your agent ──MCP──► Reflex (reflex.mcp.getdoable.ai) ──► Doable ──► real browser ──► your live URL
```

- **Instruction-driven:** every tool response carries an `instruction` field;
  the agent follows it until the run reaches a terminal verdict. No hardcoded
  workflow.
- **Bearer-as-data:** your Doable key rides in the `Authorization` header,
  is forwarded to Doable, and is never stored.
- **The deployment must be publicly reachable** by Doable's browsers — disable
  any auth wall / preview gate (e.g. Vercel deployment protection) on the
  target URL.

Full details: [`docs/usage.md`](docs/usage.md) and the skill at
[`skills/reflex/`](skills/reflex/).

## License

[MIT](LICENSE) © Doable (getdoable)
