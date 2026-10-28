# Languages and code-mix

How to build a Scenario that speaks a language other than English, or mixes two
in one sentence (Hinglish: Hindi and English). Applies to every pipeline.
Hindi and Hinglish are the tested case; for another pair, see Other language
pairs. Cascade-only speech rules stay in
[../recipes/cascade-tts.md](../recipes/cascade-tts.md).

## Contents

- Language codes
- Hearing the caller, per pipeline
- Script rule
- LANGUAGE block (switch triggers)
- Grammatical gender
- Respect and numbers
- Accent
- Voicemail, extraction, and app screens
- Other language pairs
- Checklist

## Language codes

Two fields, two jobs:

- `appearance.language_code` — the language the agent speaks (the schema:
  speech synthesis). It defaults to `en-US`; set it for any other locale. Valid
  values: [control.md](../scenario/control.md#appearance).
- `transcribe_config.language` — the language the transcriber listens for. It
  overrides the user's own language preference.

Before a non-English Scenario, check what supports the language: the MCP
resource `ttai://reference/speech-support` (or the
[supported languages](https://app.toughtongueai.com/docs/guides/supported-languages)
page) lists which realtime model, voice provider (`tts_provider`), and speech
recognizer (`stt_provider`) cover each language.

| The Scenario speaks                                          | `language_code` | `transcribe_config.language` |
| ------------------------------------------------------------ | --------------- | ---------------------------- |
| One language (Indian English, Hindi, Tamil, …)               | that code       | the same code                |
| Hindi-first Hinglish: Hindi grammar, everyday English words  | `hi-IN`         | `hi-IN`                      |
| English-first Hinglish: mostly English, Hindi when they want | `en-IN`         | `hi-IN`                      |

For one language, match the two. For code-mix, set them deliberately: the
speaking locale and the listening language can differ. Tested Hinglish phone
callers use the English-first row (an Indian English voice, a Hindi listener)
on both Cascade and Gemini Live. Before launch, test a short Hindi reply and a
short English reply.

In a `super` Scenario, a flow with its own model can override both with
`voice_language` and `transcription_language`.

## Hearing the caller, per pipeline

- **Realtime** (Galaxy, Ocean, Landmass `gemini`, `gemini-stable`, `openai`) —
  the model hears the audio. The caller's transcript comes from Deepgram in
  `transcribe_config.language` (left empty, Deepgram detects the language).
  `stt_provider`, `stt_mode`, and `stt_keyterms` are Cascade fields with no
  documented effect here.
- **Realtime with separate TTS** — as above, plus a `tts_voice_id` that speaks
  both languages. Never invent one
  ([model-selection.md](../scenario/model-selection.md)).
- **Cascade** — STT writes the transcript the LLM reads:
  - Hinglish: `stt_provider: "sarvam"` with `stt_mode: "codemix"`, which keeps
    Hinglish as Devanagari Hindi + Latin English, the same script the prompt
    uses. Other STT providers ignore `stt_mode`.
  - `stt_keyterms` — brand, product, and place names. The schema documents it
    for Deepgram (Nova-3); with another STT provider, also list likely
    mishearings in the prompt
    ([../recipes/cascade-tts.md](../recipes/cascade-tts.md), STT errors).
  - Choose a TTS voice that speaks both languages.

## Script rule

Write Hindi words in Devanagari and English words in Latin script. Never write
romanized Hindi. This applies on every pipeline, to every line the agent may
say:

- examples and scripted lines, including the FLOW opening;
- filler words, in the prompt and in `strategy.filler_words` (Cascade);
- the voicemail message;
- the agent's name when it should sound Hindi (मीरा, not Meera).

Tone only:

- Right: आपको best schools suggest करने के लिए बस कुछ quick details चाहिए।
- Wrong: Aapko best schools suggest karne ke liye bas kuch quick details
  chahiye.

TTS reads romanized Hindi as English, so pronunciation breaks. English domain
words (plan, app, callback) stay English inside a Hindi line. Text a person
reads instead of hears (extraction, reports) follows its reader; see below.

## LANGUAGE block (switch triggers)

Write switching as rules the model can check, not "switch naturally". Pick one
default and state the mix, as a ratio (about seventy percent English, thirty
percent Hindi) or as grammar (Hindi grammar, everyday English words). Copy and
fill:

```
## LANGUAGE
- Default: <English | mostly English with a light Hindi touch every turn or
  two | Hindi grammar with everyday English words>.
- Hindi words in Devanagari, English words in Latin script. Never romanized
  Hindi.
- They ask for Hindi or English: switch from your next line, ask your pending
  question again in that language, and never switch back. Never repeat their
  request back. A request beats every rule below.
- They speak only Hindi: simple everyday Hindi; drop English jargon such as
  motive, impact, specific.
- Their last two answers are mostly English sentences (not single words like
  okay or yes): simple Indian English from your next turn.
- A language you do not support: say once that you understand it less, then
  continue in <Hindi | English>. Tone only: uh.. मुझे वो language थोड़ी कम समझ
  आती है, क्या हम Hindi में बात कर सकते हैं?
- After a switch, translate the scripted lines too; keep their meaning.
- Otherwise never announce or discuss the language choice.
```

- Keep only the triggers the Scenario needs: an English-first caller rarely
  needs the English trigger; a Hindi-first caller needs it.
- Language known per lead before the call: branch on a variable with
  `{% case %}` ([dynamic-variables.md](../scenario/dynamic-variables.md)).
- Say which languages work in `user_instructions` (Answer in English or Hindi).

## Grammatical gender

Hindi verbs mark the speaker's gender, so a wrong form is heard at once. Fix the
agent's gender and keep the caller neutral until it is clear:

```
## GENDER
- You are female: always बोल रही हूँ, समझ गई, बता दूँगी, never the male forms.
- Never decide the caller's gender from their voice: phone audio misleads, and
  the phone can change hands. Until it is clear, speak gender-neutral: आप with
  the respectful plural (आप क्या करते हैं?), and जी instead of sir or ma'am.
- Switch only when two signals agree, and at least one is their own verb forms
  (मैं करती हूँ = woman, मैं करता हूँ = man) or a clearly gendered name. Never
  voice alone. Then use the matching forms, and sir or ma'am, to the end.
- Never ask their gender. If another person comes on the line, or the signals
  conflict, go back to neutral.
```

For a male agent, swap the examples (बोल रहा हूँ, समझ गया).

## Respect and numbers

- Respectful आप, never तुम.
- Numbers, dates, times, and prices in words, never digits: दो मिनट, not 2
  मिनट.
- State which language numbers use: Hindi words in Hindi lines (पाँच हज़ार
  रुपये), or English words everywhere (four hundred ninety-nine rupees) when
  that is how the audience says prices and dates. Use lakh and crore where the
  audience does.
- Phone numbers and IDs: slowly, digit by digit, in short groups.

## Accent

- Realtime: set `language_code` to the Indian locale (`en-IN` or `hi-IN`, not
  the `en-US` default) and state the accent at the top of the prompt. Tone
  only: You are a young Indian woman from Pune. Speak with a natural Indian
  English accent, never American or British, from your first word to the end
  of the call.
- Cascade or separate TTS: the accent is the voice's. Choose an Indian voice
  for `tts_voice_id`.

## Voicemail, extraction, and app screens

- **Voicemail** — `meeting_config.amd_config.voicemail.message` is spoken by
  TTS in the Scenario's voice. Write it in the default language, under the
  script rule. A voicemail line inside `ai_instructions` follows the same
  rule.
- **Extraction and reports** — write `extraction_vars` descriptions and values
  in the language the team reads, usually English. Tell the extractor the
  transcript may be Hinglish and garbled, so it infers intent. Hindi quotes go
  in Latin transliteration when the reader may not read Devanagari. A
  `language` variable (English, Hinglish, or Hindi, and whether they asked for
  Hindi) tells the team which language to follow up in.
- **App screens** — an Agent Desktop app shows whatever strings the agent
  writes. Decide one screen-language policy and state it in both the app's
  README and the prompt: one screen language whatever is spoken, or the
  translated strings for each language written into the prompt. A README that
  says English while the prompt says translate leaves the agent guessing. App
  files are the `ttai-agent-apps` skill's job.

## Other language pairs

- Same structure: the native script for native words, Latin for English words,
  codes from the valid list (`ta-IN`, `bn-IN`, `mr-IN`, …), a LANGUAGE block,
  and a gender block where the language marks gender.
- `stt_mode: "codemix"` is documented for Hinglish only.
- Check the speech-support reference first: a pair needs the realtime model,
  or the voice provider and recognizer, to list both languages.
- Only Hindi and Hinglish have been tested this way. For another pair, run test
  calls in both languages and one mid-call switch before launch.

## Checklist

- [ ] `appearance.language_code` set (default `en-US`);
      `transcribe_config.language` matched for one language, chosen from the
      code-mix table otherwise
- [ ] Cascade Hinglish: `stt_provider: "sarvam"`, `stt_mode: "codemix"`,
      `stt_keyterms` for names; no `stt_*` reliance on realtime models
- [ ] Hindi in Devanagari and English in Latin in every spoken line, filler,
      and voicemail message
- [ ] LANGUAGE block: default and mix stated; switch triggers written as
      rules; a request beats the triggers
- [ ] Agent gender fixed with verb examples; caller neutral until two signals
      agree; never asked
- [ ] आप; numbers in words, in a stated language
- [ ] Accent in the prompt (realtime) or an Indian voice (TTS)
- [ ] Extraction in English; Hindi quotes transliterated for English readers
- [ ] One screen-language policy, in both the app README and the prompt
- [ ] Test calls: one in Hindi, one in English, one switch mid-call

## Key Files

- [model-selection.md](../scenario/model-selection.md) — pipeline and voice
- [control.md](../scenario/control.md) — `appearance`, `transcribe_config`
- [ai-instructions.md](../scenario/ai-instructions.md) — prompt shape
- [../recipes/cascade-tts.md](../recipes/cascade-tts.md) — Cascade speech rules
