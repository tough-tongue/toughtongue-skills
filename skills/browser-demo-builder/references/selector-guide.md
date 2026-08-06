# Selector Guide — Rules, Forbidden Patterns, Verification

How to write selectors that replay deterministically. The replay engine
supports a specific XPath subset; selectors outside it are silently
simplified and match the WRONG element — the classic symptom is a demo that
always clicks the first item in a list.

## Contents

- Selector preference order
- Never use
- Supported XPath subset (the contract)
- Three forbidden patterns and their fixes
- Text-matching quirk you can rely on
- DevTools console snippets (harvest + verify)
- Interaction gotchas

---

## Selector preference order

Stop at the first that exists and is unique on the screen it runs on:

1. **Stable `#id`** — `xpath=//input[@id='password']`. Watch for generated
   ids (`:r1:`, `rc_select_3`, uuid-ish) — those are NOT stable.
2. **`aria-label` / `title`** — `xpath=//button[@aria-label='add item']`.
3. **Text-anchored structural XPath** — anchor on visible text plus a role
   or container: `xpath=//button[@role='tab'][contains(text(),'Featured')]`.

Uniqueness is per-screen: `//button[@type='submit']` is fine if the screen
it runs on has exactly one submit button — verify with the count snippet
below.

## Never use

- **Build-generated class hashes** (`sc-eNfBWa`, `css-1q2w3e`) — they change
  every build.
- **Positional-only paths** (`/div[3]/div[2]/button`) — break on any layout
  change and are useless as retry context.
- **Visible-text assumptions** — CSS `text-transform` can show "SIGN IN"
  while the DOM text is "Sign In". `text()` matches the DOM, not what you
  see. When unsure, prefer attributes (`@type='submit'`).

## Supported XPath subset (the contract)

The replay engine parses selectors with its own XPath engine. Write every
selector within this subset:

- Plain child/descendant steps: `//div//button`, `//tbody/tr`
- Own-element predicates: `@attr='v'`, `contains(@attr,'v')`,
  `contains(text(),'v')`, `text()='v'`
- Chained bracket predicates (AND filter): `//button[@role='tab'][contains(text(),'Featured')]`
- Step-level index: `//tr[contains(text(),'completed')][1]`

**The trap:** unsupported predicates are silently dropped — no error, no
retry trigger. The step "succeeds" against the first element matching only
the surviving (simpler) part of the path.

## Three forbidden patterns and their fixes

1. **Nested relative-path predicates** — `div[.//h3[contains(text(),'X')]]`
   (filtering a wrapper by a descendant's text).
   **Fix:** anchor directly on the text-bearing element itself
   (`//h3[contains(text(),'X')]`) and rely on the click bubbling up to the
   wrapper's handler — the replay engine dispatches a real, bubbling mouse
   event at the resolved element's visual center.
2. **Grouped + indexed expressions** — `(//tr[...])[1]//button[...]`.
   **Fix:** drop the parens and chain plain steps with own-element
   predicates plus a step-level `[n]` index:
   `//tr[contains(text(),'completed')][1]//button[contains(text(),'Analyze')]`.
3. **`and`-joined predicates containing bare `normalize-space()`** —
   `button[@role='tab' and normalize-space()='Featured']`. A failed
   sub-parse drops the WHOLE and-group — leaving `//button` with zero
   predicates, i.e. the first button in the document.
   **Fix:** split into separate bracket predicates, each individually
   supported: `//button[@role='tab'][contains(text(),'Featured')]`.

## Text-matching quirk you can rely on

The engine's `text()` / `.` predicates read the element's full
`textContent` (recursive), not just direct text children. So
`tr[contains(text(),'completed')]` correctly matches a row whose status
badge is a nested `<div>` — even though that deviates from the XPath spec.

Consequence for verification: native `document.evaluate` disagrees with the
engine exactly here (native `contains(text(),'v')` only sees direct child
text nodes). Verify behavior by replicating the predicate with
`element.textContent.includes(needle)`, not with `document.evaluate` alone.

## DevTools console snippets (harvest + verify)

Run these in the browser console on the screen the selector targets.

Dump all buttons (id, class, aria, text):

```js
(() => [...document.querySelectorAll('button')].map(b => ({
  id: b.id, cls: b.className, aria: b.getAttribute('aria-label'),
  text: (b.innerText || '').slice(0, 60),
})))()
```

Dump all inputs:

```js
(() => [...document.querySelectorAll('input, textarea, select')].map(i => ({
  id: i.id, type: i.type, name: i.name, ph: i.placeholder,
  aria: i.getAttribute('aria-label'),
})))()
```

Find non-button clickables containing some text (tiles, cards, rows):

```js
(() => [...document.querySelectorAll('[role="button"], [data-testid], li, a, h3')]
  .filter(e => (e.innerText || '').includes('NEEDLE'))
  .map(e => ({ tag: e.tagName, id: e.id, cls: e.className,
    role: e.getAttribute('role'), html: e.outerHTML.slice(0, 300) })))()
```

Resolve an XPath and count matches (uniqueness check — count must be 1):

```js
(() => {
  const p = "//button[@role='tab'][contains(text(),'Featured')]";
  const first = document.evaluate(p, document, null,
    XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue;
  const count = document.evaluate(`count(${p})`, document, null,
    XPathResult.NUMBER_TYPE, null).numberValue;
  return { count, match: first ? first.tagName + '#' + first.id + ' | ' +
    (first.innerText || '').slice(0, 40) : null };
})()
```

Engine-faithful text check (recursive textContent, matching the replay
engine's semantics):

```js
(() => [...document.querySelectorAll('tbody tr')]
  .filter(tr => tr.textContent.includes('completed'))
  .map((tr, i) => ({ i, text: tr.textContent.slice(0, 80) })))()
```

## Interaction gotchas

- **Authenticated context ≠ fresh visit.** A persistent logged-in context
  shows different screens (saved-account tile vs full login form). Harvest
  in the state the demo will see; record fallback steps for the other
  state.
- **Submit disabled until fill.** Harmless — the replayed `fill` dispatches
  input events that enable the button. Order actions fill-then-click.
- **Cascading dropdowns.** Later selects populate only after earlier picks.
  Keep the pick order from the real UI; drop flaky auto-populated fields
  when defaults suffice.
- **Portal-rendered dropdowns.** Many UI libraries render dropdown options
  at `body` level, and hidden ones stay in the DOM. Scope the option
  selector to the *visible* dropdown container, and always record select
  widgets as two actions: open the select, then click the option.
- **Screen transitions are async.** After a click, the next screen's
  elements may take a moment to exist — verify each selector on the screen
  it actually runs on, walking the flow in order.
- **Sandbox clocks.** Demo sandboxes run their own dates. Never record
  date-picker math; use the form's default dates and note that in the
  scenario narration.
