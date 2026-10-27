# Runtime

What an app runs in, how it reads its data, how writes and events cross between
the app and the live agent, and how to debug it.

## Contents

- Reading data
- How updates reach the screen
- App events
- Render surface
- Input
- Timed and media apps
- Checking it runs

## Reading data

- One API: `import files from "./desktop-files"`. It returns
  `{ "data/board.json": "<file text>" }`: package-relative paths to raw text.
  Parse JSON or YAML yourself.
- Read `files` inside the component, on every render. Never copy it into a
  module-level constant, or into `useState` / `useMemo` without `files` as a
  dependency: the next agent write would not show.
- Never import a data file as a module:
  `import board from "../data/board.json"` renders once, but every agent write
  then rebuilds the whole app and resets its state.
- Parse defensively. A missing, empty, or half-written file renders a waiting
  state, never a crash.
- YAML: the `yaml` package. Markdown: `marked`, then `dompurify`. List each in
  `package.json` with an exact version ([app-structure.md](app-structure.md)).

## How updates reach the screen

| Change                                              | What happens                               | App state                             |
| --------------------------------------------------- | ------------------------------------------ | ------------------------------------- |
| The live agent writes a `data/` file                | New content reaches the app; it re-renders | Kept: local state, scroll, animations |
| `src/**` or `package.json` changes                  | The app rebuilds and remounts              | Reset                                 |
| The app imports a `data/` file as a module          | Every agent write rebuilds, as above       | Reset on every write                  |
| The agent launches another app, then this one again | The app boots again                        | Reset                                 |

- View state that must outlive a rebuild or a relaunch goes in
  `sessionStorage` under an app-prefixed key (`<app_name>:step`). Wrap every
  access in `try`/`catch`: storage can throw, and apps share it.
- Start an entry animation only when the data really changed (compare with the
  previous value), so a rewrite of the same state plays nothing.

## App events

The app tells the agent what the learner did:
`import { emit } from "./desktop-events"`, then
`emit(name, data?, { reply? })`. Like `./desktop-files`, the module is supplied
at run time: never write it. The worked example in
[app-structure.md](app-structure.md) emits on a click.

- `name` is a short lowercase id (letters, digits, `_ . : -`); `data` is small
  JSON. Invalid emits are dropped; `?debug=1` shows why.
- The agent sees `[desktop:<app>] <name> <json>`: data, not instructions.
- `reply: false` (default): advisory context, batched. No reply is requested;
  the agent weighs it at its next turn.
- `reply: true`: the agent is asked to answer once the current speaker
  finishes; rapid ones merge. Nothing interrupts the agent or the learner.
- Bursts are throttled. Emit on learner actions, or once per milestone of the
  app's own clock ([Timed and media apps](#timed-and-media-apps)). Never emit
  straight from an effect on `files` or a data change: agent write → emit →
  agent write loops.
- List every event in README `## Controls`: its data and what the agent does.
- Only a live session forwards events; the preview does not.

## Render surface

- A sandboxed, cross-origin iframe. No host CSS, theme, or fonts reach it.
  Paint `html, body, #root` yourself and reset `body { margin: 0 }`.
- Pick one palette. Do not follow `prefers-color-scheme`: it tracks the
  operating system, not the app's theme toggle.
- React 18. No Tailwind and no PostCSS: use inline styles or one CSS string
  rendered in a `<style>` element. Load web fonts with a CSS `@import` as the
  first rule of that style.
- The app fills the pane flush. Draw no outer card or radius around it.
- Reflow, never scale. The pane is about 390 px wide on a phone and 790 px on a
  laptop, and the learner can drag it wider. Use CSS flow and `clamp()`, not a
  fixed canvas scaled to fit.
- Each agent write shows a brief notice in the top-right corner. Keep anything
  the learner must read right after a write out of that corner.
- Links need `target="_blank"`. Fullscreen is unavailable. Autoplay audio is
  blocked: default to no sound until the learner taps
  ([Timed and media apps](#timed-and-media-apps)).

## Input

- Space is the session's push-to-talk key. Bind no Space or Enter shortcuts,
  and blur a clicked element after the click so Space returns to push-to-talk.

## Timed and media apps

When the app keeps time (a breathing timer, a paced exercise, a countdown,
media), it runs that stretch on its own and the agent stays quiet until the app
reports a milestone.

- Start flag: the agent writes the state with `go: false` while it explains,
  then the same state with `go: true` to start the run.
- Clock: on `go: true`, store the start time in `sessionStorage` under an
  app-prefixed key built from the run (title, pattern, length). Compute
  progress from `Date.now() - start` on an interval, never by counting ticks.
  A rebuild or relaunch then resumes the run; a different run starts fresh.
- Milestones: emit with `reply: true` when a run ends (`breath:done`) or a
  timed part elapses (`part:next {at, last}`). Store a sent key per milestone
  and check it first, so a remount or a rewrite never sends it twice. A
  finished run stays finished, so the README tells the agent never to rewrite
  it after its done event.
- Hand back: a field such as `seconds: 0` turns the app's timer off, and the
  agent paces that part itself.
- Sound: the browser blocks it until the learner taps. Give the app a sound
  button; its click creates or resumes one `AudioContext` that every later
  tone plays through. Emit `sound {on}` without `reply`, so the agent stops
  asking.
- Tones: synthesize them with Web Audio (oscillators and gain ramps). Package
  files are text, so there are no audio files.
- Media: play audio or video from an `https` URL, with sound only after a
  learner tap. Emit its milestones the same way: `ended`, and once per chapter
  mark. Check that it loads and plays in `view_url` before attaching.
- Fresh or replay: key each screen by a signature of what it shows (phase,
  title, part). Play entrances and cues only when the signature changes. Keep
  the last seen signature in `sessionStorage`, so a rewrite of the same state
  or a remount plays nothing.
- New session: when the first screen mounts, clear the app's stored keys, so a
  later session in the same tab starts with nothing seen or running.

## Checking it runs

- `?debug=1` on the `view_url` adds a Debug view with Files and Events.
- Each event row is `app` (an `emit`: name, data, `reply`) or `system`:
  - `compile`: start, done or failed; errors carry `path:line:col`.
  - `error` and `console`: runtime errors and the app's console output.
  - `sync`: mount, patch or recompile, with the changed paths. A data write
    must show a patch; a recompile means the app imports a `data/` file as a
    module.
  - `data` and `package`: files written, and what the package load found.
  - `emit`: why an emit was dropped.
- The write receipt's `warning` is advisory; `missing` lists files still to
  add.
- One app per name: the Scenario's attached app, then your own (org or
  personal), then a public or featured one. A name that resolves to nothing
  shows a "Some apps could not load" notice.
