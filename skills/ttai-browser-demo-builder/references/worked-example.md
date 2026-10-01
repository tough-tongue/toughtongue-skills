# Worked Example — Platform Tour Demo

## Contents

- The demo
- The steps
- Why each selector looks this way
- The matching instructions
- What was left out

---

## The demo

An AI SDR demos the Tough Tongue AI web app live: library, a course, then a
session analysis. Four steps cover the scripted clicks; everything else is
narration, `goto`, and three capture milestones. Treat the selectors as the
pattern — page labels may differ in the current app.

## The steps

Merged into the `tool_settings` read from the Scenario (`initialUrl` and
`contextId` carried over unchanged — see [steps-format.md](steps-format.md)):

```json
{
  "library_show_coaches": {
    "description": "On the library: select the Featured tab and bring the coaching row on screen",
    "actions": [
      {
        "description": "Featured tab on the library page",
        "method": "click",
        "selector": "xpath=//button[@role='tab'][contains(text(),'Featured')]",
        "arguments": []
      },
      {
        "description": "Google PM Interviewer card in the featured library grid",
        "method": "scrollIntoView",
        "selector": "xpath=//h3[contains(text(),'Google PM Interviewer')]",
        "arguments": []
      }
    ]
  },
  "library_show_tutors": {
    "description": "Bring the tutors and language coaches row on screen",
    "actions": [
      {
        "description": "Interactive Python 101 card in the featured library grid",
        "method": "scrollIntoView",
        "selector": "xpath=//h3[contains(text(),'Interactive Python 101')]",
        "arguments": []
      }
    ]
  },
  "open_course": {
    "description": "On the courses page: select Featured and open the Sales Coaching course",
    "actions": [
      {
        "description": "Featured tab on the courses page",
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
    "description": "On the sessions page: open the analysis of the newest completed session",
    "actions": [
      {
        "description": "Analyze button on the first completed session row",
        "method": "click",
        "selector": "xpath=//tbody/tr[contains(.,'completed')][1]//button[contains(.,'Analyze')]",
        "arguments": []
      }
    ]
  }
}
```

## Why each selector looks this way

- `//button[@role='tab'][contains(text(),'Featured')]` — separate brackets, not
  `[@role='tab' and normalize-space()='Featured']` (forbidden pattern 3 would
  collapse it to the first button on the page).
- `//h3[contains(text(),'…')]` — targets the heading, not `div[.//h3[...]]`
  (forbidden pattern 1). Clicks bubble to the card.
- `scrollIntoView` for the "scroll" steps: the visible effect is the smooth
  scroll replay performs before any XPath action. `scrollTo` would only move the
  heading's own (nonexistent) scrollbar.
- `//tbody/tr[contains(.,'completed')][1]//button[…]` — a step-level `[1]`, not
  `(//tr[...])[1]` (forbidden pattern 2). `.` reads the row's full text in both
  engines, so the nested status badge matches; `text()` would miss it on a plain
  page.
- Clicking Featured first is a harmless no-op when it is already active and a
  guard when the page remembered another tab.

## The matching instructions

Abridged to the browser parts:

```text
## BROWSER TOOL INSTRUCTIONS
The browser is pre-opened at https://app.toughtongueai.com/. Use `runBrowserCommand`:
- `command: "step"` + `name` — pre-recorded batch. PRIMARY command.
- `command: "goto"` + `url` — direct navigation.
- `command: "capture"` — screenshot. Only at the 3 milestones below.
- `command: "observe"` / `command: "act"` — off-script moments only.
Steps run in the background — keep talking. If a step fails you are told
which action broke: `capture`, then recover with `act`.
None of the steps take variables.

## DEMO FLOW
### Phase 1: Home
1. `capture` — MILESTONE 1: see the home page, then introduce the platform.

### Phase 2: Library
1. `goto` "https://app.toughtongueai.com/library"
2. Narrate: sales and SDR agents you can remix.
3. `step` "library_show_coaches" — narrate coaching and interview agents.
4. `step` "library_show_tutors" — narrate language coaches and tutors.

### Phase 3: Courses
1. `goto` "https://app.toughtongueai.com/course"
2. `step` "open_course" — narrate: agents bundle into courses with progress.
3. `capture` — MILESTONE 2: describe the course page.

### Phase 4: Analytics
1. `goto` "https://app.toughtongueai.com/sessions"
2. `step` "analyze_first_session" — narrate while it runs.
3. `capture` — MILESTONE 3: walk through the analysis.
```

Rhythm: every page change is a `goto`, every scripted click is a `step`,
narration covers execution, captures only where the agent must see the page.

## What was left out

- **Typing.** No `fill` actions; the demo shows forms and frames typing as the
  prospect's turn.
- **Date pickers and dropdown cascades.** Only tabs, cards, and one button.
- **Variables.** Every step is fixed, and the instructions say so.
