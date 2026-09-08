# Coaching Scenario Patterns

Rules for scenarios where the **AI teaches skills through interactive
exercises** (not roleplay). The AI is the trainer/mentor.

## Contents

- Coach vs roleplay decision
- The three patterns (A Situation-First, B Teach-First, C Reflective)
- Two-part ai_instructions structure (FLOW + CONTENT)
- Visual-tool discipline and mandatory rules
- Pattern A / B / C step-by-step flows with templates
- user_instructions structure
- Technical config, anti-patterns, quick checklist

Key facts:

- Realtime: Ocean `medium-stable` when card/MCQ (or other artifacts) are on;
  otherwise Galaxy `medium-stable`. Fall back to Galaxy if Ocean is rejected.
  See [../entities/scenario/model-selection.md](../entities/scenario/model-selection.md).
- Three patterns: A (Situation-First), B (Teach-First), C (Reflective)
- Every scenario needs FLOW (steps with tool names) + CONTENT (topics)
- Choose a deliberate visual palette: `card`, `image_generation`, or
  `slide_generation`. Combine tools only when each has a distinct job.
- When a visual introduces a decision: visual before MCQ, then "Why did you
  choose this?" before reveal
- Coaching is about _knowing what to do_; roleplay is about _doing it_

---

## Coach vs Roleplay

| Format       | Use when                                                                       | AI role                                        |
| ------------ | ------------------------------------------------------------------------------ | ---------------------------------------------- |
| **Coach**    | Skill = _knowing what to do_. Teaching knowledge, frameworks, decision-making. | AI is the trainer/mentor                       |
| **Roleplay** | Skill = _the conversation itself_. Practice talking to a real person.          | AI plays a character (see `sales-roleplay.md`) |

---

## The Three Patterns

Pick ONE pattern based on what you're teaching.

| Pattern | Name            | Use when                                                              | Flow                                                                                            |
| ------- | --------------- | --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **A**   | Situation-First | Handling specific situations: sales objections, customer interactions | Greet → Present Situation → MCQ → Discuss → Practice Pitch → Teach → Next                       |
| **B**   | Teach-First     | Concepts/frameworks to learn BEFORE applying                          | Greet → Teach Concept → MCQ → Discuss → Present Situation → Practice → Next Topic               |
| **C**   | Reflective      | Internal skills: awareness, regulation, listening. No "right answer." | Greet + Warm-Up Q → Present Scenario → MCQ → Deep Discussion → Teach Framework → Reflect → Next |

---

## Two-Part ai_instructions Structure — CRITICAL

| Part        | Purpose                      | What to write                                                                                                                                         |
| ----------- | ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **FLOW**    | Step order the AI follows    | Numbered steps (`### 1. 2. 3.`). Each step names the **tool** and **action**: "Use the card tool to…", "Use MCQ tool with 4 options…"                 |
| **CONTENT** | Topics/scenarios the AI uses | Separate section: `## THE N SCENARIOS` or `## TEACHING CONTENT` — one block per topic with card text, MCQ options, discussion points, ideal responses |

The flow says _when_ to use which tool. The content says _what_ to show. Both
are required — the AI does not infer tools; it follows explicit instructions.

---

## Visual-Tool Discipline

Start with one visual presentation tool. Add another only when the learner
needs a different representation—for example, an image to establish a scene
and a card to make the framework scannable. Do not stack tools that repeat
the same information.

| Visual tool        | Default for               | Best at                                          |
| ------------------ | ------------------------- | ------------------------------------------------ |
| `card`             | Pattern A, Pattern B      | Situations, objections, concept cards            |
| `image_generation` | Pattern A (retail/visual) | Painting a scene — makes the situation feel real |
| `slide_generation` | Pattern C                 | Framework teaching, side-by-side models          |

`mcq` is useful for knowledge checks; `emoji_reaction` is optional. Register
`end_session` when the session has a planned close.

### Image generation notes

- Image style is **per-org**, NOT always Ghibli. Match the brand/domain:
  sports → Ghibli scenes, education → warm illustration, sales → not used.
- 2-3 images per session max. More dilutes impact.
- Prompt pattern: `"[style] illustration of [setting] showing [specific
situation], [people involved], [emotional tone], [lighting]"`

---

## Mandatory Rules

1. **Numbered FLOW with tool names in every step.**
2. **Visual before MCQ** when a visual introduces the decision.
3. **Use an MCQ only when a check is useful** — 3–4 plausible options.
4. **"Why did you choose this?" before reveal** — probe reasoning, then explain.
5. **Close after a summary** when the session has a planned end: say the
   goodbye, wait for the learner's reply, then call `end_session`.

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
### 7. Next or Wrap — After 3-4 scenarios, summarize and follow the closing turn rule
```

### Sample ai_instructions layout (Pattern A)

```
## ROLE
- You are Coach [Name], experienced [domain] trainer. Warm, encouraging.

## FLOW
### 1. Greeting & Context (1 min)
### 2. Present Situation — Use card tool
### 3. MCQ — Use mcq tool with 4 options
### 4. Explore Reasoning — "Why did you choose this?"
### 5. PRACTICE PITCH ⭐ — trainee speaks, coach plays customer
### 6. Teach Best Approach
### 7. Next or Wrap — summary, goodbye, then follow the closing turn rule

## CONTEXT / CONTENT
## THE 6 SCENARIOS (pick one randomly per round)
[Each: persona, situation, 4 MCQ options, best approach]

## TOOLS
card + mcq. When to call each. When to end_session.

## GUARDRAILS
## STYLE
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
### 8. Wrap-Up — Summary card, goodbye, then follow the closing turn rule
```

### Sample ai_instructions layout (Pattern B)

```
## ROLE
- You are [Coach Name], [title] at [Org].

## FLOW
### 1. Greeting (1 min)
### 2. Teach Concept — card tool
### 3. MCQ — test understanding
### 4. Discuss — "Why?" BEFORE revealing
### 5. Present Situation — card tool. WAIT.
### 6. Next Topic — Repeat 2-5
### 7. Final Practice Round ⭐
### 8. Wrap-Up — Summary card, goodbye, then follow the closing turn rule

## CONTEXT / CONTENT
### Topic 1: [Name]
- Concept Card / Key Teaching Points / MCQ / Situation / Ideal response

## TOOLS
## GUARDRAILS
## STYLE
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
### 6. Wrap-Up + Transfer Task (1-2 min) — "This week, try X in one conversation";
say goodbye, then follow the closing turn rule
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
  "ai_model_config": {
    "provider": "Ocean",
    "model": "medium-stable"
  },
  "strategy": {
    "skip_auto_start": false,
    "system_instructions_template": "minimal",
    "silence": { "silence_threshold": 8000, "end_session": false, "force_agent_to_speak": true },
    "conductor": {
      "enabled": true,
      "messages": [
        {
          "time_seconds": 1000,
          "message": "Wrap up the session with a summary card and goodbye. Wait for the learner's reply before end_session.",
          "end_turn": true
        }
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
  "session_analysis": { "is_auto_analysis": true, "is_auto_submit": true },
  "appearance": { "voice": "Aoede" },
  "memory": { "is_memory": true }
}
```

Ocean/medium-stable for card/MCQ coaching — fall back to Galaxy if rejected.
No TTS/STT/LLM sub-fields on either. Enable an additional visual tool only
when it has a distinct role in the FLOW.

---

## Anti-Patterns

| Anti-pattern                                  | Instead                                  |
| --------------------------------------------- | ---------------------------------------- |
| Combining card + image + slide for one job    | Use the one representation that advances the step |
| No practice step (A) or practice round (B)    | Always include verbal practice           |
| Teaching situation before concept (Pattern B) | Concept → MCQ → THEN situation           |
| Vague scenarios ("A customer objects")        | Full persona: name, age, exact words     |
| Exact card text in instructions               | Give guidance: topic, format, key points |
| `slide_generation` for Pattern A              | Use `card` for situations                |
| Skipping "Why?" after MCQ                     | Always probe reasoning before revealing  |

---

## Quality Checklist

- [ ] One of the 3 patterns chosen and followed
- [ ] FLOW: numbered steps, each naming its tool
- [ ] CONTENT section: one detailed block per topic/scenario
- [ ] Visual tools have distinct jobs; duplicate presentation tools disabled
- [ ] Visual before decision MCQ; "Why?" discussion before every reveal
- [ ] Practice step present (A: pitch practice; B: final round; C: transfer task)
- [ ] Summary, goodbye, and a response-gated `end_session` at the end
- [ ] Rubric weights match the pattern

## Key Files

- [../scenario-authoring.md](../scenario-authoring.md) — durable teaching and tool rules
- [../entities/scenario/model-selection.md](../entities/scenario/model-selection.md) — model stamp
- [../entities/scenario/control.md](../entities/scenario/control.md) — tools / strategy
- [../entities/scenario/ai-instructions.md](../entities/scenario/ai-instructions.md) — prompt shape
- [../entities/scenario/rubrik.md](../entities/scenario/rubrik.md) — evaluation design
