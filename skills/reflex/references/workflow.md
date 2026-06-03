# Reflex workflow — detailed steps & decision rules

Read this when you need the full step-by-step, what each response
carries, or how the engine decides "fix vs done". The golden rule still
holds: **after every call, follow the `instruction` field** — this page
explains *why* it points where it does, not a sequence to hardcode.

## Step by step

1. **`init`** → returns `run_id` (UUID), a PRD template, and the first
   `instruction` (normally: call `start`). Keep the `run_id`.

2. **`start`** with `run_id`, `prd` (object or string), `base_url` (the
   deployed URL). **Blocks 1–5 minutes** while Doable's LLM generates
   browser test cases. Returns `test_case_ids` and a polling instruction.

3. **Retrieve results.** Follow the instruction — usually
   **`wait_for_findings`**, which polls Doable server-side until the run
   reaches a results-bearing or terminal phase (optional
   `max_wait_seconds` — default 1800, capped at 1800 = 30 min; and
   `poll_interval_seconds`, 10–600). If it returns still non-terminal,
   call it again. You can instead poll `get_status` yourself if you want
   event-by-event progress.

4. **`get_findings`** → the findings (each with `severity`,
   `test_case_id`, title, expected/actual, repro steps), the engine's
   `recommendation`, `must_fix_ids`, and `suggested_prompts`.

5. **If a fix is recommended:** apply the fix, redeploy, then
   **`submit_fix`** with `new_base_url` (the corrected deploy);
   optionally `commit_sha`, `notes`. Reflex re-runs only the
   failed-and-blocking test cases and maps each result back to a
   resolution: `fixed` / `still_open` / `not_retested`. Then go back to
   step 3 to retrieve the verify results. The loop is capped (see below).

6. **`finalize`** with the terminal `decision` the instruction indicates:
   - `pass` — verification accepted.
   - `skip_fix` — known non-blocking findings, ship anyway.
   - `fail` — reject the deploy (e.g. blocking issues remain).
   (`fix` is **not** a finalize value — that's what `submit_fix` is for.)

7. **`get_report`** (`format=md` default, or `json`) for the full
   decision trail, findings, resolutions, and test-execution summary.
   Only available in terminal phases.

## Decision rules (what `recommendation` will say)

**After the initial test:**
- zero findings → `pass`
- any critical/high finding → `fix`
- only medium/low findings → `skip_fix`

**After a verify (`submit_fix`):**
- blocking issues still open **and** attempts < cap → `fix` again
- all retested issues `fixed` → `pass`
- cap reached with issues still open → `fail`

The fix-attempt cap is the server's `REFLEX_MAX_FIX_ATTEMPTS` (default 2).

## State machine

```
initialized → tested → fix_attempted → verified → reported (terminal)
                            ▲              │
                            └──────────────┘  (capped fix→verify loop)

any non-terminal phase ──► canceled   (user calls cancel)
any non-terminal phase ──► failed     (cap exhausted / unrecoverable / timed out)
```

## `test_case_id` is the stable link

Initial findings carry a `test_case_id`. On `submit_fix`, Reflex re-runs
those specific failed-and-blocking cases and keys the resolutions back by
`test_case_id`. This is how "was *this* bug fixed?" is answered precisely
rather than by re-running everything.

## Staleness / timeouts

A non-terminal run that goes too long is handled automatically on read:
past a soft threshold it's flagged `stale: true` with advice; past a hard
ceiling the next `get_status`/`wait_for_findings` finalizes it to `failed`
(`run_timed_out`). If you see this, the run won't recover — start a fresh
run.
