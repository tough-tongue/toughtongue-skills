# Scenario model selection

How to stamp `ai_model_config`. Load the `ttai` tool schema before calling — it
is the type source of truth.

Sibling files: [control.md](control.md) (strategy, tools, appearance),
[ai-instructions.md](ai-instructions.md). On update, `ai_model_config` is
replaced whole — resend the full stamp, including `tts_voice_id`.

## Contents

- Three pipelines (realtime, separate TTS, Cascade)
- Realtime tuning (Gemini Live, OpenAI realtime)
- Voice-to-voice on the phone
- Need-based overrides
- Situation → starting stamp
- Name pipeline variants
- Cartesia voice IDs
- Catalog families

---

Always set `provider` + `model`. Three **families**: Galaxy, Ocean, Landmass.
The runtime builds one of three pipelines. Prefer the **situation stamp** table
for create defaults; use need-based rows for overrides.

`cascade` is current; never author legacy `cascade-01`. Never invent a
`tts_voice_id`. Galaxy and Ocean both support SIP. The server enforces model
access: some plans cannot save Ocean models (`403`), and `super` Scenarios
require Landmass.

## 1. Realtime — model speaks

Native transport. Only `provider` + `model`. Do not set `tts_*` / `llm_*` /
`stt_*`.

```json
{ "provider": "Galaxy", "model": "medium-stable" }
```

```json
{ "provider": "Ocean", "model": "medium-stable" }
```

Landmass realtime (`super` Scenarios, live avatars, or other Landmass-only
features) — still no `tts_*`:

```json
{ "provider": "Landmass", "model": "gemini" }
```

Galaxy `*-stable*` → Landmass `gemini-stable`. Ocean `medium-stable` → Landmass
`openai`.

## 2. Realtime with separate TTS

Landmass realtime (`gemini`, `gemini-stable`, or `openai`) + `tts_*` when the
user needs a selected/cloned voice but still wants a realtime listener. Load
[../recipes/cascade-tts.md](../recipes/cascade-tts.md) **only** for full Cascade
— not this path.

```json
{
  "provider": "Landmass",
  "model": "gemini",
  "tts_provider": "cartesia",
  "tts_voice_id": "[SELECTED_VOICE_ID]"
}
```

## 3. Cascade — STT + LLM + TTS

Landmass `cascade` for full pipeline control, selected external voice, or
text-first speech. Set component fields when choosing deliberately; omitted ones
default to OpenAI TTS, a Google Gemini LLM (the catalog default model), and
Deepgram STT. Load [../recipes/cascade-tts.md](../recipes/cascade-tts.md) into
`ai_instructions`.

```json
{
  "provider": "Landmass",
  "model": "cascade",
  "tts_provider": "cartesia",
  "tts_voice_id": "[SELECTED_VOICE_ID]",
  "llm_provider": "google",
  "llm_model": "gemini-3.5-flash-lite",
  "stt_provider": "deepgram"
}
```

`tts_provider`: `cartesia` | `openai` | `elevenlabs` | `sarvam`.\
`llm_provider`: `google` | `google_vertex` | `openai` | `cerebras` | `sarvam`.\
`stt_provider`: `deepgram` | `cartesia` | `sarvam` | `google`.

- `tts_voice_id` — the provider's own ID: a Cartesia UUID, or an ElevenLabs
  voice ID the user supplies.
- `llm_model` — pin it to keep the model you tested; omitted, the catalog
  default runs.
- `stt_keyterms` — names and jargon to boost; Deepgram only.
- `stt_mode` — Sarvam only: `transcribe` (default) or `codemix`, which keeps
  Hinglish as Devanagari Hindi plus Latin English. A Hinglish phone caller
  pairs `stt_provider: sarvam` with `stt_mode: codemix`.
- `speed` — TTS rate multiplier (0.25–4); Cartesia clamps it to 0.6–2.0,
  ElevenLabs to 0.8–1.2.

## Realtime tuning

`gemini_realtime_config` tunes Gemini Live (Landmass `gemini*` only):
`start_of_speech_sensitivity` / `end_of_speech_sensitivity` (`low`|`high`),
`silence_duration_ms` (100–5000), `prefix_padding_ms` (0–1000),
`activity_handling` (`interrupt`|`no_interruption`), `thinking_level`
(`minimal`|`low`|`medium`|`high`), `thinking_budget` (0–24576; `0` turns
thinking off), `proactivity`, `enable_affective_dialog`. Unsupported settings
are dropped server-side.

`openai_realtime_config` tunes OpenAI realtime (Ocean and Landmass `openai`) in
OpenAI's own session shape: `audio.input.turn_detection`,
`audio.input.noise_reduction`, `audio.input.transcription`, `reasoning`, and
more. Set only what should differ; unknown top-level keys are rejected. To
spell names and jargon right in the transcript, give the transcription model a
prompt:

```json
{
  "provider": "Landmass",
  "model": "openai",
  "openai_realtime_config": {
    "type": "realtime",
    "audio": {
      "input": {
        "transcription": {
          "language": "en",
          "model": "gpt-4o-mini-transcribe",
          "prompt": "Acme, Priya Raman, Northwind, SOC 2, RFP"
        }
      }
    }
  }
}
```

## Voice-to-voice on the phone

For open phone conversations (research calls, confirmations, follow-ups),
stamp Landmass `gemini-stable` with a tuned `gemini_realtime_config`. These are
starting values from live phone callers; adjust them from transcripts.

```json
{
  "provider": "Landmass",
  "model": "gemini-stable",
  "gemini_realtime_config": {
    "start_of_speech_sensitivity": "low",
    "end_of_speech_sensitivity": "low",
    "silence_duration_ms": 900,
    "thinking_budget": 0,
    "proactivity": false,
    "enable_affective_dialog": false
  }
}
```

- `start_of_speech_sensitivity: low` — line noise and backchannels ("hmm",
  "yes") cut the agent off less often.
- `end_of_speech_sensitivity: low` — the agent waits for a finished thought
  instead of jumping in at a short pause.
- `silence_duration_ms` — the pause that ends their turn. Short confirmation
  calls run 700 ms; long research calls run 900 ms so slow speakers can finish.
  Raise it if the agent talks over people; lower it if replies feel slow.
- `thinking_budget: 0` — no thinking delay before each reply.
- `proactivity: false`, `enable_affective_dialog: false` — the agent answers
  every turn in a steady style, instead of skipping input it judges irrelevant
  or mirroring the caller's mood.

The voice is `appearance.voice`; add no `tts_*`. Silence starting values for
each phone stamp are in [control.md](control.md#strategy).

## Need-based overrides

| Need                                  | Start with                            |
| ------------------------------------- | ------------------------------------- |
| Roleplay / multilingual chat          | Galaxy `medium-stable`                |
| Browser tools / interactive artifacts | Ocean `medium-stable`                 |
| Selected or cloned external voice     | Landmass realtime + `tts_*`           |
| Polished TTS / full STT·LLM·TTS       | Landmass `cascade`                    |
| `super` Scenario / live avatar        | Landmass `gemini` or `gemini-stable`  |
| Silent meeting assistant              | `meet_assist`; verify its live schema |

## Situation → starting stamp

Starting defaults:

| Situation                          | Stamp                               | Pipeline |
| ---------------------------------- | ----------------------------------- | -------- |
| Outbound phone — scripted lines    | Landmass `cascade` + external voice | Cascade  |
| Outbound phone — open conversation | Landmass `gemini-stable`, tuned     | Realtime |
| Sales roleplay                     | Galaxy `medium-stable`              | Realtime |
| Coaching                           | Ocean `medium-stable`               | Realtime |
| Demo — browser                     | Ocean `medium-stable`               | Realtime |
| Demo — slides                      | Landmass `cascade` + Cartesia       | Cascade  |
| Text-only                          | Ocean `medium-stable`               | —        |
| High-quality text                  | Ocean `medium-super`                | —        |

Outbound phone calls have two proven stamps. Pick by the call:

- **Scripted lines → Cascade.** Pick it for a client's tested lines said as
  written, a fixed flow, a chosen Cartesia or ElevenLabs voice, Hinglish that
  needs Sarvam `codemix` transcription, or a pinned LLM. Load
  [../recipes/cascade-tts.md](../recipes/cascade-tts.md).
- **Open conversation → Gemini Live voice-to-voice.** Research calls,
  confirmations, follow-ups: lines are tone references, the agent listens more
  than it talks, and turn-taking matters more than exact wording. Use the tuned
  stamp above.

Slide demos use Cascade too: narrated AI speech wants controllable TTS
(Cartesia) + explicit STT/LLM. Load
[../recipes/cascade-tts.md](../recipes/cascade-tts.md).

Overrides:

- Outbound phone → Ocean `medium-stable` only if the account cannot use either
  Landmass stamp. Never Galaxy: phone voice-to-voice is Landmass
  `gemini-stable`, which takes `gemini_realtime_config`.
- Unsure which phone stamp fits → ask the user, or build both as named
  variants (below) and compare.
- Coaching → Galaxy if Ocean is unavailable.
- Sales also needs `skip_auto_start: true` (see [control.md](control.md)).

## Name pipeline variants

To compare pipelines on the same call, run one Scenario per pipeline with the
same task, and label each the same way:

- `name` ends with the pipeline: `Acme Renewals - Gemini Live`,
  `Acme Renewals - Cascade`.
- The last line of `user_friendly_description` names it: "Voice-to-voice:
  Gemini Live native audio." or "Cascade: Deepgram transcription, Gemini 3.5
  Flash-Lite and a Cartesia voice."

Results stay comparable per Scenario, and anyone reading a session sees which
pipeline ran.

## Cartesia `tts_voice_id` quick-reference

| Persona         | Locale | `tts_voice_id`                         |
| --------------- | ------ | -------------------------------------- |
| Indian female   | en-IN  | `343e5d21-9db1-4efa-a1bc-196777cce1bb` |
| American female | en-US  | `f786b574-daa5-4673-aa0c-cbe3e8534c02` |

Use these or a voice ID the user supplies; never invent one. Native Galaxy/Ocean
voices live in `appearance.voice` ([control.md](control.md)): `Aoede`, `Puck`,
`Kore`, `Charon`, `Fenrir`. Landmass Gemini Live reads it too and takes other
Gemini voice names (live phone callers use `Leda`); use a name the user picked,
never a guess.

## Catalog families

- Galaxy: `medium`, `medium-stable`, `medium-super`, and noise-cancelling
  `medium-nc`, `medium-stable-nc`
- Ocean: `medium-stable`, `medium-super`
- Landmass: `gemini`, `gemini-stable`, `openai`, `cascade`

Other codes may appear on older Scenarios (`cascade-01`, `medium-preview`,
`openai-1`, `openai-1.5`, `gemini-preview`); they are legacy or internal — do
not author them.

Tool schemas and server validation remain authoritative.

## Key Files

- [control.md](control.md) — strategy, tools, appearance, flags
- [../recipes/cascade-tts.md](../recipes/cascade-tts.md) — Cascade speech rules
