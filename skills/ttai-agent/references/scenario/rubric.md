# Rubric

How a session is evaluated. The API spells the fields `rubrik` (free-form text
you write) and `processed_rubrik` (structured criteria derived from it).

## Contents

- `rubrik`
- `processed_rubrik`
- Reading scores
- Agent rules

## `rubrik`

Author-facing text describing what "good" looks like and what the report must
contain. Recipes carry a starting pattern per situation (lead screening, rep
scorecard, demo intelligence, candidate evidence).

Declare the subject first: the human participant, the AI's counterpart, or
business facts extracted from the conversation. In multi-party sessions also set
`session_analysis.evaluation_target` (for example `"Sales Rep"`).

## `processed_rubrik`

Weighted criteria the evaluation pipeline scores against:

| Field                     | Meaning                                               |
| ------------------------- | ----------------------------------------------------- |
| `criteria[]`              | `topic`, `weight` (0–100), `prompt`, `requires_video` |
| `report_instructions`     | How to write the overall report                       |
| `next_steps_instructions` | How to write next steps                               |

The platform derives it from `rubrik`, and people refine it in the web app; a
manual edit is treated as intentional. Prefer writing clear `rubrik` text. Send
`processed_rubrik` only when the user asks for exact criteria, and then send the
whole object: it is replaced, not merged.

## Reading scores

Analysis runs only when `session_analysis.is_auto_analysis` is true.

- Compact: `ttai:list_resources(sessions)` with `include_fields: ["evaluation"]`.
- Full detail: `ttai:get_resource(sessions)` (`evaluation_results.final_score`,
  `evaluation_results.report_card`).
- Aggregates: `ttai:get_workspace_info(sections: [usage])`.
- A rubric change applies to sessions scored afterwards. Past scored sessions
  keep their results: `ttai:analyze_session` only fills missing results and
  never re-scores. Test a new rubric with a fresh session.

## Agent rules

1. Name who is evaluated; keep it consistent with ROLE, FLOW outcomes, and
   `user_instructions`.
2. Score observable stages, not vague traits. Weight the moments that matter.
3. Keep learning feedback (how the human did) separate from business
   intelligence (what happened).
4. Never claim criteria were reprocessed unless a tool result shows it.

## Key Files

- [authoring.md](authoring.md) — evaluation principles
- [control.md](control.md) — `session_analysis` settings
- [ai-instructions.md](ai-instructions.md) — FLOW outcomes the rubric mirrors
