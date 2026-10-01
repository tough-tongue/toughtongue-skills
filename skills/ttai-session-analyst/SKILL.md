---
name: ttai-session-analyst
description: >-
  Turns Tough Tongue AI sessions into evidence-backed reports. Reads
  sessions, scores, report cards, strengths, weaknesses, and transcripts from
  practice roleplays, AI phone calls, and meeting-bot calls for a Scenario,
  person, or date range, then writes team performance, Scenario health, or
  individual coaching reports with action items. Can also ingest an external
  call transcript or recording as a scored session (ttai:create_session) and
  fill in missing analysis (ttai:post_process_session), each after the user
  confirms. Use when the user asks "how is my team doing?", "top improvement
  areas for scenario X", "lowest-scoring sessions", "coaching report for
  Priya", or "session trends this month". Not for changing a Scenario,
  rubric, or prompt (ttai-agent).
when_to_use: >-
  Also when the user wants an external sales or support call scored against a
  coaching Scenario.
---

# Session Analyst

Pull sessions → aggregate per Scenario → write an evidence-backed report →
optionally hand it to the user's deck, email, or docs tools.

Practice runs, AI phone calls, and meeting-bot joins all land as **sessions**.
Read-only by default. The only writes are `ttai:create_session` and
`ttai:post_process_session`, and both need explicit confirmation.

## Hard rules

- **Guide first.** In a fresh conversation, call
  `ttai:callme_before_using_tough_tongue_mcp` unless its guide is in context.
  Some clients show tools as `mcp__ttai__<name>`. The `ttai-agent` skill holds
  the full tool catalog and entity model.
- **Scope first.** For organization data, call `ttai:list_organizations` and
  pass the returned `id` as `org_id` on every call. Never pass a slug. Omit
  `org_id` for personal scope.
- **V3 first.** Default to `ttai:v3_list_sessions`. Use legacy
  `ttai:list_sessions` only when you need a person (`user_email`), a date
  window, `hasLearning`, `duration`, or `analytics_url`.
- **Count cheaply.** For a count, call `ttai:v3_list_sessions` with
  `limit: 1, include_total: true`. Never page through results just to count.
- **Request only needed fields.** V3 returns IDs, status, and timestamps unless
  you ask for `participant`, `evaluation`, `transcript`, `recording`, or
  `processing` in `include_fields`.
- **Transcripts come from two places only:** `ttai:v3_list_sessions` (`ids` +
  `transcript`) or `ttai:get_session`. `ttai:list_sessions` and
  `ttai:get_sessions_batch` never return transcripts.
- **One rubric per aggregate.** Report-card topics and weights come from each
  Scenario's rubric. Aggregate per Scenario. Compare Scenarios qualitatively.
- **Confirm before writing.** Before `ttai:create_session` or
  `ttai:post_process_session`, state the Scenario ID and session IDs (or
  transcript source) and wait for a yes. See
  [references/ingest-and-reprocess.md](references/ingest-and-reprocess.md).
- **Scenario faults go to `ttai-agent`.** This skill diagnoses. It never edits a
  Scenario.
- **Privacy.** Coaching reports name people. Confirm the audience before sending
  per-person results to a group. Pass `include_recording_url: false` to
  `ttai:get_session` unless the user needs the signed recording link.

## Workflow

### 1. Scope

1. Resolve the workspace: `ttai:list_organizations` → `org_id`, or personal.
2. Resolve the Scenario. Reuse an ID already in context. Otherwise make one
   `ttai:list_scenarios` call with `search`, take the ID, and continue on V3.
3. Pin down the population: Scenario(s), date window, people, and how many
   sessions. When the window is vague, use the last 30 days and say so.

### 2. Pull

Pick the read path by filter. Parameters and limits:
[references/data-model.md](references/data-model.md).

- **Recent sessions for a Scenario** — `ttai:v3_list_sessions`: `scenario_ids`,
  `include_fields: ["participant", "evaluation"]`, `limit` ≤ 100, follow
  `next_cursor`.
- **Counts by status** — `ttai:v3_list_sessions`: `scenario_ids`, `statuses`,
  `limit: 1`, `include_total: true`.
- **One person or a date window** — `ttai:list_sessions`: `scenario_id`,
  `user_email`, `from_date` / `to_date`, `is_org: true` for org-wide, `page` /
  `limit` ≤ 500.
- **Scenario-learning flags** — `ttai:list_sessions`: `hasLearning: "issue"`.
- **Transcripts for chosen sessions** — `ttai:v3_list_sessions`: `ids`,
  `include_fields: ["transcript", "evaluation"]`.
- **Full detail for one session** — `ttai:get_session`: `session_id`,
  `include_recording_url: false`.
- **Review links for chosen sessions** — `ttai:get_sessions_batch`:
  `session_ids` → `analytics_url`.
- **Usage, minutes, member activity** — `ttai:get_analytics`:
  `is_org_wide: true`, `start_date` / `end_date`.
- **Phone call or bot outcomes** — `ttai:list_sip_calls` /
  `ttai:list_meeting_bots` → `session_id` per record.

Notes:

- Legacy `from_date` / `to_date` filter on `updated_at`, not creation time.
  Mention this when the window edge matters.
- Sessions without `evaluation_results.final_score` are unanalyzed. Exclude them
  from score math and report the count. Fill them in only if the user needs
  completeness (step 5).

### 3. Aggregate

Compute per Scenario:

- **Score distribution.** Mean, median, and range of
  `evaluation_results.final_score` (0–10, already weighted by the rubric). State
  the analyzed count and the excluded count.
- **Per-topic breakdown.** Group `report_card[]` by `topic` and average its
  `score` (0–10) across sessions. Show each topic's `weight` (percent of the
  final score). Low score on a high-weight topic is the top improvement area.
- **Recurring weaknesses.** `weaknesses` and `improvement_results`
  (`improvement_areas`, `action_items`) are markdown text. Cluster them into
  themes and count sessions per theme. Name a theme by its behavior ("pitches
  price before discovery"), not a category ("communication").
- **Trend.** Weekly buckets when the window spans 3+ weeks. Per-person averages
  for team views.
- **Evidence.** For each top theme, pull 1–2 transcripts for chosen IDs and
  quote the shortest excerpt that shows it. Reports without evidence read as
  opinion.

Fewer than ~10 analyzed sessions → report observations, not statistics, and say
so.

### 4. Report

Use the matching template in
[references/report-templates.md](references/report-templates.md):

- **Team performance:** "how is my team doing?"
- **Scenario health:** "is this Scenario working?"
- **Individual coaching:** one person, at most three focus areas

Every report states the population and window, the score summary, the top 3–5
improvement areas with evidence, concrete actions, and `analytics_url` links for
sessions worth a human review.

### 5. Fill gaps or ingest (only on request)

Missing analysis → confirm IDs, `ttai:post_process_session` per session, poll.
External call → confirm Scenario and source, `ttai:create_session`. Steps,
polling, and failure handling:
[references/ingest-and-reprocess.md](references/ingest-and-reprocess.md).

### 6. Distribute (optional)

Hand the report to the user's deck, email, or docs tools: one improvement area
per slide or section, evidence quote included.

## Recipes

- **"Top 5 improvement areas for Scenario X, last 50 sessions."**
  `ttai:v3_list_sessions` (`scenario_ids`, `limit: 50`,
  `include_fields: ["evaluation"]`) → per-topic averages and weakness themes →
  pull transcripts for 1–2 sessions per theme → Team performance report.
- **"The 5 lowest-scoring sessions and what went wrong."** Collect the
  population (V3 with `evaluation`, or legacy for a date window) → sort by
  `final_score` ascending → take 5 → transcripts via V3 `ids` → separate
  Scenario faults from user-skill gaps → Scenario faults go to the `ttai-agent`
  skill with the diagnosis and transcript quotes.
- **"How did priya@example.com do this month?"** `ttai:list_sessions`
  (`user_email`, `from_date` = first of month, `is_org: true` in an org) →
  per-topic averages and trend → Individual coaching report with actions drawn
  from `improvement_results.action_items`.
- **"Is this Scenario working?"** Status counts via V3 `include_total` (count
  old `active` sessions as abandoned) → `ttai:list_sessions` with
  `hasLearning: "issue"` for flagged transcripts → Scenario health report.

## Pitfalls

- **Status ≠ processing.** Session `status`: `active`, `completed`, `archived`,
  `terminated` (use for completion rates). `post_session_status.state`: `idle`,
  `running`, `failed`.
- **Stale `active`.** Legacy reads and `ttai:get_session` report an `active`
  session older than 40 minutes as `terminated`. V3 returns the stored status,
  so abandoned sessions can still read `active` there.
- **Math on numbers.** Use `score` and `final_score`; `score_str` and
  `overall_score` are display text.
- **V3 scope.** `ttai:v3_list_sessions` returns other people's sessions only
  when the caller holds EDIT or higher on the Scenario (pass `scenario_ids`) or
  in the selected organization. Otherwise it returns only the caller's own. An
  empty page can mean missing permission.
- **Advanced filters are REST-only.** `meta_*` and `$gte_created_at`-style
  filters do nothing over MCP. Filter such fields after fetching.

## Key Files

- [references/data-model.md](references/data-model.md): tools, parameters,
  limits, response fields, score scales, status values
- [references/report-templates.md](references/report-templates.md): team,
  Scenario health, and coaching report formats
- [references/ingest-and-reprocess.md](references/ingest-and-reprocess.md):
  `ttai:create_session` ingest, `ttai:post_process_session` backfill,
  confirmation and polling
