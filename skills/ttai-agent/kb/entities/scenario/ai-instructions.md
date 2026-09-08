# scenario.ai_instructions

The agent's system prompt — the most important long-form field. This file is
the **shape**. Situation recipes under
[../../scenario-recipes/](../../scenario-recipes/index.md) fill in the content.
Do not copy a recipe's sections into every scenario.

To create or update this field, use MCP
([../../../features/mcp/](../../../features/mcp/index.md)) —
`create_scenario` / `update_scenario`. In Scenario Studio it is the AI
Instructions field.

## Contents

- What it is
- Required shape
- Fundamental sections (not situation-specific)
- Voice rules that always apply
- Dynamic variables
- What to load next

---

## What it is

`ai_instructions` is the compiled brain of the scenario. The model sees it
every turn. Every extra sentence is a sentence it may ignore — be dense,
structured, and free of unresolved placeholders (except intentional
`{{ dynamic_vars }}`).

Write enough to specify the role, reality model, flow, tools, boundaries, and
evaluation target. Do not use a word-count target: concise, unambiguous rules
outperform a long prompt with duplicated instructions.

## Required shape

- Use `##` headings. One idea per section. No essay walls.
- Write as **speech**, not as a document — this is spoken aloud.
- **One question per turn.** Never stack questions.
- Name the party's role clearly: who the AI is, who the human is.
- **Two beats then wait.** Identity / reason / permission → STOP → branch on
  the reply. Do not collapse the opening into one breath.
- **Follow the thread.** If they say something interesting, pursue it — do
  not jump phases robotically.
- **Silence can be the job.** Meeting observers and notetakers stay quiet
  until addressed. Do not fill pauses.
- Every `{{ var }}` has a missing-value fallback in the same file
  ("If `lead_name` is blank, open with …"). Otherwise the agent speaks the
  placeholder.

Do **not** put a quoted opening line anywhere. Runtime greets with
"your opening line" from FLOW when `skip_auto_start` is false. Write Beat 1
as a directive in FLOW ("Start with: … Then STOP and wait."). Quoted
openings are delivered robotically and restart on interruption. There is no
`strategy.welcome_instructions` field — do not send one.

## Fundamental sections

Use this skeleton. Rename or drop a section only when the situation file
says so. Do not add Cascade/STT blocks unless the model is Landmass
`cascade`.

```
## CONTEXT
Who, company, product, why this conversation exists. Real names beat
invented ones. Template variables for per-session facts.

## ROLE
Who the AI plays, register (warm SDR, skeptical buyer, patient coach),
language + locale.

## FLOW
Ordered beats. First spoken turn (if skip_auto_start is false): a
directive — "Start with: … Then STOP and wait." Never quote the opening.
Then branches. What must never be skipped. One question per turn.

## TOOLS
When to call `end_session` (and any other enabled tools). Timing rules —
without them the agent never hangs up or hangs up mid-sentence.

## GUARDRAILS
Never reveal you are an AI unless asked. Never speak tool names. Never
invent facts that contradict CONTEXT.

## STYLE
Spoken cadence. Short sentences. Fragments fine. What "good" sounds like
in one good/bad pair.
```

Situation files add domain sections (outcomes, objections, rubric target)
on top of this skeleton — they do not replace it. Map `SESSION FLOW` /
`YOUR ROLE` to FLOW / ROLE if you see those names in an older scenario.

Before drafting, load [../../scenario-authoring.md](../../scenario-authoring.md)
to define the human outcome, reality model, difficulty, and evidence of
success. The skeleton is the structure; that contract supplies its content.

## Voice rules (every spoken scenario)

- Plain text in the model's **output**. No markdown, bullets, or URLs in
  what will be spoken.
- Spell out numbers and symbols when they will be read aloud.
- Realtime (Galaxy, Ocean, Landmass `gemini`/`openai`) hear audio. Landmass
  Cascade sees **transcripts** — only then load
  [cascade-tts.md](../../scenario-recipes/cascade-tts.md) (STT errors, SSML, beats).

## Dynamic variables

`{{ var_name }}` is filled from the session URL (`?t_company=Acme`).
Document the fallback next to the first use. Prefer `appearance.voice` for
Realtime. Cascade `tts_voice_id` lives in [model-selection.md](model-selection.md).

## Load next

| Need                               | File                                       |
| ---------------------------------- | ------------------------------------------ |
| Model / voice stamp | [model-selection.md](model-selection.md) |
| Strategy, tools, appearance | [control.md](control.md) |
| Cold call / SDR                    | [../../scenario-recipes/cold-call.md](../../scenario-recipes/cold-call.md)               |
| Sales roleplay                     | [../../scenario-recipes/sales-roleplay.md](../../scenario-recipes/sales-roleplay.md)     |
| Coaching                           | [../../scenario-recipes/coaching.md](../../scenario-recipes/coaching.md)                 |
| Demo                               | [../../scenario-recipes/demo.md](../../scenario-recipes/demo.md)                         |
| Interview                          | [../../scenario-recipes/facilitated/interview.md](../../scenario-recipes/facilitated/interview.md) |
| Meeting facilitation               | [../../scenario-recipes/facilitated/meeting-facilitation.md](../../scenario-recipes/facilitated/meeting-facilitation.md) |
| Landmass Cascade speech            | [cascade-tts.md](../../scenario-recipes/cascade-tts.md)           |
| Create / edit / refine a scenario  | **scenario-maker**                             |
| Create / update on the backend     | **ttai-agent features/mcp**                  |

## Key Files

- [../../scenario-authoring.md](../../scenario-authoring.md) — design contract
- [model-selection.md](model-selection.md) — model stamps
- [control.md](control.md) — strategy and tools
- [../../scenario-recipes/index.md](../../scenario-recipes/index.md) — situation patterns
- [../../../features/mcp/connect.md](../../../features/mcp/connect.md) — create/update rules
