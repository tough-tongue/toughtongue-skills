# Processed rubrik

Structured evaluation criteria derived from the scenario’s raw
[`rubrik`](rubrik.md). Used when Tough Tongue AI scores a **session**.

## What it is

| Idea      | Meaning                                              |
| --------- | ---------------------------------------------------- |
| Source    | Free-form `rubrik` (and related authoring)           |
| Result    | Weighted criteria with standalone evaluation prompts |
| Consumers | Post-session analysis / report cards                 |

Creators can edit processed criteria in Scenario Studio. A user edit is
treated as intentional — automatic reprocessing should not silently wipe it.

## MCP today

Public MCP focuses on scenario create/update of author fields and on
reading session analyses. Do **not** invent a `processed_rubrik` payload
unless the live tool schema includes it. Prefer:

1. Set clear `rubrik` text when creating/updating.
2. Run sessions; read scores via `ttai:get_session` /
   `ttai:list_sessions` / `ttai:get_analytics`.
3. Point power users at Scenario Studio for fine-grained criteria edits.

## Related

- Session results channels → [../scenario-engine.md](../scenario-engine.md)
- Raw rubrik guidance → [rubrik.md](rubrik.md)

## Key Files

- [rubrik.md](rubrik.md) — author-owned evaluation intent
- [ai-instructions.md](ai-instructions.md) — scenario outcomes
- [../scenario-engine.md](../scenario-engine.md) — read session results
