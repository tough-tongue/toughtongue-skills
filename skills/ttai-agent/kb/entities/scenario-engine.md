# Scenario engine

The **scenario engine** is how Tough Tongue AI **runs** a scenario (voice
agent) across channels and how you **fetch results** afterward.

A scenario is only a definition. A **session** is one execution.

## Contents

- Channels
- Starting a run
- Fetching results
- Agent checklist

---

## Channels

| Channel            | What the user experiences                   | How it is usually started                                                    |
| ------------------ | ------------------------------------------- | ---------------------------------------------------------------------------- |
| **Web app**        | Browser practice / roleplay at the run link | Open `https://app.toughtongueai.com/run/<scenario_id>`                       |
| **iframe / embed** | Same runtime inside a host product          | Embed the run experience; respect [auth-and-sharing.md](auth-and-sharing.md) |
| **SIP / phone**    | Outbound or inbound PSTN call               | `ttai:list_sip_trunks` → `ttai:create_sip_call` (or batch)                   |
| **Meeting bot**    | Agent joins **Google Meet, Zoom, or Teams** | `ttai:schedule_meeting_bot`                                                  |
| **Incoming**       | Inbound telephony / configured entry points | Trunk + scenario routing in the product; MCP lists trunks/calls              |

Choose the model pipeline for the voice and interaction, not the channel:
Galaxy, Ocean, and Landmass scenarios can run over SIP. Ocean is a strong
realtime starting point for phone-call realism; Landmass is appropriate for
full STT/LLM/TTS control or a selected external voice. See
[scenario/model-selection.md](scenario/model-selection.md).

Silent `meet_assist` overlays are Studio-managed scenario types. MCP can run
an existing scenario through a meeting bot but cannot create or set that type.

## Starting a run

| Goal                  | Do this                                                                                  |
| --------------------- | ---------------------------------------------------------------------------------------- |
| Human practice        | Return the run link (mint a SAT if private)                                              |
| Outbound call         | Trunk id + scenario id + E.164 via `create_sip_call`                                     |
| Many calls            | `create_sip_batch`                                                                       |
| Meeting               | `schedule_meeting_bot` with the meeting URL                                              |
| Scripted browser demo | Scenario + **browser-demo-builder** workflow; `authenticate_browser` when login persists |

## Fetching results

| Need            | Tool                                                     |
| --------------- | -------------------------------------------------------- |
| Recent runs     | `ttai:list_sessions` (prefer a larger `limit`)           |
| One run         | `ttai:get_session`                                       |
| Many ids        | `ttai:get_sessions_batch`                                |
| Re-run analysis | `ttai:post_process_session` (async — poll `get_session`) |
| Aggregates      | `ttai:get_analytics`                                     |

Session rows carry transcripts, scores, and report-card style fields when
analysis has finished. Edits to the scenario do not rewrite old sessions.

## Agent checklist

1. Confirm the scenario id and workspace.
2. Pick the channel; do not invent trunk or meeting ids — list first.
3. After the run, pull sessions; wait for analysis before claiming scores.
4. Diagnose failures with evidence → workflow skill **scenario-maker**
   (refine) or **session-analyst** (team reports).

## Key Files

- [scenario/model-selection.md](scenario/model-selection.md) — pipeline stamps
- [resources.md](resources.md) — operational attachments
- [auth-and-sharing.md](auth-and-sharing.md) — access and sharing
- [../../features/mcp/tools-by-resource.md](../../features/mcp/tools-by-resource.md) — action catalog
