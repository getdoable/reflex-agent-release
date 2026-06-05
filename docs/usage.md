# Reflex MCP — Usage Guide

**Audience:** AI coding agents (Claude Code, Cursor, etc.) and the developers
wiring them up to verify a deployed web feature through Reflex's Model Context
Protocol (MCP) server.

For the fast path, see the [README](../README.md). This is the complete
reference.

---

## 1. What this is

Reflex is a **post-deployment verification service**. You give it a PRD (what
the feature should do) and a live URL; it drives Doable's browser-based
testing on your behalf and tells you — in plain, machine-followable
instructions — whether the feature actually works for real users, and what to
fix if it doesn't.

The MCP server exposes Reflex as **ten `reflex__*` tools** at `POST /mcp`. Your
agent calls them; each response carries an `instruction` field telling the
agent which tool to call next. **You never hardcode the workflow** — you follow
the instructions until the run reaches a terminal phase.

```
your agent ──MCP──► Reflex MCP server ──► Doable ──► real browser ──► your live URL
                  (reflex.mcp.getdoable.ai)
```

### Key properties

| Property | Meaning for you |
|---|---|
| **No-auth service** | Reflex has no accounts or API keys of its own. |
| **Bearer-as-data** | Your **Doable** API key rides in the `Authorization` header per request, is forwarded to Doable, and is **never persisted or logged**. |
| **Instruction-driven** | Read `instruction.next_action` / `next_endpoint` in every response and act on it. |
| **Synchronous** | All Doable work happens inline. `start` blocks 1–5 min; there are no webhooks or background jobs. |
| **Stateful per run** | Each run is identified by a `run_id` returned by `init`. |

---

## 2. Prerequisites

1. A **Doable API key** (Bearer). The only credential involved.
2. A **deployed, publicly reachable** feature URL. If it's on Vercel, disable
   deployment protection for that project so Doable's browsers can reach it.
3. Your MCP client connected to Reflex (see §3).

---

## 3. Connecting

The Reflex MCP server is a **hosted endpoint** (`https://reflex.mcp.getdoable.ai/mcp`),
so it isn't tied to any folder — register it once in the scope you want and it
works from your own project directory. See the [README](../README.md) for the
three install paths (plugin / `claude mcp add` / project `.mcp.json`); in short:

- **All projects:** `claude mcp add --transport http --scope user reflex
  https://reflex.mcp.getdoable.ai/mcp --header "Authorization: Bearer <key>"`,
  or install the Claude Code plugin (registers server + skill).
- **One project:** put this block in that project's `.mcp.json` (the release
  repo ships it as `.mcp.json.example` — copy to `.mcp.json`, which is
  gitignored, and set your key):

```json
{
  "mcpServers": {
    "reflex": {
      "type": "http",
      "url": "https://reflex.mcp.getdoable.ai/mcp",
      "timeout": 660000,
      "headers": { "Authorization": "Bearer ${QA_DOABLE_API_KEY}" }
    }
  }
}
```

> **Tool-call timeout:** the `timeout` field (milliseconds) is required-in-
> practice — `start` blocks 1–5 min, so without a generous per-server timeout
> the client cuts the call off. `660000` (11 min) covers it. `MCP_TIMEOUT`
> only governs server startup, not tool execution.

Supply your Doable key one of two ways:

**(a) Environment variable (recommended).** Keep the `${QA_DOABLE_API_KEY}`
placeholder and export the value in the shell you launch your agent from:

```bash
cp .env.example .env        # set QA_DOABLE_API_KEY=...
set -a; source .env; set +a
claude                      # launch from THIS shell
```

Env vars don't cross terminal sessions — a new tab needs the export again. If
the var is unset, the Bearer resolves to empty and Doable-touching tools fail
with `missing_doable_key`.

**(b) Hardcode.** Replace `${QA_DOABLE_API_KEY}` in `.mcp.json` with your
literal key. No per-terminal step; just don't commit the edited file.

> For Claude Code, a third option is `"env": {"QA_DOABLE_API_KEY": "..."}` in
> `.claude/settings.local.json` (gitignored) — no per-terminal step, key out
> of `.mcp.json`.

Verify:

```bash
claude mcp list
# reflex: https://reflex.mcp.getdoable.ai/mcp (HTTP) - ✓ Connected
```

Your client now exposes ten tools, namespaced `mcp__reflex__*`.

---

## 4. The ten tools

Each Doable-touching tool accepts an optional `doable_api_key`; omit it to use
the session-level `Authorization` Bearer (the normal case). All `run_id`
values are the UUID returned by `init`.

| Tool | Required args | Purpose |
|---|---|---|
| `reflex__init` | — | Create a run + Doable suite. Returns `run_id` + first instruction. |
| `reflex__start` | `run_id`, `prd`, `base_url` | Bind PRD + URL, kick off the initial Doable test. **Blocks 1–5 min.** |
| `reflex__get_status` | `run_id` | One Doable poll + state update. Safe from any phase; may surface findings. |
| `reflex__wait_for_findings` | `run_id` | Server-side poll loop until a results/terminal phase or `max_wait_seconds`. |
| `reflex__get_findings` | `run_id` | Findings + recommendation + suggested prompts. `kind=initial\|verify`. |
| `reflex__submit_fix` | `run_id`, `new_base_url` | You fixed + redeployed: trigger a Doable verify against the new URL. |
| `reflex__finalize` | `run_id`, `decision` | Terminal verdict: `pass` / `skip_fix` / `fail`. |
| `reflex__cancel` | `run_id` | Terminate the run as `canceled`. |
| `reflex__get_run` | `run_id` | Read-only run snapshot. No Bearer, no Doable call. |
| `reflex__get_report` | `run_id` | Final report, `format=md` (default) or `json`. Terminal phases only. |

### Optional args worth knowing

- `reflex__start` → `prd` accepts an object **or** a string; `base_url` is the
  deployed feature URL.
- `reflex__submit_fix` → `new_base_url` (corrected deploy), `commit_sha`,
  `notes`. Increments `fix_attempt_count`, capped at the server's fix cap
  (default 2).
- `reflex__wait_for_findings` → `max_wait_seconds` (default 1800; **capped at 1800 = 30 min**, min 60), `poll_interval_seconds` (10–600). To wait longer than 30 min, call it again.
- `reflex__get_status` → `since` (last-seen `event_id`; replays only newer
  events).

---

## 5. The workflow

```
init ──► start ──► (wait_for_findings | get_status) ──► get_findings
                                                            │
                          ┌─────────────── recommendation ──┤
                          ▼                                  ▼
                  fix needed?                          no fix needed
                  submit_fix ──► wait_for_findings ──► finalize(pass|skip_fix)
                          │                                  │
                          ▼                                  ▼
                  (loop, capped)                        get_report
                  finalize(fail) ◄── cap reached
```

**Phases:** `initialized → tested → fix_attempted → verified → reported`
(terminal). Any non-terminal phase can also reach `canceled` (you call
`cancel`) or `failed` (cap exhausted / unrecoverable / timed out).

### Step by step

1. **`init`** → get `run_id` + a PRD template + the first instruction.
2. **`start`** with `prd` + `base_url`. Blocks while Doable's LLM generates
   test cases (1–5 min). Returns `test_case_ids` and a polling instruction.
3. **Retrieve results** as the instruction directs — `wait_for_findings`
   (server polls for you) is simplest; or poll `get_status` yourself.
4. **`get_findings`** → read the findings, severities, the engine's
   `recommendation` (`pass` / `fix` / `skip_fix`), and `must_fix_ids`.
5. **If a fix is recommended:** apply your fix, redeploy, then **`submit_fix`**
   with `new_base_url`. Repeat the retrieve → findings loop. Reflex re-runs
   only the failed-and-blocking test cases and maps each back to `fixed` /
   `still_open` / `not_retested`.
6. **`finalize`** with the terminal `decision` the instruction indicates
   (`pass`, `skip_fix`, or `fail`).
7. **`get_report`** for the full markdown (or JSON) report.

**Golden rule:** after every tool call, read `instruction.next_action` and do
what it says. The engine adapts the sequence to the findings and the fix cap.

---

## 6. Decision rules (what the engine recommends)

- **After the initial test:** zero findings → `pass`; any critical/high →
  `fix`; only medium/low → `skip_fix`.
- **After a verify:** if blocking issues remain **and** attempts < cap → `fix`
  again; if all `fixed` → `pass`; cap reached with issues still open → `fail`.

The fix-attempt cap defaults to 2.

---

## 7. Errors & edge cases

All errors use the envelope `{code, message, hint?, run_id?}`.

| Code | Meaning | What to do |
|---|---|---|
| `missing_doable_key` | No Bearer reached the server | Set your Doable key (§3) and reconnect. |
| `doable_auth_rejected` | Doable rejected the Bearer (401) | Check the key is valid for the target org. |
| `doable_timeout` | Doable transport cut off (504) | Retry; if `start` repeatedly times out, the deploy URL may be unreachable/gated. |
| `doable_unreachable` | Can't reach the Doable backend (502) | Network issue upstream; retry. |
| `doable_protocol_error` | Doable returned a malformed/unexpected response (502) | Usually transient — retry. If it persists, the Doable API contract may have changed. |
| `validation_failed` | Bad request body (422) | Fix the argument shape (bad URL, malformed PRD). |
| `payload_too_large` | PRD exceeds the size limit (413) | Trim the PRD (default limit 256 KB). |
| `run_not_found` | Unknown `run_id` (404) | Use the `run_id` from `init`. |
| `phase_conflict` | Tool not valid for the current phase (409) | Follow the `instruction` — call what it tells you. |
| `fix_attempt_limit_reached` | `submit_fix` past the cap | `finalize` with `fail` (or `skip_fix`); no more fix attempts. |
| `not_yet_available` | Report/findings requested too early | Wait for the phase the instruction names. |
| `rate_limited` | Quota exceeded (429) | Honor `Retry-After` (delta-seconds). |
| `run_timed_out` | Run exceeded the hard ceiling | The run auto-failed; start a fresh run. |

### Completion & known Doable-side behaviors

- **A run completes when there's no active work** — Doable cancels cases whose
  prerequisite failed and leaves them unrun; those don't block the run. It
  finalizes on what actually ran (a blocking failure still yields a `fix`
  recommendation) instead of waiting out the timeout.
- **Initial test generation can take the full 5 minutes** — `start` blocking
  is expected.
- **The browser run can stall.** If `wait_for_findings` runs well past ~10 min
  with no progress, `cancel` the run and start fresh. Reflex also auto-fails
  runs that exceed the hard time ceiling (~24 h backstop).

---

## 8. Example agent prompts

### 8.1 The intended UX — a "real user" prompt (recommended)

Because the service is instruction-driven, you should **not** need to know the
tools, the order, or the decision rules. Stating only the goal and the inputs
is enough — the `instruction` field carries the rest.

> I just deployed a feature and I want to confirm it actually works for real
> users before I call it done. Use the Reflex tools (`mcp__reflex__*`) to
> check my deployment.
>
> - My deployed app: `<your live URL>`
> - What it's supposed to do: `<paste the PRD, or point to a PRD file>`
>
> I don't know the exact steps — just use the tools and walk me through
> whatever the service tells you to do next, until you reach a final result.
> If something's broken and I need to fix and redeploy, assume my corrected
> redeploy is at `<your redeployed URL>` and continue from there.
>
> When you're finished, tell me the final verdict and show me the report.

### 8.2 Explicit variant (when you want to steer)

> Use the Reflex MCP tools (`mcp__reflex__*`) to verify my deployed feature.
> It's instruction-driven — every response has an `instruction` field telling
> you the next call; follow it until the run hits a terminal phase. Don't
> hardcode the sequence.
>
> - Deployed URL: `<your live URL>`
> - PRD: `<path to your PRD JSON, or inline the PRD object>`
>
> Flow: `init` → `start` (blocks 1–5 min, that's normal) → retrieve findings
> (`wait_for_findings`). If a fix is recommended, apply it, redeploy, then
> `submit_fix` with `new_base_url` and continue the verify loop. `finalize`
> when directed, then `get_report`.
>
> At the end, report: run_id, phases traversed, findings, resolutions, and the
> final verdict.

---

## 9. See also

- The skill: [`../skills/reflex/SKILL.md`](../skills/reflex/SKILL.md) and its
  references (`workflow.md`, `troubleshooting.md`, `connect.md`).
- The Agent Skills format: <https://agentskills.io/>
