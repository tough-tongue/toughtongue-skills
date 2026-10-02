---
name: ttai-agent-apps
description: >-
  Creates and maintains Agent Desktop apps for Tough Tongue AI voice agents:
  small packages (spec.json, React code, and one editable data file) that the
  agent displays and updates beside a live session, such as a board,
  checklist, or scorecard. Creates, writes, verifies, previews, enables, and
  deletes apps with ttai:create_app, ttai:write_app_files,
  ttai:get_app_files, ttai:get_app_url, ttai:update_app, and
  ttai:delete_app, then attaches one to a Scenario's agent_desktop tool with
  ttai:update_scenario. Use when the user says "build an app my voice agent
  can edit", "make a board the agent updates during the call", "attach this
  app to my scenario", or "preview my app". Not for designing the Scenario's
  conversation (ttai-agent) or analyzing sessions (ttai-session-analyst).
when_to_use: >-
  Also when the user asks to change what an existing app shows or who can
  open it.
---

# Tough Tongue AI Agent Apps

Design → create → write files → verify → open → attach. An app is a file package
rendered next to a live voice session; the voice agent rewrites only its
**data** file.

## Contents

- Prerequisites
- Tool map
- Workflow
- Rules
- Checklist
- Key Files

## Prerequisites

- Resolve scope first: omit `org_id` for the personal workspace. For an
  organization, call `ttai:list_organizations` and pass the chosen `id` as
  `org_id`; an app created there belongs to it and is visible to its members.
- Load each tool's input schema before calling; the live `tools/list` wins.
- Read [references/app-structure.md](references/app-structure.md) before writing
  any file.

## Tool map

| Intent             | Tool                                                                   |
| ------------------ | ---------------------------------------------------------------------- |
| Discover           | `ttai:list_apps`, `ttai:get_app` (`app_id`)                            |
| Create             | `ttai:create_app`                                                      |
| Edit catalog entry | `ttai:update_app` (`title`, `description`, `visibility`, `is_enabled`) |
| Read / write files | `ttai:get_app_files`, `ttai:write_app_files`                           |
| Editor / preview   | `ttai:get_app_url`                                                     |
| Remove             | `ttai:delete_app` — destructive, confirm first                         |

## Workflow

### 1. Design

One round of questions; skip what is stated: what the app shows, which values
the voice agent changes mid-call, who may open it (me / my organization /
public), and which Scenario uses it. Split before writing:

- **Code** — `src/**`, fixed; the agent never edits it.
- **Data** — one small, flat JSON file the agent rewrites whole.

### 2. Create

`ttai:create_app` with `app_name`, `title`, optional `description`, and optional
`visibility` (`private` default, `org`, `public`).

- `app_name`: lowercase letters, digits, and hyphens; 2–64 characters; starts
  with a letter or digit. Sessions look apps up by name, so pick one no other
  app in `ttai:list_apps` uses, and never a built-in name (`canvas`, `maxx`,
  `brainstorm`).
- Inside an organization the app always starts as `org`, and every member sees
  it whatever `visibility` says. To open it to everyone, set `public` afterwards
  with `ttai:update_app`.
- Keep the returned `id`; every later call takes it as `app_id`.

### 3. Write files

`ttai:write_app_files` with `app_id` and a map of **package-relative** paths
(`spec.json`, `src/App.tsx`, `data/board.json`) to contents.

- `spec.json` **must** list every other file in `files`. Unlisted files never
  load, and a listed file that is missing stops the whole app from loading.
  Adding a file means adding it to `files` in the same write.
- Writes upsert per path and delete nothing. Later edits send only the changed
  files.

### 4. Verify

`ttai:get_app_files` and check: `spec.json` parses; every `files` entry was
returned; every `filesystem.editableGlobs` pattern matches an existing file;
`src/App.tsx` is listed and default-exports a component.

### 5. Open

`ttai:get_app_url` → give the user the **editor** link (sign-in required;
iterate on code) and the **preview** link (render check). Preview links expire
after 20 minutes; request a fresh one instead of reusing it. The preview link
carries an access token, so share it only with the user.

### 6. Attach to a Scenario

Fetch the Scenario first. `ttai:update_scenario` merges `tools_config.tools` per
tool, so other tools stay as they are, but it replaces `tool_settings` whole:
copy the existing `agent_desktop.tool_settings` keys (such as `mode`) into your
write. Send `id` and:

```json
{
  "tools_config": {
    "tools": {
      "agent_desktop": {
        "should_register": true,
        "tool_settings": { "apps": { "idea-board": true } }
      }
    }
  }
}
```

`apps` maps `app_name` to `true`; the first key is the app that opens at start.
Then add usage rules to `ai_instructions`: which file the agent may rewrite and
when to show the app. The app must be enabled and visible to the session's
participant; otherwise the session silently falls back to the default canvas.

### 7. Iterate or retire

- Content change → `ttai:write_app_files` on the data file only.
- Hide → `ttai:update_app` with `is_enabled: false`; Scenarios that list it fall
  back to the canvas.
- `ttai:delete_app` → confirm, and warn which Scenarios still list the app.

## Rules

- Confirm scope and visibility before creating or widening access.
- No secrets, tokens, or personal data in app files.
- The agent's edits during a session stay in that session. Every new session
  starts from the stored files, including code changes.
- Do not invent `spec.json` keys; unknown keys are ignored.

## Checklist

- [ ] `spec.json` `files` lists every file, including `src/App.tsx`
- [ ] Data file is the only `editableGlobs` target
- [ ] `spec.json` `instructions` name the exact file to rewrite
- [ ] Re-read verified; user has editor + preview links
- [ ] `agent_desktop.tool_settings` merged, not replaced; `ai_instructions`
      updated
- [ ] Destructive or visibility-widening actions confirmed

## Key Files

- [references/app-structure.md](references/app-structure.md) — package layout,
  `spec.json`, worked example, limits
