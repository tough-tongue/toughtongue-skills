---
name: ttai-agent-apps
description: >-
  Creates and maintains Agent Desktop apps for Tough Tongue AI voice agents:
  small React packages (spec.json, code, and an editable data or control
  file) that the agent opens and drives beside a live session, such as a
  board, checklist, scorecard, or presentation deck. Finds, creates, writes,
  verifies, and previews apps with ttai:list_resources, ttai:create_filebase,
  ttai:write_filebase, ttai:get_resource, and ttai:create_token, declares the app's `controls` so
  the voice agent knows how to operate it, then attaches it to a Scenario's
  agent_desktop tool with ttai:update_scenario. Use when the user says
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
rewriting one small **data** or **control** file, following the `controls` the
app declares.

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
- Read [references/app-structure.md](references/app-structure.md) before writing
  any file.

## Tool map

| Intent               | Call                                                            |
| -------------------- | --------------------------------------------------------------- |
| Discover             | `ttai:list_resources(type: filebases, filters: {kind: app}, query?)` |
| Inspect / read files | `ttai:get_resource(type: filebases, id, paths?: [...])` (≤ 20 files) |
| Create               | `ttai:create_filebase(kind: app, name, title, description?)`     |
| Write / delete files | `ttai:write_filebase(id, files?, delete?)` — destructive, confirm |
| Preview link         | `ttai:create_token({type: filebase_view, filebase_id})`          |
| Attach               | `ttai:update_scenario` (`tools_config.tools.agent_desktop`)     |

Visibility changes, disabling, and deleting an app happen in the web app's
app editor, not over MCP.

## Workflow

### 1. Design

One round of questions; skip what is stated: what the app shows, what the voice
agent changes mid-call (a slide, a status, a list item), and which Scenario
uses it. Split before writing:

- **Code** — `src/**`, fixed; the agent never edits it.
- **Data / control** — one small, flat JSON file the agent rewrites whole.

### 2. Create

`ttai:create_filebase` with `kind: app`, `name`, `title`, optional `description`.

- `name`: lowercase letters, digits, and hyphens; 2–64 characters; starts with
  a letter or digit; unique in the workspace (the call refuses a duplicate).
  Sessions look apps up by name, and it becomes the live agent's tool prefix
  (`idea-board` → `idea_board_launch`, `idea_board_write`, …). Never use a
  built-in name (`canvas`, `maxx`, `brainstorm`).
- Keep the returned `id`; every later call takes it.

### 3. Write files

`ttai:write_filebase` with `id` and `files`: a map of **package-relative** paths
(`spec.json`, `src/App.tsx`, `data/board.json`) to full contents.

- Each file is replaced whole. Send only the files that change; `delete`
  removes paths.
- `spec.json` `files` is synced to the package tree on every write, so a new
  file loads without editing the manifest. Write `spec.json` in the first
  write.
- Write `controls` in `spec.json`: short lines that tell the live agent how to
  operate the app. They are added to the Scenario's system prompt for every
  session that enables the app.

### 4. Verify

`ttai:get_resource(type: filebases, id, paths: ["spec.json", ...])` and check:
`spec.json` parses; the returned tree has every file `src/App.tsx` imports;
every `filesystem.editableGlobs` pattern matches an existing file; `controls`
name the exact file and values to write.

### 5. Preview

`ttai:create_token({type: "filebase_view", filebase_id: id, valid_for_hours})`
→ `url` lists the app's files without signing in, for up to 24 hours (default 1). To see the app run, open a scenario that attaches it.
Request a fresh one instead of reusing it. It carries an access token, so share it only
with the user.

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
start. `add_to_system_prompt: true` is what puts the app's `controls` in the
prompt. `ai_instructions` then only needs the when (which phase shows which
view), not the how. The app must be enabled and visible to the session's
participant; otherwise the session silently falls back to the default canvas.

### 7. Iterate

Content or control change → `ttai:write_filebase` on that file only. Code change →
write the changed `src/` files, then preview again.

## Rules

- Confirm before `ttai:write_filebase` on an app a Scenario already uses: live
  sessions pick up the new files on their next app launch.
- No secrets, tokens, or personal data in app files.
- The agent's edits during a session stay in that session. Every new session
  starts from the stored files.
- Do not invent `spec.json` keys; unknown keys are ignored.

## Checklist

- [ ] `spec.json` written; tree has every file `src/App.tsx` imports
- [ ] Data / control file is the only `editableGlobs` target
- [ ] `controls` name the exact file, its fields, and allowed values
- [ ] Re-read verified; user has a fresh preview link
- [ ] `agent_desktop.tool_settings` merged, not replaced;
      `add_to_system_prompt: true`

## Key Files

- [references/app-structure.md](references/app-structure.md) — package layout,
  `spec.json`, `controls`, worked example, limits
