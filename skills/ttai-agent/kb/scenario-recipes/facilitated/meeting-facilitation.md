# Meeting Facilitation Scenario Patterns

Rules for an AI that facilitates, observes, or synthesizes a human meeting.
The humans own the discussion. The AI should speak only when its defined role
adds value.

## Contents

- Facilitation contract
- Conversation states
- Silence and nudges
- Synthesis and evaluation
- Tools, closure, and quality checks

## Facilitation contract

Define these before writing `ai_instructions`:

| Element      | Decide                                                                        |
| ------------ | ----------------------------------------------------------------------------- |
| Meeting goal | Decision, discovery, planning, review, or stand-up                            |
| Agenda       | Ordered topics, timeboxes, and the owner or decision for each                 |
| AI mode      | Active facilitator, silent observer, or on-demand notetaker                   |
| Invocation   | Wake word, direct-question rule, and handoff back to observation              |
| Nudge policy | What counts as a stall, drift, unresolved decision, or time risk              |
| Synthesis    | Required decisions, owners, actions, disagreements, and open questions        |
| Evaluation   | Whether the session assesses facilitation quality, meeting output, or neither |

Do not borrow the silence policy from a call agent. A useful observer remains
quiet through productive discussion.

## Conversation states

Write the flow as explicit states, even when the scenario does not use a
multi-stage runtime:

1. **Open** — frame the purpose, agenda, timebox, and how participants invoke
   the AI.
2. **Observe** — listen without speaking. Track decisions, unresolved points,
   ownership, and agenda progress.
3. **Intervene** — speak only when invoked or when a defined nudge threshold
   is met. Keep the intervention short and return to observation.
4. **Synthesize** — on an explicit wrap-up request or the defined time
   boundary, state the agreed output once.
5. **Close** — use the response-gated close rule in
   [scenario-authoring.md](../../scenario-authoring.md).

Specify the transition trigger for every state. For example, a direct question
may move Observe → Intervene; an answer must move Intervene → Observe, not
start an AI-led side conversation.

## Silence and nudges

- Silence is correct while humans are productively discussing the agenda.
- Define a nudge threshold in observable terms: repeated disagreement without
  a decision, discussion that leaves the agenda, an unanswered owner, or a
  timebox risk.
- State the nudge channel: a short spoken question, a visible card, or no
  intervention. Use a card only when the runtime and meeting format support it.
- Make every nudge actionable and neutral. Ask for a decision, owner,
  assumption, or next step; do not lecture or dominate.
- Limit cadence and repetition. The AI should not convert a quiet meeting into
  a stream of status updates.
- Do not infer consensus from silence. Name uncertainty in the final synthesis.

## `ai_instructions` additions

Add these sections to the shared
[instruction shape](../../entities/scenario/ai-instructions.md):

```text
## MEETING CONTRACT
Goal, agenda, timebox, AI mode, and participant invocation rule.

## OBSERVATION
Track decisions, owners, unresolved questions, risks, and agenda progress.
Remain silent unless a defined trigger occurs.

## NUDGES
Trigger, channel, cadence, and one-sentence intervention pattern.

## SYNTHESIS
Decisions, action items with owners, unresolved disagreements, and next steps.
Deliver once.
```

When a workflow requires distinct opening, observer, speaker, or closer roles,
use an existing Studio-managed multi-stage scenario only if those handoffs are
necessary. Keep a single-flow scenario when explicit state rules are enough.

## Synthesis and evaluation

Deliver one concise synthesis:

1. Decisions and their rationale.
2. Action items with owners and due dates when stated.
3. Unresolved disagreements or assumptions.
4. The next discussion or follow-up needed.

Do not fabricate ownership, commitments, or consensus. Separate a participant's
proposal from an agreed decision.

If evaluated, make the target explicit:

- **Facilitation quality** — agenda framing, appropriate silence, useful nudges,
  accurate synthesis, and respectful intervention.
- **Meeting output** — decision clarity, ownership, risks surfaced, and
  follow-up quality.

Do not use a generic sales or coaching rubrik for a meeting.

## Tools, closure, and quality checklist

- Enable a card or visual tool only if it provides an agenda, decision log, or
  synthesis the participants can use.
- Register `end_session` when the meeting should close, then follow the shared
  response-gated close rule.
- Keep direct AI responses to one or two sentences unless the synthesis calls
  for a longer structured summary.

- [ ] Goal, agenda, timebox, AI mode, and invocation rule are explicit.
- [ ] Observation mode stays quiet during productive human discussion.
- [ ] Nudge triggers, channel, cadence, and return-to-observation behavior are explicit.
- [ ] Synthesis separates decisions, owners, disagreements, and unknowns.
- [ ] Rubrik evaluates the stated target only.

## Key Files

- [../../scenario-authoring.md](../../scenario-authoring.md) — situation-neutral design and closure rules
- [../../entities/scenario/model-selection.md](../../entities/scenario/model-selection.md) — model stamp
- [../../entities/scenario/control.md](../../entities/scenario/control.md) — tools and runtime control
- [../../entities/scenario/ai-instructions.md](../../entities/scenario/ai-instructions.md) — prompt structure
- [../../entities/scenario/rubrik.md](../../entities/scenario/rubrik.md) — evaluation contract
