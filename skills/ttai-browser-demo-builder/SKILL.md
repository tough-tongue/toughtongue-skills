---
name: ttai-browser-demo-builder
description: >-
  Scripts deterministic browser demo steps
  (tools_config.tools.browser.tool_settings.steps) on an existing Tough
  Tongue AI Scenario, so its voice agent clicks exact XPath selectors instead
  of improvising. Signs the demo browser in with ttai:authenticate_browser,
  converts a pasted browser recording or a walked-through flow into steps,
  saves them with ttai:update_scenario without losing the saved login, and
  repairs steps that click the wrong element. Use when the user says "record
  browser demo steps", "turn my recording into steps", "make my demo
  deterministic", "the demo clicks the wrong thing", or "log the demo browser
  in". Not for creating the Scenario (ttai-agent) or analyzing sessions
  (ttai-session-analyst).
when_to_use: >-
  Also when a demo session reports a failed or misfiring browser step.
---

# Browser Demo Builder

Log in → capture the flow (recording or walkthrough) → convert to steps →
read-merge-write → verify. Output: named steps the voice agent replays with
`runBrowserCommand` (`command: "step"`), exact selectors first, no AI per click.

## How steps replay (respect this)

- Actions run in order. No wait, no `goto`, no branching inside a step.
- Each action replays its exact `selector` with its `method` and `arguments`.
- An action that fails gets ONE retry: the `description` is used as a prompt to
  rediscover the element. If that fails too, the step aborts and the agent is
  told which action broke.
- XPath targets are scrolled into view and shown with an animated cursor before
  the action. Use XPath for every selector.
- `%name%` placeholders in `selector` or `arguments` are filled from the step
  call's `variables`. A step with placeholders is rejected without them.
- Live demo sessions run at **1280×720** and open at `initialUrl`. Without
  `initialUrl`, no browser opens.
- One browser command runs at a time; a second is rejected until the first
  reports back.

## Hard rules

1. **Never write `tool_settings` partially.** `ttai:update_scenario` replaces
   the browser tool's `tool_settings` object whole. Omitting `contextId` deletes
   the saved login. Every write = read → merge → write the complete object.
   Details: [references/steps-format.md](references/steps-format.md).
2. **Never write `contextId` yourself.** `ttai:authenticate_browser` creates it
   and stores it on the Scenario. Carry the existing value through unchanged.
3. **No credentials anywhere.** Not in steps, descriptions, arguments, or
   `ai_instructions`. Scenario config is plaintext. Sign-in goes through the
   saved login only.
4. **No positional XPath.** Never ship `/html[1]/body[1]/div[3]/…` (the
   recorder's fallback). Rewrite to text-, label-, or role-anchored XPath:
   [references/selector-guide.md](references/selector-guide.md).
5. **Recordings are evidence, not steps.** Never paste recorder output into
   `steps` unconverted.

## Phase 0 — Scope and target

- Workspace: reuse a verified org; otherwise call `ttai:list_organizations` and
  pass `org_id` on every call for an organization-owned Scenario. Omit `org_id`
  for personal Scenarios. Never pass a slug.
- Scenario: reuse a known ID; otherwise `ttai:list_scenarios(search=...)`. A
  title is not an ID.
- Read it: `ttai:v3_get_scenario_version(scenario_id)`. Confirm
  `tools_config.tools.browser.should_register` is `true` and note `initialUrl`,
  `contextId`, and existing `steps`.
- No Scenario yet, or the browser tool is off? Hand off to the `ttai-agent`
  skill to create or configure it, then return here.

## Phase 1 — Log in (login demos only)

1. Call
   `ttai:authenticate_browser(scenario_id, initial_url?, viewport, enable_recording_tool)`.
   Needs EDIT access to the Scenario.
   - `initial_url` defaults to the saved `initialUrl`.
   - `viewport`: `"compact"` (1280×800) is closest to the live 1280×720; the
     default `"wide"` (1920×1080) can render a different responsive layout than
     the demo will see.
   - `enable_recording_tool: true` adds a **Workflows** tab for recording.
2. Hand the user the returned `embed_url`. The link expires in 20 minutes
   (`expires_at`); the live browser it opens closes after about 7 minutes, so
   open it when ready.
3. User signs in, then clicks **Save & close session**. That click saves the
   login. Closing the tab without it saves nothing.
4. Re-read the Scenario and confirm `tool_settings.contextId` exists. The call
   creates `contextId` before anyone signs in, so its presence does not prove a
   saved login: confirm the user clicked **Save & close session**.

## Phase 2 — Capture the flow

**Recording-first (preferred).** Sources:

- The **Workflows** tab in the `authenticate_browser` embed.
- The browser tool configuration in the Tough Tongue AI dashboard (start
  recording → stop → review; can also be sent to Edit with AI as a Markdown
  note).

No MCP tool reads recordings. Ask the user to copy the review's **View as
document** Markdown (or the event list) and paste it. In the embed, recorded
events are discarded on Close — copy first.

**Hand-authored (alternative).** Interview, then walk the flow and harvest
selectors with your browser automation or the user's DevTools. Interview in one
round, skipping what is known:

1. Target URL and whether login is needed (→ Phase 1).
2. The flow screen by screen, as a salesperson would click it.
3. Demo data: fix every value; use `%placeholders%` only for per-session values.
4. Drop flaky parts: date pickers, auto-assigned records, optional panels.

Harvest in the same logged-in state the demo sees, at a 1280-wide window.

## Phase 3 — Convert to steps

Follow the playbook:
[references/recording-to-steps.md](references/recording-to-steps.md). Core
moves:

- Merge consecutive `input` events on one field into one `fill` (last value).
- `keydown` Enter → `press` with `["Enter"]` right after the field's `fill`.
- `navigation` → step boundary (steps cannot navigate; the agent uses `goto`).
- Drop `scroll`, `note`, `wait`, `dom-snapshot`; drop `scrollTo` unless
  scrolling a panel's own scrollbar.
- Rewrite every selector from the candidates into stable XPath.

Step design:

- Short steps (typically 3–6 actions), each ending at a screen change.
- Name steps by intent: `open_reports`, `filter_by_status`.
- Each `description` names its element standalone (it is the retry prompt).
- Wire `ai_instructions`: a browser-tool block, one phase per step, ≤3 `capture`
  milestones, fallback steps for state-dependent screens. Templates in
  [references/steps-format.md](references/steps-format.md).

## Phase 4 — Read, merge, write

1. Re-read with `ttai:v3_get_scenario_version` right before writing (the login
   may have changed since Phase 0).
2. Copy `tool_settings` whole. Add or replace only your step keys. Keep
   `contextId`, `initialUrl`, and unrelated steps byte-for-byte.
3. `ttai:update_scenario` with `id`, `tools_config.tools.browser.tool_settings`
   (complete), and `ai_instructions` if changed.

## Phase 5 — Verify

1. Re-read. Confirm `contextId` unchanged and every step key present.
2. Ask the user to run one test session end-to-end.
3. A failed step names its broken action: fix that one selector, then repeat
   Phase 4. For selector bugs (e.g. "always clicks the first card"), start at
   the forbidden patterns in the selector guide.

## Checklist

- [ ] `contextId` identical before and after every write
- [ ] Every selector is `xpath=…`, unique on its screen, non-positional, no
      hashed classes
- [ ] No step crosses a navigation; typically 3–6 actions each
- [ ] Descriptions static and standalone; no credentials anywhere
- [ ] `%placeholders%` only where truly per-session, and listed in instructions
- [ ] ≤3 capture milestones; fallback steps wired for variant screens
- [ ] User ran one live test session successfully

## Key Files

- [references/steps-format.md](references/steps-format.md) — schema, methods,
  read-merge-write payload, `ai_instructions` templates
- [references/selector-guide.md](references/selector-guide.md) — XPath subset,
  forbidden patterns, DevTools checks
- [references/recording-to-steps.md](references/recording-to-steps.md) —
  event-to-action mapping with a full conversion
- [references/worked-example.md](references/worked-example.md) — a complete
  four-step demo with instructions
