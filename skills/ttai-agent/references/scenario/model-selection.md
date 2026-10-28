# Scenario model selection

How to stamp `ai_model_config`. Load the `ttai` tool schema before calling — it
is the type source of truth.

Sibling files: [control.md](control.md) (strategy, tools, appearance),
[ai-instructions.md](ai-instructions.md). On update, `ai_model_config` is
replaced whole — resend the full stamp, including `tts_voice_id`.

## Contents

- Three pipelines (realtime, separate TTS, Cascade)
- Need-based overrides
- Situation → starting stamp
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
default to OpenAI TTS, a Google `gemini-3.1-flash-lite` LLM, and Deepgram STT.
Load [../recipes/cascade-tts.md](../recipes/cascade-tts.md) into
`ai_instructions`.

```json
{
  "provider": "Landmass",
  "model": "cascade",
  "tts_provider": "cartesia",
  "tts_voice_id": "[SELECTED_VOICE_ID]",
  "llm_provider": "google_vertex",
  "llm_model": "gemini-3.1-flash-lite",
  "stt_provider": "deepgram"
}
```

`tts_provider`: `cartesia` | `openai` | `elevenlabs` | `sarvam`.\
`llm_provider`: `google` | `google_vertex` | `openai` | `cerebras` | `sarvam`.\
`stt_provider`: `deepgram` | `cartesia` | `sarvam` | `google`. `stt_keyterms`:
terms to boost Deepgram recognition. `stt_mode` (Sarvam only): `codemix` keeps
Hinglish as Devanagari Hindi plus Latin English. `speed`: TTS rate multiplier
(Cartesia 0.6–2.0, ElevenLabs 0.8–1.2).

`gemini_realtime_config` tunes Gemini Live (Landmass `gemini*` only):
`start_of_speech_sensitivity` / `end_of_speech_sensitivity` (`low`|`high`),
`silence_duration_ms`, `prefix_padding_ms`, `activity_handling`
(`interrupt`|`no_interruption`), `thinking_level`
(`minimal`|`low`|`medium`|`high`), `thinking_budget`, `proactivity`,
`enable_affective_dialog`. Unsupported settings are dropped server-side.
`openai_realtime_config` tunes OpenAI realtime (Ocean, Landmass `openai`) in
OpenAI's session shape; `audio.input.transcription.prompt` lists names to spell
right.

### Voice-to-voice on the phone

For open phone conversations (research calls, confirmations, follow-ups),
stamp Landmass `gemini-stable` with these starting values, then tune from
transcripts:

```json
{
  "provider": "Landmass",
  "model": "gemini-stable",
  "gemini_realtime_config": {
    "start_of_speech_sensitivity": "low",
    "end_of_speech_sensitivity": "low",
    "silence_duration_ms": 900,
    "thinking_budget": 0,
    "proactivity": false
  }
}
```

Low sensitivities stop line noise and "hmm" from cutting the agent off, and
keep it from jumping in at short pauses. `silence_duration_ms` is the pause
that ends their turn: raise it if the agent talks over people, lower it if
replies feel slow. `thinking_budget: 0` removes thinking delay. The voice is
`appearance.voice`; add no `tts_*`.

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

| Situation         | Stamp                         | Pipeline |
| ----------------- | ----------------------------- | -------- |
| Phone, scripted   | Landmass `cascade` + Cartesia | Cascade  |
| Phone, open talk  | Landmass `gemini-stable`      | Realtime |
| Sales roleplay    | Galaxy `medium-stable`        | Realtime |
| Coaching          | Ocean `medium-stable`         | Realtime |
| Demo — browser    | Ocean `medium-stable`         | Realtime |
| Demo — slides     | Landmass `cascade` + Cartesia | Cascade  |
| Text-only         | Ocean `medium-stable`         | —        |
| High-quality text | Ocean `medium-super`          | —        |

Outbound phone: Cascade when lines must be said as written (a tested script, a
chosen external voice, Hinglish code-mix STT); tuned Gemini Live
(voice-to-voice on the phone, above) for open conversations. Slide demos use
Cascade too. Load [../recipes/cascade-tts.md](../recipes/cascade-tts.md) for
Cascade.

Overrides:

- Outbound phone → Ocean only if neither Landmass stamp is available.
- Coaching → Galaxy if Ocean is unavailable.
- Sales also needs `skip_auto_start: true` (see [control.md](control.md)).

## Cartesia `tts_voice_id` quick-reference

| Persona         | Locale | `tts_voice_id`                         |
| --------------- | ------ | -------------------------------------- |
| Indian female   | en-IN  | `343e5d21-9db1-4efa-a1bc-196777cce1bb` |
| American female | en-US  | `f786b574-daa5-4673-aa0c-cbe3e8534c02` |

Use these or a voice ID the user supplies; never invent one. Native Galaxy/Ocean
voices live in `appearance.voice` ([control.md](control.md)): `Aoede`, `Puck`,
`Kore`, `Charon`, `Fenrir`, …

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
