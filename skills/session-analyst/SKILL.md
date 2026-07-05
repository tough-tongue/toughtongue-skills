---
name: session-analyst
description: >
  Analyze ToughTongue AI practice-session performance and build reports via
  the ttai MCP server. Pulls sessions with scores, strengths, and weaknesses,
  aggregates patterns across a team or scenario, and produces structured
  reports with improvement areas and action items. Use when the user asks
  "how is my team doing?", "top improvement areas for scenario X", "pull the
  lowest-scoring sessions", "build me a coaching report", "session trends
  this month", or wants session data turned into a deck, email, or dashboard.
---

# Session Analyst

Pull session data → aggregate patterns → produce a structured report →
optionally hand off to slides/email tools for distribution.

## Prerequisites

- The **ttai** MCP server must be connected. Tool references below use the
  `ttai:` server prefix (e.g. `ttai:list_sessions`); some agents surface
  these as `mcp__ttai__list_sessions`. If the tools are missing, point the
  user at the repo README and <https://app.toughtongueai.com/developer> for a
  `TTAI_PAT` token.

## Data model (what a session gives you)

Each session from `ttai:list_sessions` / `ttai:get_sessions_batch` includes:

- Identity: `scenario_id`, `scenario_name`, `user_name`, `user_email`
- Lifecycle: `status`, `created_at`, `completed_at`, `duration_minutes`
- `evaluation_results`: `final_score`, `strengths`, `weaknesses`, and
  `report_card[]` — per-topic `{topic, score, note, weight}`
- `improvement_results`: `improvement_areas`, `action_items`, `resources`
- `extraction_results`: structured variables (if the scenario extracts them)
- `transcript_url` (signed URL — fetch it for the conversation text) and
  `analytics_url` (human-viewable analysis page)

`report_card` topics are the backbone of aggregation: they are consistent
within a scenario because they come from its rubric.

## Workflow

### Step 1: Scope

1. Call `ttai:list_organizations`. Team analysis almost always needs an
   `org_id` — pass it on every call, along with `is_org: true` on
   `ttai:list_sessions` for org-wide data.
2. Resolve the scenario: `ttai:list_scenarios` if the user gave a name, not
   an ID.
3. Confirm the window and population: which scenario(s), which date range
   (`from_date` / `to_date`), which people (`user_email` filter), how many
   sessions.

### Step 2: Pull

- `ttai:list_sessions` with `scenario_id`, date filters, and pagination
  (`page`, `limit`). Iterate pages until you have the requested population —
  check the page metadata rather than assuming one page is everything.
- Sessions missing `evaluation_results`: either exclude them from scoring
  aggregates (note the count), or backfill — call `ttai:post_process_session`
  for each, then re-fetch after a wait and check that `evaluation_results`
  appeared. Backfill only when the user needs completeness.
- Deep dives (outliers, disputed scores): `ttai:get_sessions_batch` with the
  session IDs, then fetch `transcript_url` contents for the actual
  conversation.

### Step 3: Aggregate

Compute, at minimum:

- **Score distribution**: mean, median, range of `final_score`; flag the
  count of unanalyzed sessions excluded.
- **Per-topic breakdown**: average `report_card` score per topic, weighted by
  `weight`. The lowest topics are the improvement areas.
- **Recurring weaknesses**: cluster `weaknesses` and `improvement_areas` text
  across sessions into themes; count occurrences. Name each theme by the
  behavior, not an abstraction ("jumps to price before discovery" beats
  "communication issues").
- **Trend**: score over time if the window is long enough (week buckets work
  well); per-person averages for team views.
- **Evidence**: for each top theme, pull 1-2 short transcript quotes from
  representative sessions. Reports without evidence read as opinion.

For org-wide rollups (usage, member breakdown, time series),
`ttai:get_analytics` with `is_org_wide: true` complements per-session
aggregation.

### Step 4: Report

Use the matching template from
[references/report-templates.md](references/report-templates.md):

- **Team performance report** — "how is my team doing?"
- **Scenario health report** — "is this scenario working?" (pairs with the
  scenario-refiner skill when the answer is no)
- **Individual coaching report** — one person, one skill gap, action items

Always include: population and window, score summary, top 3-5 improvement
areas with evidence, concrete action items, and `analytics_url` links for
sessions worth reviewing by a human.

### Step 5: Distribute (optional)

If the user wants a deck, email, or document, hand the report content to
their connected tools (slides MCP, email MCP, docs). Keep the structure:
one improvement area per slide/section, evidence quote included.

## Recipes

### "Top 5 improvement areas for scenario X, last 50 sessions"

`ttai:list_scenarios` (resolve ID) → `ttai:list_sessions` (scenario_id,
limit 50, org context) → aggregate report_card topics + weakness themes →
Team performance report → deck if asked.

### "Pull the 5 lowest-scoring sessions and find out what went wrong"

`ttai:list_sessions` (scenario_id + window) → sort by
`evaluation_results.final_score` ascending, take 5 →
`ttai:get_sessions_batch` → fetch transcripts → diagnose common failure
patterns → if the fault is in the scenario (not the users), hand off to the
**scenario-refiner** skill with the diagnosis.

### "How did [person] do this month?"

`ttai:list_sessions` (user_email + from_date) → per-topic averages, trend
across their sessions → Individual coaching report with action items from
`improvement_results`.

### Automated post-call coaching (webhook-driven)

For teams wiring this into pipelines (e.g. every real sales call gets a
coaching report): see the recipe in
[references/report-templates.md](references/report-templates.md) —
`ttai:create_session` ingests an external transcript against a coaching
scenario, `ttai:post_process_session` triggers analysis, poll until
`evaluation_results` appears, then format and send the report.

## Pitfalls

- **Don't average across different scenarios' report cards** — topics and
  weights differ per rubric. Aggregate per scenario, compare qualitatively.
- **Small samples**: below ~10 analyzed sessions, report observations, not
  statistics — and say so.
- **Session status**: only `completed` sessions have meaningful duration and
  results; exclude `in_progress` and `failed` from aggregates.
- **Privacy**: coaching reports name individuals. Confirm the audience before
  distributing anything per-person to a group channel.
