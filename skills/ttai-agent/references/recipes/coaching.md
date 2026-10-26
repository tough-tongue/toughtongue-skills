# Coaching Scenario Patterns

Rules for scenarios where the **AI teaches skills through interactive
exercises** (not roleplay). The AI is the trainer/mentor.

## Contents

- Coach vs roleplay decision
- The three patterns (A Situation-First, B Teach-First, C Reflective)
- Two-part ai_instructions structure (FLOW + CONTENT)
- Visual-tool discipline and mandatory rules
- Judging spoken answers (meaning, not wording)
- Pattern A / B / C step-by-step flows with templates
- user_instructions structure
- Technical config, anti-patterns, quick checklist

Key facts:

- Realtime: Ocean `medium-stable` when card/MCQ (or other artifacts) are on;
  otherwise Galaxy `medium-stable`. Fall back to Galaxy if Ocean is rejected.
  See [../scenario/model-selection.md](../scenario/model-selection.md).
- Three patterns: A (Situation-First), B (Teach-First), C (Reflective)
- Every scenario needs FLOW (steps with tool names) + CONTENT (topics)
- Choose a deliberate visual palette: `card`, `image_generation`, or
  `slide_generation`. Combine tools only when each has a distinct job.
- When a visual introduces a decision: visual before MCQ, then "Why did you
  choose this?" before reveal
- MCQ pacing: never explain and show an MCQ in the same turn. End the
  explanation with a check-in and wait; after the reply, show the MCQ, then read
  the question. Write it into every MCQ step (Mandatory Rules 3–4)
- Judge spoken answers by meaning, not wording, unless an item must be exact
  (Judging Spoken Answers)
- Coaching is about _knowing what to do_; roleplay is about _doing it_

---

## Coach vs Roleplay

- **Coach** — skill = _knowing what to do_: teaching knowledge, frameworks,
  decision-making. The AI is the trainer/mentor.
- **Roleplay** — skill = _the conversation itself_: practice talking to a real
  person. The AI plays a character (see `sales-roleplay.md`).

---

## The Three Patterns

Pick ONE pattern based on what you're teaching.

### A — Situation-First

- **Use when:** handling specific situations: sales objections, customer
  interactions.
- **Flow:** Greet → Present Situation → MCQ → Discuss → Practice Pitch → Teach →
  Next

### B — Teach-First

- **Use when:** concepts/frameworks to learn BEFORE applying.
- **Flow:** Greet → Teach Concept → MCQ → Discuss → Present Situation → Practice
  → Next Topic

### C — Reflective

- **Use when:** internal skills: awareness, regulation, listening. No "right
  answer."
- **Flow:** Greet + Warm-Up Q → Present Scenario → MCQ → Deep Discussion → Teach
  Framework → Reflect → Next

---

## Two-Part ai_instructions Structure — CRITICAL

- **FLOW** — the step order the AI follows. Write numbered steps
  (`### 1. 2. 3.`). Each step names the **tool** and **action**: "Use the card
  tool to…", "Use MCQ tool with 4 options…"
- **CONTENT** — the topics/scenarios the AI uses. Write a separate section:
  `## THE N SCENARIOS` or `## TEACHING CONTENT` — one block per topic with card
  text, MCQ options, discussion points, ideal responses.

The flow says _when_ to use which tool. The content says _what_ to show. Both
are required — the AI does not infer tools; it follows explicit instructions.

---

## Visual-Tool Discipline

Start with one visual presentation tool. Add another only when the learner needs
a different representation—for example, an image to establish a scene and a card
to make the framework scannable. Do not stack tools that repeat the same
information.

| Visual tool        | Default for       | Best at                                 |
| ------------------ | ----------------- | --------------------------------------- |
| `card`             | Patterns A and B  | Situations, objections, concept cards   |
| `image_generation` | A (retail/visual) | A scene that makes the situation real   |
| `slide_generation` | Pattern C         | Framework teaching, side-by-side models |
| `google_slides`    | The user's deck   | Teaching slide by slide from a set deck |

With a Google Slides deck, follow Google Slides decks in
[../scenario/control.md](../scenario/control.md): `/pubembed` link, SLIDE MAP,
a slide cue in every FLOW step, other visual tools off.

`mcq` is useful for knowledge checks; `emoji_reaction` is optional. Register
`end_session` when the session has a planned close.

### Image generation notes

- Match the image style to the brand and audience (for example, warm
  illustration for education). Sales coaching rarely needs images.
- 2-3 images per session max. More dilutes impact.
- Prompt pattern:
  `"[style] illustration of [setting] showing [specific
situation], [people involved], [emotional tone], [lighting]"`

---

## Mandatory Rules

1. **Numbered FLOW with tool names in every step.**
2. **Visual before MCQ** when a visual introduces the decision.
3. **Check in, wait, then the MCQ.** Never explain and show an MCQ in the same
   turn. The explanation before an MCQ (a card, slide, image or spoken concept)
   ends with a check-in, and the coach waits for the reply. Answer any question
   it raises, then run the MCQ. If the FLOW already checks in after every
   slide or card, this check-in takes that place, so it adds no turn.
   - Write it into **every FLOW step that runs an MCQ**, as a plain directive,
     not a quoted line (a quoted line is recited the same way every time).
   - Why per step: a realtime voice model can call a tool while its speech is
     still playing, so a step that says "explain X, then run the MCQ" puts the
     MCQ on screen mid-sentence. A general rule elsewhere ("only after the
     learner replies") does not stop it: any earlier reply, even a "yes" before
     the slide, satisfies it.
4. **Show the MCQ, then read it.** Call the MCQ tool first; read the question
   aloud only once it is on screen. Never say the options are on screen before
   the tool call. Put this in the TOOLS or MCQ section once.
5. **Use an MCQ only when a check is useful** — 3–4 plausible options.
6. **"Why did you choose this?" before reveal** — probe reasoning, then explain.
7. **Judge spoken answers by meaning**, not wording, unless an item must be
   exact. See Judging Spoken Answers below.
8. **Close after a summary** when the session has a planned end: say the
   goodbye, wait for the learner's reply, then call `end_session`.

Rules 3–4 in a FLOW (adapt the content; keep the shape):

```
Weak:   4. Show slide 4 and explain it. Run MCQ 1.
Strong: 4. Show slide 4 and explain it. Ask if they have any questions or are
           ready for a quick check, and wait. Answer any question, then run
           MCQ 1.
Strong: 6. Show slides 8 to 12, one per turn. After slide 12, ask if they have
           any questions or are ready for a quick check, and wait. Answer any
           question, then run MCQ 3.
MCQ section: Show the MCQ with the mcq tool, then read the question once it is
on screen.
```

---

## Judging Spoken Answers

Applies whenever the coach judges what the learner says: a teach-back, a
practice pitch, the reasoning behind an MCQ pick, or any open question. MCQ
picks are clicks and need no leniency; spoken answers do.

**Judge by meaning** when the check is about understanding or skill. Speech
arrives imperfect: recognition errors, accents, mixed languages, cut-off turns.
A coach that compares the learner's words with the slide marks correct answers
wrong and feels rigid.

**Keep exact** what must be exact: a price, number, name or date, a required
disclosure, a safety step, or a term the learner must use word for word. Name
these items in the check; judge only them by wording.

For each spoken check, CONTENT carries:

- **CORE IDEA** — one sentence: what the listener must take away.
- **Passing answers** — one or two, worded differently, labelled as examples.
- **Not passing** — one answer, with the reason it fails.
- **TIP** (optional) — a bonus to offer after a pass, never a correction.

Example check (adapt the content; keep the shape):

```
TEACH-BACK 2 — The late refund request
Trigger, as the customer: Can I still get my money back? It's been six weeks.
CORE IDEA: the customer hears an honest no, plus one real option the agent
will act on now.
Passing answers (examples): The refund window was thirty days, so a refund
isn't possible, but I can move you to the cheaper plan today. / Honestly, the
thirty days have passed; what I can do right now is pause your billing.
Not passing: Let me check with my team and get back to you. It answers
nothing and commits to nothing.
Must be exact: thirty days.
TIP: offer the option before they ask for one.
```

Judging routine — put it in the prompt once and adapt the bracketed names:

```
Judge the CORE IDEA by meaning: would [the listener] take it away? Their own
words, a different order, a mixed-language answer or a clumsy transcript all
pass when the idea is there. Never compare their words with the slide or the
example answers. Be lenient: a close variation of the idea counts as landed;
offer the fuller version as a TIP. Check only the items marked Must be exact
by their exact value.
- Landed: praise the specific thing that worked; add one TIP at most; move on.
- Missing: praise what worked, name the missing idea in plain words, and
  invite one more try in their own words.
- Still missing after that try: explain the idea briefly, give one way to
  say it, and move on. Never ask them to repeat a line word for word.
If their words look cut off or garbled, ask them to say it again instead of
marking anything missing.
```

Write the rubric the same way, or the report contradicts the coach: say it
judges meaning, not wording, and name the must-be-exact items
([../scenario/rubric.md](../scenario/rubric.md)).

---

## Pattern A: Situation-First

Session: 10-15 min. 4-8 case studies, pick 3-4 randomly. Visual: `card`
(default) or `image_generation` (retail).

```
### 1. Greeting (1 min) — Set expectation, pick ONE scenario randomly
### 2. Present Situation (1 min) — Use card tool to show situation, then check
    in and wait
### 3. MCQ (2-3 min) — After their reply: show it, then read it. 3-4 options
    (poor → excellent), emoji_reaction
### 4. Discuss (2-4 min) — "Why?" before reveal, probe follow-ups
### 5. Practice Pitch (3-5 min) ⭐ — Trainee speaks, coach plays customer,
    repeat until natural
### 6. Teach (1-3 min) — Ideal script, good vs improve
### 7. Next or Wrap — After 3-4 scenarios, summarize and follow the closing
    turn rule
```

### Sample ai_instructions layout (Pattern A)

```
## ROLE
- You are Coach [Name], experienced [domain] trainer. Warm, encouraging.

## FLOW
### 1. Greeting & Context (1 min)
### 2. Present Situation — Use card tool, then check in and wait
### 3. MCQ — After their reply, use mcq tool with 4 options, then read it
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
- **Core idea**: [the one idea a good response must carry]
- **Best script**: [one ideal response, labelled as an example, not a script
  to match word for word]
- **Common mistakes**: Arguing, defensive, no next step
```

Rubric weights: MCQ 15% | Reasoning 20% | Pitch Practice 35-40% | Feedback
Absorption 15-25%

---

## Pattern B: Teach-First

Session: 15-20 min. 4-5 topics in logical sequence (not random). Visual: `card`.

```
### 1. Greeting (1 min) — Set the frame for what will be learned
### 2. Teach Concept (2-3 min) — card tool with concept (table/bullets/timeline),
    then check in and wait
### 3. MCQ (2 min) — After their reply: show it, then read it. Test concept
    grasp (not situation yet)
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
### 2. Teach Concept — card tool, then check in and wait
### 3. MCQ — After their reply, show it, then read it. Test understanding
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
- **Ideal response**: CORE IDEA, one or two passing answers, one that does
  not pass (see Judging Spoken Answers)
```

Rubric weights: Practice Execution 30% | Concept Understanding 25% | Situation
Handling 25% | Feedback Absorption 20%

---

## Pattern C: Reflective

Session: 10-15 min. 2-3 deeper scenarios (not 4-8 quick ones). Visual:
`slide_generation`.

```
### 1. Greeting + Warm-Up Q (2 min) — Reflective question setting
    introspective tone
### 2. Present Scenario (1 min) — slide_generation with power dynamics,
    emotional stakes, then check in and wait
### 3. MCQ (2-3 min) — After their reply: show it, then read it.
    Reactive/compliant to authentic/regulated (no obvious "right answer")
### 4. Deep Discussion (4-6 min) — 3-4 questions exploring patterns
### 5. Teach Framework (3-5 min) — slide_generation for visual, evidence not
    motivation
### 6. Wrap-Up + Transfer Task (1-2 min) — "This week, try X in one
    conversation"; say goodbye, then follow the closing turn rule
```

Rubric weights: MCQ ~33% | Reasoning & Self-Reflection ~33% | Absorbing Better
Practices ~33%

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
    "silence": {
      "silence_threshold": 8000,
      "end_session": false,
      "force_agent_to_speak": true
    },
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
      "emoji_reaction": {
        "should_register": true,
        "add_to_system_prompt": false
      },
      "slide_generation": {
        "should_register": false,
        "add_to_system_prompt": false
      },
      "image_generation": {
        "should_register": false,
        "add_to_system_prompt": false
      }
    }
  },
  "session_analysis": { "is_auto_analysis": true, "is_auto_submit": true },
  "appearance": { "voice": "Aoede" },
  "memory": { "is_memory": true }
}
```

Ocean/medium-stable for card/MCQ coaching — fall back to Galaxy if rejected. No
TTS/STT/LLM sub-fields on either. Enable an additional visual tool only when it
has a distinct role in the FLOW.

---

## Anti-Patterns

Each line: anti-pattern → instead.

- Combining card + image + slide for one job → use the one representation that
  advances the step.
- No practice step (A) or practice round (B) → always include verbal practice.
- Teaching situation before concept (Pattern B) → Concept → MCQ → THEN
  situation.
- Vague scenarios ("A customer objects") → full persona: name, age, exact words.
- Exact card text in instructions → give guidance: topic, format, key points.
- `slide_generation` for Pattern A → use `card` for situations.
- Skipping "Why?" after MCQ → always probe reasoning before revealing.
- Explanation and MCQ in one step with no wait ("Explain X. Run MCQ 1.") → the
  MCQ appears mid-sentence; end the explanation with a check-in, wait, then run
  the MCQ.
- MCQ pacing only as a general rule ("start an MCQ only after the learner
  replies") → write the check-in into each MCQ step; any earlier reply
  satisfies a general rule.
- A quoted check-in line → a plain directive; quoted lines are recited
  identically every time.
- Reading the question, or saying the options are on screen, before the MCQ
  tool call → show the MCQ first, then read it.
- Checking a spoken answer against the slide's wording → judge the CORE IDEA
  by meaning; keep only the must-be-exact items exact.
- Marking a garbled or cut-off answer as wrong → ask them to say it again.
- Asking the learner to repeat a model line word for word → explain the idea,
  give one way to say it, move on.

---

## Quality Checklist

- [ ] One of the 3 patterns chosen and followed
- [ ] FLOW: numbered steps, each naming its tool
- [ ] CONTENT section: one detailed block per topic/scenario
- [ ] Visual tools have distinct jobs; duplicate presentation tools disabled
- [ ] Visual before decision MCQ; "Why?" discussion before every reveal
- [ ] Every MCQ step ends its explanation with a check-in and a wait, written
      in the step as a plain directive; the MCQ is shown before it is read
- [ ] Every spoken check has a CORE IDEA, passing and not-passing examples,
      and its must-be-exact items; the judging routine is in the prompt once;
      the rubric also judges meaning
- [ ] Practice step present (A: pitch practice; B: final round; C: transfer
      task)
- [ ] Summary, goodbye, and a response-gated `end_session` at the end
- [ ] Rubric weights match the pattern

## Key Files

- [../scenario/authoring.md](../scenario/authoring.md) — durable teaching and
  tool rules
- [../scenario/model-selection.md](../scenario/model-selection.md) — model stamp
- [../scenario/control.md](../scenario/control.md) — tools / strategy
- [../scenario/ai-instructions.md](../scenario/ai-instructions.md) — prompt
  shape
- [../scenario/rubric.md](../scenario/rubric.md) — evaluation design
