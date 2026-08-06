# Worked Example — Tough Tongue AI Live Demo (Paris)

A complete, real pre-recorded demo: "Paris", an AI SDR that demos the Tough
Tongue AI platform itself by navigating <https://app.toughtongueai.com> live.
Four pre-recorded steps cover the scripted flow; everything else is
narration, `goto`, and three capture milestones.

Use it as the pattern for structure, selector style, and how
`ai_instructions` and `steps` fit together.

## Contents

- The steps payload (`ttai:update_scenario`)
- Why each selector looks the way it does
- The matching `ai_instructions` demo flow
- What was deliberately left out

---

## The steps payload

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
            "initialUrl": "https://app.toughtongueai.com/",
            "contextId": "[BROWSER_CONTEXT_ID]",
            "steps": {
              "library_scroll_coaches": {
                "description": "On the library page: ensure the Featured tab is active, then scroll down to the coaching & interviewing agents row",
                "actions": [
                  {
                    "description": "Featured tab on the library page",
                    "method": "click",
                    "selector": "xpath=//button[@role='tab'][contains(text(),'Featured')]",
                    "arguments": []
                  },
                  {
                    "description": "Google PM Interviewer scenario card in the featured library grid",
                    "method": "scrollTo",
                    "selector": "xpath=//h3[contains(text(),'Google PM Interviewer')]",
                    "arguments": ["0%"]
                  }
                ]
              },
              "library_scroll_tutors": {
                "description": "Scroll the library further down to the tutors & language coaches row",
                "actions": [
                  {
                    "description": "Interactive Python 101 for Data Science scenario card in the featured library grid",
                    "method": "scrollTo",
                    "selector": "xpath=//h3[contains(text(),'Interactive Python 101')]",
                    "arguments": ["0%"]
                  }
                ]
              },
              "open_course": {
                "description": "On the course page: ensure the Featured tab is active, then open the Sales Coaching: Varied Collection course",
                "actions": [
                  {
                    "description": "Featured tab on the course page",
                    "method": "click",
                    "selector": "xpath=//button[@role='tab'][contains(text(),'Featured')]",
                    "arguments": []
                  },
                  {
                    "description": "Sales Coaching: Varied Collection course card",
                    "method": "click",
                    "selector": "xpath=//h3[contains(text(),'Sales Coaching: Varied Collection')]",
                    "arguments": []
                  }
                ]
              },
              "analyze_first_session": {
                "description": "On the sessions page: click Analyze on the most recent completed session (opens its analysis page)",
                "actions": [
                  {
                    "description": "Analyze button on the first completed session row",
                    "method": "click",
                    "selector": "xpath=//tbody/tr[contains(text(),'completed')][1]//button[contains(text(),'Analyze')]",
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

`[BROWSER_CONTEXT_ID]` comes from running `ttai:authenticate_browser` for
the scenario and logging in once via the returned `embed_url` — the demo
navigates authenticated pages (sessions history), so it needs the persisted
login.

## Why each selector looks the way it does

Each one maps to a rule in [selector-guide.md](selector-guide.md):

- `//button[@role='tab'][contains(text(),'Featured')]` — chained bracket
  predicates instead of
  `//button[@role='tab' and normalize-space()='Featured']` (forbidden
  pattern 3: a bare `normalize-space()` inside an `and`-group would drop
  ALL predicates and click the first button on the page).
- `//h3[contains(text(),'Google PM Interviewer')]` — anchors on the
  text-bearing element itself instead of the card wrapper
  (`div[.//h3[...]]` is forbidden pattern 1). For `open_course` the click
  on the `<h3>` bubbles up to the card's handler; for the scrolls the
  heading is the scroll target directly.
- `//tbody/tr[contains(text(),'completed')][1]//button[contains(text(),'Analyze')]`
  — plain chained steps with a step-level `[1]` instead of
  `(//tr[...])[1]//button[...]` (forbidden pattern 2). It also relies on
  the engine's recursive text matching: the "completed" status is a nested
  badge inside the row, and `contains(text(),…)` still matches it.
- Both tab clicks exist because the library and course pages persist the
  tab choice — clicking Featured is a harmless no-op when it's already
  active, and a guard when it isn't. That's determinism over cleverness.

## The matching `ai_instructions` demo flow

Abridged to the browser-relevant parts — the full scenario also has
persona, qualifying questions, and product context sections:

```text
## BROWSER TOOL INSTRUCTIONS
The browser is pre-opened at https://app.toughtongueai.com/. Use `runBrowserCommand`:
- `command: "step"` + `name` — runs a PRE-RECORDED batch of actions INSTANTLY
  (zero AI processing). This is your PRIMARY command for the scripted flow.
- `command: "goto"` + `url` — navigate to a URL (fast, direct)
- `command: "capture"` — take a screenshot to SEE the page visually
- `command: "observe"` / `command: "act"` — ONLY for off-script moments

**Speed rules (IMPORTANT — the demo must feel snappy):**
- ALWAYS use `step` for the scripted flow. Never rebuild a scripted action with observe/act.
- `capture` is EXPENSIVE. Use it ONLY at the 3 milestones marked below.
- Steps run in the background and send you a completion notification —
  keep talking while they execute, narrating from the script.
- If a step FAILS, you'll get a notification saying which action broke.
  Then (and only then): `capture` to see the page, and recover with `act`.

## DEMO FLOW (Follow This Sequence)

### Phase 1: Home Page Introduction
1. `capture` FIRST — **MILESTONE 1**: see the home page before describing it
2. Introduce the platform; preview the sections coming up

### Phase 3: INTERACT — Library & Courses
2. `goto`: "https://app.toughtongueai.com/library"
3. No capture needed — narrate from this script: the top rows show sales &
   SDR agents you can remix.
4. `step` — name: "library_scroll_coaches" — smoothly scrolls to the
   coaching & interviewing row. Narrate while it runs: coaching and
   interviewing agents that score you against real rubrics.
5. `step` — name: "library_scroll_tutors" — scrolls further to the tutors
   row. Narrate while it runs: language coaches and an Interactive Python
   tutor.
7. `goto`: "https://app.toughtongueai.com/course"
8. `step` — name: "open_course" — opens the Sales Coaching course.
   Narrate while it runs: agents bundle into courses — structured learning
   paths with progress tracking.

### Phase 4: LEARN — Analytics
1. `goto`: "https://app.toughtongueai.com/sessions"
2. `step` — name: "analyze_first_session" — clicks Analyze on the most
   recent completed session. Narrate while it runs.
3. `capture` — **MILESTONE 3**: see the analysis page, walk through the
   actual content
```

Note the rhythm: every `step` call is sandwiched between a narration cue
("narrate while it runs") so the agent keeps talking during execution, and
captures appear only where the agent genuinely needs to see the page to
speak about it.

## What was deliberately left out

- **No typing steps.** The create-an-agent phase shows the page via `goto`
  + `capture` and explicitly tells the agent NOT to fill the inputs — the
  scenario frames typing as the user's turn. Fewer `fill` actions, fewer
  ways to break.
- **No date pickers, no dropdown cascades.** The flow was trimmed to
  scrolls, tab clicks, and card clicks — all stable against app state.
- **Variables.** None: every step is fixed. The instructions even say
  "none of the steps need variables" so the agent never invents a
  `variables` param.
