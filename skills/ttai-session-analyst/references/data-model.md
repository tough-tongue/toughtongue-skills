# Session Data Model

## Contents

- Read tools and parameters
- Session fields by tool
- Evaluation shape and score scales
- Extraction results
- Status values
- Phone call and meeting-bot records
- Analytics rollup

All account tools accept an optional `org_id` (an `organizations[].id` from
`ttai:get_workspace_info`). Omit it for personal scope. Date filters take an ISO
8601 date or timestamp. A date-only lower bound starts at 00:00:00 UTC and a
date-only upper bound ends at 23:59:59.999999 UTC.

## Read tools and parameters

Session and bot reads are `ttai:list_resources` or `ttai:get_resource`;
type-specific parameters go in `filters`. Usage is `ttai:get_workspace_info`. Paging at a glance:

| Read                                   | Paging | `limit`           |
| -------------------------------------- | ------ | ----------------- |
| list `sessions` (typed)                | cursor | 1–100, default 25 |
| list `sessions` with a legacy filter   | cursor | 1–500, default 25 |
| get `sessions`                         | none   | —                 |
| workspace `usage`                      | none   | —                 |
| list `bots`                            | cursor | 1–100, default 25 |

**List `sessions` (typed)**

- Filters: `ids`, `scenario_ids`, `statuses`. Plus `include_fields`, `cursor`,
  `limit`, `include_total`.
- Newest first by `created_at`. `total` only with `include_total: true`.

**List `sessions` (legacy enriched)** — any of these filters selects it:

- Filters: `user_email` (comma-separated allowed), `hasLearning` (`with` /
  `issue` / `ok` / `none`), `from_date`, `to_date`, `is_org`; at most one
  `scenario_ids` value. No `query` or `include_fields`.
- Paging: pass `next_cursor` back as `cursor`; `limit` ≤ 500; `total` with
  `include_total: true`.
- Dates filter `updated_at`. `is_org: true` returns org-wide data; needs
  `org_id` and an OWNER or EDIT org role, otherwise the call is rejected.

**Get `sessions`**

- `id`, optional `include_fields` (`participant`, `recording` →
  `recording_present`, `transcript`, `evaluation`, `extraction`, `processing`).
  `recording_url` instead returns the enriched record with a signed recording
  URL.

**Workspace `usage`** — `ttai:get_workspace_info(sections: ["usage"])`

- Arguments: `org_id`, `start_date`, `end_date`.
- Default window is the last 90 days. Org-wide view is automatic for EDIT or
  OWNER org roles; otherwise it is your own usage in the scope.

**List `bots`** — meeting bots and phone calls in one view

- Filters: `ids`, `scenario_ids`, `session_ids`, `batch_ids`, `statuses`,
  `kinds` (`meeting`, `phone`), `phone_directions` (`inbound`, `outbound`),
  `created_at_gte`, `created_at_lte`. `include_fields`: `meeting`, `failure`.
- Bot records, not sessions.

The typed `sessions` list with `scenario_ids` covers other people's sessions
only when the caller holds EDIT or higher on the Scenario, or org-wide EDIT in
the selected organization. Otherwise it returns the caller's own sessions.

## Session fields by tool

**Typed list and get.** Always returned: `id`, `scenario_id`,
`scenario_version_id`, `status`, `created_at`, `updated_at`, `completed_at`.
Opt-in via `include_fields`:

- `participant` — `user_name`, `user_email`.
- `evaluation` — `evaluation_results`, `improvement_results`,
  `learning_results`.
- `transcript` — `transcript` (finalized, or the running transcript if not
  finalized).
- `extraction` — `extraction_results`.
- `recording` — `recording_present`.
- `processing` — `post_session_status`.

Typed reads also return `duration_seconds` (`created_at` to `completed_at`; on
a phone call that includes ringing), but not `analytics_url` or
`scenario_name`. A field you did not request comes back `null`.

**Legacy enriched list.** `id`, `scenario_id`,
`scenario_name`, `created_at`, `completed_at`, `status`, `user_name`,
`user_email`, `duration` (seconds), `analytics_url`, `recording_present`,
`metadata`, `user_metadata`, `evaluation_results`, `improvement_results`,
`extraction_results`, `learning_results`. No transcript and no
`post_session_status`. Ignore the deprecated flat fields `evaluation_score`,
`report_card`, `report_card_topics`, `duration_minutes`, and `evaluation_note`;
read the nested `evaluation_results` instead. On phone sessions, `user_email`
holds the dialled number and `metadata.dynamic_vars` the call's variables;
`duration` measures differently from the typed `duration_seconds`, so never
mix the two.

## Evaluation shape and score scales

```
evaluation_results:
  final_score: number        # 0–10, weight-averaged across report_card
  overall_score: string      # 1–2 sentence verdict (text, not a number)
  strengths: string          # markdown bullets
  weaknesses: string         # markdown bullets
  detailed_feedback: string  # per-topic markdown
  report_card:
    - topic: string          # rubric criterion, stable within a Scenario
      score: number          # 0–10
      score_str: string      # display form, such as a grade or "7/10"
      weight: number         # percent share of final_score
      note: string
improvement_results:
  improvement_areas: string  # markdown
  action_items: string       # markdown
  resources: string          # markdown
learning_results:            # present when Scenario learning is enabled
  status: ok | issue
  issue, evidence, suggestion, summary: string
```

Any of these can be `null`.

## Extraction results

Scenarios that record outcomes instead of scores turn on
`session_analysis.enable_extraction` and list `extraction_vars`: `name`,
`description`, and `type` (`text`, `number`, `boolean`, `list`, `date`). Many
voice agents that take calls also have no rubric and
`is_auto_analysis: false`, so their sessions never carry
`evaluation_results`. Read the variables with
`ttai:get_resource(scenarios, id)`.

```
extraction_results:   # typed: include_fields ["extraction"]; legacy: always
  <extraction_vars[].name>: value | null
```

- **Outcome variable.** The one whose `description` lists its values ("Return
  exactly one of: …"). Other variables hold a fixed format (a callback date
  and time), a count, a language, or free text such as a summary for the
  person who follows up.
- **Null.** Every declared variable comes back as a key, `null` when there was
  nothing to extract. Treat a missing key as `null`. A declared string value
  such as `"None"` or `"Unknown"` is an answer, not a null.
- **Dials that never connected** still get a session whose variables are all
  `null`. Take reach from the bots, not from sessions.
- **`extraction_results: null`** on a row where you requested it means no
  extraction ran: check `post_session_status` and the Scenario's
  `enable_extraction`.
- **Off-list and off-format values.** Values are model output: expect a
  different case, extra words, or a sentence where a time belongs. Match the
  declared list ignoring case and spacing; count the rest as off-list or
  off-format and quote one example.
- `boolean` variables arrive as JSON `true` / `false`.

## Status values

- Session `status`: `active`, `completed`, `archived`, `terminated`. The
  legacy list reports an `active` session older than 40 minutes as
  `terminated`; typed reads return and filter the stored status.
- `post_session_status.state`: `idle` (nothing running; results may or may not
  exist), `running` (processing holds the lock), `failed` (the last run errored;
  see `failed_reason`). `start_ts` marks the last run.

## Phone call and meeting-bot records

`bots` records: `id`, `kind` (`meeting` / `phone`), `phone_direction`
(`outbound` dialled, `inbound` received), `scenario_id`,
`scenario_version_id`, `session_id`, `status`, `batch_id`,
`scheduled_ts`, `bot_joined_at`, `call_ended_at`, `created_at`, `updated_at`;
`meeting_provider` and `bot_name` with `include_fields: ["meeting"]`;
`failure_reason` (`busy`, `no_answer`, `voicemail`, `unavailable`,
`service_error`, `unknown`, …) with `["failure"]`.
Phone numbers and meeting URLs are never returned.

- `status` values: `pending`, `scheduled`, `joining_call`, `in_waiting_room`,
  `in_call_not_recording`, `in_call_recording`, `call_ended`, `done`,
  `terminated`, `failed`.
- An answered phone call reads `call_ended` with `bot_joined_at` set. A dial
  that never connected reads `failed` with a `failure_reason`; the agent never
  spoke on it.
- A voicemail can read `call_ended`: the line was answered. The Scenario's
  outcome variable, not `failure_reason`, marks it.
- `call_ended_at` can be empty on ended calls. Take call length from the
  session.

To score calls or read their outcomes, collect their `session_id` values and
read those sessions. Failed dials have a `session_id` too, with nothing in it;
records without one have no session to read.

## Analytics rollup

`ttai:get_workspace_info(sections: ["usage"])` returns, under `usage`:

- `scenarios[]`: `name`, `session_count`, `total_duration_seconds`.
- `org_dashboard` (org-wide only): `summary` (`total_sessions`,
  `completed_sessions`, `total_minutes`, `wallet_balance`), `time_series[]`
  (`date`, `session_count`, `total_minutes`).
- `member_usage.members[]` (org-wide only): `name`, `email`, `session_count`,
  `total_minutes`.

It has no scores. Use it for volume and engagement. Use session reads for
performance.
