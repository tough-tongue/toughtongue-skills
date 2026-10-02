# Scenario workflow: create, edit, refine

Turn a brief, a requested change, or a bad session into a Scenario that is
better on the next run. This file is the procedure; field meaning lives in the
sibling Scenario references.

## Contents

- Choose the job
- Ground the action
- Create
- Edit
- Refine from evidence
- Report and hand off

## Choose the job

| User has                                | Do         |
| --------------------------------------- | ---------- |
| A new outcome or no live Scenario       | **Create** |
| A known Scenario and a requested change | **Edit**   |
| A complaint, transcript, or poor result | **Refine** |

Create finishes with `ttai:create_scenario`; Edit finishes with
`ttai:update_scenario`; Refine gathers evidence, then makes a surgical
`ttai:update_scenario`.

Advice only? Stop at a recommendation. A named write request authorizes the
smallest matching change.

## Ground the action

1. Follow [../operating-model.md](../operating-model.md): guide tool once, then
   workspace. Pass `org_id` for organization work.
2. Resolve the Scenario. A known ID → `ttai:v3_get_scenario_version`. A title
   only → one `ttai:list_scenarios(search=...)`, then switch to V3 with the
   returned ID. `ttai:v3_list_scenarios` is for inventory and counts, not title
   lookup.
3. Load the live `ttai:create_scenario` / `ttai:update_scenario` input schema.
   Never invent fields, model IDs, voice IDs, or linked-resource IDs.

## Create

Load, in order:

1. [authoring.md](authoring.md) — human outcome, reality model, evidence.
2. One recipe matching the AI's role (see the router in SKILL.md).
3. [model-selection.md](model-selection.md) — `ai_model_config` stamp.
4. [control.md](control.md) — `strategy`, `appearance`, `tools_config`,
   `session_analysis`.
5. [ai-instructions.md](ai-instructions.md) — prompt shape.
6. [rubric.md](rubric.md) — who is scored and how.

Build from supplied facts, not generic filler. The recipe decides the AI role,
resistance, success evidence, and rubric subject; the references decide
pipeline, controls, and prompt shape.

Call `ttai:create_scenario` with no `id`; `name` and `ai_instructions` are
required. Always send an explicit `tools_config`: when it is omitted, the server
registers a broad default tool set. On a validation error, fix the named field
and retry.

Return the run link `https://app.toughtongueai.com/run/<scenario_id>`. For a
private Scenario, offer a Scenario Access Token only when the user wants to
share it
([../entities/account-and-access.md](../entities/account-and-access.md)).
Recorded browser steps → the `ttai-browser-demo-builder` skill.

## Edit

1. Fetch the current Scenario with `ttai:v3_get_scenario_version`.
2. Change only the requested fields. Most nested objects merge one level, but a
   tool's `tool_settings` and the whole `ai_model_config` are replaced: send the
   full current object plus your change. Full merge rules:
   [../mcp/tools.md](../mcp/tools.md#updating-a-scenario).
3. Call `ttai:update_scenario` with `id` plus those fields. Pass
   `save_as_version` (a 1–100 character label) to archive the prior state first
   when the change is risky.
4. Re-fetch with V3 and state exactly what changed.

## Refine from evidence

1. Fetch the Scenario; read the instructions and controls the complaint touches.
2. Get evidence. Use a supplied transcript when present. Otherwise call
   `ttai:v3_list_sessions` with `scenario_ids` and a small `limit`; add
   `include_fields: ["transcript"]` or `["evaluation"]` only for the selected
   sessions. `ttai:get_session` gives one full detail. Legacy
   `ttai:list_sessions` covers person, date, or learning filters and session
   `duration`.
3. Read [runtime.md](runtime.md). Name one concrete cause that the evidence
   supports; separate what the Scenario prescribes from what happened. Do not
   layer more prose onto a weak prompt.
4. Update the narrowest field that improves it, then re-fetch.

## Report and hand off

Report **Diagnosis** (refine only), **Change** (fields touched, before → after
in one line each), the run link, and the reminder that edits affect new sessions
only — never an active or past session.

- Population-wide trends, scorecards → the `ttai-session-analyst` skill.
- Deterministic browser walkthroughs → the `ttai-browser-demo-builder` skill.

Never request a token, and never expose or fabricate private linked-resource
configuration.

## Key Files

- [runtime.md](runtime.md) — evidence for refining runtime behavior
- [control.md](control.md) — control fields
- [../mcp/tools.md](../mcp/tools.md) — create/update semantics
