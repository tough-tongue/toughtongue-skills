---
name: scenario-maker
description: >
  Create, edit, or refine Tough Tongue AI scenarios via the ttai MCP server
  — outbound callers, demo agents, screeners, and practice roleplay.
  Classifies the job, loads ttai-agent, then calls ttai:create_scenario
  or ttai:update_scenario. Use when the user says "create a scenario",
  "create a voice agent", "build a practice scenario", "edit the scenario",
  "refine the scenario", "fix the scenario", "the agent said X instead of
  Y", "it ended the call too early", "make it sound more natural", "I have
  a call in 30 minutes, help me rehearse", or pastes a brief, transcript,
  or complaint.
when_to_use: >
  User wants a Tough Tongue AI scenario created, edited, or fixed from evidence.
---

# Scenario Maker

One workflow for **new**, **edit**, and **refine**. Load **ttai-agent**
for datastore + scenario shape; load `ttai-agent/features/mcp/` before any
`ttai:` call. Do not invent fields; load the tool schema.

| Job        | Signal                                                   | Tool                                         |
| ---------- | -------------------------------------------------------- | -------------------------------------------- |
| **Create** | No live scenario, or "build / make / rehearse"           | `ttai:create_scenario` (no `id`)             |
| **Edit**   | Named scenario, "change X to Y", no failure evidence     | `ttai:get_scenario` → `ttai:update_scenario` |
| **Refine** | Complaint, transcript, low scores, "too early / robotic" | Evidence → surgical `ttai:update_scenario`   |

If ambiguous: create when nothing exists to fetch; otherwise fetch first.

## 1. Account context

Load `ttai-agent/kb/operating-model.md`. Reuse a current, verified workspace
context when supplied by the consumer; otherwise call `ttai:list_organizations`
and use `org_id` for team work. The public MCP cannot fetch a user profile or
effective platform plan—do not treat `list_subscriptions` as either.

## 2. Create

Classify → situation file → draft → validate → create.

| Type                             | AI plays                                         | Load from **ttai-agent**                                   |
| -------------------------------- | ------------------------------------------------ | ---------------------------------------------------------- |
| Cold call / SDR / outbound phone | Outbound caller                                  | `kb/scenario-recipes/cold-call.md`                         |
| Sales roleplay                   | Prospect                                         | `kb/scenario-recipes/sales-roleplay.md`                    |
| Coaching                         | Trainer / mentor                                 | `kb/scenario-recipes/coaching.md`                          |
| Demo                             | Product demo agent                               | `kb/scenario-recipes/demo.md`                              |
| Other                            | Interview, support, negotiation, clone, observer | `model-selection.md` + `control.md` + `ai-instructions.md` |

Signals: "AI calls the customer" / SDR / SIP → cold call. "Practice selling"
→ sales roleplay. "Coach my team" → coaching. "Demo my product" → demo.

Always load `kb/scenario-authoring.md`, `model-selection.md`, `control.md`, and
`ai-instructions.md`. Load `kb/scenario-recipes/cascade-tts.md` only when the
selected stamp is full Cascade.

Scripted browser walkthroughs: create first, then **browser-demo-builder**.

### Gather + defaults

URLs, transcripts, and other MCP tools beat invented facts. Ask only when
the brief is silent:

| Question           | Default                                                       |
| ------------------ | ------------------------------------------------------------- |
| Language           | `en-US`                                                       |
| External TTS voice | Use a user-supplied or provider-returned ID; never invent one |
| Cold-call sub-type | Warm lead                                                     |
| Coaching pattern   | A (Situation-First)                                           |
| Public or private  | `is_public: true`                                             |

### Draft (`ttai:create_scenario`, no `id`)

1. `name`
2. `ai_model_config` — stamp from the **ttai-agent**
   `kb/entities/scenario/model-selection.md` situation table (cold call / slides →
   Landmass `cascade` + Cartesia; sales → Galaxy `medium-stable`; coaching /
   browser demo → Ocean `medium-stable`). Never author `cascade-01` or invent
   a `tts_voice_id`. Super-agent requires Landmass.
3. `ai_instructions` — shape from `kb/entities/scenario/ai-instructions.md`;
   content from the recipe. Specify the full role, reality model, flow,
   tools, and constraints without padding. Two-beat opening:
   identity/reason → STOP and wait. One question per turn.
4. `user_instructions`, `rubrik` (correct party), `user_friendly_description`
5. `strategy`, `tools_config`, `session_analysis`, `appearance`
6. `is_recording: true` for voice

Fast path: `ttai:generate_scenario` then create. Prefer full authoring for
team-run scenarios.

### Create checklist

- [ ] `name`, `ai_instructions`; `user_friendly_description` for humans
- [ ] `ai_model_config` matches the voice, interaction, and channel contract
- [ ] `##` sections; `{{ vars }}` have fallbacks
- [ ] Opening lives in FLOW (directive, never quoted). No `welcome_instructions`
- [ ] `end_session` registered with `add_to_system_prompt: true`
- [ ] Talkative `strategy.silence` (omit = 120s hang up)
- [ ] Auto-analysis + auto-submit on
- [ ] Rubric evaluates the correct party
- [ ] Full Cascade: voice-pipeline + STT blocks. Separate-TTS realtime:
      `tts_*` only. Native realtime: no TTS/STT sub-fields

`ttai:create_scenario` → return `https://app.toughtongueai.com/run/<id>`.
Private: mention `ttai:create_scenario_access_token`.

## 3. Edit (no failure evidence)

`ttai:get_scenario` (resolve id via `ttai:list_scenarios` if they gave a
name). Change only what they asked. Partial `ttai:update_scenario`: `id` +
changed fields. Re-fetch to confirm.

## 4. Refine (from evidence)

Diagnose → plan → surgical edit → verify. Replace or tighten before adding.
Flag the user if `ai_instructions` grows more than ~50 tokens. One issue =
one edit.

Read [references/runtime-behavior.md](references/runtime-behavior.md) before
blaming prompt assembly, tools, conductor, or silence.

1. Fetch the scenario (`ttai:get_scenario`). Read all of `ai_instructions`,
   plus `strategy`, `tools_config`, `session_analysis`.
2. Evidence: pasted transcript/complaint, or `ttai:list_sessions` →
   `ttai:get_sessions_batch` → `transcript_url`. Quote the prescribed turn
   vs what the agent did. Do not translate Hindi/Hinglish.
3. Classify:

| Symptom                                | Fix                                                                        |
| -------------------------------------- | -------------------------------------------------------------------------- |
| `end_session` too early / late / never | End-of-call block or `tools_config.tools.end_session`                      |
| Wrong branch / skipped step            | FLOW triggers; NEVER skip                                                  |
| Two questions in one turn / robotic    | STYLE — bold one rule; do not add a section                                |
| Revealed AI or spoke a tool name       | GUARDRAILS                                                                 |
| ~2 min silence then hang up            | Talkative `strategy.silence`                                               |
| Wrap-up mid-call                       | `strategy.conductor.messages`                                              |
| Wrong voice / locale                   | Cascade: `tts_voice_id`. Native: `appearance.voice`. Match `language_code` |
| Spoke `{{ firstname }}`                | CONTEXT fallback                                                           |
| Browser click misses                   | **browser-demo-builder** selector guide                                    |
| Robotic / restarting opening           | FLOW as a directive                                                        |

4. Imperative voice. Bind rules to triggers. Send only `id` + changed
   fields. For `ai_instructions`, send the full updated string.
5. Re-fetch. Report: **Diagnosis**, **Change**, **Token delta**, reminder
   that **new sessions only** pick up the edit.

## Pitfalls

- Stamp from `model-selection.md` (cold call / slides → Cascade; sales →
  Galaxy; coaching / browser → Ocean). Do not freestyle from channel alone.
- Cold-call rubrics score the **lead**; sales rubrics score the **rep**.
- Missing `end_session` timing → never hangs up, or hangs up mid-sentence.
- Never embed `TTAI_PAT` in anything you generate.

## Key Files

- [../ttai-agent/SKILL.md](../ttai-agent/SKILL.md) — intelligence-layer entry
- [../ttai-agent/kb/entities/scenario/model-selection.md](../ttai-agent/kb/entities/scenario/model-selection.md) — stamps
- [../ttai-agent/kb/scenario-authoring.md](../ttai-agent/kb/scenario-authoring.md) — quality principles
- [references/runtime-behavior.md](references/runtime-behavior.md) — refine from evidence
