# MCP tools

The `ttai` MCP server (`https://api.toughtongueai.com/api/public/mcp`) is the
action surface: 15 tools, named `verb_resource`. Tools
appear as `ttai:<name>` (some clients show `mcp__ttai__<name>`). The connected
`tools/list` and each tool's input schema are authoritative; this file is the
map.

## Contents

- Tools
- Reading: resource types
- Read recipes
- Confirm before acting
- Updating a Scenario
- FileBases
- Tool notes
- Directory endpoint

## Tools

Records are read through two tools; each write family keeps its own tools
because risk hints are per tool.

| Do               | Tools                                                    |
| ---------------- | -------------------------------------------------------- |
| Guide            | `ttai:read_guide`                                        |
| Read records     | `ttai:list_resources`, `ttai:get_resource`               |
| Read a schema    | `ttai:get_schema` (`"scenario"`)                         |
| Scenarios        | `ttai:create_scenario`, `ttai:update_scenario`           |
| Sessions         | `ttai:upload_session`, `ttai:analyze_session`            |
| FileBases        | `ttai:create_filebase`, `ttai:write_filebase`                |
| Bots             | `ttai:schedule_bot`, `ttai:cancel_bot` (meeting and phone) |
| Access           | `ttai:create_token`, `ttai:authenticate_browser`         |

Skip any tool prefixed `int_` or titled "Internal…": it is not part of the
public API. Every tool except `ttai:read_guide` and `ttai:get_schema` accepts an optional `org_id`; omit it for the personal
workspace. Rate limit: 30 calls per minute per token; respect a live
rate-limit response.

## Reading: resource types

`ttai:list_resources(type, query?, filters?, include_fields?, cursor?, limit?,
include_total?)` lists; `ttai:get_resource(type, id?, filters?,
include_fields?, paths?)` reads one record or a singleton. The server's MCP
instructions name every type, and `ttai:read_guide` lists each type's filter
names — read it when unsure. These skills write
`ttai:list_resources(sessions)` as shorthand for `ttai:list_resources` with
`type: "sessions"`; extra words in the parentheses name the argument to set.

| Type                                   | Holds                                                    |
| -------------------------------------- | -------------------------------------------------------- |
| `organizations`                        | orgs you belong to; a returned `id` is your `org_id`     |
| `scenarios` · `scenario-versions`      | agent definitions and their history                      |
| `sessions`                             | runs and results; `include_fields: [transcript]` for text |
| `bots`                                 | meeting bots and phone calls (`kinds`: meeting, phone)   |
| `filebases`                            | app and knowledge FileBases (filter `kind`); `paths` returns file text |
| `sip-trunks` · `collections`           | dial-out trunks, courses                                 |
| `knowledge-bases` · `custom-functions` | legacy RAG bases, HTTP tools                             |
| `avatar` · `user-preferences`          | system faces, your preferences                           |
| `account` · `usage`                    | balance + entitlements, analytics (no `id`)              |
| `platform`                             | public config (no `id`)                                  |

- `filters` is a map of the type's own filter names, e.g.
  `{"scenario_ids": ["…"], "statuses": ["completed"]}`. One value where a list
  is expected is accepted.
- `query` is free text where the type supports it; on `scenarios` it is the
  title search.
- Typed lists return `{items, next_cursor, total?}`; default `limit` 25, max
  100. Pass `next_cursor` back as `cursor`.
- Directory types (`sip-trunks`, `knowledge-bases`, `custom-functions`,
  `avatar`, `user-preferences`) return common metadata; `data` is empty until
  you request that type's advertised `include_fields`. Credentials are never
  returned.
- Date filters accept ISO 8601 dates or timestamps; a date-only lower bound
  means 00:00 UTC, an upper bound 23:59:59 UTC. No timezone means UTC.

## Read recipes

- **Exact count** → `list_resources` with `limit: 1, include_total: true`. Never
  page through results to count. For an org-wide Session count pass `org_id`;
  do not list every Scenario and echo its IDs.
- **Scenario by title** → `list_resources(scenarios, query: "…")`, then
  `get_resource(scenarios, id)`.
- **Scenario history** (needs edit access) →
  `list_resources(scenario-versions, filters: {scenario_id})`, then
  `get_resource(scenarios, id, filters: {version_id})`. Omit the filter for
  current.
- **Sessions by person, date, or learning result** → `list_resources(sessions,
  filters: {user_email | from_date | to_date | hasLearning | is_org | page})` —
  these switch to the enriched legacy list (`is_org: true` for the org-wide
  admin view).
- **Transcript** → `get_resource(sessions, id, include_fields: [transcript])`.
  Several known IDs → `list_resources(sessions, filters: {ids: […]},
  include_fields: [...])`.
- **Calls and bots** → `list_resources(bots, filters: {kinds: [phone]})`
  (`include_fields: [failure]` for why a call failed).
- **Usage, durations, top Scenarios, member activity** →
  `get_resource(usage, filters: {is_org_wide, start_date, end_date})`.
- **Balance and plan limits** → `get_resource(account)`.

## Confirm before acting

State the exact target (phone number, meeting URL, Scenario ID, audience,
timing) and get the user's go-ahead before any of these. Never dial or dispatch
a bot from inferred data. The server's tool annotations flag the same risks
(destructive: `update_scenario`, `write_filebase`, `cancel_bot`; open world:
every tool below except `write_filebase` and `create_token`); confirm
even when a client hides them.

- `ttai:schedule_bot` — `kind: "phone"` dials real phones (one unscheduled
  entry rings immediately); `kind: "meeting"` sends a visible bot into a real
  meeting.
- `ttai:cancel_bot` — destructive; cancels a call or meeting bot only before it
  starts.
- `ttai:update_scenario` — overwrites the fields you send; changes the live
  agent.
- `ttai:write_filebase` — replaces or deletes files in a FileBase.
- `ttai:analyze_session` — fills only missing results (never re-scores); may
  fire the Scenario's post-session hooks.
- `ttai:upload_session` — fetches a remote recording when `recording_url` is
  given.
- `ttai:authenticate_browser` — opens a live browser on the Scenario's saved
  login profile.
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
  - `save_as_version` (1–100 characters) archives the prior state under that
    label before applying the change.
- `get_resource(scenarios, id)` omits linked-resource IDs
  (`knowledge_base_ids`, `custom_function_ids`, `pre_connect`); the
  create/update response returns the full authoring state.
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

- `ttai:create_filebase(kind: app | knowledge, name, title, description?)` → an
  empty FileBase in the current org or personal space. `name` is a slug, unique per space; for
  an app it is the runtime name and the live agent's tool prefix.
- `ttai:write_filebase(id, files, delete)` → whole-file upserts by relative path
  (no leading `/`), then deletes, and returns the new `revision`. For apps,
  `spec.json` `files` is kept equal to the tree automatically.
- List: `list_resources(filebases, filters: {kind: app}, query?)`. Read before
  rewriting: `get_resource(filebases, id, paths: [...])` (≤ 20 files) returns
  `{filebase, files}`.
- View link: `create_token({type: "filebase_view", filebase_id,
  valid_for_hours})` (≤ 24 h) renders the FileBase without signing in (apps
  today).
- The ttai-agent-apps skill covers the app package and the `controls` contract.

## Tool notes

- `ttai:read_guide` returns the server's usage guide, the same text as the
  `ttai://guide/mcp-agent` resource. Call it once at the first action in a
  conversation unless the guide is already in context.
- On connect the server also sends MCP `instructions`: a short index of this
  map. `tools/list` is ordered by importance (the guide first, `int_` tools
  last).
- `get_resource(platform)` needs no special access. System avatars:
  `list_resources(avatar, filters: {type: static | hybrid | live, provider})`.
- `get_resource(account)` returns `balance` (personal minutes, or the
  organization's shared balance plus the caller's quota when quotas are
  enabled) and `entitlements` (role, plan tier identity/status, effective
  limits — no price, wallet, or billing history).
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
- `session_credentials: "bot_uc"` gives a call or bot run the deploying user's
  identity for the Scenario's MCP tools. Omit it unless requested.
- `ttai:upload_session` is for automations that use Tough Tongue AI's analysis
  and extraction on calls held elsewhere (dialer, Gong, Zoom): it uploads a
  transcript or `recording_url` as a completed session, then the Scenario's
  rubric and extraction variables run in the background. No live agent runs.
- `ttai:analyze_session` queues analysis/extraction as configured on the
  Scenario. Poll `get_resource(sessions, id, include_fields: [processing])`
  until `post_session_status.state` leaves `running` (`idle` = done, `failed` =
  error), then read the results.
- `ttai:create_token` takes one `request` with a `type` and returns `{type,
  token, expires_at, url}`:
  - `type: "scenario_access"` — `scenario_id`, `valid_for_hours` (1–168),
    optional `email` or `self_billed: true` (runs and bills as you; omit
    `email`). `url` is an iframe embed.
  - `type: "filebase_view"` — `filebase_id`, `valid_for_hours` (≤ 24). `url`
    renders the FileBase.
- `ttai:authenticate_browser` needs edit access and a Scenario whose `browser`
  tool is registered. It returns a single-use `embed_url` (about 20 minutes)
  where a human logs in on the Scenario's saved browser profile; logins persist
  there. `enable_recording_tool: true` adds a tab for recording replayable
  steps. Hand the link to the user; the agent cannot log in for them.
- Collections are read-only through MCP.

## Directory endpoint

`https://api.toughtongueai.com/api/public/mcp/directory` is a policy-limited
surface for connector directories. It has the same 15 tools, but its
`ttai:schedule_bot` accepts only `kind: "meeting"` (no outbound dialing). Inspect
`tools/list` before inferring what it can do. Place calls from a full MCP
client.

## Key Files

- [clients.md](clients.md) — connect a client, OAuth, troubleshooting
- [../operating-model.md](../operating-model.md) — scope and action loop
- [../entities/scenario-engine.md](../entities/scenario-engine.md) — channels
  and results
