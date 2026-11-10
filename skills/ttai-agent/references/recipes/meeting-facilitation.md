# Meeting Scenario Patterns

Rules for an AI that joins a human meeting: to facilitate it, to observe and
synthesize it, or to sit in as a participant that speaks only when called. The
humans own the discussion. The AI should speak only when its defined role adds
value.

Pick the pattern first:

- **Runs the meeting** (opens, nudges, wraps up) — facilitator: the contract,
  states, and nudges below.
- **Listens and reports afterwards** — observer: the same contract, with no
  spoken turns until the synthesis.
- **Stays silent until someone says its name** — go to
  [Participant that speaks only when called](#participant-that-speaks-only-when-called).

## Contents

- Facilitation contract
- Conversation states
- Silence and nudges
- `ai_instructions` additions
- Participant that speaks only when called
- Synthesis and evaluation
- Tools, closure, and quality checklist

## Facilitation contract

Define these before writing `ai_instructions`:

- **Meeting goal** — decision, discovery, planning, review, or stand-up.
- **Agenda** — ordered topics, timeboxes, and the owner or decision for each.
- **AI mode** — active facilitator, silent observer, or on-demand notetaker.
- **Invocation** — wake word, direct-question rule, and handoff back to
  observation. If the AI must be silent until invoked, use the participant
  pattern, not a prompt rule.
- **Nudge policy** — what counts as a stall, drift, unresolved decision, or time
  risk.
- **Synthesis** — required decisions, owners, actions, disagreements, and open
  questions.
- **Evaluation** — whether the session assesses facilitation quality, meeting
  output, or neither.

Do not borrow the silence policy from a call agent. A useful observer remains
quiet through productive discussion.

## Conversation states

Write the flow as explicit states, even when the scenario does not use a
multi-stage runtime:

1. **Open** — frame the purpose, agenda, timebox, and how participants invoke
   the AI.
2. **Observe** — listen without speaking. Track decisions, unresolved points,
   ownership, and agenda progress.
3. **Intervene** — speak only when invoked or when a defined nudge threshold is
   met. Keep the intervention short and return to observation.
4. **Synthesize** — on an explicit wrap-up request or the defined time boundary,
   state the agreed output once.
5. **Close** — use the response-gated close rule in
   [authoring.md](../scenario/authoring.md#closure-and-re-engagement).

Specify the transition trigger for every state. For example, a direct question
may move Observe → Intervene; an answer must move Intervene → Observe, not start
an AI-led side conversation.

## Silence and nudges

- Silence is correct while humans are productively discussing the agenda.
- Define a nudge threshold in observable terms: repeated disagreement without a
  decision, discussion that leaves the agenda, an unanswered owner, or a timebox
  risk.
- State the nudge channel: a short spoken question, a visible card, or no
  intervention. Use a card only when the runtime and meeting format support it.
- Make every nudge actionable and neutral. Ask for a decision, owner,
  assumption, or next step; do not lecture or dominate.
- Limit cadence and repetition. The AI should not convert a quiet meeting into a
  stream of status updates.
- Do not infer consensus from silence. Name uncertainty in the final synthesis.

## `ai_instructions` additions

Add these sections to the shared
[instruction shape](../scenario/ai-instructions.md):

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

A facilitator with distinct opening and closing roles can also use a `super`
Scenario (one flow per role). Keep a single-flow Scenario when the AI speaks on
its own schedule and explicit state rules are enough. When the AI must stay
silent until it is called, use the participant pattern below: a prompt rule
alone does not hold.

## Participant that speaks only when called

Use this when the user asks for an AI teammate, expert, or assistant that sits
in a group meeting and "only talks when we ask it", "stays quiet until I say
its name", "has a wake word", or "stops when I tell it to be quiet".

### Why a prompt rule is not enough

A `default` Scenario answers every turn it hears. In a group meeting every
sentence anyone says is a turn. An instruction such as "speak only when your
name is called" lowers the rate but does not stop it: the agent still starts
replies to side conversations, apologises when told to be quiet, and talks
over the room. Do not ship that version.

The fix is structural. Give the Scenario two modes and let the platform mute
one of them:

- **Supervisor** — the voice. Answers when called, then hands the meeting back
  to the observer.
- **Observer** — a flow with `ai_model: { "provider": "standby", "model":
"standby" }`. A standby flow hears the meeting and can hand off, but the
  platform discards everything it would say. It cannot make a sound.

The agent spends most of the meeting in the observer, so silence is the
default and speech is the exception.

### Collect before writing

- **Wake name** — the name people will say, in the language they will say it
  ("Maya", "村井さん"). Include common variants.
- **Quiet words** — what people say to stop it ("be quiet", "one moment",
  "黙って"). Always include the plain-language ones even if the user names
  none.
- **After an answer** — go silent at once (default), or accept a direct
  follow-up first.
- **First words** — the one-line introduction for the first time it is called,
  or none.
- **What it may answer from** — the facts, knowledge FileBase, or tools it
  uses. A participant with nothing to answer from invents.
- **Leaving** — whether it should leave when asked (needs `end_session`), or
  stay until the host removes it.

### Scenario shape

One stage, one flow. The stage is the supervisor; the flow is the observer.

```json
{
  "name": "Maya (meeting participant)",
  "type": "super",
  "ai_instructions": "Super agent scenario. Instructions come from stages.",
  "stages": [
    {
      "name": "Maya",
      "role": "Product specialist sitting in the meeting",
      "goal": "Stay silent in the meeting; answer briefly when called by name; go silent again",
      "common_instructions": "<identity, speaking rule, facts, guardrails, style>",
      "instructions": "<supervisor rules>",
      "flows": [
        {
          "name": "observer",
          "role": "Silent listener",
          "goal": "Hand back to the supervisor only when Maya is called and asked something",
          "tool_ids": [],
          "ai_model": { "provider": "standby", "model": "standby" },
          "instructions": "<observer rules>"
        }
      ]
    }
  ],
  "strategy": {
    "skip_auto_start": true,
    "max_duration_seconds": 3600,
    "silence": {
      "silence_threshold": 1200000,
      "end_session": false,
      "force_agent_to_speak": false
    }
  },
  "tools_config": {
    "tools": { "end_session": { "should_register": false } }
  }
}
```

Add `ai_model_config`, `appearance`, and `transcribe_config` from
[model-selection.md](../scenario/model-selection.md). `common_instructions` is
shared by the supervisor and the observer, so the facts and the speaking rule
live there once.

The platform gives the supervisor and every flow a handoff tool and adds the
standby rules (no speech, hand off only when addressed) to the observer's
prompt. Write the instructions in plain words ("hand off to observer", "hand
back to the supervisor"); do not name tool arguments.

### Supervisor instructions

Write per-turn rules, in this order. Replace the names and phrases.

```text
You are the speaking side of Maya. The silent side is the observer flow.

## WHEN THE SESSION STARTS
Say nothing. Do not greet. Hand off to observer at once.
Memo: "Meeting started. Not introduced yet."

## WHEN THE OBSERVER HANDS BACK
Read the memo and do exactly one of these:
1. First time you are called: say "<one-line introduction>". If the memo
   holds a question, answer it next.
2. A question or request: answer in one to three sentences from FACTS.
   Do not introduce yourself again.
3. No question or request in the memo: say nothing and hand off to observer.

## AFTER EVERY ANSWER
Do not ask a question back and do not keep talking. In the same turn, hand
off to observer. Memo: what you answered, and "introduced".

## QUIET WORDS (highest priority)
If anyone says "be quiet", "stop", "one moment", or "hold on", stop at once,
even mid-sentence. Do not reply and do not apologise. Hand off to observer.
Memo: "Asked to be quiet."

## CLOSING WORDS
If someone says "thanks", "got it", or "that's all", say "Understood." and
hand off to observer. No goodbye and no further offer.
```

If `end_session` is not registered, add: "You cannot leave the meeting. If
asked to leave, say 'Understood. The host can remove me from the participant
list.' and hand off to observer." If it is registered, follow the
response-gated close rule in
[authoring.md](../scenario/authoring.md#closure-and-re-engagement) instead.

### Observer instructions

```text
## GOAL
Listen to the meeting in silence. Hand back to the supervisor only when Maya
is called by name and asked for something.

## HAND BACK when one of these is clearly true
- Someone says "Maya" (or "Maia", "Hey Maya") followed by a question, a
  request, or an invitation to speak: "Maya, go ahead", "Maya, what do you
  think?", "Maya, can it export to PDF?"
- Right after Maya answered, someone asks a follow-up clearly meant for Maya.

## STAY SILENT (do not hand back)
- People talking to each other, acknowledgements, noise, thinking aloud.
- Maya is mentioned but not asked: "Maya is an AI", "we set Maya up last week".
- The name alone with nothing after it ("Maya..."). Wait for the next
  sentence; hand back if it is a question or request.
- A quiet word, even with the name: "Maya, be quiet." Stay silent.
- When unsure, stay silent.

## MEMO WHEN HANDING BACK
- Who asked and the question in their words.
- Whether Maya has introduced itself yet.
- One sentence on what the meeting was discussing.
```

The examples of what is and is not a call do most of the work. Write them in
the meeting's language, using phrases the team would say.

### `common_instructions`

```text
## IDENTITY
Who the agent is, its name, the language and register it speaks.

## SPEAKING RULE (highest priority)
Silent by default. Speak only when called by name and asked for something.
Say nothing in response to side conversations, mentions, acknowledgements,
or noise.

## FLOW MAP
- supervisor (Maya, speaks): answers when called, then hands off to observer.
- observer (silent): listens; hands back only when Maya is called and asked.
Switch silently. Never tell the participants about the switch.

## FACTS
What the agent may answer from. State what it must not guess, and the one
line it says instead.

## STYLE
Short spoken sentences. No lists, links, or emoji.
```

### Settings that matter

| Setting                                                  | Value                                                                       | Why                                                                                                                                                                                            |
| -------------------------------------------------------- | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type`                                                   | `super`                                                                     | Only a `super` Scenario has flows, and only a flow can be standby.                                                                                                                             |
| Observer `ai_model`                                      | `{ "provider": "standby", "model": "standby" }` exactly                     | Any other value makes the observer speak.                                                                                                                                                      |
| `ai_model_config`                                        | Landmass realtime; add `tts_provider` and `tts_voice_id` for a chosen voice | `super` requires Landmass. This pattern is tested on Landmass realtime, with and without an external voice; start there, not on `cascade`.                                                     |
| `strategy.skip_auto_start`                               | `true`                                                                      | The agent must not greet the meeting when it joins.                                                                                                                                            |
| `strategy.silence`                                       | long threshold, `end_session: false`, `force_agent_to_speak: false`         | The default ends the session after two minutes without speech. A meeting has long quiet stretches.                                                                                             |
| `strategy.max_duration_seconds`                          | the longest meeting expected                                                | The session ends at the cap.                                                                                                                                                                   |
| `end_session`                                            | not registered by default                                                   | The agent stays until the host removes it. Register it only if the user wants it to leave on request, and never tell the agent to call a tool that is not registered: it loops on its goodbye. |
| `appearance.language_code`, `transcribe_config.language` | the meeting's language                                                      | Set both for a non-English meeting ([model-selection.md](../scenario/model-selection.md)).                                                                                                     |
| `ai_model_config.stt_keyterms`                           | the wake name and product terms                                             | Helps the transcript spell the name.                                                                                                                                                           |

Minutes are spent for the whole time the agent is in the meeting, silent or
not. Say so when the user plans to leave it in long meetings.

### Send it into a meeting

`ttai:schedule_bot` with `kind: "meeting"` and the meeting link, after the
user confirms the exact link. A private Scenario needs
`session_credentials: "bot_uc"`. Send one bot per meeting.

### Test before hand-off

Run it once on the web link, then once in a real meeting with two people.

1. Join and say hello to each other. Expected: silence.
2. Say the name and a question. Expected: the introduction, then the answer.
3. Talk to each other again. Expected: silence.
4. Say the name and "be quiet" while it is speaking. Expected: it stops and
   says nothing.
5. Mention the name without asking anything. Expected: silence.

Read the session with `ttai:get_resource(sessions, id, include_fields:
["transcript"])`, and have the user confirm what they heard: silence is judged
on the call, not on the transcript alone.

### Limits to state plainly

- Whether a sentence was addressed to the agent is a model judgment. Expect
  an occasional missed or false wake in a noisy room or with a misheard name.
- There is no chime or visual cue when it wakes. It answers a moment after
  the question.
- Each new question needs the name, unless it directly follows an answer.
- Without `end_session`, only the host can remove it.

### Participant checklist

- [ ] `type` is `super`, one stage, one flow named `observer` with the exact
      standby `ai_model`.
- [ ] Supervisor: hands off silently at start, one action per hand-back,
      hands off after every answer, quiet words stop it without a reply.
- [ ] Observer: concrete "hand back" and "stay silent" examples in the
      meeting's language, including name-only and name-plus-quiet-word.
- [ ] Facts and the speaking rule are in `common_instructions`.
- [ ] `skip_auto_start`, `silence`, and `max_duration_seconds` set as above.
- [ ] No instruction refers to a tool that is not registered.
- [ ] Tested with the five steps, in a real meeting before hand-off.

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

Do not use a generic sales or coaching rubric for a meeting. When you write a
`rubrik`, send `"session_analysis": { "is_auto_analysis": true }` in the same
call, or the meeting is never evaluated.

## Tools, closure, and quality checklist

- Enable a card or visual tool only if it provides an agenda, decision log, or
  synthesis the participants can use.
- Register `end_session` when the meeting should close, then follow the shared
  response-gated close rule.
- Keep direct AI responses to one or two sentences unless the synthesis calls
  for a longer structured summary.

- [ ] Goal, agenda, timebox, AI mode, and invocation rule are explicit.
- [ ] Observation mode stays quiet during productive human discussion.
- [ ] Nudge triggers, channel, cadence, and return-to-observation behavior are
      explicit.
- [ ] Synthesis separates decisions, owners, disagreements, and unknowns.
- [ ] Rubric evaluates the stated target only, and is sent with
      `session_analysis.is_auto_analysis: true`.

## Key Files

- [../scenario/authoring.md](../scenario/authoring.md) — situation-neutral
  design and closure rules
- [../scenario/model-selection.md](../scenario/model-selection.md) — model stamp
- [../scenario/control.md](../scenario/control.md) — tools and runtime control
- [../scenario/ai-instructions.md](../scenario/ai-instructions.md) — prompt
  structure
- [../scenario/rubric.md](../scenario/rubric.md) — evaluation contract
