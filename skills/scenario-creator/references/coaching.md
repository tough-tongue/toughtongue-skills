# Coaching Scenario Patterns

Rules for scenarios where the **AI teaches skills through interactive
exercises** (not roleplay). The AI is the trainer/mentor.

Key facts:

- Three patterns: A (Situation-First), B (Teach-First), C (Reflective)
- Every scenario needs SESSION FLOW (steps with tool names) + CONTENT (topics)
- ONE visual tool per scenario: `card`, `image_generation`, or `slide_generation`
- Always: visual before MCQ, "Why did you choose this?" before reveal
- Coaching is about *knowing what to do*; roleplay is about *doing it*

---

## Coach vs Roleplay

| Format | Use when | AI role |
|--------|----------|---------|
| **Coach** | Skill = *knowing what to do*. Teaching knowledge, frameworks, decision-making. | AI is the trainer/mentor |
| **Roleplay** | Skill = *the conversation itself*. Practice talking to a real person. | AI plays a character (see `sales-roleplay.md`) |

---

## The Three Patterns

Pick ONE pattern based on what you're teaching.

| Pattern | Name | Use when | Flow |
|---------|------|----------|------|
| **A** | Situation-First | Handling specific situations: sales objections, customer interactions | Greet → Present Situation → MCQ → Discuss → Practice Pitch → Teach → Next |
| **B** | Teach-First | Concepts/frameworks to learn BEFORE applying | Greet → Teach Concept → MCQ → Discuss → Present Situation → Practice → Next Topic |
| **C** | Reflective | Internal skills: awareness, regulation, listening. No "right answer." | Greet + Warm-Up Q → Present Scenario → MCQ → Deep Discussion → Teach Framework → Reflect → Next |

---

## Two-Part ai_instructions Structure — CRITICAL

| Part | Purpose | What to write |
|------|---------|---------------|
| **SESSION FLOW** | Step order the AI follows | Numbered steps (`### 1. 2. 3.`). Each step names the **tool** and **action**: "Use the card tool to…", "Use MCQ tool with 4 options…" |
| **CONTENT** | Topics/scenarios the AI uses | Separate section: `## THE N SCENARIOS` or `## TEACHING CONTENT` — one block per topic with card text, MCQ options, discussion points, ideal responses |

The flow says *when* to use which tool. The content says *what* to show. Both
are required — the AI does not infer tools; it follows explicit instructions.

---

## One-Visual-Tool Rule

Every scenario uses exactly **ONE** visual presentation tool. Disable unused
ones in `tools_config`.

| Visual tool | Default for | Best at |
|-------------|-------------|---------|
| `card` | Pattern A, Pattern B | Situations, objections, concept cards |
| `image_generation` | Pattern A (retail/visual) | Painting a scene — makes the situation feel real |
| `slide_generation` | Pattern C | Framework teaching, side-by-side models |

Interaction tools (`emoji_reaction`, `mcq`, `end_session`) should always be
registered for coaching scenarios.

---

## Mandatory Rules

1. **Numbered SESSION FLOW with tool names in every step.**
2. **Visual before MCQ** — trainee sees something first via the chosen visual tool.
3. **MCQ after every visual** — 3-4 options to check understanding.
4. **"Why did you choose this?" before reveal** — probe reasoning, then explain.
5. **End with summary + `end_session`.**

---

## Pattern A: Situation-First

Session: 10-15 min. 4-8 case studies, pick 3-4 randomly. Visual: `card`
(default) or `image_generation` (retail).

```
### 1. Greeting (1 min) — Set expectation, pick ONE scenario randomly
### 2. Present Situation (1 min) — Use card tool to show situation
### 3. MCQ (2-3 min) — 3-4 options (poor → excellent), emoji_reaction
### 4. Discuss (2-4 min) — "Why?" before reveal, probe follow-ups
### 5. Practice Pitch (3-5 min) ⭐ — Trainee speaks, coach plays customer, repeat until natural
### 6. Teach (1-3 min) — Ideal script, good vs improve
### 7. Next or Wrap — After 3-4 scenarios, end_session
```

Case study template (one per scenario in the CONTENT section):

```
### Scenario N: "[Objection / Title]"
- **Customer**: Name, age, profile
- **Context**: Plan discussed, price quoted, situation
- **Objection**: Exact words (natural language)
- **Root cause**: Why they're really saying this
- **Nuanced discussion points**: Point 1, Point 2, Point 3
- **Best script**: "[Full ideal response]"
- **Common mistakes**: Arguing, defensive, no next step
```

Rubric weights: MCQ 15% | Reasoning 20% | Pitch Practice 35-40% | Feedback Absorption 15-25%

---

## Pattern B: Teach-First

Session: 15-20 min. 4-5 topics in logical sequence (not random). Visual: `card`.

```
### 1. Greeting (1 min) — Set the frame for what will be learned
### 2. Teach Concept (2-3 min) — card tool with concept (table/bullets/timeline)
### 3. MCQ (2 min) — Test concept grasp (not situation yet)
### 4. Discuss (1-2 min) — "Why?" before reveal, explain reasoning
### 5. Apply — Situation (2-3 min) — card tool with realistic scenario
### 6. Next Topic — Repeat steps 2-5
### 7. Final Practice Round (3-5 min) ⭐ — Apply ALL topics in one exercise
### 8. Wrap-Up — Summary card, end_session
```

Topic template:

```
### Topic N: [Name]
- **Concept Card**: [format, key data to show]
- **Key Teaching Points**: [2-3 things to explain verbally]
- **MCQ**: [what to test, correct principle, common mistake]
- **Situation**: [realistic scenario to apply concept]
- **Ideal response**: [what a good answer covers]
```

Rubric weights: Practice Execution 30% | Concept Understanding 25% | Situation Handling 25% | Feedback Absorption 20%

---

## Pattern C: Reflective

Session: 10-15 min. 2-3 deeper scenarios (not 4-8 quick ones). Visual:
`slide_generation`.

```
### 1. Greeting + Warm-Up Q (2 min) — Reflective question setting introspective tone
### 2. Present Scenario (1 min) — slide_generation with power dynamics, emotional stakes
### 3. MCQ (2-3 min) — Reactive/compliant to authentic/regulated (no obvious "right answer")
### 4. Deep Discussion (4-6 min) — 3-4 questions exploring patterns
### 5. Teach Framework (3-5 min) — slide_generation for visual, evidence not motivation
### 6. Wrap-Up + Transfer Task (1-2 min) — "This week, try X in one conversation", end_session
```

Rubric weights: MCQ ~33% | Reasoning & Self-Reflection ~33% | Absorbing Better Practices ~33%

---

## user_instructions Structure

```
**Your Goal:** [What the trainee will learn/practice]
## Session Overview
- [Duration, number of topics, what to expect]

## How to Get the Most Out of This
- Think before answering MCQs — your reasoning matters more than being "right"
- During practice: speak naturally, don't read a script
- Ask questions when something isn't clear

## What Scores High Points
## What Loses Points
```

---

## Technical Config

```json
{
  "strategy": {
    "skip_auto_start": false,
    "system_instructions_template": "minimal",
    "silence": { "silence_threshold": 8000, "end_session": false, "force_agent_to_speak": true },
    "conductor": {
      "enabled": true,
      "prefix": "CONDUCTOR:",
      "messages": [
        { "time_seconds": 1000, "message": "Wrap up the session — summary card, then end_session.", "end_turn": true }
      ]
    }
  },
  "tools_config": {
    "tools": {
      "end_session": { "should_register": true, "add_to_system_prompt": true },
      "card": { "should_register": true, "add_to_system_prompt": true },
      "mcq": { "should_register": true, "add_to_system_prompt": true },
      "emoji_reaction": { "should_register": true, "add_to_system_prompt": false },
      "slide_generation": { "should_register": false, "add_to_system_prompt": false },
      "image_generation": { "should_register": false, "add_to_system_prompt": false }
    }
  },
  "session_analysis": { "is_auto_analysis": true, "is_auto_submit": true }
}
```

(Swap which visual tool is enabled per the One-Visual-Tool Rule.)

---

## Anti-Patterns

| Anti-pattern | Instead |
|-------------|---------|
| Combining card + image + slide in one step | Pick ONE visual tool; disable the rest |
| No practice step (A) or practice round (B) | Always include verbal practice |
| Teaching situation before concept (Pattern B) | Concept → MCQ → THEN situation |
| Vague scenarios ("A customer objects") | Full persona: name, age, exact words |
| Exact card text in instructions | Give guidance: topic, format, key points |
| `slide_generation` for Pattern A | Use `card` for situations |
| Skipping "Why?" after MCQ | Always probe reasoning before revealing |

---

## Quality Checklist

- [ ] One of the 3 patterns chosen and followed
- [ ] SESSION FLOW: numbered steps, each naming its tool
- [ ] CONTENT section: one detailed block per topic/scenario
- [ ] ONE visual tool chosen; unused visual tools disabled in `tools_config`
- [ ] MCQ after every visual, "Why?" discussion after every MCQ
- [ ] Practice step present (A: pitch practice; B: final round; C: transfer task)
- [ ] Summary + `end_session` at the end
- [ ] Rubric weights match the pattern
