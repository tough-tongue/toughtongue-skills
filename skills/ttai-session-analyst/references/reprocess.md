# Reprocess

One write path: `ttai:analyze_session`. The MCP server marks it as reaching
outside the account (it can fire the Scenario's webhooks). Confirm it with the
user before calling.

## Contents

- Confirm before acting
- Fill missing analysis: `ttai:analyze_session`
- Poll for results
- External calls

## Confirm before acting

State the exact target and wait for an explicit yes: the session IDs, their
Scenario, and what is missing. Example: "Run analysis on 7 sessions of Scenario
`<id>` that have no evaluation? This can fire the Scenario's post-session
webhooks."

Never infer consent from an earlier, broader request such as "build me a
report".

## Fill missing analysis: `ttai:analyze_session`

Argument: `session_id` only.

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
- A 429 "Analysis limit reached for this session." is final: report it. Any
  other 429 is a rate limit: wait, then retry.

## Poll for results

Poll with `ttai:list_resources(sessions)` (`ids`,
`include_fields: ["evaluation", "processing"]`) or `ttai:get_resource(sessions)`. Back off
between reads, starting at about 15 seconds, and stop after a few minutes.

- **`state: running`** — in progress. Keep polling.
- **`state: failed`** — the last run errored (`failed_reason`). Report it. Retry
  once with `ttai:analyze_session` only if the user approves.
- **`state: idle` and `evaluation_results` present** — done. Build the report.
- **`state: idle`, no `evaluation_results` after a few polls** — not configured,
  or nothing ran. Tell the user the Scenario may not have analysis enabled. Do
  not retry in a loop.

## External calls

No MCP tool ingests an external call transcript or recording. Scoring one is a
REST API call that needs an API token set up outside this skill: send the user
to [Create session](https://app.toughtongueai.com/docs/api-reference/sessions/create).
Once it exists, its session reads and reprocesses like any other.
