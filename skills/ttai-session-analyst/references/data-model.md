# Session Data Model

## Contents

- Read tools and parameters
- Session fields by tool
- Evaluation shape and score scales
- Status values
- Phone call and meeting-bot records
- Analytics rollup

All account tools accept an optional `org_id` (the `id` from
`ttai:list_organizations`). Omit it for personal scope. Date filters take an ISO
8601 date or timestamp. A date-only lower bound starts at 00:00:00 UTC and a
date-only upper bound ends at 23:59:59.999999 UTC.

## Read tools and parameters

Paging at a glance:

| Tool                      | Paging | `limit`               |
| ------------------------- | ------ | --------------------- |
| `ttai:v3_list_sessions`   | cursor | 1–100, default 25     |
| `ttai:list_sessions`      | page   | default 50, max 500   |
| `ttai:get_sessions_batch` | none   | `session_ids` max 500 |
| `ttai:get_session`        | none   | —                     |
| `ttai:get_analytics`      | none   | —                     |
| `ttai:list_sip_calls`     | page   | default 50, max 500   |
| `ttai:list_meeting_bots`  | page   | default 50, max 500   |

**`ttai:v3_list_sessions`**

- Parameters: `ids[]`, `scenario_ids[]`, `statuses[]`, `include_fields[]`,
  `cursor`, `limit`, `include_total`.
- Paging: cursor. `limit` 1–100, default 25. Follow `next_cursor`.
- Newest first by `created_at`. `total` only with `include_total: true`. Typed
  output.

**`ttai:list_sessions`**

- Parameters: `scenario_id`, `user_email` (comma-separated allowed),
  `hasLearning` (`with` / `issue` / `ok` / `none`), `from_date`, `to_date`,
  `is_org`, `page`, `limit`.
- Paging: page. `page` from 1. `limit` default 50, max 500. `page_meta.total`.
- Dates filter `updated_at`. `is_org: true` returns org-wide data; needs
  `org_id` and an OWNER or EDIT org role, otherwise the call is rejected.
  Returns JSON text.

**`ttai:get_sessions_batch`**

- Parameters: `session_ids[]` (max 500). No paging.
- Same shape as `ttai:list_sessions`. No transcript.

**`ttai:get_session`**

- Parameters: `session_id`, `include_recording_url` (default `true`). No paging.
- Full detail, including transcript.

**`ttai:get_analytics`**

- Parameters: `is_org_wide`, `start_date`, `end_date`. No paging.
- Default window is the last 90 days. Org-wide view needs an EDIT or OWNER role.

**`ttai:list_sip_calls`**

- Parameters: `call_type` (`sip_call`, `sip_inbound`, comma-separated),
  `status`, `scenario_id`, `batch_id`, `from_date`, `to_date`, `page`, `limit`.
- Paging: page. `limit` default 50, max 500.
- Call records, not sessions.

**`ttai:list_meeting_bots`**

- Parameters: `status`, `scenario_id`, `from_date`, `to_date`, `page`, `limit`.
- Paging: page. `limit` default 50, max 500.
- Bot records, not sessions.

`ttai:v3_list_sessions` with `scenario_ids` covers other people's sessions only
when the caller holds EDIT or higher on the Scenario, or org-wide EDIT in the
selected organization. Otherwise it returns the caller's own sessions.

## Session fields by tool

**`ttai:v3_list_sessions`.** Always returned: `id`, `scenario_id`,
`scenario_version_id`, `status`, `created_at`, `updated_at`, `completed_at`.
Opt-in via `include_fields`:

- `participant` — `user_name`, `user_email`.
- `evaluation` — `evaluation_results`, `improvement_results`,
  `learning_results`.
- `transcript` — `transcript` (finalized, or the running transcript if not
  finalized).
- `recording` — `recording_present`.
- `processing` — `post_session_status`.

V3 does not return `analytics_url`, `duration`, `scenario_name`, or
`extraction_results`.

**`ttai:list_sessions` / `ttai:get_sessions_batch`.** `id`, `scenario_id`,
`scenario_name`, `created_at`, `completed_at`, `status`, `user_name`,
`user_email`, `duration` (seconds), `analytics_url`, `recording_present`,
`metadata`, `user_metadata`, `evaluation_results`, `improvement_results`,
`extraction_results`, `learning_results`. No transcript and no
`post_session_status`. Ignore the deprecated flat fields `evaluation_score`,
`report_card`, `report_card_topics`, `duration_minutes`, and `evaluation_note`;
read the nested `evaluation_results` instead.

**`ttai:get_session`.** Session record plus `transcript`, `duration`,
`post_session_status`, `extraction_results`, `scenario_name`, a
`scenario_overview`, and a signed `recording_url` unless
`include_recording_url: false`. No `analytics_url`; get links from
`ttai:get_sessions_batch`.

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

Any of these can be `null`. Configured extraction variables arrive as the
free-form `extraction_results` object on legacy reads.

## Status values

- Session `status`: `active`, `completed`, `archived`, `terminated`.
  `ttai:list_sessions`, `ttai:get_sessions_batch`, and `ttai:get_session` report
  an `active` session older than 40 minutes as `terminated`.
  `ttai:v3_list_sessions` returns and filters the stored status.
- `post_session_status.state`: `idle` (nothing running; results may or may not
  exist), `running` (processing holds the lock), `failed` (the last run errored;
  see `failed_reason`). `start_ts` marks the last run.

## Phone call and meeting-bot records

`ttai:list_sip_calls` records: `id`, `status`, `scenario_id`, `session_id`,
`phone_number`, `user_name`, `batch_id`, `scheduled_ts`, `error_message`,
`call_failure_reason`. `ttai:list_meeting_bots` records: `id`, `status`,
`scenario_id`, `session_id`, `meeting_url`, `meeting_provider`, `bot_name`,
`scheduled_ts`, `bot_joined_at`, `error_message`.

The `status` filter documents `pending`, `scheduled`, `in_call_recording`,
`call_ended`, and `failed`. Records can also carry intermediate or terminal
values such as `joining_call`, `in_waiting_room`, `in_call_not_recording`,
`done`, and `terminated`. To score calls, collect their `session_id` values and
read those sessions. Records without a `session_id` have no session to score.

## Analytics rollup

`ttai:get_analytics` returns:

- `scenarios[]`: `name`, `session_count`, `total_duration_seconds`.
- `org_dashboard` (org-wide only): `summary` (`total_sessions`,
  `completed_sessions`, `total_minutes`, `wallet_balance`), `time_series[]`
  (`date`, `session_count`, `total_minutes`), and `usage_by_member[]`.
- `member_usage.members[]` (org-wide only): `name`, `email`, `session_count`,
  `total_minutes`.

It has no scores. Use it for volume and engagement. Use session reads for
performance.
