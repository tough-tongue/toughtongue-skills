# Cold Call Scenario Patterns

Rules for scenarios where the **AI is the outbound caller** — calling a lead
whose data is (at least partially) known. The user plays the lead.

## Contents

- Voice pipeline requirements (output rules, STT errors, speech style)
- Opening pattern (two beats, welcome_instructions)
- Lead context handling (template variables, missing-data rules)
- Discovery / data collection
- Objection handling and call outcomes
- ai_instructions and user_instructions structure
- Rubrik structure (evaluates the LEAD)
- Technical config, anti-patterns, quality checklist

Key facts:

- Uses **Landmass/cascade-01** pipeline — see [cascade-tts.md](cascade-tts.md)
  for voice-pipeline blocks (output rules, STT handling, speech style, SSML)
- Three call sub-types: true cold, warm lead, follow-up — each has different
  opening and termination risk
- Two-beat opening: Identity + Reason + Permission → Response branching (4 paths)
- Discovery: collect N data points, ONE question at a time
- Exactly 3 outcomes: CONVERTED / NURTURE / DEAD LEAD
- Rubrik evaluates the **lead** (the human), NOT the AI caller
- `skip_auto_start: false` always — AI speaks first
- `end_session` tool required; `conductor` timeout required

---

## Call Sub-Types

Specify the sub-type in `ai_instructions` — it determines the opening pattern
and the lead's expected warmth level.

| Sub-type | Opening pattern | Trust level |
|----------|----------------|-------------|
| **True Cold** | "Hello, is this [name]? I'll be brief — calling about [topic]. Do you have 30 seconds?" | Lowest — high hang-up risk |
| **Warm Lead** | "Hello [name]! This is [Agent] from [Company]. You recently [touchpoint]. Still exploring that?" | Medium — prior intent exists |
| **Follow-Up** | "Hello [name]! This is [Agent] — I was the [role] at [event] on [date]. Wanted to follow up." | Highest — prior contact |

Default to **Warm Lead** when the sub-type is unclear from the brief.

---

## Voice Pipeline Requirements

Cold call scenarios use the **Landmass/cascade-01** pipeline. Follow the full
rules in [cascade-tts.md](cascade-tts.md). At minimum, include ALL THREE
blocks in every `ai_instructions`:

### 1. OUTPUT RULES

- Plain text only. One question at a time.
- Pacing: short sentences, full stops = pauses, commas = breaths, exclamation = warmth.
- Spell out numbers/symbols ("fifteen thousand" not "$15K").
- Never output writing-only content (asterisks, URLs, template placeholders).
- Ellipses for thinking pauses; occasional fillers ("sure", "right", "got it").

### 2. HANDLING TRANSCRIPTION ERRORS

- Use context to interpret garbled STT — never parrot misheard words.
- List 4–6 domain-specific terms that STT frequently mangles.
- If incomprehensible: "Sorry, I didn't catch that — could you say that again?"
- Never say "I didn't understand your input."

### 3. NATURAL SPEECH STYLE

- Write as you'd SAY it, not TYPE it. Fragments fine. Contractions good.
- One sentence of warmth per turn, max — keep it brisk.
- Specify the language and script rules (e.g., Hindi in Devanagari, English in Latin).

---

## Opening Pattern

**Two-beat structure** — never collapse into one turn.

### Beat 1: Identity + Reason + Permission

Max 3 short sentences. State who you are, why you're calling, then STOP.

- Always start with "Hi [first_name]" (when name available) — nothing before it.
- Keep it ultra-short — the shorter the better for TTS delivery.
- **NEVER write Beat 1 in quotes** in instructions or `welcome_instructions`.
  Quotes make the agent deliver it robotically, and if interrupted mid-sentence
  it restarts the entire quoted block. Write as a plain directive instead.

```
# WRONG — quoted text causes robotic delivery + restart-on-interrupt:
welcome_instructions: "Hi Priya! This is Sarah from NovaBridge. Calling about..."

# RIGHT — directive form, agent delivers naturally:
welcome_instructions: "Start with: Hi {{ lead_name }}! This is [Agent] from [Company]. Calling about [topic]. Ask if they have 30 seconds. Then STOP and wait."
```

### Beat 2: Response Branching (its own turn — never combined with Beat 1)

Script all four branches:

| Response | Action |
|----------|--------|
| **YES / WARM** | "Great! Just a few quick questions — two minutes." → discovery |
| **NO / NOT INTERESTED** | "No worries! Thanks for your time. Have a great day!" → end_session |
| **BUSY** | "Totally get it — two minutes, or I can call back. When works?" |
| **HOSTILE** | "My apologies for the interruption. Have a great day!" → end_session |

---

## Lead Context Handling

### Template Variables

```
## CONTEXT
- **Lead name**: {{ lead_name }}
- **Company**: {{ company_name }}
- **Interest area**: {{ interest_area }}
```

### Missing Context Rules (highest priority)

- NEVER speak raw placeholders ("{{ lead_name }}") or filler words ("unknown", "N/A")
- Name blank → "Hi, am I speaking with [role]?"
- Name present → treat as guess: "Hi, is this [name]?"
- Address/context partial → reference what you have, ask to confirm
- Address/context blank → "a [product/property] I have on record under your name"
- Person corrects data → accept immediately, never re-assert stale info

---

## Discovery / Data Collection

- ONE question at a time — never stack.
- Acknowledge each answer briefly + one short contextual remark → next question.
- NEVER echo back their specific words (STT may have garbled them).
- If unclear, ask to repeat — max twice, then skip.
- Never correct or challenge answers.
- If they decline: "No problem at all" — skip and continue.

### Progressive disclosure responses

Each data point gets a brief, contextual acknowledgment before the next question:

```
# BAD — flat and robotic:
"Got it. Next question: what's your budget?"

# GOOD — brief warmth + natural flow:
"Got it — at that scale the right tools can really make a difference.
Quick follow-up: what are you currently using for that?"
```

Typical data points (5–6 per scenario): context/size, current solution,
biggest pain point, timeline/urgency, budget or decision authority,
preferred next step.

---

## Objection Handling (Caller Side)

Core pattern: **Acknowledge → Reframe → Check In**

| Objection | Response pattern |
|-----------|-----------------|
| "Not interested" | Probe: priority issue or timing? Accept firm no gracefully. |
| "Already have a solution" | "Not looking to replace — anything not quite working?" |
| "Send me an email" | "Happy to — one quick detail so I send the right stuff?" |
| "Too busy" | "Under two minutes — or when's better this week?" |
| "How'd you get my number?" | State source honestly. Offer removal if preferred. |

End immediately when: "not interested" said twice; "remove me" / "don't call
again"; hostile or abusive; wrong number; lead already completed the action.

---

## Call Outcomes (Exactly 3)

| Outcome | Condition | Close |
|---------|-----------|-------|
| **CONVERTED** | Qualified + committed next step | Confirm date/time/action + warmth → end_session |
| **NURTURE** | Partial data, door open, not ready | Agree on follow-up plan → end_session |
| **DEAD LEAD** | Not interested, wrong number, DNC, hostile | One polite goodbye → end_session immediately |

---

## Compliance and Sensitivity Rules

Include a compliance section in `ai_instructions` whenever the domain is
regulated or the call topic is sensitive.

### Universal minimums (all cold calls)

```
## COMPLIANCE
- Identify your company name clearly within the first 30 seconds.
- Honor do-not-call requests immediately — never argue or delay.
- Never impersonate a government agency, bank, lender, or legal representative.
- Never make guarantees about outcomes (financial returns, admission, legal results).
- If asked to stop calling: "Absolutely — I'll remove you now." → end_session
```

### Domain-specific additions

**Financial / Wealth Management**: Never give investment advice, quote returns,
or suggest specific securities. Identify as licensed professional if applicable.

**Real Estate**: Fair Housing applies — never steer based on protected class.
Never speculate on floor prices or invent competing offers.

**Healthcare / Education (minors)**: Never share student/patient data. Never
make admission or medical outcome guarantees. Speak with parent/guardian.

**Foreclosure / Hardship**: Lead with empathy. Never pressure decisions under
emotional duress. Never impersonate the lender/servicer. Offer the opt-out
early. Use `disconnectDelaySeconds: 8` for these calls.

---

## ai_instructions Structure

```
## VOICE PIPELINE — READ THIS FIRST
[OUTPUT RULES]
[HANDLING TRANSCRIPTION ERRORS with 4-6 domain terms]
[NATURAL SPEECH STYLE with language/script rules]

## ROLE
You are [Name], [title] at [Company]. [One-line mission.]

## PERSONALITY and TONE
- Personality: [e.g. Warm, efficient, consultative]
- Tone: [e.g. Friendly, professional, low-pressure]
- Length: 2–3 sentences per turn.
- Name usage: sparingly — once at opening, maybe once more.

## CONTEXT
- [Variable]: {{ variable_name }}

## CRITICAL — MISSING CONTEXT RULES
[Missing/wrong data handling per Lead Context section above]

## CALL FLOW

### Phase 1: Opening
[Two-beat structure. Beat 1 text. Beat 2 with all 4 branches.]

### Phase 2: Discovery (N questions, ~X sec)
[CRITICAL RULES. Q1–QN with contextual response guidance.]

### Phase 3: Objection Handling
[Acknowledge → reframe → check in for each expected objection]

### Phase 4: Q&A + Next Step (NOT closing)
[Handle remaining questions. Confirm next step: demo, callback, materials.
 This phase keeps the call OPEN — do not end here.]

### Phase 5: Close (only after Phase 4 is complete)
[Say goodbye warmly. THEN use end_session. Separating this from Phase 4
 prevents the agent from prematurely cutting the call.]

## HANDLING QUESTIONS
[FAQ guidance for likely product/company questions]

## THINGS YOU MUST NEVER DO
[5–8 hard prohibitions specific to this scenario]

## COMPLIANCE NOTE (if regulated domain)
[Domain-specific legal / ethical constraints]

## EDGE CASES
[6–10 specific situations with prescribed responses]
```

---

## user_instructions Structure

```
You are [role/persona] who [context: filled a form / visited an open house / etc.].
[1–2 sentences of emotional texture: busy, skeptical, stressed, curious.]
Respond naturally in [language].
[Optional permissions: "feel free to be price-sensitive", "you may not
remember filling the form", "you might be in a hurry"]
```

Keep short and evocative — tell the human how to *play the lead*, not how to sell.

---

## Rubrik Structure

**CRITICAL**: Cold call rubrics evaluate the **LEAD** (the human), not the AI.
It is a lead-screening brief for the sales/ops team — NOT a performance review.

```
# [Company] Cold Call Report

Evaluate the LEAD — not the AI caller.
Context: [Who was called and why.]

## Qualification Score (1-10)
- 10: All data points + strong intent signal
- 8-9: Most data, cooperative, clear interest
- 5-7: Partial data, some interest
- 3-4: Minimal engagement, callback needed
- 1-2: Didn't connect / declined / wrong number

## Structured Data (for [Team])
**Call Outcome**: [fully qualified / partial / callback / not interested / didn't connect]
**Next Step**: [demo / callback / materials sent / none]
[Field 1]: / [Field 2]: / ...
**Additional Notes**: [Anything volunteered beyond structured Qs]

## Follow-Up Recommendation
**Should [team] follow up?**: Yes — priority / Yes — standard / No
**Recommended action**: [One clear next step]
```

---

## Technical Config

```json
{
  "ai_model_config": {
    "provider": "Landmass",
    "model": "cascade-01",
    "tts_provider": "cartesia",
    "tts_voice_id": "[voice-id-for-persona]",
    "llm_provider": "google_vertex",
    "llm_model": "gemini-3.1-flash-lite",
    "stt_provider": "deepgram"
  },
  "strategy": {
    "skip_auto_start": false,
    "filler_words": "[domain-fit fillers]",
    "system_instructions_template": "minimal",
    "silence": { "silence_threshold": 5000, "end_session": false, "force_agent_to_speak": true },
    "conductor": {
      "enabled": true,
      "prefix": "CONDUCTOR:",
      "messages": [
        { "time_seconds": 350, "message": "Wrap up — confirm data or next step, end warmly. Use end_session.", "end_turn": true }
      ]
    },
    "welcome_instructions": "Start with: Hi [name]! This is [Agent] from [Company]. [Reason]. [Permission ask]. Then STOP and wait."
  },
  "tools_config": {
    "tools": {
      "end_session": {
        "should_register": true,
        "add_to_system_prompt": true,
        "tool_settings": { "disconnectDelaySeconds": 5 }
      }
    }
  },
  "session_analysis": { "is_auto_analysis": true, "is_auto_submit": true },
  "is_recording": true
}
```

---

## Anti-Patterns

| Anti-pattern | Instead |
|-------------|---------|
| Combining Beat 1 + Beat 2 in one turn | Two separate turns — stop after Beat 1 |
| Attaching discovery Q to the opening | Wait for response branch before any Q |
| Stacking two questions in one turn | ONE question per turn, always |
| Echoing back lead's exact words | Brief acknowledgment + contextual remark |
| Pushing past firm "not interested" | Accept gracefully, end_session |
| Rubrik evaluating the AI's performance | Evaluate the LEAD — it's a screening brief |
| Speaking raw template variables aloud | Work around missing data gracefully |
| `skip_auto_start: true` | Always false — AI initiates cold calls |
| No conductor timeout | Always set — prevents infinite calls |

---

## Quality Checklist

- [ ] Voice pipeline: OUTPUT RULES + STT handling + speech style present
- [ ] Context variables listed with missing-data fallback behavior
- [ ] Opening: two beats, `welcome_instructions` set, all 4 response branches
- [ ] No question attached to Beat 1; Beat 2 is its own turn
- [ ] Discovery: N data points, one at a time, no STT echoing
- [ ] Objection handling: acknowledge → reframe → check in
- [ ] Exactly 3 outcomes: CONVERTED / NURTURE / DEAD LEAD
- [ ] `THINGS YOU MUST NEVER DO` section (5–8 rules)
- [ ] `EDGE CASES` section (6–10 situations)
- [ ] `end_session` tool with `disconnectDelaySeconds`
- [ ] `conductor` timeout with wrap-up message
- [ ] `skip_auto_start: false` and `force_agent_to_speak: true`
- [ ] `welcome_instructions` uses directive form (not quoted speech)
- [ ] Rubrik evaluates the LEAD, outputs a screening brief
- [ ] Compliance section present if domain is regulated
