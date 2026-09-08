# Scenario control fields

Key **non-prose** fields other than model selection. Load the `ttai` tool
schema before calling — it is the type source of truth.

Not in this file:

- `ai_model_config` → [model-selection.md](model-selection.md)
- `ai_instructions` → [ai-instructions.md](ai-instructions.md)
- `rubrik` → [rubrik.md](rubrik.md) + a recipe under
  [../../scenario-recipes/](../../scenario-recipes/index.md)

There is **no** `strategy.welcome_instructions` field. The first spoken turn
comes from `ai_instructions` FLOW (`skip_auto_start: false`) plus a runtime
prompt: "Greet the user with your opening line." Put the opening in FLOW.

## Contents

- Top-level
- strategy (silence defaults, conductor, no prefix)
- appearance
- tools_config
- session_analysis, memory, transcribe
- What MCP cannot set

---

## Top-level

Create **requires** `name` and `ai_instructions`. Everything else is optional
on the wire — still set control fields below for a quality scenario.

| Field                       | Notes                                         |
| --------------------------- | --------------------------------------------- |
| `id`                        | Update only. Omit on create                   |
| `name`                      | Short display title                           |
| `type`                      | See type rules below                          |
| `user_friendly_description` | 1–2 public sentences                          |
| `user_instructions`         | What the human reads before starting          |
| `pdf_context`               | Extra context document                        |
| `is_public`                 | Default `true`. `false` needs an access token |
| `is_recording`              | Default `false`. Set `true` for voice         |
| `passcode`                  | Optional gate                                 |
| `analysis_access`           | `"default"` \| `"always"` \| `"never"`        |
| `user_metadata`             | Filterable string key-value                   |
| `mcp_server_ids`            | Voice-agent MCP ids. Do not invent            |
| `meeting_config`            | Optional: meet bot, notetaker, SIP AMD        |

`type` via MCP: `"default"` (usual), `"super"` (needs `stages`),
`"composite"` (needs `constituent_scenarios`). Never `super_agent`.
Studio-only types (`meet_assist`, …) cannot be set here.

Always set `ai_model_config` — see [model-selection.md](model-selection.md).

---

## strategy

```json
{
  "skip_auto_start": false,
  "system_instructions_template": "minimal",
  "silence": {
    "silence_threshold": 6000,
    "end_session": false,
    "force_agent_to_speak": true
  },
  "conductor": {
    "enabled": true,
    "messages": [
      {
        "time_seconds": 300,
        "message": "Wrap up and say goodbye. Wait for the user's reply before end_session.",
        "end_turn": true
      }
    ]
  }
}
```

| Field                          | Effect                                                             |
| ------------------------------ | ------------------------------------------------------------------ |
| `skip_auto_start`              | `false` = AI speaks first from FLOW. `true` = wait for human       |
| `system_instructions_template` | `"minimal"` (~50 tok) or omit (STANDARD ~600–800)                  |
| `silence`                      | **Omit → 120s then hang up.** Always set talkative for practice    |
| `conductor.messages[]`         | Timed wrap-up. No deprecated `prefix`. On-demand → `ask_conductor` |
| `filler_words`                 | Cascade only; ignored on Galaxy/Ocean                              |
| `max_duration_seconds`         | Hard wall-clock cap                                                |

Recommended silence for practice: 5–8s, `end_session: false`,
`force_agent_to_speak: true`.

### Strategy defaults by situation

| Type           | Auto-start           | Wrap-up   | Opening               |
| -------------- | -------------------- | --------- | --------------------- |
| Cold call      | AI first (`false`)   | 300–450s  | FLOW Beat 1 directive |
| Sales roleplay | Human first (`true`) | 600–900s  | FLOW as prospect      |
| Coaching       | AI first (`false`)   | 900–1200s | FLOW greeting + intro |

No `welcome_instructions` field — opening lives in FLOW only.

---

## appearance

```json
{ "voice": "Aoede", "language_code": "en-US" }
```

`voice` is the product voice name (`Aoede`, `Puck`, `Kore`, `Charon`,
`Fenrir`, …). Galaxy uses it natively; Ocean maps it to an OpenAI voice.
Not a TTS UUID — Cartesia IDs live in [model-selection.md](model-selection.md).

`language_code` is BCP 47 and must match the scenario locale. Common
supported codes:

`en-US en-GB en-IN en-AU de-DE es-US es-ES fr-FR fr-CA hi-IN pt-BR ar-XA
id-ID it-IT ja-JP tr-TR vi-VN bn-IN gu-IN kn-IN ml-IN mr-IN ta-IN te-IN
nl-NL ko-KR cmn-CN pl-PL ru-RU th-TH`

Match `transcribe_config` to the same locale when you set one.

---

## tools_config

Each tool: `should_register` (can the AI call it) and
`add_to_system_prompt` (is it told when). `end_session` on every scenario
with a stop condition. `disconnectDelaySeconds` defaults to **15** if omitted.

```json
{
  "tools": {
    "end_session": {
      "should_register": true,
      "add_to_system_prompt": true,
      "tool_settings": { "disconnectDelaySeconds": 8 }
    }
  }
}
```

| Tool                                    | When                                         |
| --------------------------------------- | -------------------------------------------- |
| `end_session`                           | Any scenario that can hang up                |
| `card` / `mcq`                          | Coaching content + checks                    |
| `image_generation` / `slide_generation` | Teaching visuals; one primary tool (not SIP) |
| `browser`                               | Live product demo                            |
| `google_slides`                         | Slide demo (`embedUrl` in settings)          |
| `knowledge_base_search`                 | Only if KB already attached in Studio        |
| `collect_data` / `cold_transfer`        | SIP / phone                                  |
| `memory_search`                         | When `memory.is_memory` is on                |

### Tool defaults by situation

| Tool                    | Cold | Sales | Coaching        |
| ----------------------- | ---- | ----- | --------------- |
| `end_session`           | yes  | yes   | yes             |
| `card`                  | no   | no    | yes (A/B)       |
| `mcq`                   | no   | no    | yes             |
| `emoji_reaction`        | no   | no    | yes             |
| `slide_generation`      | no   | no    | Pattern C       |
| `image_generation`      | no   | no    | optional        |
| `memory_search`         | no   | no    | if memory on    |
| `knowledge_base_search` | no   | no    | if KB in Studio |

`end_session` is required whenever termination rules exist. Prefer
`disconnectDelaySeconds: 8` for emotional or sensitive calls.

**SIP / phone** only runs server tools: `end_session`, `knowledge_base_search`,
`collect_data`, `cold_transfer` (and on-demand `ask_conductor` if configured).
Visual tools are silently unavailable — do not prescribe them for SIP.

If `tools_config` is omitted on create, `end_session` and `memory_search`
default to on.

---

## session_analysis / memory / transcribe

```json
{
  "is_auto_analysis": true,
  "is_auto_submit": true,
  "enable_extraction": false
}
```

Always set `is_auto_analysis: true` and `is_auto_submit: true` when the team
will read reports (session-analyst depends on this).
`evaluation_target` for multi-party sessions (e.g. `"Sales Rep"`).
`enable_extraction` + `extraction_vars` for structured capture, e.g.
`[{"name": "sentiment", "description": "...", "type": "text"}]`.
Types: `text` \| `number` \| `boolean` \| `list` \| `date`.
`memory`: `{ "is_memory": true }` when the coach should recall prior sessions.
`transcribe_config` must match `appearance.language_code` if set.

Rubrik _templates_ live in the recipes (cold call → lead screening; sales →
rep scorecard; coaching → engagement-weighted). See
[rubrik.md](rubrik.md).

---

## Dynamic variables

`{{ var }}` in instructions is filled from `?t_var=` URL params. Every
variable needs a missing-value fallback in `ai_instructions`.

## What MCP cannot set

Public create/update **omits** `knowledge_base_ids` and `custom_function_ids`.
Do not invent those fields. Attach KBs and custom functions in Scenario
Studio, then enable the matching tool flags here.

Public `type` is only `default` | `super` | `composite`. Do not send
`meet_assist`, `quiz`, or `coding`.

## Key Files

- [model-selection.md](model-selection.md) — `ai_model_config` stamps
- [ai-instructions.md](ai-instructions.md) — instruction shape
- [../../scenario-authoring.md](../../scenario-authoring.md) — authoring principles
- [../../scenario-recipes/index.md](../../scenario-recipes/index.md) — situation patterns
- [../../../features/mcp/connect.md](../../../features/mcp/connect.md) — tool rules
