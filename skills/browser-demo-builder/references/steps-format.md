# Steps Format — Field Guide, Payload, and Instruction Templates

The contract for pre-recorded browser steps: what each field means (same
fields in the scenario editor UI and the JSON payload), the full
`ttai:update_scenario` payload template, and the `ai_instructions` blocks
that make the voice agent use the steps well.

## Contents

- Steps vs actions — the mental model
- Field guide (creator-facing)
- Method + arguments recipes
- Good vs bad descriptions
- Full `tools_config` payload template
- BROWSER TOOL INSTRUCTIONS block for `ai_instructions`
- Flow phase template
- Fallback steps for state-dependent screens

---

## Steps vs actions — the mental model

A **step** is a chapter of the demo: a named batch the voice agent runs
with one `runBrowserCommand` call (`command: "step"`, `name: "<step>"`). An
**action** is one concrete browser move inside that chapter — a click, a
fill, or a scroll.

> Description is the **what**, method is the **how**, selector is the
> **where**, arguments are the **with-what**.

The scenario editor UI (Steps → Actions → Arguments) and the JSON payload
below are the same contract — a step edited in the UI and a step pushed via
`ttai:update_scenario` are interchangeable.

## Field guide (creator-facing)

Per step:

| Field | What it is | Guidance |
|---|---|---|
| step name (the key) | Machine id the agent calls | Short, stable, one phrase: `login`, `open_course`, `fill_signup_form` |
| `description` | Intent of the whole batch | One sentence for the screen transition, not the individual clicks |

Per action:

| Field | What it is | Guidance |
|---|---|---|
| `description` | Human name of the target element | Name the thing a person would point at. If the selector breaks, the system retries ONCE using this text as the prompt — it must stand alone |
| `method` | The verb | `click`, `fill`, or `scrollTo` |
| `selector` | Exact element address | `xpath=…`, unique on the screen it runs on. Rules: [selector-guide.md](selector-guide.md) |
| `arguments` | Payload for the method | Array of strings; see recipes below |

## Method + arguments recipes

| Method | `arguments` | Example |
|---|---|---|
| `click` | `[]` | Click a button, tab, card, or menu item |
| `fill` | `["text to type"]` | Type into an input; dispatches real input events (enables submit buttons) |
| `scrollTo` | `["0%"]` | Scroll the element into view (centered); `"0%"` is the standard value |

Order fill-then-click when a submit button starts disabled — the fill's
input events enable it.

## Good vs bad descriptions

The description is the retry prompt when a selector breaks. It must
identify the element with no other context.

| Good | Bad | Why |
|---|---|---|
| "Analyze button on the first completed session row" | "Analyze button" | Which one? Rows repeat |
| "Password field on the login form" | "the input" | Not identifiable |
| "Sales Coaching: Varied Collection course card" | "the card we want" | No standalone meaning |

## Full `tools_config` payload template

Push with `ttai:update_scenario` — send only `id` plus the fields you
changed:

```json
{
  "scenario_data": {
    "id": "[SCENARIO_ID]",
    "tools_config": {
      "tools": {
        "browser": {
          "should_register": true,
          "add_to_system_prompt": true,
          "tool_settings": {
            "initialUrl": "[PRODUCT_URL]",
            "contextId": "[BROWSER_CONTEXT_ID]",
            "steps": {
              "login": {
                "description": "Log in with the demo account: fill email and password, submit",
                "actions": [
                  {
                    "description": "Email field on the login form",
                    "method": "fill",
                    "selector": "xpath=//input[@type='email']",
                    "arguments": ["demo@example.com"]
                  },
                  {
                    "description": "Password field on the login form",
                    "method": "fill",
                    "selector": "xpath=//input[@type='password']",
                    "arguments": ["[DEMO_PASSWORD]"]
                  },
                  {
                    "description": "Sign In button on the login form",
                    "method": "click",
                    "selector": "xpath=//button[@type='submit']",
                    "arguments": []
                  }
                ]
              }
            }
          }
        }
      }
    }
  }
}
```

Notes:

- `contextId` is the persistent authenticated browser context. Get it by
  running `ttai:authenticate_browser` for the scenario and having the user
  log in once via the returned `embed_url`. Omit the field for demos that
  need no login.
- Anything in `arguments` is stored in plaintext scenario config — sandbox
  or demo credentials only, never real accounts.
- `%placeholders%` inside arguments are substituted from the `variables`
  param of the step command at runtime. Use sparingly — every placeholder
  is something the voice agent must supply correctly mid-demo.

## BROWSER TOOL INSTRUCTIONS block for `ai_instructions`

```text
## BROWSER TOOL INSTRUCTIONS
The browser is pre-opened at [PRODUCT_URL]. Use `runBrowserCommand`:
- `command: "step"` + `name` — runs a PRE-RECORDED batch of actions INSTANTLY
  (zero AI processing). This is your PRIMARY command.
- `command: "capture"` — screenshot to SEE the page (EXPENSIVE — milestones only)
- `command: "goto"` + `url` — direct navigation
- `command: "observe"` / `command: "act"` — ONLY for off-script moments

**Speed rules (IMPORTANT):**
- ALWAYS use `step` for the scripted flow. Never rebuild a scripted action with observe/act.
- `capture` ONLY at the milestones marked below (max 3).
- Steps run in the background — keep talking while they execute.
- If a step FAILS, you'll be told which action broke. Then: `capture` to see
  the page, and recover with `act`.
```

## Flow phase template (one per step)

```text
### Phase N: <Name>
1. Explain: "<one narration line — why this step matters>"
2. `step` — name: "<step_name>"
   - If it FAILS at "<first action>", <fallback: run step "<alt_step>" / recover with act>.
3. Narrate while it runs (no capture): "<what's happening on screen>"
   — or — `capture` — **MILESTONE N**: <what to confirm and describe>.
```

## Fallback steps for state-dependent screens

When a screen can appear in two states (e.g. a saved-account tile vs the
full login form), record a step for each state and wire the fallback into
the instructions:

```text
2. `step` — name: "login"
   - If it FAILS at "Saved account tile", the profile is signed out — run
     `step` — name: "login_first_time" instead, then continue.
```

The step-failure notification names the broken action, so the agent can
route to the right fallback without a screenshot.
