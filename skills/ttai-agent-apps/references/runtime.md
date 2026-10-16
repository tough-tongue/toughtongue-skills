# Runtime

What an app runs in, how it reads its data, how a write from the live agent
reaches the screen, and what the learner can do with it.

## Contents

- Reading data
- How updates reach the screen
- Render surface
- Input
- Delivery
- Checking it runs
- Limits

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
- YAML: the allowlisted `yaml` package. Markdown: `marked`, then `dompurify`.

## How updates reach the screen

| Change                                               | What happens                                                          | App state                             |
| ---------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------- |
| The live agent writes a `data/` file                 | New content is posted into the running app; it re-renders in a few ms | Kept: local state, scroll, animations |
| `src/**` or `package.json` changes                   | The app rebuilds and remounts (up to about 1.5 s)                     | Reset                                 |
| The app imports a `data/` file as a module           | Every agent write rebuilds, as above                                  | Reset on every write                  |
| The learner switches apps or minimizes, then reopens | The app boots again                                                   | Reset                                 |

- View state that must outlive a rebuild or a reopen goes in `sessionStorage`
  under an app-prefixed key (`<app_name>:step`). Wrap every access in
  `try`/`catch`: storage can throw, and all apps share one storage origin.
- Start an entry animation only when the data really changed (compare with the
  previous value), so a rewrite of the same state plays nothing.
- Each agent write shows a host notice in the app's top-right corner for about
  3 seconds. Keep roughly 360×72 px there free of anything the learner must
  read right after a write.

## Render surface

- A sandboxed, cross-origin iframe. No host CSS, theme, or fonts reach it.
  Paint `html, body, #root` yourself and reset `body { margin: 0 }`.
- Pick one palette. Do not follow `prefers-color-scheme`: it tracks the
  operating system, not the app's theme toggle.
- React 18, forced. No Tailwind and no PostCSS: use inline styles or one CSS
  string rendered in a `<style>` element.
- `index.html` is replaced, so load web fonts with a CSS `@import` as the first
  rule of your style.
- The host already frames the app (border, rounded corners, taskbar). Do not
  draw an outer card or radius around the whole app.
- Reflow, never scale. The pane is about 388×470 px on a phone and 789×624 px
  on a 1280×720 laptop, and wider when the learner drags the split. Use CSS
  flow and `clamp()`, not a fixed canvas scaled to fit.
- Links need `target="_blank"`. Fullscreen is unavailable. Autoplay audio is
  blocked: default to no sound.

## Input

- The live agent never sees clicks, drags, or typing inside the app. Prefer
  state the voice drives. If something is clickable, it works locally, and the
  README `## Controls` tell the agent to ask the learner what they picked.
- Space is the session's push-to-talk key. Bind no Space or Enter shortcuts,
  and blur a clicked element after the click so Space returns to push-to-talk.

## Delivery

- App files are served without caching. After `ttai:write_filebase`, reload
  the session page: a page that is already open keeps the files it loaded.
- A preview (`view_url`, `filebase_view`) shows the stored files. To see the
  live agent drive the app, start a new session of a Scenario that attaches it.
- The agent's writes during a session stay in that session. Every new session
  starts from the stored files.
- An app that is disabled, or not visible to the participant (an organization
  app opened by someone outside the organization), silently falls back to the
  default canvas.

## Checking it runs

- `ttai:create_filebase`, `ttai:write_filebase` and `ttai:get_resource` return
  `view_url` (+ `view_expires_at`, 20 min) for an app. After a write, open it
  in a browser: the app must compile and render. If the result has no
  `view_url` (older server), mint a link with
  `ttai:create_token({type: "filebase_view", filebase_id})` (up to 24 h).
- Append `&debug=1` to the link for the runtime Events log, newest first:
  - `compile`: start, done or failed with ms; errors carry `path:line:col`.
  - `error` and `console`: runtime errors and the iframe's console output.
  - `sync`: mount, patch or recompile, with the changed paths. A data write
    must show a patch; a recompile means the app imports a `data/` file as a
    module.
  - `data` and `package`: files written, and what the package load found.
- The write result's `warning` is advisory: it flags a `spec.json` ("not
  read; put agent guidance in README.md") and a package past the advisory size
  cap. Fix what it names.

## Limits

- One `ttai:write_filebase` call: 1–200 paths (files plus deletes). One read:
  up to 20 paths.
- During a session the agent can write files up to 128 KB. Keep data files to
  a few KB: every write posts all data files into the app.
- Paths use letters, digits, `.`, `_`, `-`, and `/`; any other character
  becomes `_`.
- App names: lowercase letters, digits, and hyphens; 2–64 characters; starting
  with a letter or digit.
- Every file is stored as plain text: no binaries (inline SVG or data URIs for
  images), no secrets, no personal data.
