# Reflex — Agent Skill & MCP (Doable in the Loop)

Verify that your **deployed** web feature actually works for real users —
before you call it done. Reflex drives a real browser against your live URL,
checks it against your PRD, and tells your AI coding agent — in plain,
machine-followable instructions — what's broken and what to fix.

This repo packages Reflex for any skills-compatible AI agent (Claude Code,
Cursor, Copilot, and [others](https://agentskills.io/clients)):

- an **Agent Skill** (`skills/reflex/`) following the [Agent Skills](https://agentskills.io/) format, and
- a pre-wired **`.mcp.json`** pointing at the hosted Reflex MCP server.

Download it, set one API key, and start verifying.

---

## Install

Reflex has two pieces, and **neither is tied to this repo's folder** — you
install them into whatever scope you work in, then use Reflex from **your own
project directory**:

- the **skill** — instructions your agent loads on demand, and
- the **MCP server** — a **hosted endpoint** (`https://reflex.mcp.getdoable.ai/mcp`)
  your agent calls. Because it's a URL, you just register it once; it then works
  from any directory.

Pick the path that matches how widely you want it available.

### Option A — Claude Code plugin (one step, every project)

Installs the skill **and** registers the MCP server at the user level, so both
are available in every project you open. Nothing to copy into your repo.

```bash
claude plugin marketplace add getdoable/reflex-agent-release
claude plugin install reflex@reflex-agent-release
```

### Option B — register the server + skill yourself

Register the hosted MCP server once. Use `--scope user` for **all** your
projects, or `--scope project` to write a `.mcp.json` into the current project
(commit it to share with your team):

```bash
claude mcp add --transport http --scope user reflex \
  https://reflex.mcp.getdoable.ai/mcp \
  --header "Authorization: Bearer <your-doable-api-key>"
```

Install the skill globally (drop `--global` to scope it to the current project):

```bash
npx skills add getdoable/reflex-agent-release --global
```

### Option C — copy the config into your project

Copy the `reflex` block from this repo's [`.mcp.json`](.mcp.json) into your own
project's `.mcp.json`, and run `npx skills add getdoable/reflex-agent-release`
in that project. Good when you want the config version-controlled with your app.

> **Just trying it out?** Clone this repo and launch your agent inside it — the
> bundled `.mcp.json` works as-is. For real work, install into your own project
> (above) so you verify your app from your own codebase.

## Set your Doable API key

Reflex forwards this key to Doable per request; it's never persisted or logged.
Whatever ends up in the MCP server's `Authorization` header is the key.

- If you registered with `claude mcp add ... --header "Authorization: Bearer <key>"`
  (Option B), the key is already baked in — nothing more to do.
- If you're using a `.mcp.json` with the `${QA_DOABLE_API_KEY}` placeholder
  (Options A/C, or the bundled file), provide the value from the environment in
  the shell you launch your agent from, or replace the placeholder with the
  literal key:

```bash
export QA_DOABLE_API_KEY=<your-doable-api-key>   # or add it to your shell profile
```

Verify it's connected (from your project):

```bash
claude mcp list
# reflex: https://reflex.mcp.getdoable.ai/mcp (HTTP) - ✓ Connected
```

A `Missing environment variables: QA_DOABLE_API_KEY` warning means the
placeholder didn't resolve — export the key (above) or bake it into the header.

> Don't have a key? Get one from your Doable account.

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
├── .mcp.json                      # Pre-wired Reflex MCP server (hosted)
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
