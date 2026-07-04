# Scenario Field Reference

Field schema, valid enum values, and config defaults for `create_scenario` /
`update_scenario` payloads. Load the MCP tool schema before calling — this file
explains what to put in each field; the tool schema is the source of truth for
types.

---

## Top-Level Fields

| Field | Required | Notes |
|---|---|---|
| `id` | update only | Omit for `create_scenario`; required for `update_scenario` |
| `name` | create | Short display title |
| `type` | | `"default"` (almost always), `"super"` (multi-stage), `"composite"`. Never write `super_agent` — the schema rejects it |
| `user_friendly_description` | recommended | 1-2 public-facing sentences |
| `ai_instructions` | create | The AI system prompt — the most important field |
| `user_instructions` | recommended | Pre-session guidance for the human |
| `rubrik` | recommended | Evaluation criteria for session analysis |
| `pdf_context` | | Extra context document text |
| `is_public` | | Default `true`. `false` = only accessible via access token |
| `is_recording` | | Always `true` for voice scenarios |
| `passcode` | | Optional passcode gate |
| `analysis_access` | | `"default"` (run page only), `"always"`, `"never"` |
| `user_metadata` | | Filterable key-value metadata |

---

## ai_model_config

```json
{
  "provider": "Landmass",
  "model": "cascade-01",
  "tts_provider": "cartesia",
  "tts_voice_id": "f786b574-daa5-4673-aa0c-cbe3e8534c02",
  "llm_provider": "google_vertex",
  "stt_provider": "deepgram"
}
```

Valid values:

- `provider`: `Ocean` | `Galaxy` | `Landmass`
- `model`: `medium` | `medium-super` | `medium-stable` | `cascade-01`
- `tts_provider`: `cartesia` | `openai` | `elevenlabs`
- `llm_provider`: `google` | `google_vertex` | `openai` | `cerebras`
- `stt_provider`: `deepgram` | `cartesia`

### When to use which

| Scenario | provider | model | tts |
|---|---|---|---|
| Voice (cold call, sales, coach) | Landmass | cascade-01 | cartesia |
| Text-only | Ocean | medium | — |
| High-quality text | Ocean | medium-super | — |

If unsure, omit `ai_model_config` entirely — platform defaults are sensible.

### Voice quick-reference (Cartesia `tts_voice_id`)

| Persona | Gender | Locale | Voice ID |
|---|---|---|---|
| Indian female | F | en-IN | `343e5d21-9db1-4efa-a1bc-196777cce1bb` |
| American female | F | en-US | `f786b574-daa5-4673-aa0c-cbe3e8534c02` |
| American male | M | en-US | `a167e0f3-df7e-4d52-a9c3-f949145f52ef` |
| German male | M | de-DE | `384b625b-da5a-4b3a-8c5d-b9e1a5f30497` |

---

## strategy

```json
{
  "skip_auto_start": false,
  "system_instructions_template": "minimal",
  "welcome_instructions": "Start with: Hi [name]! This is [Agent] from [Company]. [Reason]. [Permission ask]. Then STOP and wait.",
  "filler_words": "hmm, right, okay, sure, mm, got it",
  "silence": {
    "silence_threshold": 5000,
    "end_session": false,
    "force_agent_to_speak": true
  },
  "conductor": {
    "enabled": true,
    "prefix": "CONDUCTOR:",
    "messages": [
      {
        "time_seconds": 300,
        "message": "Wrap up the call — confirm next step, end warmly. Use end_session tool.",
        "end_turn": true
      }
    ]
  }
}
```

| Field | Effect |
|---|---|
| `skip_auto_start` | `false` = AI speaks first; `true` = user speaks first |
| `welcome_instructions` | The AI's exact first beat. Directive form, never quoted speech |
| `system_instructions_template` | `"minimal"` keeps the compiled prompt lean (recommended) |
| `silence.silence_threshold` | ms of silence before action (4000-8000 typical) |
| `silence.end_session` | `true` = disconnect on silence; `false` = nudge and continue |
| `silence.force_agent_to_speak` | `true` = AI re-engages on silence |
| `conductor.messages[]` | Timed mid-call directives (wrap-up timers) |
| `filler_words` | Comma-separated TTS filler phrases — cascade models only |

### Strategy defaults by type

| Type | skip_auto_start | conductor wrap-up | welcome_instructions |
|---|---|---|---|
| Cold call (AI calls) | `false` | 300-450s | Beat 1 directive |
| Sales roleplay (AI is prospect) | `true` | 600-900s | Opening line as prospect |
| Coaching | `false` | 900-1200s | Greeting + session intro |

---

## appearance

```json
{
  "voice": "Aoede",
  "language_code": "en-US",
  "avatar_url": null
}
```

`language_code` (BCP 47) must match the scenario locale. Supported:
`en-US en-GB en-IN en-AU de-DE es-US es-ES fr-FR fr-CA hi-IN pt-BR ar-XA id-ID
it-IT ja-JP tr-TR vi-VN bn-IN gu-IN kn-IN ml-IN mr-IN ta-IN te-IN nl-NL ko-KR
cmn-CN pl-PL ru-RU th-TH`

---

## tools_config

Every tool has two booleans: `should_register` (AI can call it) and
`add_to_system_prompt` (AI is told when/how to use it).

```json
{
  "tools": {
    "end_session": {
      "should_register": true,
      "add_to_system_prompt": true,
      "tool_settings": { "disconnectDelaySeconds": 5 }
    }
  }
}
```

### Tool defaults by scenario type

| Tool | Cold call | Sales | Coaching |
|---|---|---|---|
| `end_session` | **yes** | **yes** | **yes** |
| `card` | no | no | **yes** (Pattern A/B) |
| `mcq` | no | no | **yes** |
| `emoji_reaction` | no | no | yes |
| `slide_generation` | no | no | Pattern C only |
| `image_generation` | no | no | optional (visual scenes) |
| `memory_search` | no | no | if memory enabled |
| `knowledge_base_search` | no | no | optional |

`end_session` is required for every scenario with termination rules. Use
`disconnectDelaySeconds: 8` for emotional or sensitive calls.

---

## session_analysis

```json
{
  "is_auto_analysis": true,
  "is_auto_submit": true,
  "enable_extraction": false,
  "evaluation_target": null
}
```

- Always set `is_auto_analysis: true` and `is_auto_submit: true` — this is what
  makes sessions analyzable by the session-analyst skill.
- `evaluation_target`: set for multi-participant sessions (e.g. `"Sales Rep"`).
- `enable_extraction` + `extraction_vars`: structured data capture, e.g.
  `[{"name": "sentiment", "description": "...", "type": "text"}]`.
  Types: `text`, `number`, `boolean`, `list`, `date`.

---

## rubrik patterns

Structure: 4-6 categories with weights summing to 100%, observable criteria,
performance thresholds.

**Cold call / SDR — evaluates the LEAD (the human playing the prospect):**

```
# [Company] — Lead Qualification Report
Evaluate the LEAD (the prospect) — not the AI caller.
## Qualification Score (1-10) [criteria per band]
## Structured Data [fields the team needs]
## Follow-Up Recommendation
```

**Sales roleplay — evaluates the REP (the human user):**

```
# [Company] — Sales Performance Report
Evaluate the SALES REP (the human user) — not the AI prospect.
## Score (1-10) [performance criteria]
## Skills Assessment [discovery, objection handling, closing]
## Improvement Areas
```

**Coaching — engagement-weighted:**

```
## Evaluation Dimensions
### Practice Execution (35%)
### Concept Understanding (25%)
### Situation Handling (25%)
### Feedback Absorption (15%)
```

---

## Dynamic variables

`{{ var_name }}` placeholders in `ai_instructions` are filled per session from
URL parameters (`?t_company=Acme`). Every variable MUST have a documented
missing-value fallback in the instructions ("If `lead_name` is blank, open
with 'Hi, am I speaking with the homeowner?'") — otherwise the agent speaks
the raw placeholder aloud.
