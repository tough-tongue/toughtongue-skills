# Cascade TTS Rules

Rules to include in `ai_instructions` when the scenario uses the cascade voice
pipeline (Landmass/cascade-01 with external TTS/STT). These supplement the
scenario's core instructions — they do not replace the call flow or domain
content.

## Contents

- Voice pipeline preamble (mandatory)
- Output rules with response length by scenario type
- STT transcription error handling (with customization template)
- Natural speech style
- SSML emotion tags (Cartesia only)
- User names and dynamic variables
- Language and code-switching (Hindi/Hinglish rules)
- Response pacing and turn-taking
- Concise opening pattern (Beat structure)
- Checklist

---

## What Is the Cascade Pipeline?

Cascade = STT → LLM → TTS. The customer speaks, speech-to-text transcribes it,
the LLM generates a text response, and text-to-speech reads it aloud. The LLM
never hears audio directly — it sees imperfect transcripts and produces text
that will be spoken. Every rule below exists because of this architecture.

---

## 1. Voice Pipeline Preamble

Always open `ai_instructions` with this block:

```
## VOICE PIPELINE — READ THIS FIRST
You are in a voice call pipeline: the user's speech is transcribed by STT,
you generate text, and TTS reads it aloud. You are NOT in a text chat.
```

---

## 2. Output Rules (mandatory for all cascade scenarios)

TTS reads the model's output verbatim. Anything that looks fine in text but
sounds wrong when spoken must be eliminated.

```
OUTPUT RULES:
- Respond in plain text only.
- Spell out numbers and symbols — say "ten to twelve hours" not "10-12 hours",
  say "rupees zero" not "₹0".
- Never output anything that only makes sense in writing — no asterisks, no URLs,
  no parenthetical notes.
```

### Response length by scenario type

| Scenario type | Target length | Guidance |
|---|---|---|
| Cold call / screening | 1-3 sentences, under 15 sec | Speed matters — they'll hang up |
| Onboarding / education | 2-4 sentences, under 30 sec | Slightly longer per turn |
| Customer service | 1-3 sentences, under 25 sec | Concise answers for stressed callers |
| Demo / walkthrough | 3-5 sentences, under 30 sec | More room, but still conversational |

### Common spell-out patterns

| Written | Spoken |
|---|---|
| `₹999` | "rupees nine ninety nine" |
| `10-12 hours` | "ten to twelve hours" |
| `$299` | "two hundred ninety nine dollars" |
| `4-5 PM` | "four to five PM" |
| `Rs 199/year` | "rupees one ninety nine per year" |

---

## 3. Handling STT Transcription Errors

STT produces garbled text on most calls — wrong homophones, dropped syllables,
mangled names. Include a section tailored to the scenario's domain terms.

```
## HANDLING TRANSCRIPTION ERRORS
The customer's words arrive via speech-to-text, which WILL make mistakes —
garbled words, dropped syllables, wrong homophones.

CRITICAL RULES:
- Use conversation context and common sense to figure out what they likely meant.
  Don't take garbled text literally.
- NEVER parrot back or repeat obviously misheard words.
- Domain terms get misheard often: "[TERM]" may come through as "[VARIANT1]",
  "[VARIANT2]", "[VARIANT3]". Recognize these and proceed normally.
- If you genuinely cannot understand, ask naturally: "Sorry, I didn't catch that —
  could you say that again?"
  Do NOT say robotic things like "I didn't understand your input."
- When in doubt, respond to what makes sense given where you are in the conversation,
  not to the literal transcript words.
```

Customize with 4-6 domain-specific misheard terms per scenario.

---

## 4. Natural Speech Style

```
## NATURAL SPEECH STYLE
Write exactly as you would SAY it, not as you would TYPE it.
Short sentences. Fragments are fine. Sound like a real person, not a script.

WHAT GOOD OUTPUT SOUNDS LIKE:
- Bad: "I can definitely help you understand that feature."
  Good: "Yeah so basically, let me explain how it works."
- Bad: "Unfortunately, that is not possible at the moment."
  Good: "Oh no, that won't work right now. But let me see what I can do."
- Bad: "Would you like me to provide more information?"
  Good: "Want me to tell you more about it?"
```

Adjust the register per scenario — a customer service agent sounds different
from a cold-calling SDR — but the principle is the same: spoken cadence, not
written prose.

---

## 5. SSML Emotion Tags (Cartesia TTS Only)

Cartesia supports SSML emotion tags. ElevenLabs and OpenAI TTS do NOT — skip
this section entirely for non-Cartesia scenarios.

```
## VOICE & EMOTION
Your TTS supports SSML tags for emotion and speed — use them to make the
call feel alive.
- **Opening / greeting**: Use `<emotion value="excited" />` — warm, upbeat.
- **Core pitch / explanation**: Use `<emotion value="enthusiastic" />` — genuine confidence.
- **Closing / wrap-up**: Use `<emotion value="content" />` — calm, warm, reassuring.
- **Handling concerns**: Use `<emotion value="sympathetic" />` — understanding and patient.
- Keep tags natural — only switch emotion when the conversational tone genuinely shifts.
```

| TTS provider | Include emotion section? |
|---|---|
| Cartesia (`tts_provider: "cartesia"`) | Yes |
| ElevenLabs (`tts_provider: "elevenlabs"`) | No — tags render as literal text |
| Other | No |

---

## 6. User Names and Dynamic Variables

STT frequently mangles proper names beyond recognition. Never infer or correct
a user's name from the transcript — trust the template variable.

```
The user's first name is provided as a dynamic variable: {{ first_name }}.
ALWAYS use this variable for the name. Do NOT attempt to catch, infer, or guess
the user's name from their speech — STT frequently mangles names.
```

---

## 7. Language and Code-Switching

For multilingual scenarios (Hindi, Hinglish, Indian English):

```
Speak in [default language] by default. If the customer switches to [other language],
switch naturally. Support [mixed variety] if they mix both.
```

### Hindi-specific rules

- Always use respectful "आप" — never "तुम"
- Spell out Hindi numbers: "दो मिनट" not "2 मिनट"
- Domain terms in English are fine even in Hindi responses

### Devanagari script rule (CRITICAL for Hindi/Hinglish)

Always write Hindi words in **Devanagari script** — never romanized
transliteration. English words stay in Latin script. This produces
dramatically better TTS pronunciation.

```
**CRITICAL — Script rule**: Always write Hindi words and phrases in **Devanagari script**
(e.g. "आपको best schools suggest करने के लिए बस कुछ quick details चाहिए").
NEVER use romanized Hindi (e.g. "Aapko best schools suggest karne ke liye").
English words stay in Latin script. This is essential for correct TTS pronunciation.
```

| Output style | Example | TTS result |
|---|---|---|
| Devanagari + English (correct) | "आपको best schools suggest करने के लिए..." | Natural Hindi with clean English |
| Romanized Hindi (wrong) | "Aapko best schools suggest karne ke liye..." | Broken pronunciation |

Apply to ALL Hindi text: `ai_instructions` examples, `welcome_instructions`,
filler words, Beat 1 intros, edge-case dialogues.

---

## 8. Response Pacing and Turn-Taking

Cascade pipelines have inherent latency (STT + LLM + TTS). Keep responses tight.

- **One question per turn.** Never stack questions. Wait for a response.
- **Pause after key information.** Let the customer absorb before moving on.
- **Wait for their goodbye.** Never end a call after your own last line.
- **Don't front-load.** Spread information across turns.

---

## 9. Concise Opening Pattern

Cold calls and outbound scenarios live or die in the first 5 seconds. Structure
the opening as discrete **beats** — short, breathable lines.

```
**Beat 1 (intro):** Three short fragments, one per line:
 - Greeting + name
 - Identity + company
 - Context check — end with a question

**Beat 2 (response-gated):** Branch on YES / NO / BUSY.
 - YES → set expectation ("बस दो minute")
 - NO → thank and end_session immediately
 - BUSY → offer callback or quick pitch
```

Never combine Beat 1 and Beat 2 in one turn. The AI speaks Beat 1, then
STOPS and waits for a response.

---

## Checklist: Adding Cascade TTS Rules

- [ ] Voice pipeline preamble at the top of `ai_instructions`
- [ ] Output rules: plain text, spell-out numbers, no markdown/emojis
- [ ] Response length calibrated to scenario type
- [ ] STT error handling with 4-6 domain-specific mishearing examples
- [ ] Natural speech style block with good/bad examples
- [ ] SSML emotion tags (if Cartesia TTS — skip for ElevenLabs/other)
- [ ] Dynamic variable for user name (`{{ first_name }}` or `{{ name }}`)
- [ ] Language/code-switching guidance (if multilingual)
- [ ] Devanagari script rule (if Hindi/Hinglish scenario)
- [ ] Concise opening — Beat 1 / Beat 2 structure (if outbound / cold call)
- [ ] One-question-per-turn pacing rules
- [ ] `system_instructions_template: minimal` in strategy

---

## Anti-Patterns

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| Markdown in output (bold, bullets, headers) | TTS reads asterisks or skips formatting | Plain text only |
| Long monologue responses | Customer thinks the call froze | Cap at 3 sentences / 15-30 sec |
| Numeric symbols (`₹`, `$`, `%`, `10-12`) | TTS mispronounces or skips symbols | Spell out everything |
| Parroting back garbled STT text | Confuses the customer, breaks trust | Infer from context |
| Stacking multiple questions in one turn | Customer only answers the last one | One question, then wait |
| SSML emotion tags with ElevenLabs | Tags render as literal text | Only use with Cartesia |
| Inferring user name from transcript | STT mangles names constantly | Trust `{{ first_name }}` variable |
| Formal/written tone ("I shall assist you") | Sounds robotic when spoken | "Let me help with that" |
| Romanized Hindi ("Aapko...") | TTS reads as English — broken pronunciation | Devanagari script always |
