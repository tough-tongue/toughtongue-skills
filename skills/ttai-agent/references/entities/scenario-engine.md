# Scenario engine: runs and results

How Tough Tongue AI runs a Scenario across channels and how to read results. A
Scenario is a definition; a **session** is one run of it.

## Contents

- Channels
- Starting a run
- Calling campaigns
- Reading results
- Agent checklist

## Channels

Each channel: what the user experiences, then how it starts.

- **Web app** — browser roleplay or agent at the run link. Starts at
  `https://app.toughtongueai.com/run/<scenario_id>`.
- **Embed** — the same runtime inside a host product (iframe). Starts from the
  `create_token` (`scenario_access`) `url`; same visibility rules.
- **Phone (SIP)** — outbound or inbound call. Starts with `ttai:list_resources(sip-trunks)`
  → `ttai:schedule_bot` with `kind: "phone"`.
- **Meeting bot** — the agent joins Google Meet, Zoom, or Teams. Starts with
  `ttai:schedule_bot` with `kind: "meeting"`.
- **Ingested** — an external transcript or recording, analyzed. Starts with
  `ttai:upload_session`.

Inbound calls route by trunk configuration in the web app; MCP lists trunks and
calls. Choose the model for the voice and interaction, not the channel
([../scenario/model-selection.md](../scenario/model-selection.md)). Phone runs
use only server-side tools ([../scenario/runtime.md](../scenario/runtime.md)).

Per-run variables: the run link takes `?t_<name>=value` for `{{ name }}` (the
web app otherwise prompts the user for missing ones); calls and bots take
`dynamic_vars`. Calls and bots also accept `scenario_version_id` to run an
archived version (`ttai:list_resources(scenario-versions)`).

`meet_assist` Scenarios are silent text overlays for meetings, not ordinary
voice agents; configure them only from the live schema.

## Starting a run

- **Human practice** — return the run link; mint a SAT if private.
- **One outbound call** — trunk ID + Scenario ID + E.164 number →
  `ttai:schedule_bot` with `kind: "phone"` and one entry (immediate unless
  `scheduled_ts`).
- **Many calls** — `ttai:schedule_bot` with several entries: a calling
  campaign (below).
- **Meeting** — `ttai:schedule_bot` with `kind: "meeting"`, URL, provider, and an
  AI-identifying `bot_name`.
- **Live browser demo** — browser tool + `ttai:authenticate_browser` login link;
  recorded steps via the `ttai-browser-demo-builder` skill.

Confirm the target first
([../mcp/tools.md](../mcp/tools.md#confirm-before-acting)).

## Calling campaigns

One `ttai:schedule_bot` call with `kind: "phone"` and many `entries` is a
batch: one Scenario, trunk, start time, and `call_policy` for every number.
Leads in different timezones go in separate batches.

**Prepare**

1. Check SIP access and minutes
   (`ttai:get_workspace_info(sections: [plan, balance])`), then list outbound
   trunks (`ttai:list_resources(sip-trunks)`) and let the user pick one.
2. Read the Scenario: its `{{ var }}` names and its outcome extraction
   variable, if any (such as `call_result`, with a value for voicemail).
3. Pin `scenario_version_id` to an archived version
   (`ttai:list_resources(scenario-versions)`) holding the content you will
   test, so edits made mid-campaign do not reach queued calls. If that content
   exists only as the current copy, a `ttai:update_scenario` with
   `save_as_version` archives it (a confirmed write).
4. Place one test call to the user's own number with that version and one
   lead's `dynamic_vars`. Read its transcript and extraction before dialing
   anyone else. Before relying on voicemail retries, let a second test call
   go to the user's voicemail and read the bot (`include_fields: [failure]`)
   and its session's extraction; build the retry policy on what they record.

**Entries** — E.164 `phone_number` (`+14155550100`), optional `user_name`,
and per-lead `dynamic_vars` (string values) keyed to the Scenario's
`{{ var }}` names. Use only the user's data; flag rows with invalid numbers
instead of guessing. `scheduled_ts` starts the batch (ISO time with an offset,
`2026-11-03T10:00:00+05:30`); omit it to start now.

**Office hours** — `call_policy.office_hours` is the window for first calls and
retries, in the callee's timezone:

- `timezone` — an IANA name (`Asia/Kolkata`, `America/New_York`), not an
  offset.
- `weekdays` — one or more of `mon` … `sun`.
- `start`, `end` — `HH:MM`; one window that does not cross midnight.

**Retries** — `call_policy.retry`:

- `max_retries` (1–5) after the first call; `delays_minutes` holds one delay
  per retry, each at least 1 (`[60, 60]` = two retries an hour apart).
- `retry_on_failed` (default `true`) — retries a call that fails (such as a
  busy line) or that the platform terminates.
- `extraction_conditions` (up to 10 `{variable, operator, value}`; operators
  `eq`, `neq`, `contains`, `gt`, `lt`) — retries when every condition matches
  the call's post-session extraction. Key them to the outcome variable, using
  the exact value its extraction returns (`call_result` `eq` `Voicemail`).
  Conditions are AND, so two `eq` values on one variable never both match; to
  retry several outcomes, list the outcomes that should stop retries with
  `neq` instead. Extraction must be on (`enable_extraction`,
  [../scenario/control.md](../scenario/control.md)). No outcome variable: say
  so and offer to add one before the campaign.
- `check_recent_calls: true` — before each retry, skips it when a newer call
  with the same number and Scenario (inbound, call-now, or another schedule) no
  longer matches the conditions. Use it when leads may call back.

**Voicemail** — one handler, never two. Answering machine detection
(`meeting_config.amd_config`) on: it hangs up or leaves its message, and the
prompt has no voicemail branch. AMD off: the prompt needs one. Details:
[../scenario/control.md](../scenario/control.md) (`meeting_config`).

**Confirm** — show, then wait for an explicit yes before `ttai:schedule_bot`:

- Scenario, pinned version label, and trunk.
- Number count, the first and last three numbers, and any rows left out.
- Start time and office hours in the callee's timezone (and the user's, if
  different).
- Retries in words ("up to 2 retries, 60 minutes apart, while `call_result` is
  `Voicemail`; failed calls retried") and the voicemail handling.

**Track and stop**

- Progress: `ttai:list_resources(bots, filters: {batch_ids: [...]},
  include_fields: [failure])`; narrow with `statuses` or `kinds`. Without the
  `batch_id`, filter by `scenario_ids` and `created_at_gte`. `bots` has no
  `include_total`: page with `cursor` to count.
- Stop: `ttai:cancel_bot(id)` for each call that has not started, after the
  user confirms; one call per bot.
- Results: each bot's `session_id` leads to the session and its extraction
  (`ttai:list_resources(sessions, filters: {ids}, include_fields:
  [extraction])`). Reports across the campaign: the `ttai-session-analyst`
  skill.

## Reading results

| Need                          | Tool                                          |
| ----------------------------- | --------------------------------------------- |
| Recent runs, counts           | `ttai:list_resources(sessions)` ¹              |
| Selected transcripts / scores | `ttai:list_resources(sessions)` with `ids` ²   |
| One full detail               | `ttai:get_resource(sessions)` ²               |
| By person or date window      | `ttai:list_resources(sessions)` legacy filters |
| Fill missing analysis         | `ttai:analyze_session` ³                      |
| Aggregates                    | `ttai:get_workspace_info` (`usage`)           |

1. Request optional fields only when needed.
2. Add `include_fields` for the transcripts or scores you need.
3. Then poll `ttai:get_resource(sessions)`.

Results appear only after configured processing finishes
(`session_analysis.is_auto_analysis`, `enable_extraction`). Scenario edits do
not rewrite past sessions.

## Agent checklist

1. Confirm Scenario ID and workspace.
2. List trunks or bots first; never invent trunk, call, or bot IDs.
3. Wait for analysis before claiming scores.
4. Diagnose behavior with [../scenario/runtime.md](../scenario/runtime.md), then
   refine ([../scenario/workflow.md](../scenario/workflow.md)) or hand team-wide
   reporting to the `ttai-session-analyst` skill.

## Key Files

- [../scenario/runtime.md](../scenario/runtime.md) — failure diagnosis
- [account-and-access.md](account-and-access.md) — access and sharing
- [../mcp/tools.md](../mcp/tools.md) — action catalog
