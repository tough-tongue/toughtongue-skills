# Ingest and Reprocess

Two write paths. The MCP server marks both as reaching outside the account (they
can fire the Scenario's webhooks). Confirm each one with the user before
calling.

## Contents

- Confirm before acting
- Fill missing analysis: `ttai:post_process_session`
- Score an external call: `ttai:create_session`
- Poll for results
- Automated post-call coaching

## Confirm before acting

State the exact target and wait for an explicit yes:

- `ttai:post_process_session`: the session IDs, their Scenario, and what is
  missing. Example: "Run analysis on 7 sessions of Scenario `<id>` that have no
  evaluation? This can fire the Scenario's post-session webhooks."
- `ttai:create_session`: the Scenario ID, the participant name or email, and the
  transcript or recording source. Recording URLs are fetched by the server, and
  an organization's session-completed webhook fires.

Never infer consent from an earlier, broader request such as "build me a
report".

## Fill missing analysis: `ttai:post_process_session`

Arguments: `session_id`.

- The Scenario's analysis settings decide what runs: evaluation, variable
  extraction, and Scenario learning. If none is configured, the call does
  nothing and still returns OK.
- It fills only what is missing. A session that already has `evaluation_results`
  is not re-scored. It cannot regrade under a changed rubric.
- It runs in the background. An OK response only means the attempt was queued.
  If processing is already `running`, the attempt is skipped.
- Use it for sessions with no `evaluation_results`, or after
  `post_session_status.state` is `failed`.
- Do not call it while the state is `running`, and do not loop it.

## Score an external call: `ttai:create_session`

Arguments: `scenario_id` (required), plus `transcript` or `recording_url` (at
least one; a recording URL overrides the transcript). Optional: `user_name`,
`user_email`, `user_metadata` (filterable string, number, or bool values),
`metadata`, `dynamic_vars`, `force_analysis`.

- Returns `session_id`, `analytics_url`, and `created_at`. The session is
  created as `completed`.
- Analysis is queued automatically when the Scenario has analysis enabled. Do
  not follow up with `ttai:post_process_session`; poll instead.
- Requires VIEW access to the Scenario. An error mentioning balance means the
  workspace balance is too low. Report it and stop.
- The Scenario's rubric scores the call. Use a coaching Scenario whose rubric
  evaluates the rep on real calls. If none exists, hand off to the `ttai-agent`
  skill to build one, then reuse its ID for every call.

## Poll for results

Poll with `ttai:v3_list_sessions` (`ids`,
`include_fields: ["evaluation", "processing"]`) or `ttai:get_session`. Back off
between reads, starting at about 15 seconds, and stop after a few minutes.

- **`state: running`** — in progress. Keep polling.
- **`state: failed`** — the last run errored (`failed_reason`). Report it. Retry
  once with `ttai:post_process_session` only if the user approves.
- **`state: idle` and `evaluation_results` present** — done. Build the report.
- **`state: idle`, no `evaluation_results` after a few polls** — not configured,
  or nothing ran. Tell the user the Scenario may not have analysis enabled. Do
  not retry in a loop.

## Automated post-call coaching

For "every real sales call gets a coaching report" pipelines, such as one
triggered by a call-recording platform's webhook:

1. **Ingest.** `ttai:create_session` with the call's transcript or recording URL
   against the coaching Scenario. Put the rep's email in `user_email` and the
   external call ID in `user_metadata`.
2. **Poll** as described above.
3. **Format** the Individual coaching report from `evaluation_results` and
   `improvement_results`.
4. **Deliver** through the team's email or chat tooling, with the
   `analytics_url`.

Unattended pipelines authenticate with the `TTAI_PAT` environment variable. Keep
it in the server environment, never in client code or webhook payloads. The
confirmation step happens once, when the user approves the pipeline.
