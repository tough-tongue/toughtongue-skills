---
name: ttai-agent
description: >-
  Creates, edits, and refines Tough Tongue AI Scenarios, the definitions
  behind voice agents (sales calls, lead qualification, demos, screening,
  support, booking) and practice roleplays (negotiations, interviews,
  coaching). Writes ai_instructions, ai_model_config, strategy, tools_config,
  and rubrik through ttai:create_scenario and ttai:update_scenario. Also
  resolves the personal or organization workspace, checks plan and balance,
  places or cancels SIP calls, schedules or cancels meeting bots, mints
  private Scenario links, and troubleshoots ttai MCP sign-in. Use when the
  user wants a voice agent or practice Scenario created or improved ("create
  a voice agent", "the agent ends calls too early", "make it sound more
  natural"), a call placed, a meeting bot sent, or a Tough Tongue AI account
  question answered. Not for reports across many sessions
  (ttai-session-analyst) or browser demo steps (ttai-browser-demo-builder).
when_to_use: >-
  Also when the user pastes a brief or an example conversation to turn into a
  Scenario, or a ttai tool call fails.
---

# Tough Tongue AI Agent

Tough Tongue AI is the platform for tough conversations. Some the AI takes:
voice agents that call, demo, screen, support, and book. Others you nail:
realistic roleplay for negotiations, interviews, sales, and coaching.

This skill turns a person's intent into the smallest correct action on their
Tough Tongue AI workspace through the `ttai` MCP server. Live tool schemas and
tool results always win over this text.

## Contents

- Mental model
- Scenario workflow (create / edit / refine)
- Job router
- Hard rules
- Key Files

## Mental model

- **Workspace** — a personal account or an organization (`org_id` from
  `ttai:list_organizations`).
- **Scenario** — one agent definition: instructions, model stamp, controls,
  rubric.
- **Run** — a session on the web link, an embed, a SIP call, or a meeting bot.
- **Result** — transcript, evaluation, extraction; read per session or in
  aggregate.
- **Access** — server-decided role, plan, and balance; never inferred from a
  Scenario.

## Scenario workflow

The core job. Full procedure:
[references/scenario/workflow.md](references/scenario/workflow.md).

- **Create** — the user has a new outcome, brief, or transcript. Finish with
  `ttai:create_scenario` (no `id`).
- **Edit** — the user has a known Scenario and a requested change. Finish with
  `ttai:update_scenario` (`id` + changed fields).
- **Refine** — the user has a complaint or a poor session. Finish with evidence
  → one diagnosis → surgical update.

Loop: resolve workspace → fetch current Scenario
(`ttai:v3_get_scenario_version`) → load the matching recipe and Scenario
references → draft → load the live write schema → write → re-fetch → report the
change, the run link `https://app.toughtongueai.com/run/<scenario_id>`, and that
edits affect new sessions only.

## Job router

Read only the file the job needs. Each file is self-contained. Each entry gives
the job, the file, and when to read it.

### Account, access, and runs

- **Scope, focus, first-run orientation** —
  [operating-model.md](references/operating-model.md). Any stateful request;
  "what can I do here?"
- **What a workspace holds** —
  [entities/resources.md](references/entities/resources.md). Listing or
  explaining resources.
- **Workspaces, plans, balance, sharing** —
  [entities/account-and-access.md](references/entities/account-and-access.md).
  Org vs personal, entitlements, private links, SATs.
- **Run a Scenario, read results** —
  [entities/scenario-engine.md](references/entities/scenario-engine.md). Web
  link, embed, SIP, meeting bot, session results.

### Scenario authoring

- **Create / edit / refine a Scenario** —
  [scenario/workflow.md](references/scenario/workflow.md). Any Scenario write.
- **Design a good Scenario** —
  [scenario/authoring.md](references/scenario/authoring.md). Before drafting
  content.
- **Write `ai_instructions`** —
  [scenario/ai-instructions.md](references/scenario/ai-instructions.md). Prompt
  shape, opening, variables.
- **Pick `ai_model_config`** —
  [scenario/model-selection.md](references/scenario/model-selection.md). Model
  family, pipeline, voice.
- **Set controls** — [scenario/control.md](references/scenario/control.md).
  `strategy`, `appearance`, `tools_config`, analysis, links.
- **Write the rubric** — [scenario/rubric.md](references/scenario/rubric.md).
  `rubrik`, `processed_rubrik`, who is scored.
- **Diagnose runtime behavior** —
  [scenario/runtime.md](references/scenario/runtime.md). Hang-ups, silence,
  wrap-up timing, tools over SIP.

### Recipes

- **Outbound caller** — [recipes/cold-call.md](references/recipes/cold-call.md).
  AI calls a lead; SDR, qualification, reminders.
- **Prospect for sales practice** —
  [recipes/sales-roleplay.md](references/recipes/sales-roleplay.md). User
  practices selling, objections, negotiation.
- **Trainer or mentor** — [recipes/coaching.md](references/recipes/coaching.md).
  Teaching, feedback, certification.
- **Product demo agent** — [recipes/demo.md](references/recipes/demo.md).
  Browser or slide demo.
- **Interviewer** — [recipes/interview.md](references/recipes/interview.md).
  Case, behavioral, screening interviews.
- **Meeting facilitator or observer** —
  [recipes/meeting-facilitation.md](references/recipes/meeting-facilitation.md).
  Agenda, nudges, synthesis.
- **Add-on: Cascade speech rules** —
  [recipes/cascade-tts.md](references/recipes/cascade-tts.md). Only when the
  model is Landmass `cascade`.

### MCP

- **MCP tools, read policy, confirmations** —
  [mcp/tools.md](references/mcp/tools.md). Choosing or calling any `ttai:` tool.
- **Connect a client, fix auth** — [mcp/clients.md](references/mcp/clients.md).
  Install, OAuth, 401, missing tools.

No matching recipe (support, intake, survey, negotiation counterpart)? Use
authoring + ai-instructions + control, and borrow the closest recipe's shape.

Sibling skills:

- `ttai-session-analyst` — population-wide trends, scorecards, team reports.
- `ttai-browser-demo-builder` — recorded browser steps and selectors for demos.

## Hard rules

- MCP tools are the contract. Call only `ttai:` tools listed in
  [mcp/tools.md](references/mcp/tools.md) and present in the live `tools/list`.
- At the first Tough Tongue AI action in a conversation, call
  `ttai:callme_before_using_tough_tongue_mcp` unless its guide is already in
  context.
- Resolve workspace before a stateful call; pass the returned opaque `org_id`
  for organization work. Ask only when a write's scope is ambiguous.
- Load the tool's input schema before writing. Never invent fields, model IDs,
  voice IDs, or linked-resource IDs.
- `ttai:update_scenario` replaces what you send. A tool's `tool_settings` and
  the whole `ai_model_config` are replaced, not merged: read, merge, write the
  full object.
- Confirm the exact target (phone number, meeting URL, Scenario ID, audience)
  before any call, bot, delete, share, or post-processing.
- Separate what the platform supports from what this user's plan allows; the
  server decides. Treat `403` and balance errors as real blockers.
- Advice stops at a recommendation. Never turn it into a write, call, or bot
  without the user's request.
- Never ask for, paste, or log a token. `TTAI_PAT` is named only as a headless
  fallback.
- Scenario edits affect new sessions only.

## Key Files

- [references/scenario/workflow.md](references/scenario/workflow.md) — create,
  edit, refine procedure
- [references/mcp/tools.md](references/mcp/tools.md) — tool catalog and call
  conventions
- [references/operating-model.md](references/operating-model.md) — scope,
  capability, action loop
- [references/scenario/control.md](references/scenario/control.md) — Scenario
  control fields
