# Selector Guide — Stable XPath, Forbidden Patterns, Checks

## Contents

- Format rules
- Preference order
- Never use
- Supported XPath subset
- Three forbidden patterns
- Text matching
- Iframes and shadow DOM
- Responsive layouts
- DevTools checks
- Interaction gotchas
- Component-library recipes

---

## Format rules

- Write every selector as `xpath=//…`. A bare `/…` also counts as XPath.
- Only XPath selectors get scroll-into-view and the cursor animation before the
  action. CSS selectors such as `#id`, `div > a`, or `[data-x]` fail that
  pre-step and send the action straight to the AI retry — slow and
  nondeterministic.
- Uniqueness is per screen: the selector must match exactly one element on the
  screen where the action runs.

## Preference order

Stop at the first that is stable and unique:

1. Stable id: `xpath=//input[@id='email']`. Reject generated ids (`:r1:`,
   `rc_select_3`, uuid-like, long digit runs).
2. Test or label attributes: `@data-testid`, `@aria-label`, `@name`,
   `@placeholder`, `@title`.
3. Role + visible text:
   `xpath=//button[@role='tab'][contains(text(),'Featured')]`.
4. Container-scoped text: `xpath=//nav//a[contains(text(),'Reports')]`.

## Never use

- **Positional paths**: `/html[1]/body[1]/div[3]/div[2]/button[1]`. The recorder
  emits these when an element has no id. Any layout change breaks them, and they
  give the retry nothing to work with.
- **Build-hashed classes**: `sc-eNfBWa`, `css-1q2w3e`, `_a8f3k`.
- **Rendered-case text**: CSS `text-transform` can display "SIGN IN" over DOM
  text "Sign in". `text()` matches the DOM.

## Supported XPath subset

When the page contains shadow roots, selectors are evaluated by a reduced XPath
engine; plain pages use the browser's native XPath. Write for the reduced engine
so a later-added widget cannot change behavior.

Supported:

- `/` and `//` steps, tag names, `*`
- `[@attr='v']`, `[@attr]`, `contains(@attr,'v')`, `starts-with(@attr,'v')`
- `text()='v'`, `contains(text(),'v')`, `.='v'`, `contains(.,'v')`
- `normalize-space(text())='v'`, `normalize-space(.)='v'`,
  `normalize-space(@attr)='v'`
- `and`, `or`, `not(...)` when every part is itself supported
- Chained predicates and step indexes: `//tr[contains(.,'Paid')][1]`

Not supported (silently ignored — no error, no retry):

- Axes: `ancestor::`, `parent::`, `following-sibling::`, `..`
- `position()`, `last()`, unions `|`, grouped `(…)[n]`
- Nested path predicates: `[.//h3[...]]`
- Bare `normalize-space()` with no argument

An ignored predicate leaves the step matching by tag alone, so the action hits
the first matching element. Symptom: "the demo always clicks the first item."

## Three forbidden patterns

1. **Nested path predicate** — `//div[.//h3[contains(text(),'Pro')]]`. Fix:
   target the text element itself, `//h3[contains(text(),'Pro')]`; the click
   bubbles to the card's handler.
2. **Grouped + indexed** — `(//div[contains(@class,'plan')])[3]//button`. Fix:
   anchor on content, not position: `//h3[contains(text(),'Enterprise')]` or
   `//div[contains(@class,'plan')][contains(.,'Enterprise')]//button`.
3. **`and` with an unsupported part** —
   `//button[@role='tab' and normalize-space()='Featured']`. One bad part drops
   the whole group, leaving `//button`. Fix: separate brackets,
   `//button[@role='tab'][contains(text(),'Featured')]`.

## Text matching

The two engines disagree on `text()`. Native XPath tests only the element's own
text nodes (`contains(text(),…)` checks just the first one). The reduced engine
reads the full `textContent`, descendants included. So
`//tr[contains(text(),'completed')]` matches a row with a nested status badge on
a shadow-DOM page and matches nothing on a plain page.

`.` means the full text in both engines. Use `contains(.,'…')` when the text
sits in a nested element (table rows, cards, a button wrapping a `<span>`). Keep
`contains(text(),'…')` for elements that hold the text directly, such as
headings. Put `.` on a specific tag or a class-anchored step, never a bare
`//div` or `//*`: every ancestor contains the text too, and the outermost one
matches first.

## Iframes and shadow DOM

- Iframes: an XPath continues inside a frame only at a bare `iframe` or
  `iframe[n]` step: `xpath=//iframe[1]//button[contains(text(),'Pay')]`. A
  predicate on that step (`iframe[@title='Checkout']`) stops the hop.
- Avoid the `>>` hop syntax in steps: it is not valid XPath, so the action skips
  straight to the AI retry every time.
- Shadow DOM: handled by the reduced engine; stay within the subset.
- No cursor animation or pre-scroll inside frames; verify iframe actions in a
  test session.

## Responsive layouts

Live demos render at 1280×720. A selector harvested at 1920 wide can target a
sidebar or label that collapses at 1280. Record and harvest at 1280 wide
(`viewport: "compact"` in `ttai:authenticate_browser`), and re-check selectors
for hamburger menus, truncated labels, and hidden columns.

## DevTools checks

Run on the screen the selector targets.

Count matches (must be 1):

```js
((p) =>
  document.evaluate(
    `count(${p})`,
    document,
    null,
    XPathResult.NUMBER_TYPE,
    null,
  ).numberValue)(
    "//button[@role='tab'][contains(text(),'Featured')]",
  );
```

Cross-check a row or card predicate against the text both engines see:

```js
[...document.querySelectorAll("tbody tr")]
  .filter((tr) => tr.textContent.includes("completed")).length;
```

`document.evaluate` cannot see into shadow roots. On shadow-DOM pages, rely on
the text check and a test session.

List candidate anchors for an element you can see:

```js
[...document.querySelectorAll("button, a, [role], input, textarea, select")]
  .filter((e) => (e.innerText || e.value || "").includes("NEEDLE"))
  .map((e) => ({
    tag: e.tagName,
    id: e.id,
    role: e.getAttribute("role"),
    aria: e.getAttribute("aria-label"),
    testid: e.dataset.testid,
    text: (e.innerText || "").slice(0, 60),
  }));
```

## Interaction gotchas

- **Logged-in vs fresh**: harvest in the saved-login state; a fresh visit shows
  different screens.
- **Disabled submit**: `fill` fires input events that enable it — fill, then
  click.
- **Async screens**: there is no wait action. End the step at the click that
  changes screens; start the next step on the new screen.
- **Portal dropdowns**: options render at page level and hidden ones stay in the
  DOM. Two actions: open, then click the option scoped to the visible dropdown.
- **Dates**: demo sandboxes have their own clocks. Use form defaults; never
  record date-picker math.

## Component-library recipes

Ant Design (classes are semantic and stable):

- Visible select option (one XPath; do not break the line):

  ```text
  //div[contains(@class,'ant-select-dropdown')][not(contains(@class,'ant-select-dropdown-hidden'))]//div[contains(@class,'ant-select-item-option')][@title='Deluxe Room']
  ```
- Drawer-scoped field:
  `//div[contains(@class,'ant-drawer')]//input[@id='amount']`
- Modal close: `//button[@aria-label='Close']`

MUI / Radix / Headless UI: prefer `@role` + text
(`//li[@role='option'][contains(text(),'Monthly')]`); their ids are generated.
