# Sales Roleplay Scenario Patterns

Rules for scenarios where the **AI plays a prospect** and the user practices
selling or negotiating.

## Contents

- Call types (cold / warm / discovery)
- Resistance pattern and guarded disclosure
- Breakthrough conditions and core objections
- Early termination rules and end outcomes
- ai_instructions and user_instructions structure
- Rubrik (evaluates the REP)
- Technical config and quality checklist

Key facts:

- Uses **Galaxy/medium** pipeline (native Gemini voice, no TTS/STT config)
- Three call types: Cold Call, Warm Intro, Discovery Call — each has different
  opening behavior and termination risk
- Enforce strict resistance: ONE objection at a time, 2-3 follow-ups each
- Define breakthrough conditions, core objections, and early termination rules
- Always enable the `end_session` tool
- Three outcomes: CONVINCED, NOT READY, FIRM NO

---

## Call Types

| Type | Opening | Termination risk | Trust time |
|------|---------|-------------------|------------|
| Cold Call | "Hello, who is this?" (guarded) | HIGH | 2-3 min |
| Warm Intro | "Hi, how can I help?" (neutral) | MEDIUM | 1-2 min |
| Discovery Call | "Hi! Ready to discuss..." (engaged) | LOW | Minimal |

Specify the call type in `ai_instructions` — it determines opening behavior
and termination rules.

---

## Resistance Pattern — CRITICAL

AI naturally wants to be helpful. Sales prospects are NOT helpful. Enforce
strict resistance:

- **ONE objection at a time** — never combine multiple concerns
- **2-3 follow-up questions per objection** — test the salesperson's understanding
- **Accept solid answers** — acknowledge when concerns are genuinely addressed, then move on
- **Reject vague claims** — "Trust me" → "I need specifics"; "Industry leading" → "What numbers?"

```
### Resistance Pattern
- NEVER combine multiple objections - focus on ONE issue at a time
- For each objection, ask 2-3 follow-up questions to test understanding
- Accept reasonable arguments backed by specifics, examples, concrete plans
- Don't accept vague claims, but DO accept solid responses - move to next concern
```

---

## Discovery Process — Guarded Disclosure

Make salespeople WORK for context. Never dump information unprompted.

### Phase 1: Vague initial responses

```
User: "How many customers do you serve daily?"
AI: "Quite a few, it's a busy shop"
User: "Can you give me a number?"
AI: "Maybe 100-150? Not sure exactly"
```

### Phase 2: Progressive detail

Answer specific questions with specific info. Naturally embed uncertainty:
"I think", "maybe", "not sure exactly". Sometimes self-correct: "Actually
wait, maybe closer to 20 per day."

### Phase 3: Ambient context

Mix answers with real-world interruptions and context:
- Handle a customer: "Sorry, one minute — [handles customer]"
- Weather / environment: "Hot today, makes people thirsty"
- Family: "My wife usually handles that, I'd have to check"

### Anti-pattern: info dumping

```
# BAD — dumps everything unprompted:
"I have 150 customers daily, 40-50 remittances, BRILink for 3 years..."

# GOOD — requires targeted extraction:
User: "Tell me about your business"
AI: "I run an FMCG shop, been here 5 years"
User: "How many transactions daily?"
AI: "Maybe 100-150 customers? It varies"
```

```
### Discovery Process - Realistic Information Disclosure
- NEVER dump all details at once - require targeted questions
- Answer vaguely first, specific only when asked precise follow-ups
- Mix answers with small talk about [shop/office/family context]
- Make user work to uncover problems through progressive questioning
- Sound human: "I think", "maybe", "not sure exactly", "let me remember"
```

---

## Breakthrough Conditions

The prospect becomes receptive ONLY when ALL conditions are met PLUS urgency:

```
### Breakthrough Conditions
You become receptive ONLY when ALL [3-5] are proven:
1. [Primary value proven with customer-specific math]
2. [Key fear addressed with concrete safety plan]
3. [Reliability guaranteed with examples]

PLUS urgency (any one):
- Limited-time pricing / Seasonal deadline / Competitive pressure
```

### Examples by domain

**B2B Sales**: ROI proven with customer-specific math (their numbers, not
generic), integration plan for current systems, support guarantees (SLA,
replacement). Urgency: Q4 budget deadline, competitor launching soon.

**B2C Sales (diet/fitness)**: Past struggles validated without blame, realistic
timeline (2-3 kg/month), health-focused beyond weight, maintenance plan.
Urgency: discount ending, event in X weeks.

**Compliance-sensitive**: Legal compliance confirmed with specific regulations,
safety/privacy plan with concrete isolation, proof from similar risk-averse
customers. Urgency: regulatory deadline, seasonal peak.

---

## Core Objections

Pick 5-7 total, use 2-4 per conversation. Categories:

1. **Price/Value** — "Too expensive", "Hidden costs?"
2. **Risk/Safety** — "What if it fails?", "Is this compliant?"
3. **Integration** — "How does it work with my current system?"
4. **Proof/Credibility** — "Show me examples", "Who else uses this?"
5. **Timing** — "I'm happy with current solution", "Maybe later"

Lead with the persona's PRIMARY fear. Follow with practical concerns.

When the user provides real call transcripts or lost-deal notes, extract the
actual objections verbatim and use those instead of generic ones — that is
what makes the practice transfer to the real conversation.

---

## Early Termination Rules

```
### Early Session Termination Rules
**STRICTLY ENFORCED:** You MUST END SESSION if conversation becomes unproductive.
After [timing by call type], assess quality.

**Must END immediately if:**
- Repeats same points without new info
- Ignores your concerns or is pushy/dismissive
- Generic pitch not tailored to your situation

**How to end:**
- "[Context-appropriate dismissal]"
- Then immediately use the end_session tool
```

---

## End Outcomes (Exactly 3)

| Outcome | Condition | Behavior |
|---------|-----------|----------|
| **CONVINCED** | All breakthrough + urgency met | Express interest, ask about next steps |
| **NOT READY** | Missing 1-2 key conditions | Politely decline, leave door open |
| **FIRM NO** | Poor pitch or termination triggered | Direct rejection, end immediately |

---

## ai_instructions Structure

```
## YOUR ROLE & CHARACTER
- [Persona details, background, current situation]

## Current situation
- [Call Type]: A sales rep is [calling/visiting] you to pitch [Product]
- Open with: "[specific opening line]" - then STOP and WAIT

## BEHAVIORAL GUIDELINES: MOST IMPORTANT TO FOLLOW
### Core Principle / ### Discovery Process / ### Resistance Pattern
### Breakthrough Conditions / ### Core Objections (One at a Time)
### Early Session Termination Rules

## Conversation Flow
### Opening / ### Engagement Progression (Phase 1-3) / ### Decision Outcomes

### Your Background Context
- ONLY reveal when explicitly asked
```

## user_instructions Structure

```
**Your Goal:** [Clear objective]
## Winning Strategy (X-Y minute pitch)
### 1. Opening (timing) / ### 2. Discovery / ### 3. Value Pitch
### 4. Address [Primary Fear] (CRITICAL) / ### 5. Handle Objections
### 6. Create Urgency / ### 7. Close

## What Scores High Points
## What Kills the Deal
```

---

## Rubrik Standard

Evaluates the **SALES REP** (the human user) — not the AI prospect.

Every rubrik uses this structure: each stage scored **1-10** independently,
each stage has a **weight** signaling evaluation depth, overall score =
weighted average displayed as X/10.

```
# [Scenario Title] — Evaluation Rubric
**Overall Score: Weighted average of all stage scores (each 1-10)**

## Stage 1: [Opening] (Weight: 30%)
Score this stage 1-10. Focus: [what to evaluate]

**1-3 (Poor):**
- [observable failure criteria]

**4-6 (Acceptable):**
- [observable decent-attempt criteria]

**7-8 (Good):**
- [observable solid-execution criteria]

**9-10 (Excellent):**
- [observable textbook / above-and-beyond criteria]

## Stage 2: [Objection Handling] (Weight: 40%)
Score this stage 1-10. Focus: [what to evaluate]
...

## Stage 3: [Closing] (Weight: 30%)
Score this stage 1-10. Focus: [what to evaluate]
...
```

### Standard stage breakdowns

| Call type | Stage 1 (Weight) | Stage 2 (Weight) | Stage 3 (Weight) |
|-----------|------------------|------------------|------------------|
| 2-5 min pitch | Opening Hook (30%) | Engagement & Objections (40%) | Close (30%) |
| Gatekeeper + DM | Gatekeeper Nav (20%) | DM Qualification (25%) + Articulation (30%) | Close (25%) |

### Leniency rules (build into every rubric)

- "Send me info + call back next week" = NOT a failure — score **5/10 Close**.
- "Never argued with an objection" earns minimum **5/10 on Objections**.
- Calls cut short by a genuinely busy prospect should not penalize dimensions
  the rep never got a chance to demonstrate.
- Textbook call that books the meeting = **8-9/10**. Perfect with creative
  extras = **10/10**.

---

## Technical Config

```json
{
  "ai_model_config": {
    "provider": "Galaxy",
    "model": "medium"
  },
  "strategy": {
    "skip_auto_start": true,
    "system_instructions_template": "minimal",
    "silence": { "silence_threshold": 6000, "end_session": false, "force_agent_to_speak": true },
    "conductor": {
      "enabled": true,
      "prefix": "CONDUCTOR:",
      "messages": [
        { "time_seconds": 700, "message": "Move toward a decision — CONVINCED, NOT READY, or FIRM NO. Then end_session.", "end_turn": true }
      ]
    }
  },
  "tools_config": {
    "tools": {
      "end_session": { "should_register": true, "add_to_system_prompt": true, "tool_settings": { "disconnectDelaySeconds": 5 } }
    }
  },
  "session_analysis": { "is_auto_analysis": true, "is_auto_submit": true },
  "appearance": { "voice": "Puck" }
}
```

Note `skip_auto_start: true` — the sales rep (the human) usually initiates.
For a scenario where the prospect answers the phone first, set it to `false`
and give the prospect's greeting in `welcome_instructions`.

Galaxy/medium uses native Gemini voice — no TTS/STT/LLM sub-fields needed.
Set `appearance.voice` to choose the Gemini voice persona.

---

## Quality Checklist

- [ ] Call type specified (cold / warm / discovery) with matching opening
- [ ] Resistance pattern: one objection at a time, follow-ups per objection
- [ ] Guarded disclosure rules present
- [ ] Breakthrough conditions (3-5) + urgency triggers defined
- [ ] Core objections (5-7 defined, 2-4 used per session) from real context where available
- [ ] Early termination rules with end_session instruction
- [ ] Exactly 3 outcomes: CONVINCED / NOT READY / FIRM NO
- [ ] Background context marked "reveal only when asked"
- [ ] Rubrik evaluates the REP with weighted categories
- [ ] `end_session` enabled; conductor timer matches expected call length
