---
name: reflex
description: "Use when verifying that a DEPLOYED web feature actually works for real users — post-deployment / production verification. Triggers: \"does my deploy work\", \"verify my live feature\", \"test my deployed app\", checking a Vercel/Netlify/Cloudflare deployment against a PRD or spec, confirming a feature works end-to-end in a real browser before calling it done, the Reflex MCP tools (mcp__reflex__*), \"Doable in the Loop\". NOT for local unit/integration tests — this drives a real browser against a live URL."
license: MIT
compatibility: "Requires an MCP-capable agent (e.g. Claude Code) connected to a reachable Reflex MCP server, with a Doable API key in the client's Authorization header. The deployment under test must be publicly reachable by Doable's browsers (no auth wall or preview gate)."
metadata:
  author: getdoable
  version: "0.1.0"
  product: "Reflex (Doable in the Loop)"
---

# Reflex — verify a deployed feature works for real users

Reflex is a post-deployment verification service. You give it a PRD (what
the feature should do) and a live URL; it drives Doable's real-browser
testing on your behalf and reports whether the feature actually works —
and what to fix if it doesn't. You interact with it through the
`mcp__reflex__*` MCP tools.

## Core principles

**1. The service is instruction-driven — ALWAYS follow the `instruction` field.**
Every tool response contains an `instruction` object with `next_action`
and `next_endpoint`. Read it after every call and do exactly what it
says. **Do not hardcode or assume the tool sequence** — the engine adapts
the next step to the findings and the fix-attempt cap. Your job is to
relay results to the user and follow instructions until the run reaches a
**terminal phase** (`reported`, `failed`, or `canceled`).

**2. Verify against a real, publicly reachable deployment.**
The URL must be reachable by Doable's browsers. If it's behind Vercel
deployment protection (or any auth wall / preview gate), disable that for
the target deployment first, or Doable can't load the page.

**3. `start` blocks for 1–5 minutes — that is expected.**
Doable's LLM generates browser test cases synchronously inside the
`start` call. Don't treat the wait as a hang. After it returns, retrieve
results with `wait_for_findings` (the server polls for you).

**4. The Bearer is the user's Doable API key — carried as data, never logged.**
It rides in the MCP client's `Authorization` header, is forwarded to
Doable, and is never persisted. Don't print it or write it to files.

**5. Recover from errors, don't loop.**
If a tool errors, read the error `code` (see
[references/troubleshooting.md](references/troubleshooting.md)) and act on
it. Retrying the same failing call rarely helps. After 2–3 failed
attempts, stop and surface the problem to the user.

## When to use this skill

Use it when the user wants to confirm a **deployed** feature behaves
correctly for end users — e.g. "I just shipped the checkout flow, does it
actually work?", "verify my live site against this spec", or any request
naming the `mcp__reflex__*` tools. It is **not** for running local tests.

## The workflow (follow instructions; this is the shape)

```
init ─► start ─► wait_for_findings ─► get_findings ─► (recommendation?)
                                                  │
                    pass / skip_fix ──────────────┤
                                                  ▼
                    fix needed: submit_fix(new_base_url) ─► wait_for_findings ─► … (capped loop)
                                                  ▼
                              finalize(pass|skip_fix|fail) ─► get_report
```

- **Phases:** `initialized → tested → fix_attempted → verified → reported`
  (terminal). Any non-terminal phase can also reach `canceled` (user
  abandons) or `failed` (fix cap exhausted / timed out).
- **What you need from the user up front:** the deployed URL and the PRD
  (a JSON object or a path to one). If a fix step is reached, you'll need
  the URL of their corrected redeploy for `submit_fix`.

Full step-by-step, decision rules, and what each response carries:
[references/workflow.md](references/workflow.md).

## The ten tools

Each Doable-touching tool accepts an optional `doable_api_key`; omit it to
use the session Bearer. `run_id` is the UUID returned by `init`.

| Tool | Required args | Purpose |
|---|---|---|
| `mcp__reflex__init` | — | Create a run + Doable suite. Returns `run_id` + first instruction. |
| `mcp__reflex__start` | `run_id`, `prd`, `base_url` | Bind PRD + URL, kick off the initial test. **Blocks 1–5 min.** |
| `mcp__reflex__wait_for_findings` | `run_id` | Server-side poll loop until results / terminal phase. |
| `mcp__reflex__get_findings` | `run_id` | Findings + recommendation + suggested prompts (`kind=initial\|verify`). |
| `mcp__reflex__get_status` | `run_id` | One Doable poll + state update. Safe from any phase. |
| `mcp__reflex__submit_fix` | `run_id`, `new_base_url` | You fixed + redeployed: trigger a verify against the new URL. |
| `mcp__reflex__finalize` | `run_id`, `decision` | Terminal verdict: `pass` / `skip_fix` / `fail`. |
| `mcp__reflex__cancel` | `run_id` | Terminate the run as `canceled`. |
| `mcp__reflex__get_run` | `run_id` | Read-only run snapshot (no Doable call). |
| `mcp__reflex__get_report` | `run_id` | Final report, `format=md` (default) or `json`. Terminal phases only. |

## Connecting (one-time setup)

The user's MCP client points at the Reflex server with their Doable Bearer
in the `Authorization` header. For Claude Code, `.mcp.json`:

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

Local dev points `url` at `http://127.0.0.1:8000/mcp`. `.mcp.json` is
typically gitignored, so hardcoding the key there is fine; the alternative
is a `${QA_DOABLE_API_KEY}` placeholder exported in the launch shell.
Verify with `claude mcp list` → `reflex: ✓ Connected` with no "Missing
environment variables" warning. Setup details and key-handling options:
[references/connect.md](references/connect.md).

## Reporting back to the user

When the run is terminal, summarize: the final verdict, the findings and
their resolutions, and the report (`get_report`). If the run `failed`
because the fix cap was reached with issues still open, say which issues
remain.
