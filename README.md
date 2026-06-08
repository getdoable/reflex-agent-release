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

### 1. Register the MCP server (required)

This one step is enough to use Reflex. Two ways:

**Use this repo's preconfigured config (recommended).** It already points at
the server *and* sets the per-call `timeout` the blocking `start` step needs:

```bash
git clone https://github.com/getdoable/reflex-agent-release.git
cd reflex-agent-release
cp .mcp.json.example .mcp.json   # then put your Doable key in the Authorization header
```

(Or copy the `reflex` block from [`.mcp.json.example`](.mcp.json.example) into
your own project's `.mcp.json`.)

**Or register via the CLI (no repo needed):**

```bash
claude mcp add --transport http --scope user reflex \
  https://reflex.mcp.getdoable.ai/mcp \
  --header "Authorization: Bearer <your-doable-api-key>"
```

> ⚠️ **Timeout matters.** `start` blocks 1–5 min while Doable runs the tests, so
> the MCP client needs a generous per-call timeout. The repo's
> `.mcp.json.example` sets `"timeout": 660000` (11 min) — if you register via
> the CLI, add that `timeout` field to the `reflex` entry afterward, or the
> client may cut `start` off.

Verify it connected:

```bash
claude mcp list
# reflex: https://reflex.mcp.getdoable.ai/mcp (HTTP) - ✓ Connected
```

That's all you need — the service is instruction-driven, so your agent follows
each response's next step on its own. Then jump to [Use it](#use-it).

> **Scope / key:** `.mcp.json` in a project is shared with your team (commit it
> — it holds only the `${QA_DOABLE_API_KEY}` placeholder; your real key stays
> local). `claude mcp add --scope user` registers it for every project instead.
> The key is forwarded to Doable per request and never stored by Reflex — get
> one from your Doable account.

### 2. (Optional) Add the skill

The skill gives your agent extra guidance about Reflex. Skip it and Reflex
still works (step 1 is enough). To add it, pick **one**:

- **Claude Code plugin:**
  ```bash
  claude plugin marketplace add getdoable/reflex-agent-release
  claude plugin install reflex@reflex-agent-release
  ```
- **npx skills** (`--global` = every project; omit for the current project only):
  ```bash
  npx skills add getdoable/reflex-agent-release --global
  ```

Then reload your agent.

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
