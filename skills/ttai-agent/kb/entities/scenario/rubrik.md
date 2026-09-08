# Rubrik

Free-form evaluation text on a scenario (`rubrik`). Creators use it to
describe what “good” looks like after a session.

## What it is

- Author-facing prose (or light structure) describing scoring dimensions.
- Situation recipes often include a target rubrik pattern — start from
  [../../scenario-recipes/](../../scenario-recipes/index.md).
- Not the same as **processed rubrik** (structured criteria used by the
  evaluation pipeline) — see [processed-rubrik.md](processed-rubrik.md).

## Agent rules

1. Declare who the rubrik evaluates: the human participant, the AI's
   counterpart, or the conversation's extracted business intelligence.
2. Keep rubrik aligned with FLOW outcomes in `ai_instructions`.
3. Prefer concrete, observable behaviors over vague traits.
4. When MCP schemas expose `rubrik`, send string content only — do not
   invent nested processed shapes unless the tool schema says so.
5. After material rubrik edits in Studio, processed criteria may refresh;
   do not claim you reprocessed unless a tool confirms it.

## Key Files

- [../../scenario-authoring.md](../../scenario-authoring.md) — evaluation principles
- [ai-instructions.md](ai-instructions.md) — FLOW alignment
- [processed-rubrik.md](processed-rubrik.md) — structured criteria
- [../../scenario-recipes/index.md](../../scenario-recipes/index.md) — situation patterns
