# Scenario control fields

Every non-prose field except `ai_model_config`
([model-selection.md](model-selection.md)), `ai_instructions`
([ai-instructions.md](ai-instructions.md)), and `rubrik`
([rubric.md](rubric.md)). The live `ttai:create_scenario` /
`ttai:update_scenario` schema is the type source of truth; update merge rules
are in [../mcp/tools.md](../mcp/tools.md#updating-a-scenario).

There is **no** `welcome_instructions` field. With `skip_auto_start: false` the
runtime asks the model to "greet the user with your opening line", so the
opening lives in the FLOW section of `ai_instructions`.

## Contents

- Top-level
- strategy
- appearance
- tools_config (and Google Slides decks)
- session_analysis, memory, transcribe_config
- meeting_config
- Dynamic variables and auto-updates
- Linked resources

## Top-level

Create **requires** `name` and `ai_instructions`; set the rest deliberately.

- `id` — update only.
- `name` — short display title.
- `type` — `default` (usual), `super`, `quiz`, `coding`, `meet_assist`.
- `user_friendly_description` — 1–2 public sentences.
- `user_instructions` — what the human reads before starting.
- `stages` — `super` only: stages with `flows` (sub-agents).
- `is_public` — default `true`; send it on create. `false` needs an access
  token, blocks phone calls, and needs `session_credentials: "bot_uc"` on every
  meeting bot. Private only on request
  ([../entities/account-and-access.md](../entities/account-and-access.md#scenario-visibility)).
- `is_recording` — default `false`; set `true` for voice.
- `recording_mode` — `standard` (default) or `turbo`.
- `passcode`, `starts_at`, `ends_at`, `access`, `analysis_access` — visibility;
  see [../entities/account-and-access.md](../entities/account-and-access.md).
- `user_metadata` — filterable key-value pairs (string values).
- `save_as_version` — update only: label (1–100 chars) archiving the prior
  state.

`super` needs a Landmass model and a non-empty first stage with flows (each
flow: `name`, `instructions`; stage: `name`, `instructions`, `goal`, `role`).
`coding` uses `coding_question`. Never send `super_agent`.

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

- `skip_auto_start` — `false` = AI speaks first from FLOW; `true` = waits for
  the human.
- `system_instructions_template` — `"minimal"` (~300-token wrapper) or omit for
  the standard wrapper (~550 tokens).
- `silence` — **omit → 120 s of silence, then hang up.** Set it for any
  conversation.
- `conductor` — timed or on-demand directives (below).
- `max_duration_seconds` — hard cap; a wrap-up directive fires 30 s before it.
- `filler_words` — comma-separated phrases between turns; Cascade only.
- `backchannel` — Cascade only: `enabled`, `words`, `frequency`
  (`low`/`medium`/`high`), `volume`.
- `ambient_sound` — looped background audio from an uploaded file.
- `push_to_talk` — mic muted until the user holds Space (web and embed only).
- `disable_transcription` — turns speech-to-text off.
- `ext_sess_upload` — lets users upload external sessions in the app.

**Silence:** `silence_threshold` in ms (minimum 3000); `end_session: true` hangs
up, `false` keeps the call alive; `force_agent_to_speak: true` injects a fixed
check-in, `false` lets a small model decide whether to nudge and what to say.
`nudge_text` makes every nudge that exact text, sent as written with no
`[conductor]` prefix; omit it for the default. Practice default:
5000–8000, `end_session: false`. Roleplay counterparts (prospect, interviewer)
use `force_agent_to_speak: false` so the nudge stays in character; coaches and
other AI-led hosts use `true`. Phone callers start from their stamp
([model-selection.md](model-selection.md)): Cascade at 3000 with `false`;
Gemini Live voice-to-voice at 4000 (short calls) to 6000 (long ones) with
`true`. Tune from transcripts. Observers and facilitators may want silence to
stay silent — see the meeting recipe.

**Conductor messages:** `trigger` `"time"` (needs `time_seconds`) or
`"on_demand"` (at most one; the agent calls it as a tool when it needs
re-alignment). `content_mode` `"instruction"` (verbatim, default) or `"thought"`
(rewritten by a small model; an on-demand thought must reference
`{{ transcript }}`). `end_turn: true` interrupts current speech. Do not send the
retired `prefix`.

| Situation      | `skip_auto_start` | Silence nudge   | Wrap-up timer | Opening                                        |
| -------------- | ----------------- | --------------- | ------------- | ---------------------------------------------- |
| Cold call      | `false`           | by stamp        | 300–450 s     | FLOW Beat 1 directive                          |
| Sales roleplay | `true`            | model (`false`) | 600–900 s     | Rep opens; prospect replies with one bare line |
| Coaching       | `false`           | fixed (`true`)  | 900–1200 s    | FLOW greeting + intro                          |

## appearance

```json
{ "voice": "Aoede", "language_code": "en-US" }
```

- `voice`: realtime voice name — `Aoede` (default), `Puck`, `Kore`, `Charon`,
  `Fenrir`. Galaxy uses it natively; Ocean maps it to its own voices. Not a TTS
  voice ID — those go in `ai_model_config.tts_voice_id`.
- `language_code` (BCP 47) must match the Scenario's language:
  `en-US en-GB
  en-IN en-AU de-DE es-US es-ES fr-FR fr-CA hi-IN pt-BR ar-XA id-ID it-IT
  ja-JP tr-TR vi-VN bn-IN gu-IN kn-IN ml-IN mr-IN ta-IN te-IN nl-NL ko-KR
  zh-CN pl-PL ru-RU th-TH`.
  Match `transcribe_config.language` when set.
- `start_sound`: `countdown_blip` (default), `ringtone`, `click`, `none`.

Faces come from `ttai:list_resources` with resource type `avatar` (`type`
`static` default, `hybrid`, or `live`); request `include_fields` `url` and
`avatar_id`. Keep `voice` and `language_code` when you set one.

- Static: `avatar_url` = `url`; clear `live_avatar_id` and
  `live_avatar_provider`; set `hybrid_avatar.enabled: false` if present.
- Hybrid: `hybrid_avatar: { enabled: true, thinking_url, speaking_url }` from
  the `thinking` and `speaking` slots; clear both live fields.
- Live (Landmass models only): `live_avatar_provider` = `provider` (`avatario`,
  `liveavatar` for HeyGen, `anam`, `protoface`), `live_avatar_id` = `avatar_id`,
  `avatar_url` = `url`.

## tools_config

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

- `should_register` — the AI can call the tool. `add_to_system_prompt` — the
  runtime adds the tool's usage guidance to the prompt.
- Send `tools_config` explicitly on create. When omitted, the server registers a
  broad default set (cards, quizzes, images, knowledge-base search, and more)
  and prompts only `end_session`.
- On update, `tool_settings` is replaced whole: read, merge, write.

- `end_session` — any Scenario that can close; `disconnectDelaySeconds` default
  15.
- `card`, `mcq`, `emoji_reaction` — coaching content, knowledge checks,
  reactions. An MCQ never shares a turn with an explanation: check in and wait,
  then show it, then read it
  ([../recipes/coaching.md](../recipes/coaching.md), Mandatory Rules 3–4).
- `image_generation`, `slide_generation` — teaching visuals; pick one primary
  visual tool.
- `browser` — live product demo; settings `initialUrl`, recorded `steps`;
  `contextId` is set by the platform.
- `google_slides` — teach or demo from a Google Slides deck; settings
  `embedUrl` (the published `/pubembed` link). See Google Slides decks below.
- `knowledge_base_search` — only when `knowledge_base_ids` attaches a Knowledge
  Base (legacy RAG). Knowledge FileBases need no `tools_config` entry: linking
  `knowledge_filebase_ids` gives the agent `getKnowledgeBite` and
  `listKnowledgeBites`.
- `collect_data` — phone: record structured answers.
- `cold_transfer` — phone: transfer to `tool_settings.transfer_to`.

Phone (SIP) runs only server-side tools: `end_session`, `knowledge_base_search`,
`collect_data`, `cold_transfer`, `ask_conductor` when an on-demand conductor
message exists, `getKnowledgeBite` and `listKnowledgeBites` when
`knowledge_filebase_ids` is set, and attached Custom Functions and MCP servers.
Visual tools are unavailable on phone — never prescribe them there.

| Tool                    | Cold call | Sales | Coaching          |
| ----------------------- | --------- | ----- | ----------------- |
| `end_session`           | yes       | yes   | yes               |
| `card`, `mcq`           | no        | no    | yes               |
| `emoji_reaction`        | no        | no    | optional          |
| `slide_generation`      | no        | no    | framework pattern |
| `image_generation`      | no        | no    | optional          |
| `knowledge_base_search` | if KB     | no    | if KB             |

Use `disconnectDelaySeconds: 8` for emotional or sensitive calls.

### Google Slides decks

For any Scenario that teaches, demos or interviews from a deck the user already
has. The agent cannot see the deck: the SLIDE MAP in `ai_instructions` is all
it knows about each slide.

**Link**

- `tool_settings.embedUrl` is the published link: Google Slides → File → Share
  → Publish to web, ending in `/pubembed`. For a `/pub` link, change the suffix
  to `/pubembed`.
- An `/edit` link, or a `/d/<id>/embed` link to an unpublished deck, renders
  only for people with access to the file; learners see an access wall.
- If you can open the link, check its title matches the deck before wiring it.

**Slide content**

- Build the SLIDE MAP from the deck's real text. If you cannot open the deck,
  ask the user for its text (an export or a slide-by-slide outline). Never guess
  what a slide shows.

**Prompt** — add a SLIDE MAP section, then cue every FLOW step:

```
## SLIDE MAP — google_slides (show the slide first, then talk)
The deck has exactly <N> slides. Show a slide with google_slides (command
show, slideNumber) once, right when the FLOW reaches it. Track the current
slide yourself; every session starts at slide 1. Never read a slide aloud;
explain it in your own words. If they ask to go back or repeat, finish your
current answer, then show that slide.
- Slide 1 — <what it shows, including any figure the agent must say>
- Slide 2 — <…>
```

- One line per slide, in deck order, with its key figures. A figure that is
  only on the slide does not exist for the agent.
- Start each FLOW step with its slide ("Show slide 5.") so the agent never
  infers when to advance. One slide per turn: explain it in a few sentences,
  check in, and wait.
- A slide that shows the model answer for a practice moment comes after the
  learner's attempt, not before.
- An MCQ after a slide follows Mandatory Rules 3–4 in
  [../recipes/coaching.md](../recipes/coaching.md).

**Config**

- One visual surface: with `google_slides` on, set `card`, `image_generation`
  and `slide_generation` to `should_register: false` unless each has a distinct
  job in the FLOW.
- Mention the deck in `user_instructions` and `user_friendly_description` (the
  coach walks you through N slides) so learners expect a screen.

**Converting a card Scenario to slides**

- Read the live Scenario first and keep its curriculum: slides replace card
  moments; content changes only if the user asks.
- Update only the slide pieces with `ttai:update_scenario` and a
  `save_as_version` label for the new version.

**Deck edits** — adding, removing or reordering slides breaks every
`slideNumber` cue. After any deck edit, re-count the slides and update the
SLIDE MAP and the FLOW cues together.

**Verify** — re-fetch the Scenario: `embedUrl` ends in `/pubembed`, other
visual tools are off, and the SLIDE MAP count matches the deck. Then run one
session to see the deck render; layout fixes happen in the deck, not the
Scenario.

## session_analysis, memory, transcribe_config

```json
{ "is_auto_analysis": true, "is_auto_submit": true, "enable_extraction": false }
```

- `is_auto_analysis: true` whenever anyone will read reports; analysis does not
  run otherwise. `is_auto_submit: true` uploads a recorded session without
  asking the user (it matters only with `is_recording`).
- `evaluation_target`: the participant to score in multi-party sessions.
- `enable_extraction` + `extraction_vars` (`name`, `description`, `type`:
  `text`|`number`|`boolean`|`list`|`date`) for structured capture.
- `analysis_tier`: `standard` (default) or `premium`. `multimodal_analysis`
  analyzes video/audio (plan-gated); `video_config` clips or downsamples.
- `admin_email` (comma-separated) with `email_analysis` / `email_transcript`
  mails reports.
- `post_session_function_id` and `pre_connect.pre_connect_function_id` attach
  existing Custom Functions (after analysis; before a phone session).
- `learning_config` (`user_prompt`, `analysis_limit`) drives transcript-based
  Scenario learning.
- `memory: { "is_memory": true }` lets the agent recall a returning user's prior
  sessions; `short_mem_prompt` shapes the summary kept per session.
- `transcribe_config.language` must match `appearance.language_code`.

## meeting_config

For meeting bots and phone calls: `is_notetaker` (join without AI), `audit_mode`
(notetaker: one combined session), `public_meet_bot` (allow deployment from the
public meeting embed), and `amd_config` (answering machine detection for
outbound calls; default `null`):

- `enabled` — default `true` once `amd_config` is sent.
- `voicemail.action` — `hangup` (default) or `leave_message`.
- `voicemail.message` — spoken in the Scenario's voice; without one, the call
  hangs up. Write it in the callee's language: who is calling and why. Promise
  a callback only when the campaign retries.
- `prompt` — replaces the voicemail classifier's system prompt (up to 8,000
  characters). Leave it unset unless real calls are misclassified.
- On update, `amd_config` is replaced whole: resend `voicemail`.

Voicemail has one handler. AMD on: remove any voicemail branch from
`ai_instructions`. AMD off: give the prompt a voicemail branch — on a recorded
greeting or beep, say one short message once, then end the call without
waiting for a reply.
Retries after voicemail:
[../entities/scenario-engine.md](../entities/scenario-engine.md#calling-campaigns).

## Dynamic variables and auto-updates

`{{ var }}` in instructions is filled per run: `?t_var=` on the run link,
`dynamic_vars` on calls and bots. The field runs full Liquid (`default:`
filters, `{% if %}` and `{% case %}` branches); patterns and limits in
[dynamic-variables.md](dynamic-variables.md). `auto_update_config.placeholders`
fills `[[ key ]]` placeholders from a web search or URLs on a cache schedule;
configure only from the live schema.

## Linked resources

`knowledge_base_ids` (legacy RAG), `knowledge_filebase_ids` (knowledge
FileBases, see [tools.md](../mcp/tools.md#knowledge-filebases)),
`custom_function_ids`, `pre_connect`, and `mcp_server_ids`
(`catalog:<id>` or `custom:<id>`) attach existing resources. Discover safe IDs
with `ttai:list_resources` (types in `ttai:read_guide`), or use IDs from a
trusted workspace call. The server rejects IDs you cannot see. An ID list you
send replaces the stored list. Of these, MCP creates only knowledge FileBases
(`ttai:create_filebase`), and it never reveals their sensitive configuration.

## Key Files

- [model-selection.md](model-selection.md) — `ai_model_config`
- [ai-instructions.md](ai-instructions.md) — instruction shape
- [runtime.md](runtime.md) — how controls behave at runtime
- [dynamic-variables.md](dynamic-variables.md) — variables and branching
