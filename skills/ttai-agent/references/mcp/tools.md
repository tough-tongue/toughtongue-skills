# MCP tools

The `ttai` MCP server (`https://api.toughtongueai.com/api/public/mcp`) is the
action surface: 17 public tools, named `verb_resource`. Tools
appear as `ttai:<name>` (some clients show `mcp__ttai__<name>`). The connected
`tools/list` and each tool's input schema are authoritative; this file is the
map.

## Contents

- Tools
- Reading: workspace
- Reading: resource types
- Read recipes
- Confirm before acting
- Updating a Scenario
- FileBases
- Tool notes
- Directory endpoint

## Tools

The workspace is read through one tool, records through two; each write family
keeps its own tools because risk hints are per tool.

| Do               | Tools                                                    |
| ---------------- | -------------------------------------------------------- |
| Guide            | `ttai:read_guide`                                        |
| Read workspace   | `ttai:get_workspace_info` (`sections`)                   |
| Read records     | `ttai:list_resources`, `ttai:get_resource`               |
| Read a schema    | `ttai:get_schema` (`"scenario"`)                         |
| Scenarios        | `ttai:create_scenario`, `ttai:update_scenario`, `ttai:delete_scenario_version` |
| Org settings     | `ttai:update_org` (description, logo, cover image)       |
| Sessions         | `ttai:upload_session`, `ttai:analyze_session`            |
| FileBases        | `ttai:create_filebase`, `ttai:write_filebase`                |
| Bots             | `ttai:schedule_bot`, `ttai:cancel_bot` (meeting and phone) |
| Access           | `ttai:create_token`, `ttai:authenticate_browser`         |

Skip any tool prefixed `int_` or titled "Internal…": it is not part of the
public API. Every tool except `ttai:read_guide` accepts an optional `org_id`; omit it for the personal
workspace. First calls: `ttai:read_guide`, then `ttai:get_workspace_info`.

## Reading: workspace

`ttai:get_workspace_info(org_id?, sections?, query?, start_date?, end_date?)`
answers who you are and where you work. It returns `scope` (personal |
organization), `org_id`, your `role`, and the sections you ask for (others
`null`):

| Section         | Holds                                                         | Default |
| --------------- | ------------------------------------------------------------- | ------- |
| `me`            | your id, name, email, preferences                             | yes     |
| `organizations` | every membership: `id` (your `org_id`), name, slug, role, `quota_enabled`, `editable` (what `ttai:update_org` changes) | yes |
| `plan`          | effective entitlements for the scope (SIP, models, limits)    | yes     |
| `balance`       | personal or org wallet; your quota when quotas are on         | yes     |
| `members`       | org only: id, email, name, role, `quota_available`; `query` filters name/email (max 100, `total`) | no |
| `usage`         | by Scenario; org editors add a daily series + `member_usage`; `start_date` / `end_date` | no |

Roles: `view` < `edit` (Admin in the app) < `owner`; `no_access` = end users. Rate limit: 30 calls per minute per token; respect a live
rate-limit response.

## Reading: resource types

`ttai:list_resources(type, query?, filters?, include_fields?, cursor?, limit?,
include_total?)` lists; `ttai:get_resource(type, id, filters?,
include_fields?, paths?, download?)` reads one record (`id` is always required). The server's MCP
instructions name every type, and `ttai:read_guide` lists each type's filter
names — read it when unsure. These skills write
`ttai:list_resources(sessions)` as shorthand for `ttai:list_resources` with
`type: "sessions"`; extra words in the parentheses name the argument to set.

| Type                                   | Holds                                                    |
| -------------------------------------- | -------------------------------------------------------- |
| `scenarios` · `scenario-versions`      | agent definitions and their history                      |
| `sessions`                             | runs and results; `include_fields: [transcript]` for text |
| `bots`                                 | meeting bots and phone calls (`kinds`: meeting, phone)   |
| `filebases`                            | app and knowledge FileBases (filter `kind`); `paths` returns file text |
| `sip-trunks` · `collections`           | dial-out trunks, courses                                 |
| `knowledge-bases` · `custom-functions` | legacy RAG bases, HTTP tools                             |
| `avatar`                               | system faces                                             |

- `filters` is a map of the type's own filter names, e.g.
  `{"scenario_ids": ["…"], "statuses": ["completed"]}`. One value where a list
  is expected is accepted.
- `query` is free text where the type supports it; on `scenarios` it is the
  title search.
- Typed lists return `{items, next_cursor, total?}`; default `limit` 25, max
  100. Pass `next_cursor` back as `cursor`.
- Directory types (`sip-trunks`, `knowledge-bases`, `custom-functions`,
  `avatar`) return common metadata; `data` is empty until you request that
  type's `include_fields`. Secret values come back masked.
- Date filters accept ISO 8601 dates or timestamps; a date-only lower bound
  means 00:00 UTC, an upper bound 23:59:59 UTC. No timezone means UTC.

## Read recipes

- **Exact count** → `list_resources` with `limit: 1, include_total: true`. Never
  page through results to count. For an org-wide Session count pass `org_id`;
  do not list every Scenario and echo its IDs. `bots` has no `include_total`:
  page with `cursor` to count.
- **Scenario by title** → `list_resources(scenarios, query: "…")`, then
  `get_resource(scenarios, id)`.
- **Scenario history** (needs edit access) →
  `list_resources(scenario-versions, filters: {scenario_id})`, then
  `get_resource(scenarios, id, filters: {version_id})` or `{version_name}`.
  Omit the filter for current; its list row has `is_current: true`.
- **Recently changed Scenarios** → `list_resources(scenarios, filters:
  {updated_at_gte})` (add `updated_at_lte` for a window).
- **Sessions by person, date, or learning result** → `list_resources(sessions,
  filters: {user_email | from_date | to_date | hasLearning | is_org})` — these
  switch to the enriched legacy list (`is_org: true` for the org-wide admin
  view). It takes at most one `scenario_ids` value, no `query` or
  `include_fields`, and `limit` up to 500.
- **Transcript** → `get_resource(sessions, id, include_fields: [transcript])`.
  Several known IDs → `list_resources(sessions, filters: {ids: […]},
  include_fields: [...])`.
- **Calls and bots** → `list_resources(bots, filters: {kinds: [phone]})`
  (`include_fields: [failure]` for why a call failed).
- **Usage, durations, top Scenarios, member activity** →
  `get_workspace_info(sections: [usage], start_date, end_date)`.
- **Balance and plan limits** → `get_workspace_info(sections: [plan, balance])`.
- **Org members and quotas** → `get_workspace_info(org_id, sections: [members])`.

## Confirm before acting

State the exact target (phone number, meeting URL, Scenario ID, audience,
timing) and get the user's go-ahead before any of these. Never dial or dispatch
a bot from inferred data. The server's tool annotations flag the same risks
(destructive: `cancel_bot`, `update_scenario`, `delete_scenario_version`,
`write_filebase`; open world: `schedule_bot`, `cancel_bot`,
`authenticate_browser`, `upload_session`, `analyze_session`,
`update_scenario`); confirm even when a client hides them.

- `ttai:schedule_bot` — `kind: "phone"` dials real phones (one unscheduled
  entry rings immediately); `kind: "meeting"` sends a visible bot into a real
  meeting.
- `ttai:cancel_bot` — destructive; cancels a call or meeting bot only before it
  starts.
- `ttai:update_scenario` — overwrites the fields you send; changes the live
  agent.
- `ttai:delete_scenario_version` — removes one archived version for good. The
  current version is never deleted; an archive a live Session or active Bot
  runs is refused. Finished Sessions on a deleted archive read the current
  version. Needs edit access.
- `ttai:update_org` — changes the description, logo, or cover image every
  organization member sees; show the current and new values.
- `ttai:write_filebase` — replaces or deletes files in a FileBase.
- `ttai:analyze_session` — fills only missing results (never re-scores); may
  fire the Scenario's post-session hooks.
- `ttai:upload_session` — fetches a remote recording when `recording_url` is
  given.
- `ttai:authenticate_browser` — opens a live browser on the Scenario's browser
  profile.
- `ttai:create_token` — grants access by link; a `scenario_access` email can
  create or reuse a user record.

## Updating a Scenario

- Fetch the field schema once: `ttai:get_schema("scenario")`. Both
  write tools take one `scenario` object with those fields.
- `ttai:create_scenario` rejects `id`; requires `name` and `ai_instructions`.
  Omitting `tools_config` installs a broad default tool set — send it
  explicitly.
- `ttai:update_scenario` requires `id`. Omitted fields stay unchanged; sent
  fields replace stored values, with these merge rules:
  - `strategy`, `appearance`, `memory`, `session_analysis`, `transcribe_config`,
    `meeting_config`, `access` merge one level: each key you send replaces that
    key (sending `strategy.silence` keeps `strategy.conductor`; sending
    `strategy.conductor` replaces all its `messages`).
  - `tools_config.tools.<tool>` merges `should_register`,
    `add_to_system_prompt`, and `tool_settings` per tool; `tool_settings` is
    replaced whole. Read it, merge, and write the full object, or you drop keys
    such as the browser tool's saved `contextId` and recorded `steps`.
  - Everything else is replaced whole: `ai_model_config` (resend `tts_voice_id`
    and the rest of the stamp), `stages`, `rubrik`, `processed_rubrik`,
    `auto_update_config`, ID lists, `user_metadata`.
  - `save_as_version` (1–100 characters) archives the prior state, then
    applies the change. The label names the new version you are saving; the
    archived copy keeps its own earlier label ("Initial version" if it had
    none).
- Prune archives: `list_resources(scenario-versions, filters: {scenario_id})`
  → skip the `is_current: true` row → show the rest and confirm →
  `ttai:delete_scenario_version(scenario_id, version_id)` per archive. Report
  any refusal: a live Session or active Bot still runs that version.
- `get_resource(scenarios, id)` omits linked-resource IDs
  (`knowledge_base_ids`, `knowledge_filebase_ids`, `custom_function_ids`,
  `pre_connect`); the create/update response returns the full authoring state.
- Type rules: `default` is the normal agent; `super` needs `stages` with a
  non-empty first stage and flows, and a Landmass model; `quiz`, `coding`
  (`coding_question`), and `meet_assist` need their schema-defined
  configuration. Never send `super_agent`.
- Plan checks run on write: Scenario count limits, Ocean model access, and
  `multimodal_analysis` can return `403`.
- Edits apply to new sessions only.

## FileBases

A FileBase is a folder of files with a `kind`: `app` (an Agent Desktop app) or
`knowledge` (plain files an agent reads). Read them as type `filebases`.

- `ttai:create_filebase(kind: app | knowledge, name, title, description?, visibility?)`
  → an empty FileBase in the current org or personal space. `name` is a slug,
  fixed at create, unique per space.
- Access: an org FileBase is visible to every member; its creator and role
  EDIT+ edit it. `visibility: "private"` keeps it to its creator; set it at
  create (no MCP tool changes it). Featured FileBases are read-only (`403`).
  List rows carry `visibility`, `featured`, `org_id` and `editable`; check
  `editable` before writing.
- `ttai:write_filebase(id, files?, edits?, upload?, commit?, delete?)` →
  upserts by relative path (no leading `/`), then deletes; one call bumps
  `revision` once. The receipt has `kind`, `written`, `deleted`, `files` (the
  tree after), `revision`, an advisory `warning`, and `missing`: convention
  files still to add. Write the entry file in the first call: `README.md` for
  an app, `index.md` for knowledge.
- Pick the mode by what you have; keep file bytes out of the conversation:

  | You have                          | Use                                                            |
  | --------------------------------- | -------------------------------------------------------------- |
  | A shell and the files on disk     | `upload: [paths]` → run the ticket's `script` → `commit`       |
  | A change to an existing text file | `edits: [{path, old, new}]` (`old` matches exactly once)       |
  | No shell, new small text files    | `files: {path: contents}` (binary files need a shell)          |
  | A shell, and want files locally   | `get_resource(filebases, id, paths?, download: true)` → `curl` |
  | No shell, and want to read files  | `get_resource(filebases, id, paths: [...])` (text, ≤ 20 files) |

- `upload` alone lands nothing: it returns a ticket with `upload_id`,
  `script` (POSIX `sh`; run it from the folder whose layout matches the
  paths), `form` (the same POST for any HTTP client), `expires_at` (15 minutes
  to upload), and `commit_by` (24 hours). Run the script, then call
  `write_filebase(id, commit: upload_id)`. A ticket commits once: if a commit
  fails, request a new upload. No shell to run the script → use `files`.
- Never retry an `edits` call that succeeded: it can apply again. `files`,
  `edits`, `commit`, and `delete` combine in one call; a path appears once.
  Caps per call: 200 paths, 20 MB a file, 30 MB total.
- `download: true` returns `downloads.urls` (path → GET URL for 1 hour; ≤ 64
  files, the whole tree without `paths`). Fetch each with
  `curl -sSf --create-dirs -o <path> '<url>'`, edit locally, then upload.
- List: `list_resources(filebases, filters: {kind}, query?)`. Read before
  changing: `get_resource(filebases, id, paths: [...])` (≤ 20 files) returns
  `{filebase, files}`.
- Apps: `README.md` (the agent's manual), `src/App.tsx`, and `data/**` the
  live agent rewrites; no `spec.json`. Results carry `view_url`, the preview
  page. Attach with the Scenario's `agent_desktop` tool. The ttai-agent-apps
  skill covers building, checking, and attaching apps.

### Knowledge FileBases

Not the legacy Knowledge Base (`knowledge-bases`, `knowledge_base_ids`, tool
`knowledge_base_search`), which is created in the web app only.

- Entry: root `index.md`, not `README.md`: what the base covers and which file
  answers what, linking every other file with relative paths (keep them working
  when files move). Keep it under 4 KB; a short index is inlined in the
  agent's prompt.
- Workflow: the user attaches PDFs or docs → distill them into focused
  markdown files plus `index.md` → `create_filebase(kind: knowledge)` →
  `write_filebase` (index first; `files`, or `upload` + `commit` from a shell)
  → confirm, then link the id on the Scenario.
- Link: `knowledge_filebase_ids` on `create_scenario` / `update_scenario`
  replaces the whole list, and a Scenario read omits it. Keep the ids the last
  create/update response showed. Current list unknown → say linking replaces
  any attached knowledge and get a go-ahead. Each id must be visible to you
  (yours, your org's, public, or featured; else `400`). Find featured bases
  with `list_resources(filebases, filters: {kind: knowledge})`.
- Sharing: the Scenario link is the grant. Anyone who can run the Scenario has
  the agent read the base, whatever its visibility. No `view_url` or
  `filebase_view` link.
- Runtime: no `tools_config` entry needed. The prompt names each linked base;
  the agent gets two tools:
  - `getKnowledgeBite(filepath, start?, lines?)` reads a line window of
    `<base name>/<path>`; a bare base name reads `index.md`.
  - `listKnowledgeBites(base)` lists the base's files with sizes.

## Tool notes

- `ttai:read_guide` returns the server's usage guide, the same text as the
  `ttai://guide/mcp-agent` resource. Call it once at the first action in a
  conversation unless the guide is already in context.
- The `ttai://reference/speech-support` resource lists which realtime model,
  voice provider (`tts_provider`), and speech recognizer (`stt_provider`)
  support each language. Read it before a non-English Scenario.
- On connect the server also sends MCP `instructions`: a short index of this
  map. `tools/list` is ordered by importance (the guide first, `int_` tools
  last).
- System avatars:
  `list_resources(avatar, filters: {type: static | hybrid | live, provider})`.
- `get_workspace_info` `balance` is personal minutes, or the org wallet plus
  the caller's quota when quotas are on; `plan` is role, tier identity/status,
  effective limits — no price or billing history.
- `ttai:schedule_bot` takes one `bot` object with `kind`:
  - `kind: "phone"` — `scenario_id`, `sip_trunk_id` (an outbound trunk from
    `list_resources(sip-trunks)`), `entries` of E.164 `phone_number` + optional
    `user_name` and `dynamic_vars`, optional `scheduled_ts` and
    `scenario_version_id`. One entry dials directly; several form a batch the
    dialer works through within minutes. Needs a SIP-enabled plan and enough
    balance.
  - `kind: "meeting"` — joins Google Meet, Zoom, or Teams visibly. Required:
    `scenario_id`, `meeting_url`, `meeting_provider`
    (`google-meet`|`zoom`|`teams`), and an AI-identifying `bot_name`. Needs a
    paid plan and edit access to the Scenario.
- `ttai:cancel_bot(id)` takes a bot id from `list_resources(bots)` and cancels
  either kind.
- `session_credentials: "bot_uc"` is a server-side setting: the call or bot
  runs the Scenario's attached MCP tools as the deploying Tough Tongue AI user.
  Nothing is read from the user's machine. Omit it unless requested.
- `ttai:upload_session(req_data)` is for automations that use Tough Tongue AI's analysis
  and extraction on calls held elsewhere (dialer, Gong, Zoom): it uploads a
  transcript or `recording_url` as a completed session, then the Scenario's
  rubric and extraction variables run in the background. No live agent runs.
- `ttai:analyze_session(session_id, req_data: {})` queues the missing
  analysis/extraction configured on the Scenario. `req_data` is required; its
  flags are ignored. Poll `get_resource(sessions, id, include_fields: [processing])`
  until `post_session_status.state` leaves `running` (`idle` = done, `failed` =
  error), then read the results.
- `ttai:create_token` takes one `request` with a `type` and returns `{type,
  token, expires_at, url}`:
  - `type: "scenario_access"` — `scenario_id`, `valid_for_hours` (1–168),
    optional `email` or `self_billed: true` (runs and bills as you; omit
    `email`). `url` is an iframe embed.
  - `type: "filebase_view"` — an app `filebase_id`, `valid_for_hours`
    (≤ 24). `url` opens the app without a sign-in.
- `ttai:authenticate_browser(req)` needs edit access and a Scenario whose `browser`
  tool is registered. It returns a single-use `embed_url` (about 20 minutes)
  where a human signs in to the Scenario's browser profile, then clicks
  **Save & close session** so the profile stays signed in.
  `enable_recording_tool: true` adds a tab for recording replayable steps.
  Hand the link to the user; the agent cannot log in for them.
- Collections are read-only through MCP.
- `ttai:update_org(org_id, settings)` needs role EDIT+ in the organization and
  changes only `description`, `logo_url`, and `cover_image_url` (http(s)
  URLs). Omit a field to keep it; `""` clears it. Read the current values from
  `get_workspace_info(sections: [organizations])` `editable`. Name, slug,
  billing, members, roles, domains, and SSO stay in the web app.

## Directory endpoint

`https://api.toughtongueai.com/api/public/mcp/directory` is a policy-limited
surface for connector directories. Its `ttai:schedule_bot` accepts only
`kind: "meeting"` (no outbound dialing), and its tool list can differ from the
full server's. Inspect `tools/list` before inferring what it can do. Place
calls from a full MCP client.

## Key Files

- [clients.md](clients.md) — connect a client, OAuth, troubleshooting
- [../operating-model.md](../operating-model.md) — scope and action loop
- [../entities/scenario-engine.md](../entities/scenario-engine.md) — channels
  and results
