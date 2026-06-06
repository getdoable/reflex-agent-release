# Reflex troubleshooting — error codes & known issues

All errors use the canonical envelope `{code, message, hint?, run_id?}`.
Read the `code` and act on it; don't blindly retry.

## Error codes

| Code | Meaning | What to do |
|---|---|---|
| `missing_doable_key` | No Bearer reached the server | The MCP client launched without the Doable key. Hardcode it in `.mcp.json`, or export `QA_DOABLE_API_KEY` and relaunch the client. See connect.md. |
| `doable_auth_rejected` | Doable rejected the Bearer (401) | The key is invalid or not valid for the target org. Get a fresh Doable API key. |
| `doable_timeout` | Doable transport cut off (504) | Retry once. If `start` repeatedly times out, the deployed URL may be unreachable by Doable (see "page not reachable" below). |
| `doable_unreachable` | Can't reach the Doable backend (502) | Network / endpoint problem upstream. Retry; if persistent, the Doable service may be down. |
| `doable_protocol_error` | Doable returned a malformed/unexpected response (502) | Usually transient — retry. If it persists, the Doable API contract may have changed. |
| `validation_failed` | Bad request body (422) | Fix the argument shape — malformed URL, malformed PRD. |
| `payload_too_large` | PRD exceeds the size limit (413) | Trim the PRD (default limit 256 KB). |
| `run_not_found` | Unknown `run_id` (404) | Use the `run_id` returned by `init`. |
| `phase_conflict` | Tool not valid for the run's current phase (409) | Follow the `instruction` — call what it tells you, not what you guessed. |
| `already_started` | `start` called on a run that's already starting/started (409) | `start` is one-shot — don't retry it. If `start` timed out client-side it likely still succeeded; poll `get_status`/`get_run`. To verify again, `init` a new run. |
| `fix_attempt_limit_reached` | `submit_fix` past the cap | No more fix attempts allowed. `finalize` with `fail` (or `skip_fix` if remaining issues are acceptable). |
| `not_yet_available` | Report/findings requested too early | Wait for the phase the instruction names (use `wait_for_findings`). |
| `rate_limited` | Per-IP quota exceeded (429) | Honor `Retry-After` (delta-seconds), then retry. |
| `run_timed_out` | Run exceeded the hard ceiling | The run auto-failed and won't recover. Start a fresh run. |

## "The page isn't reachable / Doable sees a login wall"

Doable drives a real browser from outside your network. The deployed URL
must be **publicly reachable with no auth gate**:

- **Vercel:** disable deployment protection for the target project/deploy
  (the preview/SSO gate blocks Doable). Production aliases on a public
  project work; protected previews don't.
- **Other hosts:** make sure there's no basic-auth, IP allowlist, or
  password page in front of the deployment.

A `start` that repeatedly `doable_timeout`s on an otherwise healthy
service is the usual symptom of an unreachable/gated URL.

## `start` "hangs"

`start` blocks 1–5 minutes by design (Doable generates test cases
synchronously). This is normal — wait for it to return. Only treat it as a
problem if it errors or runs well past ~5 minutes.

## `wait_for_findings` runs a very long time

The browser test run itself can take several minutes. If it runs well
past ~10 minutes with no progress, the upstream test run may have stalled.
`cancel` the run and start a fresh one. (Reflex also auto-fails runs that
exceed the hard time ceiling — see staleness in workflow.md.)

## Connection shows the tool but calls fail with `missing_doable_key`

The MCP server is reachable but no Bearer is being forwarded. The client
resolved the `Authorization` header to empty — fix the key (hardcode in
`.mcp.json` or export the env var) and reconnect. See connect.md.
