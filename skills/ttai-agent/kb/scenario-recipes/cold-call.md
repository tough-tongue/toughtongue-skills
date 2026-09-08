# Cold Call Scenario Patterns

Rules for scenarios where the **AI is the outbound caller** — calling a lead
whose data is (at least partially) known. The user plays the lead.

## Contents

- Voice pipeline (cascade default; half-cascade for cloned TTS + barge-in)
- Opening pattern (two beats, directive in FLOW)
- Lead context handling (template variables, missing-data rules)
- Discovery / data collection
- Objection handling and call outcomes
- ai_instructions and user_instructions structure
- Rubrik structure (evaluates the LEAD)
- Technical config, anti-patterns, quality checklist

Key facts:

- **Recommended stamp:** Landmass `cascade` + Cartesia (was `cascade-01` in
  older docs). Load [cascade-tts.md](cascade-tts.md). See
  [model-selection.md](../entities/scenario/model-selection.md). Ocean realtime is a fallback
  only when Cascade is unavailable for the account — never Galaxy.
- Three call sub-types: true cold, warm lead, follow-up — each has different
  opening and termination risk
- Two-beat opening: Identity + Reason + Permission → Response branching (4 paths)
- Discovery: collect N data points, ONE question at a time
- Exactly 3 outcomes: CONVERTED / NURTURE / DEAD LEAD
- Rubrik evaluates the **lead** (the human), NOT the AI caller
- `skip_auto_start: false` always — AI speaks first
- `end_session` tool required; add a `conductor` timeout (300–450s typical)

---

## Call Sub-Types

Specify the sub-type in `ai_instructions` — it determines the opening pattern
and the lead's expected warmth level.

| Sub-type | Opening | Trust |
| --- | --- | --- |
| **True Cold** | Brief ID + topic + 30s ask | Lowest — high hang-up |
| **Warm Lead** | Name + company + prior touchpoint | Medium — prior intent |
| **Follow-Up** | Name + prior event/role + follow-up | Highest — prior contact |

Examples (directive form in FLOW — not quoted speech):

- True Cold: Hi, is this {{ lead_name }}? Brief call about [topic]. 30 seconds?
- Warm Lead: Hi {{ lead_name }}! [Agent] from [Company]. You recently
  [touchpoint]. Still exploring?
- Follow-Up: Hi {{ lead_name }}! [Agent] — [role] at [event] on [date].
  Following up.

Default to **Warm Lead** when the sub-type is unclear from the brief.

---

## Voice Pipeline Selection

**Default:** Landmass `cascade` + Cartesia (+ Deepgram STT + Flash Lite LLM).
Include the three cascade-tts blocks (OUTPUT RULES, STT errors, NATURAL
SPEECH) from [cascade-tts.md](cascade-tts.md).

**Fallback:** Ocean `medium-stable` only if Cascade / Cartesia is unavailable
for the account. Do **not** stamp Galaxy for outbound cold calls.

**Half-cascade** (Landmass realtime + external TTS): when voice cloning
matters but you want realtime listening — no full Cascade speech blocks.

---

## Opening Pattern

**Two-beat structure** — never collapse into one turn.

### Beat 1: Identity + Reason + Permission

Max 3 short sentences. State who you are, why you're calling, then STOP.

- Always start with "Hi [first_name]" (when name available) — nothing before it.
- Keep it ultra-short — the shorter the better for TTS delivery.
- **NEVER quote Beat 1.** Quotes make the agent deliver it robotically, and
  if interrupted it restarts the quoted block. Write a directive in FLOW.
  There is no `strategy.welcome_instructions` field.

```
# WRONG — quoted text causes robotic delivery + restart-on-interrupt:
## FLOW
Say: "Hi Priya! This is Sarah from NovaBridge. Calling about..."

# RIGHT — directive form, agent delivers naturally:
## FLOW
Start with: Hi {{ lead_name }}! This is [Agent] from [Company]. Calling
about [topic]. Ask if they have 30 seconds. Then STOP and wait.
```

### Beat 2: Response Branching (its own turn — never combined with Beat 1)

Script all four branches:

| Response                | Action                                                                                      |
| ----------------------- | ------------------------------------------------------------------------------------------- |
| **YES / WARM**          | "Great! Just a few quick questions — two minutes." → discovery                              |
| **NO / NOT INTERESTED** | "No worries! Thanks for your time. Have a great day!" → follow the `end_session` turn rule  |
| **BUSY**                | "Totally get it — two minutes, or I can call back. When works?"                             |
| **HOSTILE**             | "My apologies for the interruption. Have a great day!" → follow the `end_session` turn rule |

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

| Objection                  | Response pattern                                            |
| -------------------------- | ----------------------------------------------------------- |
| "Not interested"           | Probe: priority issue or timing? Accept firm no gracefully. |
| "Already have a solution"  | "Not looking to replace — anything not quite working?"      |
| "Send me an email"         | "Happy to — one quick detail so I send the right stuff?"    |
| "Too busy"                 | "Under two minutes — or when's better this week?"           |
| "How'd you get my number?" | State source honestly. Offer removal if preferred.          |

Close when: "not interested" is said twice; a removal request is made; the
person is hostile or abusive; the number is wrong; or the lead already
completed the action. Speak the closing first, then follow the live
`end_session` instructions; do not call it in the same turn as a question or
pitch.

---

## Call Outcomes (Exactly 3)

| Outcome       | Condition                                  | Close                                                           |
| ------------- | ------------------------------------------ | --------------------------------------------------------------- |
| **CONVERTED** | Qualified + committed next step            | Confirm date/time/action + warmth; follow the closing turn rule |
| **NURTURE**   | Partial data, door open, not ready         | Agree on follow-up plan; follow the closing turn rule           |
| **DEAD LEAD** | Not interested, wrong number, DNC, hostile | One polite goodbye; follow the closing turn rule                |

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
- If asked to stop calling: state the removal confirmation, then follow the
  closing turn rule.
```

### Domain-specific additions

**Financial / Wealth Management:** Never give investment advice, quote returns,
or suggest specific securities. Identify as a licensed professional if applicable.

**Real Estate:** Fair Housing applies — never steer based on protected class.
Never speculate on floor prices or invent competing offers.

**Healthcare / Education (minors):** Never share student/patient data. Never
make admission or medical outcome guarantees. Speak with parent/guardian.

**Foreclosure / Hardship:** Lead with empathy. Never pressure decisions under
emotional duress. Never impersonate the lender/servicer. Offer the opt-out
early. Use `disconnectDelaySeconds: 8`.

---

## ai_instructions Structure

```
## PERSONALITY and TONE
- Personality: [e.g. Warm, efficient, consultative]
- Tone: [e.g. Friendly, professional, low-pressure]
- Length: 2–3 sentences per turn.
- Name usage: sparingly — once at opening, maybe once more.

## CONTEXT
- [Variable]: {{ variable_name }}
## CRITICAL — MISSING CONTEXT RULES
[Missing/wrong data handling per Lead Context section above]

## ROLE
You are [Name], [title] at [Company]. [One-line mission.]

## FLOW
### Phase 1: Opening
[Directive Beat 1. Then STOP. Beat 2: all 4 branches.]
### Phase 2: Discovery (N questions)
[ONE question per turn. Q1–QN.]
### Phase 3: Objection Handling
[Acknowledge → reframe → check in]
### Phase 4: Q&A + next step (keep the call OPEN)
### Phase 5: Close
[State the goodbye. Wait for the lead's reply, then call end_session if they
have not substantively re-opened the conversation.]

## TOOLS
Call end_session for CONVERTED / NURTURE / DEAD LEAD only after the lead's
post-goodbye reply.

## HANDLING QUESTIONS
[FAQ guidance for likely product/company questions]

## GUARDRAILS
## THINGS YOU MUST NEVER DO
[5–8 hard prohibitions]
## COMPLIANCE NOTE (if regulated)
## EDGE CASES
[6–10 situations]

## STYLE
Spoken output rules + speech style. Cascade only: STT-error handling.
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

Keep short and evocative — tell the human how to _play the lead_, not how to sell.

---

## Rubrik Structure

**CRITICAL**: Cold call rubrics evaluate the **LEAD** (the human), not the AI.
It is a lead-screening brief for the sales/ops team — NOT a performance review.

```
# [Company] Cold Call Report
Evaluate the LEAD — not the AI. Context: [who / why].

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

## Technical Config: recommended default (Cascade)

```json
{
  "ai_model_config": {
    "provider": "Landmass",
    "model": "cascade",
    "tts_provider": "cartesia",
    "tts_voice_id": "f786b574-daa5-4673-aa0c-cbe3e8534c02",
    "llm_provider": "google_vertex",
    "llm_model": "gemini-3.1-flash-lite",
    "stt_provider": "deepgram"
  },
  "strategy": {
    "skip_auto_start": false,
    "system_instructions_template": "minimal",
    "silence": { "silence_threshold": 5000, "end_session": false, "force_agent_to_speak": true },
    "conductor": {
      "enabled": true,
      "messages": [
        {
          "time_seconds": 350,
          "message": "Wrap up — confirm data or next step and say goodbye. Wait for the lead's reply before end_session.",
          "end_turn": true
        }
      ]
    }
  },
  "tools_config": {
    "tools": {
      "end_session": {
        "should_register": true,
        "add_to_system_prompt": true,
        "tool_settings": { "disconnectDelaySeconds": 8 }
      }
    }
  },
  "session_analysis": { "is_auto_analysis": true, "is_auto_submit": true },
  "is_recording": true
}
```

Cartesia ID = American female from model-selection.md. Load
[cascade-tts.md](cascade-tts.md) into `ai_instructions`.

### Ocean fallback (Cascade unavailable)

```json
{ "ai_model_config": { "provider": "Ocean", "model": "medium-stable" } }
```

Same strategy / tools / session_analysis as above. No cascade-tts blocks.

---

## Anti-Patterns

| Anti-pattern | Instead |
| --- | --- |
| Beat 1 + Beat 2 in one turn | Stop after Beat 1 |
| Discovery Q on the opening | Wait for a response branch |
| Two questions in one turn | One question per turn |
| Echoing the lead verbatim | Brief ack + context |
| Pushing past "not interested" | Graceful close → `end_session` timing |
| Rubrik scoring the AI | Score the LEAD |
| Speaking raw `{{ vars }}` | Missing-data fallbacks |
| `skip_auto_start: true` | Always `false` |
| No conductor timeout | Always set |
| Galaxy or casual Ocean stamp | Cascade default; Ocean only as fallback |

---

## Quality Checklist

- [ ] Two-beat FLOW opening; 4 branches; 3 outcomes; one Q per turn
- [ ] PERSONALITY/TONE, HANDLING QUESTIONS, EDGE CASES, NEVER list
- [ ] Missing-data fallbacks; compliance if regulated
- [ ] `end_session` + `skip_auto_start: false` + talkative silence + conductor
- [ ] Stamp: Landmass `cascade` + real Cartesia ID + cascade-tts blocks
  (or documented Ocean fallback)
- [ ] Rubrik scores the LEAD, not the AI

## Key Files

- [../scenario-authoring.md](../scenario-authoring.md) — durable design rules
- [../entities/scenario/model-selection.md](../entities/scenario/model-selection.md) — pipeline stamps
- [../entities/scenario/control.md](../entities/scenario/control.md) — strategy / tools
- [cascade-tts.md](cascade-tts.md) — full Cascade only
- [../entities/scenario/rubrik.md](../entities/scenario/rubrik.md) — evaluation text
