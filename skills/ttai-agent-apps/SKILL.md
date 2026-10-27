---
name: ttai-agent-apps
description: >-
  Creates and maintains Agent Desktop apps for Tough Tongue AI voice agents:
  small React packages (a README.md manual, src/App.tsx, and data files) that
  the agent opens and updates beside a live session, such as a board,
  checklist, scorecard, or presentation deck. Finds, creates, writes,
  verifies, and previews apps with ttai:list_resources, ttai:create_filebase,
  ttai:write_filebase, ttai:get_resource, and ttai:create_token, then attaches
  one to a Scenario's agent_desktop tool with ttai:update_scenario. Use when
  the user says "build an app my voice agent can edit", "make a board the agent
  updates during the call", "a deck the agent presents", "attach this app to my
  scenario", or "preview my app". Not for designing the Scenario's
  conversation (ttai-agent) or analyzing sessions (ttai-session-analyst).
when_to_use: >-
  Also when the user asks to change what an existing app shows or how the
  agent operates it.
---

# Tough Tongue AI Agent Apps

Design → create → write → check → attach. An app is a FileBase of kind `app`:
a React view shown beside a live voice call. The agent reads the app's README,
then rewrites small files under `data/` with five `desktop_*` tools; the app
re-renders in place. The app tells the agent what the learner did with `emit`.

## Contents

- Prerequisites
- Tool map
- Workflow
- Rules
- Checklist
- Key Files

## Prerequisites

- Scope: omit `org_id` for the personal workspace. For an organization, call
  `ttai:get_workspace_info` and pass the chosen `organizations[].id` as
  `org_id`.
- Visibility (org apps, set at create): `org` (default) lets every member list
  and read the app; its creator and editors (role EDIT+) edit it. `private`
  keeps it to its creator. Featured apps are read-only. Explain this when a
  write is refused.
- Share an app by attaching it to a Scenario: whoever can open the Scenario
  runs the app, whatever its visibility.
- Load each tool's input schema before calling; the live `tools/list` wins.
- Read [references/app-structure.md](references/app-structure.md) and
  [references/runtime.md](references/runtime.md) before writing any file.

## Tool map

| Intent               | Call                                                                      |
| -------------------- | ------------------------------------------------------------------------- |
| Discover             | `ttai:list_resources(type: filebases, filters: {kind: app}, query?)`      |
| Inspect / read files | `ttai:get_resource(type: filebases, id, paths?: [...])` (≤ 20 files)      |
| Create               | `ttai:create_filebase(kind: app, name, title, description?, visibility?)` |
| Write / delete files | `ttai:write_filebase(id, files?, delete?)`: overwrites, confirm           |
| Check it runs        | `view_url` on the create, write and get results and on list rows          |
| Share a link         | `ttai:create_token({type: filebase_view, filebase_id, valid_for_hours})`  |
| Attach               | `ttai:update_scenario` (`tools_config.tools.agent_desktop`)               |

Changing visibility after create, and deleting an app, happen in the web app's
app editor, not over MCP.

## Workflow

### 1. Design

One round of questions; skip what is stated: what the app shows, what the agent
changes mid-call (a slide, a status, a list item), what the learner can do in
it, and which Scenario uses it. Split before writing:

- **Code**: `src/**`, fixed; the agent never edits it.
- **Data**: small flat JSON or YAML files under `data/` that the agent
  rewrites whole. `src/App.tsx` reads them only through
  `import files from "./desktop-files"`, never as a module import.
- **Learner actions**: the agent sees a click only if the app calls
  `emit` (`import { emit } from "./desktop-events"`). Prefer state the voice
  drives; emit only what the agent must react to.

### 2. Create

`ttai:create_filebase` with `kind: app`, `name`, `title`, optional
`description` and `visibility`.

- `name`: lowercase letters, digits, and hyphens; 2–64 characters; starts with
  a letter or digit; fixed at create; unique in the workspace (a duplicate is
  refused). It is the `app` argument of the agent's `desktop_*` tools and the
  key in the Scenario's `apps`. Avoid featured names (`canvas`, `deck`,
  `brainstorm`, `quiz`).
- `title` and the first line of `description` reach the agent's prompt.
- Keep the returned `id`; every later call takes it.

### 3. Write files

`ttai:write_filebase` with `id` and `files`: a map of **package-relative** paths
(`README.md`, `src/App.tsx`, `data/board.json`) to full contents.

- Each file is replaced whole. Send only the files that change; `delete`
  removes paths.
- First write: `README.md`, `src/App.tsx`, and the data files. The receipt's
  `missing` must be empty.
- `README.md` is the agent's manual: `# Title`, one line, `## Data`,
  `## Controls`, `## Rules`. Layout and examples:
  [references/app-structure.md](references/app-structure.md).

### 4. Verify

`ttai:get_resource(type: filebases, id, paths: ["README.md", "src/App.tsx", ...])`
and check:

- `README.md` has all three sections; `## Controls` names the exact `data/`
  file, its fields and allowed values, and each emitted event.
- The tree has every app file `src/App.tsx` imports (`./desktop-files` and
  `./desktop-events` are supplied at run time, never written).
- No `src/` file imports a data file as a module (`from "../data/…"`).
- `package.json`, if present, lists each dependency with an exact version
  (`"yaml": "2.9.1"`, not `"^2.6.0"`), no lockfile, and no `react` (see
  app-structure.md).

### 5. Check it runs

Open the result's `view_url` in a browser, signed in: the app must compile and
render. Add `?debug=1` for the Files and Events log
([references/runtime.md](references/runtime.md)). To share it with someone
signed out, or when a result has no `view_url`: `ttai:create_token` with
`type: "filebase_view"` (`url` opens the app without signing in;
`valid_for_hours` ≤ 24, default 1). Mint a fresh link each time; it carries an
access token, so give it only to the user. To see the agent drive the app, run
a Scenario that attaches it.

### 6. Attach to a Scenario

Fetch the Scenario first. `ttai:update_scenario` merges `tools_config.tools` per
tool, so other tools stay as they are, but it replaces `tool_settings` whole:
copy the existing `agent_desktop.tool_settings` keys into your write. Send one
`scenario` object:

```json
{
  "scenario": {
    "id": "<scenario id>",
    "tools_config": {
      "tools": {
        "agent_desktop": {
          "should_register": true,
          "add_to_system_prompt": true,
          "tool_settings": { "apps": { "idea-board": true } }
        }
      }
    }
  }
}
```

`apps` maps each app `name` to `true`: the apps the agent may launch.
`add_to_system_prompt: true` adds the desktop workflow, one line per app, and
the `[desktop:<app>]` event note to the prompt. `ai_instructions` then only
needs the when (which phase shows which app), not the how. The change applies
to new sessions.

### 7. Iterate

Data or README change → `ttai:write_filebase` on that file only. Code change →
write the changed `src/` files, then check it runs again. Open sessions keep
the files they loaded: relaunch the app or reload the page.

## Rules

- Confirm before `ttai:write_filebase` on an app a Scenario already uses: every
  session that launches it after the write gets the new files. Confirm before
  `ttai:update_scenario`.
- No secrets, tokens, or personal data in app files.
- The agent's edits during a session stay in that session. Every new session
  starts from the stored files.
- Agent guidance lives in `README.md` only.

## Checklist

- [ ] `README.md` has `## Data`, `## Controls`, `## Rules`; `missing` empty;
      tree has every file `src/App.tsx` imports
- [ ] Data read via `import files from "./desktop-files"` inside the
      component; no `../data/` import
- [ ] Data files live under `data/`; `package.json` deps exact, no lockfile
- [ ] `## Controls` name the exact file, its fields, allowed values, and each
      `emit`ted event
- [ ] `view_url` opened (or a `create_token` link given); the app renders
- [ ] `agent_desktop.tool_settings` merged, not replaced;
      `add_to_system_prompt: true`

## Key Files

- [references/app-structure.md](references/app-structure.md): package layout,
  required files, the README manual, worked example
- [references/runtime.md](references/runtime.md): reading data, how updates
  reach the screen, app events, render surface, input, debugging
