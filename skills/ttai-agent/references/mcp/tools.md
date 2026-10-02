# MCP tools

The `ttai` MCP server (`https://api.toughtongueai.com/api/public/mcp`) is the
action surface. Tools appear as `ttai:<name>` (some clients show
`mcp__ttai__<name>`). The connected `tools/list` and each tool's input schema
are authoritative; this file is the map.

## Contents

- Catalog by resource
- Read policy (V3 first)
- Confirm before acting
- Updating a Scenario
- Tool notes
- Directory endpoint

## Catalog by resource

Read tools first, then write tools; "none" means MCP has no tool of that kind.

- **Orientation** — read: `ttai:callme_before_using_tough_tongue_mcp`,
  `ttai:get_public_config`. Write: none.
- **Account** — read: `ttai:list_organizations`, `ttai:v3_get_entitlements`,
  `ttai:get_balance`, `ttai:get_analytics`. Write: none.
- **Scenario** — read: `ttai:v3_list_scenarios`, `ttai:v3_get_scenario_version`,
  `ttai:v3_list_scenario_versions`, `ttai:list_scenarios` (search),
  `ttai:get_scenario` (legacy). Write: `ttai:create_scenario`,
  `ttai:update_scenario`.
- **Sharing** — read: none. Write: `ttai:create_scenario_access_token`,
  `ttai:create_self_scenario_access_token`.
- **Session** — read: `ttai:v3_list_sessions`, `ttai:list_sessions` (legacy
  filters), `ttai:get_session`, `ttai:get_sessions_batch`. Write:
  `ttai:create_session`, `ttai:post_process_session`.
- **Resource Directory** — read: `ttai:v3_list_resource_types`,
  `ttai:v3_list_resources`, `ttai:v3_get_resource`. Write: none.
- **Phone (SIP)** — read: `ttai:list_sip_trunks`, `ttai:list_sip_calls`. Write:
  `ttai:create_sip_call`, `ttai:create_sip_batch`, `ttai:delete_sip_call`.
- **Meeting bot** — read: `ttai:list_meeting_bots`. Write:
  `ttai:schedule_meeting_bot`, `ttai:delete_meeting_bot`.
- **Collection** — read: `ttai:list_collections`, `ttai:get_collection`. Write:
  none.
- **Browser** — read: none. Write: `ttai:authenticate_browser`.
- **Agent Desktop app** — read: `ttai:list_apps`, `ttai:get_app`,
  `ttai:get_app_files`, `ttai:get_app_url`. Write: `ttai:create_app`,
  `ttai:update_app`, `ttai:write_app_files`, `ttai:delete_app`.

Skip any tool prefixed `int_` or titled "Internal…": it is not part of the
public API. Every account tool accepts an optional `org_id`; omit it for the
personal workspace. Rate limit: 30 calls per minute per token; respect a live
rate-limit response.

## Read policy (V3 first)

V3 replaces older reads whenever its contract fits. If a named V3 tool is
missing from `tools/list`, use the visible legacy fallback and report the
mismatch; never invent a tool.

Each need maps to the tool to use:

- **Scenario inventory, filters, counts** → `ttai:v3_list_scenarios`.
- **Known Scenario ID → content** → `ttai:v3_get_scenario_version` (omit
  `version_id`/`version_name` for current).
- **Scenario version history** (needs edit access) →
  `ttai:v3_list_scenario_versions`.
- **Scenario by title / free text** → one `ttai:list_scenarios(search=...)`,
  then back to V3 with the ID.
- **Session inventory, Scenario/status filters, counts** →
  `ttai:v3_list_sessions`.
- **Session by person (`user_email`), date, or learning result** →
  `ttai:list_sessions` (`is_org: true` for org-wide admin view).
- **Full transcript / detail** → `ttai:get_session`; several known IDs →
  `ttai:get_sessions_batch`.
- **Usage, durations, top Scenarios, member activity** → `ttai:get_analytics`.
- **Custom Functions, Knowledge Bases, SIP trunks, avatars, user preferences** →
  `ttai:v3_list_resource_types` → `ttai:v3_list_resources` /
  `ttai:v3_get_resource`.

Notes:

- Exact count: the V3 list with `limit: 1, include_total: true`. Never page
  through results to count. For an org-wide Session count pass `org_id`; do not
  list every Scenario and echo its IDs.
- V3 lists return cursor pages (`next_cursor`); default `limit` 25, max 100.
- `ttai:v3_list_sessions` returns base records (ID, Scenario/version, status,
  timestamps). Request `participant`, `recording`, `transcript`, `evaluation`,
  or `processing` via `include_fields` only when needed.
- Resource Directory records carry common metadata; `data` is empty until you
  request that type's advertised `include_fields`. It never returns credentials.
- `ttai:v3_get_scenario_version` omits linked-resource IDs
  (`knowledge_base_ids`, `custom_function_ids`, `pre_connect`); the
  create/update response returns the full authoring state.
- Date filters accept ISO 8601 dates or timestamps; a date-only lower bound
  means 00:00 UTC, an upper bound 23:59:59 UTC. No timezone means UTC.
- Legacy `ttai:get_scenario` exists only for clients that need its old shape.
- Only the V3 reads and `ttai:list_organizations` publish an output schema; read
  other results by the field names they return.
- Metadata (`meta_*`) and operator-style date filters are not available through
  MCP; `ttai:list_sessions` and `ttai:list_scenarios` ignore them.

## Confirm before acting

State the exact target (phone number, meeting URL, Scenario ID, audience,
timing) and get the user's go-ahead before any of these. Never dial or dispatch
a bot from inferred data. The server's tool annotations flag the same risks
(destructive: `ttai:update_scenario`, `ttai:delete_sip_call`,
`ttai:delete_meeting_bot`; open world: every tool below except the access
token); confirm even when a client hides them.

Each tool and why it needs confirmation:

- `ttai:create_sip_call`, `ttai:create_sip_batch` — dials real phones; an
  unscheduled call starts immediately.
- `ttai:schedule_meeting_bot` — a visible bot joins a real meeting.
- `ttai:delete_sip_call`, `ttai:delete_meeting_bot` — destructive; only before
  the call/bot starts.
- `ttai:update_scenario` — overwrites the fields you send.
- `ttai:post_process_session` — fills only missing results (never re-scores);
  may fire the Scenario's post-session hooks.
- `ttai:create_session` — fetches a remote recording when `recording_url` is
  given.
- `ttai:authenticate_browser` — opens a live browser on the Scenario's saved
  login profile.
- `ttai:create_scenario_access_token` — grants access; an email can create or
  reuse a user record.

Also confirm `ttai:delete_app` (Scenarios may still list the app) and
`ttai:update_app` to `public` (opens the app to everyone).

## Updating a Scenario

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
- Type rules: `default` is the normal agent; `super` needs `stages` with a
  non-empty first stage and flows, and a Landmass model; `quiz`, `coding`
  (`coding_question`), and `meet_assist` need their schema-defined
  configuration. Never send `super_agent`.
- Plan checks run on write: Scenario count limits, Ocean model access, and
  `multimodal_analysis` can return `403`.
- Edits apply to new sessions only.

## Tool notes

- `ttai:callme_before_using_tough_tongue_mcp` ("Get Tough Tongue Usage Guide")
  returns the server's usage guide, the same text as the
  `ttai://guide/mcp-agent` resource. Call it once at the first action in a
  conversation unless the guide is already in context. V3 detail lives in the
  `ttai://guide/v3-tools` resource.
- On connect the server also sends MCP `instructions`: a short index of this
  tool map. `tools/list` is ordered by importance (the guide first, `int_` tools
  last).
- `ttai:get_public_config` needs no login. System avatars are not a separate
  tool: `ttai:v3_list_resources` with resource type `avatar`, `type` `static`
  (default), `hybrid`, or `live`.
- `ttai:get_balance` returns personal minutes, or the organization's shared
  balance plus the caller's quota when quotas are enabled.
- `ttai:v3_get_entitlements` returns role (in an organization), plan tier
  identity/status, and effective limits — no price, wallet, or billing history.
- SIP: `ttai:create_sip_call` takes `scenario_id`, `sip_trunk_id`, E.164
  `phone_number`, optional `scheduled_ts`, `dynamic_vars`, and
  `scenario_version_id`. `ttai:create_sip_batch` queues many `entries`
  asynchronously. Both need a SIP-enabled plan, enough balance, and an
  `outgoing` trunk.
- Meeting bots join Google Meet, Zoom, or Teams visibly. Required:
  `scenario_id`, `meeting_url`, `meeting_provider`
  (`google-meet`|`zoom`|`teams`), and an AI-identifying `bot_name`. Needs a paid
  plan and edit access to the Scenario.
- `session_credentials: "bot_uc"` gives a call or bot run the deploying user's
  identity for the Scenario's MCP tools. Omit it unless requested.
- `ttai:create_session` ingests an external transcript or `recording_url` as a
  completed session and analyzes it per the Scenario's settings.
- `ttai:post_process_session` queues analysis/extraction as configured on the
  Scenario. Poll `ttai:get_session` until `post_session_status.state` leaves
  `running` (`idle` = done, `failed` = error), then read the results.
- `ttai:authenticate_browser` needs edit access and a Scenario whose `browser`
  tool is registered. It returns a single-use `embed_url` (about 20 minutes)
  where a human logs in on the Scenario's saved browser profile; logins persist
  there. `enable_recording_tool: true` adds a tab for recording replayable
  steps. Hand the link to the user; the agent cannot log in for them.
- Collections are read-only through MCP.

## Directory endpoint

`https://api.toughtongueai.com/api/public/mcp/directory` is a policy-limited
surface for connector directories. It omits outbound SIP dialing
(`ttai:create_sip_call`, `ttai:create_sip_batch`) but keeps other writes.
Inspect `tools/list` before inferring what it can do. Place calls from a full
MCP client.

## Key Files

- [clients.md](clients.md) — connect a client, OAuth, troubleshooting
- [../operating-model.md](../operating-model.md) — scope and action loop
- [../entities/scenario-engine.md](../entities/scenario-engine.md) — channels
  and results
