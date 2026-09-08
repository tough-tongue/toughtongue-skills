# Interview Scenario Patterns

Rules for structured, case, and behavioral interview scenarios. The AI is the
interviewer; the human is the candidate. The purpose is to elicit and evaluate
the candidate's reasoning, not to teach the reference answer during the run.

## Contents

- Interview contract
- Flow and disclosure
- Interview variants
- Evaluation
- Tools, closure, and quality checks

## Interview contract

Define these before writing `ai_instructions`:

| Element          | Decide                                                                          |
| ---------------- | ------------------------------------------------------------------------------- |
| Competency       | What observable reasoning, judgment, or communication behavior is evaluated?    |
| Scenario facts   | What is known, unknown, and never to be invented?                               |
| Hidden reference | What makes an answer strong without being revealed to the candidate?            |
| Reveal gates     | Which fact, exhibit, or hint is available only after which attempt or question? |
| Hint budget      | How many neutral probes occur before the AI supplies a fact or moves on?        |
| Timebox          | Which sections are essential if the interview must shorten?                     |
| Evaluation       | Which observable evidence maps to each rubrik dimension?                        |

Accept multiple defensible approaches. State the facts and scoring evidence, not
one mandatory structure or conclusion.

## Flow and disclosure

1. **Set the frame** — identify the interview type, goal, timebox, and whether
   the candidate may ask clarifying questions.
2. **Prompt once** — give the starting situation, then stop and let the
   candidate structure or answer.
3. **Probe the reasoning** — ask one open follow-up at a time. Probe a strong
   answer one level deeper; help a weak answer with a question before supplying
   anything.
4. **Reveal deliberately** — share gated facts only when the candidate reaches
   the intended decision point, asks a valid question, or exhausts the stated
   hint budget.
5. **Test judgment** — ask the candidate to name assumptions, trade-offs,
   risks, and a recommendation when those fit the interview.
6. **Close** — thank the candidate and direct them to the session report; use
   the response-gated close rule in
   [scenario-authoring.md](../../scenario-authoring.md).

One question or action per turn. If the candidate asks for thinking time, wait
quietly. Do not fill the pause with hints or restate the prompt.

## Interview variants

### Behavioral or competency interview

- Define the competency and the evidence that demonstrates it.
- Ask for one concrete example, then probe context, the candidate's action,
  trade-offs, and outcome.
- Challenge vague claims with requests for specifics; do not supply a better
  story for them.
- Score the candidate's demonstrated behavior, not fluency alone.

### Case or problem-solving interview

- Separate the case prompt, clarifying facts, hidden calculations, exhibits,
  traps, and expected decision criteria.
- Keep the answer key hidden. A reference answer guides follow-ups; it is not
  dialogue to read to the candidate.
- State which data may be repeated, which must be requested, and what to say
  when the scenario has no answer.
- Let the candidate choose a reasonable structure. Correct only material logic
  gaps through neutral probes before revealing a missing fact.
- Gate each exhibit by an explicit condition. Never show later evidence simply
  because a previous phase ended.

### Screening or structured interview

- Specify required qualifications, disqualifiers, and permitted follow-up
  depth.
- Ask only the minimum questions needed to establish each criterion.
- Handle sensitive information with applicable consent, disclosure, and
  escalation rules. Do not infer protected characteristics.
- Keep the rubrik target explicit: candidate quality, qualification result, or
  an extracted hiring brief.

## `ai_instructions` additions

Add these sections to the shared
[instruction shape](../../entities/scenario/ai-instructions.md):

```text
## INTERVIEW CONTRACT
Role, candidate role, competency, timebox, and evaluation target.

## CASE FACTS AND DISCLOSURE
Known facts, unknown-fact response, reveal gates, and hint budget.

## FLOW
Opening → candidate attempt → response-gated probes → recommendation or close.

## GUARDRAILS
Do not reveal the reference answer early. Do not force one defensible approach.
Wait silently when the candidate asks for time.
```

Use prompt directives, not quoted scripts. The AI should react naturally to
the candidate's actual reasoning.

## Evaluation

Score observable evidence:

- **Structure** — a coherent approach appropriate to the question.
- **Analysis** — accurate use of facts, assumptions, and calculations.
- **Judgment** — trade-offs, risks, and a defensible recommendation.
- **Communication** — clear, concise reasoning and response to probes.

Adjust or omit dimensions to fit the competency. Do not invent a universal
weighting formula; the rubrik defines its own internally consistent scoring.

## Tools, closure, and quality checklist

- Enable a visual tool only when an exhibit, case card, or timed artifact is
  genuinely part of the interview. Define its reveal gate and fallback.
- Use a browser or slides only when the interview needs live material; do not
  turn every interview into a demo.
- Enable `end_session` for a closable session and follow the shared
  response-gated close rule.

- [ ] Competency, candidate role, facts, hint budget, and timebox are explicit.
- [ ] One question per turn; candidate thinking time stays quiet.
- [ ] Reference answers and later exhibits cannot leak early.
- [ ] Candidate gets credit for defensible alternatives.
- [ ] Rubrik scores observable candidate evidence.

## Key Files

- [../../scenario-authoring.md](../../scenario-authoring.md) — situation-neutral design and closure rules
- [../../entities/scenario/model-selection.md](../../entities/scenario/model-selection.md) — model stamp
- [../../entities/scenario/control.md](../../entities/scenario/control.md) — tools and runtime control
- [../../entities/scenario/ai-instructions.md](../../entities/scenario/ai-instructions.md) — prompt structure
- [../../entities/scenario/rubrik.md](../../entities/scenario/rubrik.md) — evaluation contract
