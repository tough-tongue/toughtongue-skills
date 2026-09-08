# Scenario model selection

How to stamp `ai_model_config`. Load the `ttai` tool schema before calling —
it is the type source of truth.

Sibling files: [control.md](control.md) (strategy, tools, appearance),
[ai-instructions.md](ai-instructions.md), recipes under
[../../scenario-recipes/](../../scenario-recipes/index.md).

## Contents

- Three pipelines (realtime, separate TTS, Cascade)
- Need-based overrides
- Situation → starting stamp
- Cartesia voice IDs
- Catalog families

---

Always set `provider` + `model`. Three **families**: Galaxy, Ocean, Landmass.
The runtime builds one of three pipelines. Prefer the **situation stamp**
table for create defaults; use need-based rows for overrides.

`cascade` is current; never author legacy `cascade-01`. Never invent a
`tts_voice_id`. Galaxy and Ocean both support SIP; the server enforces
model access for the account.

## 1. Realtime — model speaks

Native transport. Only `provider` + `model`. Do not set `tts_*` / `llm_*` /
`stt_*`.

```json
{ "provider": "Galaxy", "model": "medium-stable" }
```

```json
{ "provider": "Ocean", "model": "medium-stable" }
```

Landmass / LiveKit (super-agent, or user asked for LiveKit) — still no
`tts_*`:

```json
{ "provider": "Landmass", "model": "gemini" }
```

Galaxy `*-stable*` → Landmass `gemini-stable`. Ocean → Landmass `openai`.

## 2. Realtime with separate TTS

Landmass realtime (`gemini`, `gemini-stable`, or `openai`) + `tts_*` when
the user needs a selected/cloned voice but still wants a realtime listener.
Load [cascade-tts.md](../../scenario-recipes/cascade-tts.md) **only** for
full Cascade — not this path.

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
text-first speech. Set component fields when choosing deliberately. Load
[cascade-tts.md](../../scenario-recipes/cascade-tts.md) into
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

`tts_provider`: `cartesia` | `openai` | `elevenlabs` | `sarvam`.  
`llm_provider`: `google` | `google_vertex` | `openai` | `cerebras` | `sarvam`.  
`stt_provider`: `deepgram` | `cartesia` | `sarvam` | `google`.

## Need-based overrides

| Need                                  | Start with                           |
| ------------------------------------- | ------------------------------------ |
| Roleplay / multilingual chat          | Galaxy `medium-stable`               |
| Browser tools / interactive artifacts | Ocean `medium-stable`                |
| Selected or cloned external voice     | Landmass realtime + `tts_*`          |
| Polished TTS / full STT·LLM·TTS       | Landmass `cascade`                   |
| Super-agent                           | Landmass `gemini` or `gemini-stable` |
| Silent meeting assistant              | Studio `meet_assist` (not via MCP)   |

## Situation → starting stamp

Canonical defaults (pre-0.4 spirit; `cascade-01` → `cascade`):

| Situation         | Stamp                         | Pipeline |
| ----------------- | ----------------------------- | -------- |
| Cold call / SDR   | Landmass `cascade` + Cartesia | Cascade  |
| Sales roleplay    | Galaxy `medium-stable`        | Realtime |
| Coaching          | Ocean `medium-stable`         | Realtime |
| Demo — browser    | Ocean `medium-stable`         | Realtime |
| Demo — slides     | Landmass `cascade` + Cartesia | Cascade  |
| Text-only         | Ocean `medium-stable`         | —        |
| High-quality text | Ocean `medium-super`          | —        |

Cold call **and** slide demos use Cascade: outbound / narrated AI speech
wants controllable TTS (Cartesia) + explicit STT/LLM. Load
[cascade-tts.md](../../scenario-recipes/cascade-tts.md) for both.

Overrides:

- Cold call → Ocean only if Cascade / Cartesia is unavailable.
- Coaching → Galaxy if Ocean is unavailable.
- Sales was historically Galaxy `medium`; prefer `medium-stable` now.
- Sales also needs `skip_auto_start: true` (see [control.md](control.md)).

## Cartesia `tts_voice_id` quick-reference

| Persona         | Locale | `tts_voice_id`                         |
| --------------- | ------ | -------------------------------------- |
| Indian female   | en-IN  | `343e5d21-9db1-4efa-a1bc-196777cce1bb` |
| American female | en-US  | `f786b574-daa5-4673-aa0c-cbe3e8534c02` |
| American male   | en-US  | `a167e0f3-df7e-4d52-a9c3-f949145f52ef` |
| German male     | de-DE  | `384b625b-da5a-4b3a-8c5d-b9e1a5f30497` |

Cartesia may default when ID omitted; ElevenLabs requires an explicit ID.
Native Galaxy/Ocean voices live in `appearance.voice` ([control.md](control.md)):
`Aoede`, `Puck`, `Kore`, `Charon`, `Fenrir`, …

## Catalog families

- Galaxy: `medium`, `medium-stable`, `medium-super`, noise-cancellation variants
- Ocean: `medium-stable`, `medium-super`
- Landmass: `gemini`, `gemini-stable`, `openai`, `cascade`

Tool schemas and server validation remain authoritative.

## Key Files

- [control.md](control.md) — strategy, tools, appearance, flags
- [../../scenario-recipes/cascade-tts.md](../../scenario-recipes/cascade-tts.md)
- [../../scenario-recipes/index.md](../../scenario-recipes/index.md)
