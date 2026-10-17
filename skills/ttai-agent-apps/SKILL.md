---
name: ttai-agent-apps
description: >-
  Creates and maintains Agent Desktop apps for Tough Tongue AI voice agents:
  small React packages (a README.md manual, code, and an editable data or
  control file) that the agent opens and drives beside a live session, such as
  a board, checklist, scorecard, or presentation deck. Finds, creates, writes,
  verifies, and previews apps with ttai:list_resources, ttai:create_filebase,
  ttai:write_filebase, ttai:get_resource, and ttai:create_token, writes the
  README controls so the voice agent knows how to operate the app, then
  attaches it to a Scenario's agent_desktop tool with ttai:update_scenario. Use when the user says
  "build an app my voice agent can edit", "make a board the agent updates
  during the call", "a deck the agent presents", "attach this app to my
  scenario", or "preview my app". Not for designing the Scenario's
  conversation (ttai-agent) or analyzing sessions (ttai-session-analyst).
when_to_use: >-
  Also when the user asks to change what an existing app shows or how the
  agent operates it.
---

# Tough Tongue AI Agent Apps

Design → create → write files → verify → preview → attach. An app is a file
package rendered next to a live voice session; the voice agent operates it by
rewriting one small **data** or **control** file, following the `## Controls`
in the app's README.

## Contents

- Prerequisites
- Tool map
- Workflow
- Rules
- Checklist
- Key Files

## Prerequisites

- Resolve scope first: omit `org_id` for the personal workspace. For an
  organization, call `ttai:get_workspace_info` and pass the chosen
  `organizations[].id` as `org_id`; an app created there belongs to it and is
  visible to its members.
- Load each tool's input schema before calling; the live `tools/list` wins.
- Read [references/app-structure.md](references/app-structure.md) and
  [references/runtime.md](references/runtime.md) before writing any file.

## Tool map

| Intent               | Call                                                            |
| -------------------- | --------------------------------------------------------------- |
| Discover             | `ttai:list_resources(type: filebases, filters: {kind: app}, query?)` |
| Inspect / read files | `ttai:get_resource(type: filebases, id, paths?: [...])` (≤ 20 files) |
| Create               | `ttai:create_filebase(kind: app, name, title, description?)`     |
| Write / delete files | `ttai:write_filebase(id, files?, delete?)` — destructive, confirm |
| Check it runs        | `view_url` on the create / write / get result (20 min)           |
| Longer preview link  | `ttai:create_token({type: filebase_view, filebase_id})`          |
| Attach               | `ttai:update_scenario` (`tools_config.tools.agent_desktop`)     |

Visibility changes, disabling, and deleting an app happen in the web app's
app editor, not over MCP.

## Workflow

### 1. Design

One round of questions; skip what is stated: what the app shows, what the voice
agent changes mid-call (a slide, a status, a list item), and which Scenario
uses it. Split before writing:

- **Code** — `src/**`, fixed; the agent never edits it.
- **Data / control** — one small, flat JSON file under `data/` that the agent
  rewrites whole. `src/App.tsx` reads it only through
  `import files from "./desktop-files"`, never as a module import.
- **Clicks** — the agent never sees them. Let the voice drive the state, or
  have the agent ask the learner what they picked.

### 2. Create

`ttai:create_filebase` with `kind: app`, `name`, `title`, optional `description`.
The app's title and description come from here and reach the agent's prompt.

- `name`: lowercase letters, digits, and hyphens; 2–64 characters; starts with
  a letter or digit; unique in the workspace (the call refuses a duplicate).
  Sessions look apps up by name, and it becomes the live agent's tool prefix
  (`idea-board` → `idea_board_launch`, `idea_board_write`, …). Never use a
  built-in name (`canvas`, `deck`, `brainstorm`).
- Keep the returned `id`; every later call takes it.

### 3. Write files

`ttai:write_filebase` with `id` and `files`: a map of **package-relative** paths
(`README.md`, `src/App.tsx`, `data/board.json`) to full contents.

- Each file is replaced whole. Send only the files that change; `delete`
  removes paths.
- First write: `README.md`, `src/App.tsx`, and the data file. The result's
  `missing` must be empty.
- `package.json` only for extra packages, and only from the allowlist
  (`yaml`, `marked`, `mermaid`, `dompurify`, `lucide-react`). React 18 is
  forced.
- `README.md` is the agent's manual: `# Title`, a one-line description, then
  `## Data` (the `data/` files and their shapes), `## Controls` (which file to
  write to drive the running app), and `## Rules`. The live agent reads it with
  `desktop_info`, with the file list.
- No `spec.json`: it is not read. A write that includes one returns a
  `warning`; put that guidance in the README. Remove an old one with
  `delete: ["spec.json"]`.
- No manifest: the backend lists the app's files. The root is always `/<name>`;
  the live agent writes whole files only under `<root>/data/`.

### 4. Verify

`ttai:get_resource(type: filebases, id, paths: ["README.md", ...])` and check:

- `README.md` has the `## Data`, `## Controls` and `## Rules` sections; there
  is no `spec.json`; the write's `warning` is empty. The listing has every app
  file `src/App.tsx` imports. Runtime modules are not in it: `./desktop-files`
  is supplied at run time, never written.
- No `src/` file imports a data file as a module (`from "../data/…"`); data
  is read through `./desktop-files`, inside the component.
- `package.json`, if present, lists only allowlisted packages.
- `## Controls` name the exact file under `data/` and the values to write.

### 5. Check it runs, then preview

If the create / write / get result carries `view_url`, open it in a browser:
the app must compile and render. Append `&debug=1` for the runtime Events log:
compile errors with `file:line`, the iframe console, and whether each agent
write patched in place or recompiled. Without `view_url` (older server), or for
a longer link, use `ttai:create_token({type: "filebase_view", filebase_id: id,
valid_for_hours})`: `url` opens the view without signing in, for up to 24 hours
(default 1). To see the agent drive it, run a Scenario that attaches it.
Request a fresh link instead of reusing one. Each link carries an access token,
so share it only with the user.

### 6. Attach to a Scenario

Fetch the Scenario first. `ttai:update_scenario` merges `tools_config.tools` per
tool, so other tools stay as they are, but it replaces `tool_settings` whole:
copy the existing `agent_desktop.tool_settings` keys (such as `mode`) into your
write. Send one `scenario` object:

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

`apps` maps the app `name` to `true`; the first key is the app that opens at
start. `add_to_system_prompt: true` puts each app's title, root and description
in the prompt; the agent reads the README with `desktop_info`.
`ai_instructions` then only needs the when (which phase shows which view), not
the how. The app must be enabled and visible to the session's
participant; otherwise the session silently falls back to the default canvas.

### 7. Iterate

Content or control change → `ttai:write_filebase` on that file only. Code change →
write the changed `src/` files, then check it runs again (`view_url`). Either
way, reload any open session page: it keeps the files it loaded.

## Rules

- Confirm before `ttai:write_filebase` on an app a Scenario already uses: every
  session started or reloaded after the write gets the new files.
- No secrets, tokens, or personal data in app files.
- The agent's edits during a session stay in that session. Every new session
  starts from the stored files.
- Never write `spec.json`; agent guidance lives in `README.md`.

## Checklist

- [ ] `README.md` has `## Data`, `## Controls`, `## Rules`; `missing` empty;
      listing has every file `src/App.tsx` imports; no `spec.json`; `warning`
      empty
- [ ] Data read via `import files from "./desktop-files"` inside the
      component; no `../data/` import
- [ ] Data / control file lives under `data/`; `package.json` deps
      allowlisted
- [ ] `## Controls` name the exact file, its fields, and allowed values
- [ ] Re-read verified; `view_url` opened (or `create_token` link given) and
      the app renders
- [ ] `agent_desktop.tool_settings` merged, not replaced;
      `add_to_system_prompt: true`

## Key Files

- [references/app-structure.md](references/app-structure.md) — package layout,
  required files, the README manual, worked example
- [references/runtime.md](references/runtime.md) — reading data, how updates
  reach the screen, render surface, input, delivery, limits
