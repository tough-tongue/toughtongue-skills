# Scenario engine: runs and results

How Tough Tongue AI runs a Scenario across channels and how to read results. A
Scenario is a definition; a **session** is one run of it.

## Contents

- Channels
- Starting a run
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
- **Many calls** — `ttai:schedule_bot` with several entries (async; track with
  `ttai:list_resources(bots)` by `batch_id`).
- **Meeting** — `ttai:schedule_bot` with `kind: "meeting"`, URL, provider, and an
  AI-identifying `bot_name`.
- **Live browser demo** — browser tool + `ttai:authenticate_browser` login link;
  recorded steps via the `ttai-browser-demo-builder` skill.

Confirm the target first
([../mcp/tools.md](../mcp/tools.md#confirm-before-acting)).

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
