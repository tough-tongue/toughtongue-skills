---
name: scenario-maker
description: >
  Turn a Tough Tongue AI brief, transcript, or scenario complaint into the
  smallest correct Scenario action: create, edit, or evidence-backed refine.
  Uses the ttai MCP server and ttai-agent knowledge. Use when the user says
  "create a scenario", "create a voice agent", "build a practice scenario",
  "edit the scenario", "fix the scenario", "make it sound more natural",
  "it ended the call too early", or pastes a brief, transcript, or complaint.
when_to_use: >
  User wants a Tough Tongue AI Scenario created, changed, or repaired from evidence.
---

# Scenario Maker

Turn a useful brief or a bad session into a Scenario that is better on the
next run.

This is a **workflow**, not a second knowledge base. `ttai-agent` owns
Scenario fields, model stamps, prompt structure, recipes, and runtime facts.
Load the smallest relevant source; do not restate it here.

## Choose the job

| User has | Do | Finish with |
| --- | --- | --- |
| A new outcome or no live Scenario | **Create** | `ttai:create_scenario` |
| A known Scenario and a requested change | **Edit** | `ttai:update_scenario` |
| A complaint, transcript, or poor result | **Refine** | Evidence → surgical `ttai:update_scenario` |

If they want only advice, stop at a recommendation. A named write request is
authorization to make the smallest matching change.

## Ground the action

1. Load `ttai-agent/kb/operating-model.md`.
2. Reuse verified workspace and Scenario context. Otherwise call
   `ttai:list_organizations`; pass `org_id` for organization work.
3. Load the live MCP schema before a write. Do not invent fields, model IDs,
   voice IDs, or linked-resource configuration.
4. For compact discovery, read `ttai://guide/v3-tools` first. Resolve an
   exact visible Scenario name with `ttai:v3_list_scenarios`; use
   `ttai:list_scenarios` only for free-text discovery.

## Create

Load one recipe plus the four authoritative Scenario sources:

1. `ttai-agent/kb/scenario-authoring.md`
2. `ttai-agent/kb/entities/scenario/model-selection.md`
3. `ttai-agent/kb/entities/scenario/control.md`
4. `ttai-agent/kb/entities/scenario/ai-instructions.md`
5. One matching file in `ttai-agent/kb/scenario-recipes/`

Build from supplied facts, not generic filler. The recipe decides the AI role,
resistance, success evidence, and rubrik subject. The authoring sources decide
the pipeline, control fields, and prompt shape.

Call `ttai:create_scenario` with no `id`. Return the run link. For a private
Scenario, offer a Scenario Access Token only when the user wants to share it.
Use **browser-demo-builder** after creation when the Scenario needs recorded
browser steps.

## Edit

Fetch the full Scenario with `ttai:get_scenario`. Change only the requested
fields, then call `ttai:update_scenario` with `id` plus those fields. Re-fetch
and state exactly what changed.

## Refine from evidence

1. Fetch the Scenario and read the relevant instructions and controls.
2. Use the supplied transcript when available. Otherwise call
   `ttai:v3_list_sessions` scoped by `scenario_ids`, requesting `evaluation`
   only when needed; inspect selected records with `ttai:get_sessions_batch`.
   Use legacy `ttai:list_sessions` for person/date filters or full-population
   score ordering.
3. Read `ttai-agent/kb/entities/scenario/runtime.md`. Diagnose one concrete
   cause against the evidence; do not keep layering prose onto a weak prompt.
4. Update the narrowest field that fixes it, re-fetch, and report:
   **Diagnosis**, **Change**, and the reminder that edits affect **new
   sessions only**.

## Handoffs

- **browser-demo-builder** — deterministic browser walkthroughs and selectors
- **session-analyst** — population-wide trends, scorecards, and reports
- **ttai-agent** — scope, capability, channel, sharing, or resource questions

## Non-negotiables

- Never paste or request a PAT.
- Never claim a Scenario edit changes an active or historical session.
- Never expose or fabricate private linked-resource configuration.

## Key Files

- [../ttai-agent/SKILL.md](../ttai-agent/SKILL.md) — intelligence-layer entry
- [../ttai-agent/kb/entities/scenario/index.md](../ttai-agent/kb/entities/scenario/index.md) — Scenario knowledge map
- [../ttai-agent/kb/entities/scenario/runtime.md](../ttai-agent/kb/entities/scenario/runtime.md) — evidence-based repair facts
- [../ttai-agent/kb/scenario-recipes/index.md](../ttai-agent/kb/scenario-recipes/index.md) — situation selection
