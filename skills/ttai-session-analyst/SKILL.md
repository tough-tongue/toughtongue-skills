---
name: ttai-session-analyst
description: >-
  Turns Tough Tongue AI sessions into evidence-backed reports. Reads
  sessions, scores, report cards, strengths, weaknesses, and transcripts from
  practice roleplays, AI phone calls, and meeting-bot calls for a Scenario,
  person, or date range, then writes team performance, Scenario health, or
  individual coaching reports with action items. Can also ingest an external
  call transcript or recording as a scored session (ttai:upload_session) and
  fill in missing analysis (ttai:analyze_session), each after the user
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
Read-only by default. The only writes are `ttai:upload_session` and
`ttai:analyze_session`, and both need explicit confirmation.

## Hard rules

- **Guide first.** In a fresh conversation, call
  `ttai:read_guide` unless its guide is in context.
  Some clients show tools as `mcp__ttai__<name>`. The `ttai-agent` skill holds
  the full tool catalog and entity model.
- **Scope first.** For organization data, call `ttai:get_workspace_info` and
  pass an `organizations[].id` as `org_id` on every call. Never pass a slug. Omit
  `org_id` for personal scope.
- **Typed list first.** Default to `ttai:list_resources` with type `sessions`.
  Add a legacy filter (`user_email`, `from_date` / `to_date`, `hasLearning`,
  `is_org`) only when you need a person, a date window, learning flags,
  `duration`, or `analytics_url` — those switch to the enriched legacy list.
- **Count cheaply.** For a count, call `ttai:list_resources(sessions)` with
  `limit: 1, include_total: true`. Never page through results just to count.
- **Request only needed fields.** The typed list returns IDs, status, and timestamps unless
  you ask for `participant`, `evaluation`, `transcript`, `recording`, or
  `processing` in `include_fields`.
- **Transcripts** come only from `include_fields: ["transcript"]` on
  `ttai:list_resources(sessions)` (with `filters: {ids}`) or
  `ttai:get_resource(sessions)`. The legacy enriched list never returns them.
- **One rubric per aggregate.** Report-card topics and weights come from each
  Scenario's rubric. Aggregate per Scenario. Compare Scenarios qualitatively.
- **Confirm before writing.** Before `ttai:upload_session` or
  `ttai:analyze_session`, state the Scenario ID and session IDs (or
  transcript source) and wait for a yes. See
  [references/ingest-and-reprocess.md](references/ingest-and-reprocess.md).
- **Scenario faults go to `ttai-agent`.** This skill diagnoses. It never edits a
  Scenario.
- **Privacy.** Coaching reports name people. Confirm the audience before sending
  per-person results to a group. Request `recording` in `include_fields` only
  when the user needs it.

## Workflow

### 1. Scope

1. Resolve the workspace: `ttai:get_workspace_info` → `org_id`, or personal.
2. Resolve the Scenario. Reuse an ID already in context. Otherwise make one
   `ttai:list_resources(scenarios, query)` call with the title, take the ID, and continue by ID.
3. Pin down the population: Scenario(s), date window, people, and how many
   sessions. When the window is vague, use the last 30 days and say so.

### 2. Pull

Every read below is `ttai:list_resources` or `ttai:get_resource` with type
`sessions` unless named; filters go in `filters`. Parameters and limits:
[references/data-model.md](references/data-model.md).

- **Recent sessions for a Scenario** — list: `scenario_ids`,
  `include_fields: ["participant", "evaluation"]`, `limit` ≤ 100, follow
  `next_cursor`.
- **Counts by status** — list: `scenario_ids`, `statuses`,
  `limit: 1`, `include_total: true`.
- **One person or a date window** — list (legacy): `scenario_ids` (first one
  used), `user_email`, `from_date` / `to_date`, `is_org: true` for org-wide,
  `limit` ≤ 500; page with `next_cursor`. Rows carry `analytics_url`,
  `duration`, and `extraction_results`.
- **Scenario-learning flags** — list (legacy): `hasLearning: "issue"`.
- **Transcripts for chosen sessions** — list: `ids`,
  `include_fields: ["transcript", "evaluation"]`.
- **Full detail for one session** — get: `id`, `include_fields: ["participant",
  "evaluation", "transcript"]`.
- **Recording URL** — get: `id`, `include_fields: ["recording_url"]` returns the
  enriched record (transcript, recording URL, scenario overview).
- **Review links** — `analytics_url` comes on legacy-list rows; filter that
  list by person or window and match IDs.
- **Usage, minutes, member activity** — `ttai:get_workspace_info(sections:
  ["usage"])`, `start_date` / `end_date` (org-wide for EDIT+ org roles).
- **Phone call or bot outcomes** — `ttai:list_resources(bots)` (`kinds`,
  `include_fields: ["failure"]`) → `session_id` per record.

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

Missing analysis → confirm IDs, `ttai:analyze_session` per session, poll.
External call → confirm Scenario and source, `ttai:upload_session`. Steps,
polling, and failure handling:
[references/ingest-and-reprocess.md](references/ingest-and-reprocess.md).

### 6. Distribute (optional)

Hand the report to the user's deck, email, or docs tools: one improvement area
per slide or section, evidence quote included.

## Recipes

- **"Top 5 improvement areas for Scenario X, last 50 sessions."**
  `ttai:list_resources(sessions)` (`scenario_ids`, `limit: 50`,
  `include_fields: ["evaluation"]`) → per-topic averages and weakness themes →
  pull transcripts for 1–2 sessions per theme → Team performance report.
- **"The 5 lowest-scoring sessions and what went wrong."** Collect the
  population (typed list with `evaluation`, or legacy for a date window) → sort by
  `final_score` ascending → take 5 → transcripts via `ids` → separate
  Scenario faults from user-skill gaps → Scenario faults go to the `ttai-agent`
  skill with the diagnosis and transcript quotes.
- **"How did priya@example.com do this month?"** `ttai:list_resources(sessions)`
  (`user_email`, `from_date` = first of month, `is_org: true` in an org) →
  per-topic averages and trend → Individual coaching report with actions drawn
  from `improvement_results.action_items`.
- **"Is this Scenario working?"** Status counts via `include_total` (count
  old `active` sessions as abandoned) → `ttai:list_resources(sessions)` with
  `hasLearning: "issue"` for flagged transcripts → Scenario health report.

## Pitfalls

- **Status ≠ processing.** Session `status`: `active`, `completed`, `archived`,
  `terminated` (use for completion rates). `post_session_status.state`: `idle`,
  `running`, `failed`.
- **Stale `active`.** The legacy list reports an `active` session older than 40
  minutes as `terminated`. Typed reads return the stored status, so abandoned
  sessions can still read `active` there.
- **Math on numbers.** Use `score` and `final_score`; `score_str` and
  `overall_score` are display text.
- **Typed-list scope.** `ttai:list_resources(sessions)` returns other people's sessions only
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
  `ttai:upload_session` ingest, `ttai:analyze_session` backfill,
  confirmation and polling
