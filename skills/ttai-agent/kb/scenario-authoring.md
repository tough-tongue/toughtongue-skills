# Scenario authoring principles

High-quality scenarios are **decision environments**, not scripts. A scenario
gives a person realistic context, resistance, feedback, and a clear finish so
the behavior practiced transfers to the real conversation.

These principles are distilled from successful Tough Tongue AI scenario
programs. They are intentionally situation-neutral; pair them with one
[recipe](scenario-recipes/index.md) and the live scenario schema.

## Contents

- Authoring contract
- Realism and difficulty
- Teaching and tools
- Closure and re-engagement
- Evaluation and iteration
- Corpus hygiene

---

## Authoring contract

Write the following before drafting prose:

1. **Human outcome** — what can they do after this session?
2. **AI role** — caller, prospect, coach, observer, or assistant.
3. **Reality model** — known facts, unknown facts, emotional stakes, and
   what the AI may disclose only when asked.
4. **Decision path** — opening, discovery, challenge, resolution, and exit.
5. **Evidence of success** — observable behaviors the rubrik will score or
   information that the report must extract.

Keep those contracts aligned. The AI's role, `user_instructions`, FLOW,
tools, and rubrik must evaluate the same person and outcome.

## Realism and difficulty

- Prefer specific, verified context over generic personas. Use dynamic
  variables only for facts that genuinely change per session; give every
  variable a graceful missing-value fallback.
- Model **progressive disclosure**: reveal detailed context in response to
  thoughtful questions, not in an opening monologue.
- Give the counterpart an internally coherent reason to resist. Define what
  evidence changes their mind, what fails, and which outcome is the default.
- Make difficulty intentional. Vary resistance, time pressure, ambiguity,
  evidence requirements, and willingness to share — not random hostility.
- Encode non-negotiable facts as explicit verification points. State both
  the truthful answer and the false claim that must not be rewarded.
- Use concrete exit triggers. A firm opt-out, unsafe topic, or repeated
  boundary violation ends the interaction; a soft objection is not
  automatically an exit.
- Preserve uncertainty honestly. A scenario must never train people to make
  unsupported claims, manufactured urgency, or harmful guarantees.

## Situation guardrails

- **Interviews:** define the competency, evidence, case facts, hint budget,
  reveal order, and timebox. Let the candidate lead their reasoning; do not
  disclose the answer or correct path before they have attempted it.
- **Meeting facilitation:** define agenda, observation mode, nudge threshold,
  handoff state, and synthesis format. Silence can be intentional; do not
  apply a talkative call-agent silence rule to an observer or facilitator.
- **Sensitive or regulated conversations:** state approved claims, required
  disclosures, prohibited advice, consent requirements, and human escalation
  triggers before drafting roleplay dialogue.

## Teaching and tools

- Teach a **transferable pattern**, then let the human apply it to a realistic
  case. A concept without practice does not prove capability.
- For skills training, make the learner explain their reasoning before
  revealing the better approach. Feedback should name the behavior, why it
  mattered, and a sentence they can try next time.
- Use an artifact only when it changes the learning or interaction:
  a card frames a case, an MCQ tests a decision, a browser shows the product,
  and slides anchor a narrative. The live `tools_config` is the authority.
- Treat every tool as a contract. Tell the AI when to call it, what result is
  needed, and what to do if it is unavailable. Do not write prompts that
  depend on a tool not enabled on the scenario or channel.
- Keep spoken experiences conversational: one question per turn, room to
  answer, and no document-shaped output that a voice will read aloud.

## Closure and re-engagement

- Treat closure as a short state transition, not one tool call: state the
  outcome and goodbye in one turn, wait for the other person to respond, then
  call `end_session` when the conversation is still complete.
- Do not call `end_session` in the same turn as a question, pitch, summary, or
  goodbye. The person needs a natural chance to acknowledge or re-open the
  conversation.
- When a session end is pending, ordinary thanks, goodbyes, noise, or
  crosstalk do not reopen it. A substantive question, a request to wait, or a
  new topic does: cancel the pending end before responding.
- Specify the closing condition for the situation. A firm opt-out may begin an
  immediate close, but it still follows the goodbye → response → end sequence.

## Evaluation and iteration

- Score observable stages, not vague traits. Weight the moments that matter
  most, and define what weak, competent, and excellent behavior looks like.
- Keep the rubrik's subject explicit: sales practice usually scores the
  human rep; lead qualification and demo intelligence may evaluate the
  counterpart or extract business facts instead.
- Separate **learning feedback** (how the human performed) from **business
  intelligence** (what happened in the conversation). A single scenario may
  need one, both, or neither.
- Seed variation deliberately: rotate cases, objections, or profiles while
  preserving the invariant skill being evaluated.
- Refine from session evidence. Change the narrowest FLOW, guardrail, tool
  rule, or rubric criterion that explains the failure; do not keep layering
  more prose onto a weak scenario.

## Corpus hygiene

Historical scenarios are examples of behavior and product discovery, not a
configuration source of truth. Do not copy their IDs, organization data,
credentials, deprecated fields, model stamps, or customer-specific facts.

Before creating or updating a scenario, check
[entities/scenario/model-selection.md](entities/scenario/model-selection.md) +
[entities/scenario/control.md](entities/scenario/control.md) and load the
live MCP schema. Use the current product contract even when an older scenario
appears to work differently.

## Key Files

- [entities/scenario/ai-instructions.md](entities/scenario/ai-instructions.md) — prompt structure
- [entities/scenario/rubrik.md](entities/scenario/rubrik.md) — evaluation text
- [scenario-recipes/](scenario-recipes/index.md) — situation-specific patterns
- [operating-model.md](operating-model.md) — consumer-neutral action protocol
