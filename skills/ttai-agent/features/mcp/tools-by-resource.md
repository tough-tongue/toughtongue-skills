# Tools by resource

Qualified names: `ttai:tool_name`. Load schemas before calling.

| Resource                               | Read                                                 | Write                                                                                                                          |
| -------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Scenario                               | `list_scenarios`, `get_scenario`                     | `create_scenario`, `update_scenario`, `generate_scenario`, `create_scenario_access_token`, `create_self_scenario_access_token` |
| Session                                | `list_sessions`, `get_session`, `get_sessions_batch` | `create_session`, `post_process_session`                                                                                       |
| Account / analytics                    | `list_organizations`, `get_analytics`, `get_balance` | —                                                                                                                              |
| Paid scenario / collection subscribers | `list_subscriptions`                                 | —                                                                                                                              |
| Phone                                  | `list_sip_trunks`, `list_sip_calls`                  | `create_sip_call`, `create_sip_batch`, `delete_sip_call`                                                                       |
| Meeting bot                            | `list_meeting_bots`                                  | `schedule_meeting_bot`, `delete_meeting_bot`                                                                                   |
| Collection                             | `list_collections`, `get_collection`                 | —                                                                                                                              |
| Browser                                | —                                                    | `authenticate_browser`                                                                                                         |

## Notes

- Outbound SIP: `list_sip_trunks` first, then `create_sip_call` (or batch)
  with trunk + scenario + E.164. Some directory connectors omit write SIP
  tools — use a full MCP client when placing calls.
- Meeting bots join **Google Meet, Zoom, or Teams**.
- MCP **cannot** set `knowledge_base_ids` or `custom_function_ids` — attach
  those in Scenario Studio.
- MCP does not expose the caller's profile, effective platform plan, or
  feature gates. `list_subscriptions` is a list of people subscribed to the
  caller's paid products, not an account-plan lookup.
- Resource meanings → [../../kb/entities/resources.md](../../kb/entities/resources.md).
- Channels / results → [../../kb/entities/scenario-engine.md](../../kb/entities/scenario-engine.md).

Complete install + FAQ: repo [MCP.md](../../../../MCP.md).

## Key Files

- [connect.md](connect.md) — authentication and scope conventions
- [../../kb/operating-model.md](../../kb/operating-model.md) — capability protocol
- [../../kb/entities/resources.md](../../kb/entities/resources.md) — resource meanings
