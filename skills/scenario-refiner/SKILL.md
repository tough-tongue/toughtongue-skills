---
name: scenario-refiner
description: >
  Surgically refine an existing ToughTongue AI scenario based on real session
  transcripts, low scores, or observed misbehavior — via the ttai MCP server.
  Diagnoses whether the issue lives in ai_instructions, tools_config, or strategy,
  then applies the smallest possible edit with update_scenario. Use when the user
  says "refine the scenario", "fix the scenario", "the agent said X instead of Y",
  "it ended the call too early", "it skipped a step", "make it sound more natural",
  or pastes a transcript/complaint about a live scenario.
---

# Scenario Refiner

Diagnose → plan → surgical edit → verify. Never expand tokens when you can
tighten. The agent runs on a system prompt that is already long — every word
you add is a word the model has to ignore at runtime.

Read `references/runtime-behavior.md` before diagnosing anything about prompt
assembly, tool registration, conductor, or silence mechanics.

## Prerequisites

- The `ttai` MCP server is connected. If not, point the user at the repo README
  and <https://app.toughtongueai.com/developer> for a `TTAI_PAT` token.

## Workflow

### Step 1: Gather evidence

1. Call `list_organizations`; pass `org_id` on subsequent calls if the scenario
   belongs to an organization.
2. Fetch the scenario with `get_scenario` (needs the scenario ID — find it via
   `list_scenarios` if the user only gave a name). Read the **entire**
   `ai_instructions`, plus `strategy`, `tools_config`, and `session_analysis`.
3. Get the failure evidence:
   - If the user pasted a transcript or complaint, use that.
   - Otherwise pull sessions: `list_sessions` filtered by `scenario_id` (and
     date range), pick the relevant ones (e.g. lowest scores), then
     `get_sessions_batch` for details. Fetch `transcript_url` contents for
     the actual conversation text.
4. Identify the exact turn where things went wrong. Cross-reference: what did
   the scenario PRESCRIBE for that moment, and what did the agent actually DO?
   Quote both in your diagnosis.

If the transcript is in another language (Hindi, Hinglish, ...), do not
translate — the scenario is in the same language. Quote the original.

### Step 2: Diagnose root cause

Classify into one of these buckets. Each maps to a different fix location.

| Symptom | Likely cause | Fix location |
|---|---|---|
| Agent called `end_session` too early / too late / not at all | Tool timing instruction weak, or `add_to_system_prompt: false` | `ai_instructions` end-of-call block, or `tools_config.tools.end_session` |
| Agent took the wrong conversation branch | Branch trigger words too narrow, or path priority unclear | `ai_instructions` flow section (strengthen trigger list or add tie-breaker rule) |
| Agent used the wrong closing line | Closing rule not bound to the path it came from | `ai_instructions` closing section (bind closings to paths explicitly) |
| Agent skipped a prescribed step/question | "Read-the-room" rule too aggressive, or step ordering implicit | `ai_instructions` — make step 1 → step 2 a hard sequence with NEVER skip |
| Agent monologued / two questions in one turn | Style rules buried | Bold/cap a single rule in the style section; do not add a new section |
| Agent revealed AI identity or said a tool name aloud | Guardrails section missing or weak | `ai_instructions` GUARDRAILS / THINGS YOU MUST NEVER DO |
| Agent stayed silent too long, then ended the call | `silence.silence_threshold` too low, or `silence.end_session: true` | `strategy.silence` |
| Wrap-up fired during active conversation | Conductor `time_seconds` too low or `end_turn: true` | `strategy.conductor.messages` |
| Wrong voice / language / accent | Voice or language config | `appearance.voice`, `appearance.language_code`, `transcribe_config` |
| Agent said a placeholder like `{{ firstname }}` literally | Missing-context fallback not specified | `ai_instructions` CONTEXT section — add "If blank, do X" |
| Robotic opening / restarts opening when interrupted | Quoted speech in `welcome_instructions` | `strategy.welcome_instructions` — rewrite in directive form |

If the cause is architectural (template selection, tool registration,
conductor injection), open `references/runtime-behavior.md` and cite the
relevant mechanism. Don't guess.

### Step 3: Plan the edit

Before calling any tool, write the edit out — old text and new text side by
side — and check it against these principles:

**Token discipline (in priority order):**

1. **Replace** existing text > **tighten** existing text > **add** new text
2. If you must add, ask: can I delete something stale to compensate?
3. Flag the user if the total `ai_instructions` delta exceeds +50 tokens (~3 lines)
4. Never add a "while I'm here" change. One issue = one edit.

**Effective prompt writing:**

- **Imperative voice**: "NEVER call end_session before X" not "The agent
  should not call end_session before X"
- **NEVER / ALWAYS / ONLY caps markers** for hard constraints
- **One concrete example** beats three abstract rules
- **Bind rules to triggers**: "When customer says X → do Y" beats "Be empathetic"
- **Forbid the failure mode by name**: if the agent skipped a step, write
  "NEVER skip [step], even if the issue seems minor"

**Where to put a new rule inside `ai_instructions`:**

- Hard prohibition → `## GUARDRAILS` or `## THINGS YOU MUST NEVER DO`
- Conditional behavior → inside the matching phase / path
- Tool timing → right after the prescribed closing lines
- Tone / style → the style rules block near the top

### Step 4: Apply via `update_scenario`

1. Send **only** `id` plus the fields you changed — partial updates are
   supported, and untouched fields must not be re-sent (avoids clobbering
   concurrent edits).
2. For an `ai_instructions` edit: apply your surgical replacement to the full
   fetched string and send the complete updated field. Verify no collateral
   changes (whitespace, adjacent bullets).
3. Pass `org_id` if the scenario belongs to an organization.

### Step 5: Verify and report

1. Re-fetch with `get_scenario` and confirm the change landed as planned.
2. Conclude with exactly this structure:
   - **Diagnosis** (1-2 sentences) — what went wrong and why
   - **Change** — field + before/after summary
   - **Token delta** — estimate (e.g. "+38 tokens, under the 50-token threshold")
   - **Reminder** — the change applies to **new sessions only**; running
     sessions keep their compiled system prompt

## Quick recipes

### "Agent ended the call too early"

1. Search `ai_instructions` for `end_session`. Is there an explicit
   "ONLY after closing line AND customer farewell" rule?
2. If not, add a 3-4 bullet rule block right after the closing templates.
3. Check `tools_config.tools.end_session.tool_settings.disconnectDelaySeconds`
   — too low (< 8) on emotional calls feels abrupt.

### "Closing line was generic instead of path-specific"

Strengthen the closing section with: "Based on the path you actually took
above, pick the matching closing — never substitute a generic 'thank you'."

### "Scenario scores dropped after a change"

Pull sessions before and after the change date with `list_sessions`
(`from_date` / `to_date`), compare `evaluation_results`, and check whether the
prior edit introduced the regression before adding anything new.
