# Steps Format — Schema, Methods, Payload, Instructions

## Contents

- Schema
- Methods
- Read-merge-write (the only safe write)
- Placeholders
- BROWSER TOOL INSTRUCTIONS block
- Flow phase template
- Fallback steps

---

## Schema

Location: `tools_config.tools.browser.tool_settings`.

```json
{
  "initialUrl": "https://app.example.com/",
  "contextId": "<kept from the read — never invented>",
  "steps": {
    "<step_name>": {
      "description": "One sentence: the screen transition this step performs",
      "actions": [
        {
          "description": "Standalone name of the target element",
          "method": "click",
          "selector": "xpath=//button[@aria-label='Open reports']",
          "arguments": []
        }
      ]
    }
  }
}
```

- Step name (the key): what the agent passes as `name` in `command: "step"`.
  Short, stable, intent-named.
- Action `description`: the self-heal prompt when the selector fails. Must
  identify the element with no other context. Static — no placeholders.
- `arguments`: always an array of strings, `[]` when unused.
- The server does not validate steps. A typo in `method` or a malformed selector
  is only discovered at replay.
- The Scenario editor shows `click`, `fill`, `scrollTo` in its method menu but
  keeps any other valid method you write.

## Methods

Each method with its `arguments`:

- `click` `[]` — buttons, tabs, cards, links, menu items.
- `fill` `["text"]` — clears, then types into an input; fires input events.
- `type` `["text"]` — types without clearing.
- `press` `["Enter"]` — key press on the focused element; put it right after a
  fill.
- `selectOption` `["Option label"]` — native `<select>` only; custom dropdowns =
  two clicks.
- `hover` `[]` — reveal hover menus.
- `doubleClick` `[]` — double-click targets.
- `dragAndDrop` `["xpath=<target>"]` — drag selector onto the target XPath.
- `scrollIntoView` `[]` — bring an element on screen as a step's visible
  "scroll".
- `scrollTo` `["50%"]` — scroll an element's OWN scrollbar to a percentage.
- `nextChunk` / `prevChunk` `[]` — scroll an element by its own height (`/html`:
  one viewport).

`scrollTo` is not scroll-into-view. On a normal element it scrolls that
element's internal scrollbar (often a no-op). On `xpath=/html` it scrolls the
page to the percentage. Replay already scrolls XPath targets into view before
each action, so a `scrollTo` before a click is dead weight — delete it.

## Read-merge-write (the only safe write)

`ttai:update_scenario` merges per tool, shallowly: fields you send on
`tools.browser` replace the stored ones, and `tool_settings` is replaced as a
whole object. Sending `{"steps": {...}}` alone deletes `contextId` (the saved
login) and `initialUrl`.

1. Read: `ttai:v3_get_scenario_version(scenario_id)` (pass `org_id` for org
   Scenarios). Take `tools_config.tools.browser.tool_settings` verbatim.
2. Merge in memory: set or replace only your step keys under `steps`. Keep every
   other key and step unchanged.
3. Write the complete object:

```json
{
  "scenario_data": {
    "id": "<SCENARIO_ID>",
    "tools_config": {
      "tools": {
        "browser": {
          "tool_settings": {
            "initialUrl": "<from the read>",
            "contextId": "<from the read, unchanged>",
            "steps": {
              "<existing_step>": { "...": "unchanged from the read" },
              "<new_step>": { "description": "...", "actions": [] }
            }
          }
        }
      }
    }
  }
}
```

4. Re-read and diff: `contextId` identical, all step keys present.

`should_register` and `add_to_system_prompt` survive when omitted; only the keys
you send on `tools.browser` are replaced.

## Placeholders

- `%name%` in `selector` or `arguments` is replaced from the step call's
  `variables` (a JSON object string, e.g. `{"customer": "Acme"}`).
- A step whose placeholders are not all supplied is rejected before running.
- Each placeholder is a value the voice agent must produce mid-demo. Prefer
  fixed demo data. List every placeholder and its source in the instructions.

## BROWSER TOOL INSTRUCTIONS block

Add to `ai_instructions`:

```text
## BROWSER TOOL INSTRUCTIONS
The browser is pre-opened at [INITIAL_URL]. Use `runBrowserCommand`:
- `command: "step"` + `name` — runs a pre-recorded batch. PRIMARY command.
- `command: "goto"` + `url` — direct navigation (use between steps).
- `command: "capture"` — screenshot (EXPENSIVE — marked milestones only).
- `command: "observe"` / `command: "act"` — off-script moments only.

Speed rules:
- Always use `step` for the scripted flow. Never rebuild it with act.
- Steps run in the background — keep narrating while they execute.
- One browser command at a time: wait for the completion message before the
  next, or it is rejected.
- If a step FAILS you are told which action broke: `capture`, then recover
  with `act`.
```

## Flow phase template

```text
### Phase N: <Name>
1. Say: "<one narration line>"
2. `goto` "<url>"            (only if this phase starts on a new page)
3. `step` name: "<step_name>"
   - If it fails at "<first action description>": run step "<fallback>".
4. Narrate while it runs: "<what appears on screen>"
   — or — `capture` — MILESTONE N: <what to confirm>.
```

## Fallback steps

When one screen has two states (a cookie banner, a dismissible tour, an optional
panel), record a step per state and route on the failure message:

```text
3. `step` name: "open_reports"
   - If it fails at "Dismiss button on the product tour popup", run step
     "open_reports_no_tour" instead.
```

A login form appearing means the saved login expired. Stop and re-run browser
login; never add credential-typing fallback steps.
