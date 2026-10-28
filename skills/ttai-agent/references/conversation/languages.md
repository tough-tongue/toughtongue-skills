# Languages and code-mix

For a Scenario that speaks a language other than English, or mixes two in one
sentence (Hinglish). Hindi and Hinglish are the tested case. Before a
non-English Scenario, check the MCP resource `ttai://reference/speech-support`
(or the
[supported languages](https://app.toughtongueai.com/docs/guides/supported-languages)
page): it lists which model, `tts_provider`, and `stt_provider` cover each
language.

## Language codes

- `appearance.language_code` is what the agent speaks;
  `transcribe_config.language` is what the transcriber listens for (on
  realtime models, Deepgram's caller transcript).
- One language: set both to the same code.
- Code-mix: set them deliberately. Hindi-first Hinglish uses `hi-IN` for both;
  English-first Hinglish (mostly English, Hindi when the caller wants) speaks
  `en-IN` and listens `hi-IN`.
- Cascade Hinglish: `stt_provider: "sarvam"` with `stt_mode: "codemix"` keeps
  Hindi in Devanagari and English in Latin. `stt_*` fields do nothing on
  realtime models.

## Script rule

Write Hindi in Devanagari and English words in Latin script, in every line the
agent may say: examples, the opening, fillers, the voicemail message. Never
romanized Hindi: TTS reads it as English. Tone only: आपको best schools suggest
करने के लिए बस कुछ quick details चाहिए।

## LANGUAGE block

Write switching as rules the model can check, not "switch naturally":

```
## LANGUAGE
- Default: <mostly English with a light Hindi touch | Hindi grammar with
  everyday English words>. Hindi in Devanagari, English in Latin script.
- They ask for Hindi or English: switch from your next line, repeat your
  pending question in that language, and never switch back.
- Their last two answers are mostly English sentences (not single words like
  okay): simple Indian English from your next turn.
- A language you do not support: say once you understand it less, then
  continue in <Hindi | English>.
- Respectful आप, never तुम. Numbers, dates and prices in words.
```

## GENDER block

Hindi verbs mark the speaker's gender, so a wrong form is heard at once:

```
## GENDER
- You are female: always बोल रही हूँ, समझ गई; never the male forms.
- Never decide the caller's gender from their voice. Until it is clear, use
  gender-neutral आप forms and जी instead of sir or ma'am.
- Switch only when two signals agree, at least one being their own verb forms
  (मैं करती हूँ / मैं करता हूँ) or a clearly gendered name. Never ask.
```

For a male agent, swap the forms (बोल रहा हूँ, समझ गया). Before launch, place
one test call in each language and one that switches mid-call.
