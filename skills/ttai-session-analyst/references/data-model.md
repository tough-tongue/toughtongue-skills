# Session Data Model

## Contents

- Read tools and parameters
- Session fields by tool
- Evaluation shape and score scales
- Status values
- Phone call and meeting-bot records
- Analytics rollup

All account tools accept an optional `org_id` (the `id` from
`ttai:list_resources(organizations)`). Omit it for personal scope. Date filters take an ISO
8601 date or timestamp. A date-only lower bound starts at 00:00:00 UTC and a
date-only upper bound ends at 23:59:59.999999 UTC.

## Read tools and parameters

Every read is `ttai:list_resources` or `ttai:get_resource`; type-specific
parameters go in `filters`. Paging at a glance:

| Read                                   | Paging | `limit`           |
| -------------------------------------- | ------ | ----------------- |
| list `sessions` (typed)                | cursor | 1–100, default 25 |
| list `sessions` with a legacy filter   | page   | 1–100, default 50 |
| get `sessions`                         | none   | —                 |
| get `usage`                            | none   | —                 |
| list `bots`                            | cursor | 1–100, default 25 |

**List `sessions` (typed)**

- Filters: `ids`, `scenario_ids`, `statuses`. Plus `include_fields`, `cursor`,
  `limit`, `include_total`.
- Newest first by `created_at`. `total` only with `include_total: true`.

**List `sessions` (legacy enriched)** — any of these filters selects it:

- Filters: `user_email` (comma-separated allowed), `hasLearning` (`with` /
  `issue` / `ok` / `none`), `from_date`, `to_date`, `is_org`, `page`; the first
  `scenario_ids` value is used as its Scenario filter.
- Paging: `page` from 1; `page_meta.total`.
- Dates filter `updated_at`. `is_org: true` returns org-wide data; needs
  `org_id` and an OWNER or EDIT org role, otherwise the call is rejected.

**Get `sessions`**

- `id`, optional `include_fields` (`participant`, `recording`, `transcript`,
  `evaluation`, `processing`).

**Get `usage`**

- Filters: `is_org_wide`, `start_date`, `end_date`.
- Default window is the last 90 days. Org-wide view needs an EDIT or OWNER role.

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
- `recording` — `recording_present`.
- `processing` — `post_session_status`.

Typed reads do not return `analytics_url`, `duration`, `scenario_name`, or
`extraction_results`.

**Legacy enriched list.** `id`, `scenario_id`,
`scenario_name`, `created_at`, `completed_at`, `status`, `user_name`,
`user_email`, `duration` (seconds), `analytics_url`, `recording_present`,
`metadata`, `user_metadata`, `evaluation_results`, `improvement_results`,
`extraction_results`, `learning_results`. No transcript and no
`post_session_status`. Ignore the deprecated flat fields `evaluation_score`,
`report_card`, `report_card_topics`, `duration_minutes`, and `evaluation_note`;
read the nested `evaluation_results` instead.

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

- Session `status`: `active`, `completed`, `archived`, `terminated`. The
  legacy list reports an `active` session older than 40 minutes as
  `terminated`; typed reads return and filter the stored status.
- `post_session_status.state`: `idle` (nothing running; results may or may not
  exist), `running` (processing holds the lock), `failed` (the last run errored;
  see `failed_reason`). `start_ts` marks the last run.

## Phone call and meeting-bot records

`bots` records: `id`, `kind` (`meeting` / `phone`), `phone_direction`,
`scenario_id`, `scenario_version_id`, `session_id`, `status`, `batch_id`,
`scheduled_ts`, `bot_joined_at`, `call_ended_at`, `created_at`, `updated_at`;
`meeting_provider` and `bot_name` with `include_fields: ["meeting"]`;
`failure_reason` (`busy`, `no_answer`, `voicemail`, …) with `["failure"]`.
Phone numbers and meeting URLs are never returned.

To score calls, collect their `session_id` values and read those sessions.
Records without a `session_id` have no session to score.

## Analytics rollup

`ttai:get_resource(usage)` returns:

- `scenarios[]`: `name`, `session_count`, `total_duration_seconds`.
- `org_dashboard` (org-wide only): `summary` (`total_sessions`,
  `completed_sessions`, `total_minutes`, `wallet_balance`), `time_series[]`
  (`date`, `session_count`, `total_minutes`), and `usage_by_member[]`.
- `member_usage.members[]` (org-wide only): `name`, `email`, `session_count`,
  `total_minutes`.

It has no scores. Use it for volume and engagement. Use session reads for
performance.
