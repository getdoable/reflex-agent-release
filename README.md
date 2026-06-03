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

## Quick start

### 1. Get the repo

```bash
git clone https://github.com/getdoable/reflex-agent-release.git
cd reflex-agent-release
```

### 2. Set your Doable API key

Reflex forwards this key to Doable per request; it is never persisted or
logged. Pick **one**:

**Option A — environment variable (recommended).** `.mcp.json` already
references `${QA_DOABLE_API_KEY}`:

```bash
cp .env.example .env
# edit .env and set QA_DOABLE_API_KEY=<your-doable-api-key>
set -a; source .env; set +a     # export it into this shell
```

Launch your agent **from this same shell** so the variable is in scope.

**Option B — hardcode it.** Edit `.mcp.json` and replace
`${QA_DOABLE_API_KEY}` with your literal key. This repo is yours locally —
just don't commit the edited file back.

> Don't have a key? Get one from your Doable account.

### 3. Launch your agent from this folder

```bash
claude        # or your agent of choice
```

Verify the connection:

```bash
claude mcp list
# reflex: https://reflex.mcp.getdoable.ai/mcp (HTTP) - ✓ Connected
```

If you see `Missing environment variables: QA_DOABLE_API_KEY`, the key isn't
exported — redo step 2 (Option A) in the launch shell, or use Option B.

### 4. Ask it to verify your deployment

You don't need to know the steps — the service is instruction-driven and the
skill knows the flow. Just state the goal:

> I just deployed a feature and want to confirm it works for real users.
> Use the Reflex tools to verify my deployment.
> - Deployed URL: `https://my-app.example.com`
> - What it should do: `<paste your PRD, or point to a PRD file>`
>
> Walk me through whatever the service tells you to do next, until you reach
> a final verdict. If something's broken and I fix + redeploy, I'll give you
> the new URL. When done, show me the report.

That's it. The agent runs `init → start → … → finalize → report`, following
each response's `instruction` field. `start` blocks 1–5 minutes while Doable
generates and runs browser tests — that's expected.

---

## Other ways to install

### Agent Skills CLI (any compatible agent)

```bash
npx skills add getdoable/reflex-agent-release
```

You still configure the MCP server (`.mcp.json` + your Doable key) as above.

### Claude Code plugin

```bash
# 1. Add this repo as a plugin marketplace
claude plugin marketplace add getdoable/reflex-agent-release

# 2. Install the Reflex plugin
claude plugin install reflex@reflex-agent-release
```

---

## What's in here

```
reflex-agent-release/
├── .mcp.json                      # Pre-wired Reflex MCP server (hosted)
├── .claude-plugin/
│   └── marketplace.json           # Claude Code plugin manifest
├── .env.example                   # The one key you need
├── skills/
│   └── reflex/
│       ├── SKILL.md               # The skill: principles, tools, when-to-use
│       └── references/            # workflow, troubleshooting, connection
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
