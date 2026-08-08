---
name: browser-demo-builder
description: >-
  Record deterministic, pre-recorded browser demo steps for a Tough Tongue AI
  scenario via the ttai MCP server. Interviews the demo flow, harvests stable
  selectors from the product's pages, writes tools_config.tools.browser
  tool_settings.steps with ttai:update_scenario, and sets up persistent login
  with ttai:authenticate_browser. Use when the user says "record browser demo
  steps", "make my demo deterministic", "pre-record a demo flow", "the demo
  clicks the wrong thing", or "add browser steps to my scenario".
---

# Browser Demo Builder

Interview → walk the flow → harvest selectors → write `steps` via
`ttai:update_scenario` → verify with a live session. One run produces a
deterministic demo flow the voice agent replays instantly, with zero AI
processing per click.

**How steps execute** (context you must respect): each recorded action
replays through the replay engine using its exact selector — no AI calls. If
a selector breaks, ONE AI-driven retry is attempted using the action's
`description` + `method` + `arguments`. A step aborts at the first action
that fails both, and the agent is told which action broke. So: selectors
must be stable, and every description must work as a standalone retry
prompt.

Reference files (load on demand):

- [references/steps-format.md](references/steps-format.md) — field guide,
  JSON payload, `ai_instructions` templates
- [references/selector-guide.md](references/selector-guide.md) — selector
  rules, forbidden XPath patterns, verification snippets
- [references/ttai-library-demo-example.md](references/ttai-library-demo-example.md)
  — complete worked example

## Prerequisites

- The **ttai** MCP server must be connected. Tool references below use the
  `ttai:` server prefix (e.g. `ttai:update_scenario`); some agents surface
  these as `mcp__ttai__update_scenario`. If the tools are missing, point the
  user at the repo README to install the plugin or add the MCP server — the
  client runs a browser OAuth login on first use.
- A way to inspect the demo app's pages: your agent's browser automation if
  available, or the user's own browser DevTools console using the snippets
  in [references/selector-guide.md](references/selector-guide.md).

## Workflow

### Phase 1 — Interview

Ask the user (one round; skip anything already stated):

1. **Target app** — the product URL the demo walks through.
2. **Login** — does the demo need an authenticated session? If yes, plan the
   auth setup in Phase 4. Sandbox/demo credentials only if any values get
   typed by steps — they are stored in plaintext scenario config.
3. **The flow, screen by screen** — have the user narrate the demo exactly
   as a salesperson would click through it. Each screen transition is a
   candidate step boundary.
4. **Demo data** — names, amounts, dates. Push hard to FIX every value and
   bake it into the actions. `%placeholders%` (substituted from the step
   command's `variables` param) add a failure mode — use them only when a
   value genuinely varies per session.
5. **Simplify** — flag anything state-dependent or flaky (date pickers,
   auto-assigned resources, optional fields) and propose dropping it. A
   shorter deterministic demo beats a longer brittle one.
6. **Target scenario** — an existing scenario (find the ID via
   `ttai:list_scenarios`) or a new one via `ttai:create_scenario` with the
   browser tool enabled. Also decide where capture milestones belong
   (max ~3 per demo).

Call `ttai:list_organizations` first; pass `org_id` on subsequent calls if
the scenario belongs to an organization.

### Phase 2 — Walk & harvest

Walk the demo flow one screen at a time, in the SAME auth state the demo
will see (a logged-in context shows different screens than a fresh visit —
if both can occur, record a step for each and add fallback instructions in
Phase 3).

Per interaction:

1. Find the target element on the current screen.
2. Derive a selector BEFORE clicking. Preference order: stable `#id` >
   `aria-label` > text-anchored structural XPath. Full rules and console
   snippets: [references/selector-guide.md](references/selector-guide.md).
3. Verify the selector resolves **uniquely on this screen** and stays
   within the supported XPath subset (the selector guide lists three
   forbidden patterns that silently match the wrong element).
4. Perform the interaction to advance to the next screen.
5. Record the action: `description` (element name a human would recognize —
   it doubles as the retry prompt), `method` (`click` / `fill` /
   `scrollTo`), `selector` (`xpath=…`), `arguments` (fill value, else `[]`).
6. Note per-screen quirks: popups to dismiss, submit buttons disabled until
   a field is filled, dropdowns whose options load only after a prior pick.

Group recorded actions into steps at screen/intent boundaries — `login`,
`open_pricing_page`, `fill_signup_form` — each nameable in one phrase.

### Phase 3 — Write the scenario

Push the config with `ttai:update_scenario` (send only `id` plus the fields
you changed — partial updates are supported):

- `tools_config.tools.browser.tool_settings`: `initialUrl`, optional
  `contextId` (from Phase 4), and `steps` — exact shape and a full payload
  template in [references/steps-format.md](references/steps-format.md).
- `ai_instructions` additions:
  - A **BROWSER TOOL INSTRUCTIONS** block — commands and speed rules
    (`step` is primary; `capture` only at milestones; keep narrating while
    steps run).
  - **Flow phases** — one `step` call per phase with narration lines,
    capture milestones (≤3), and recovery guidance: on step failure →
    `capture` → `act`.
  - **Fallback steps** for state-dependent screens, wired to the
    step-failure notification ("if `login` fails at the account-tile
    action, run `login_first_time`").

### Phase 4 — Authenticate (login demos only)

If the demo needs a logged-in session:

1. Ensure the scenario exists and has the browser tool enabled.
2. Call `ttai:authenticate_browser` with the `scenario_id` (and optional
   `initial_url`). It returns an `embed_url`.
3. Have the user open the `embed_url` once and log in. The authenticated
   state persists for all future demo sessions of that scenario.
4. Harvest Phase 2 selectors in this same logged-in state.

### Phase 5 — Verify

1. Re-fetch with `ttai:get_scenario` and confirm the steps landed intact.
2. Have the creator run one live demo session end-to-end.
3. If a step fails, the agent's failure notification names the broken
   action — re-harvest that one selector on its screen and re-push with
   `ttai:update_scenario`.

## Checklist (before handing back)

- [ ] Every selector: unique on its screen, within the supported XPath
      subset, no build-generated class hashes, no positional-only paths
- [ ] Every action description works as a standalone retry prompt
- [ ] Demo data baked in; `%placeholders%` only where truly session-variable
- [ ] Capture milestones ≤3; narration lines cover step execution time
- [ ] Fallback steps + instructions for state-dependent screens
- [ ] Login state configured via `ttai:authenticate_browser` if needed
- [ ] Creator ran one live session end-to-end successfully
