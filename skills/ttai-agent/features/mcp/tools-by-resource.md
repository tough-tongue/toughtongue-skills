# Tools by resource

Qualified names: `ttai:tool_name`. Load schemas before calling.

| Resource                               | Read                                                 | Write                                                                                                                          |
| -------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| MCP orientation                        | `ttai:callme_before_using_tough_tongue_mcp`          | —                                                                                                                              |
| Scenario                               | `ttai:v3_list_scenarios`, `ttai:list_scenarios` (legacy search), `ttai:get_scenario`, `ttai:v3_get_scenario_version`, `ttai:v3_list_scenario_versions` | `ttai:create_scenario`, `ttai:update_scenario`, `ttai:generate_scenario`, `ttai:create_scenario_access_token`, `ttai:create_self_scenario_access_token` |
| Session                                | `ttai:v3_list_sessions`, `ttai:list_sessions` (legacy filters), `ttai:get_session`, `ttai:get_sessions_batch` | `ttai:create_session`, `ttai:post_process_session` |
| Generic Resource Directory             | `ttai:v3_list_resource_types`, `ttai:v3_list_resources`, `ttai:v3_get_resource` | — |
| Account / analytics                    | `ttai:list_organizations`, `ttai:get_analytics`, `ttai:get_balance`, `ttai:v3_get_entitlements` | — |
| Phone                                  | `ttai:list_sip_trunks`, `ttai:list_sip_calls`                  | `ttai:create_sip_call`, `ttai:create_sip_batch`, `ttai:delete_sip_call`                                                                       |
| Meeting bot                            | `ttai:list_meeting_bots`                                  | `ttai:schedule_meeting_bot`, `ttai:delete_meeting_bot`                                                                                   |
| Collection                             | `ttai:list_collections`, `ttai:get_collection`                 | —                                                                                                                              |
| Browser                                | —                                                    | `ttai:authenticate_browser`                                                                                                         |

## Notes

- For an inventory, count, or compact current-record page, use V3 first.
  `ttai:v3_list_scenarios` and `ttai:v3_list_sessions` return cursor pages; add
  `include_total: true` only when an exact authorized count is needed. Use
  legacy lists for free-text Scenario search or Session person/date/full-detail
  filters that V3 does not expose.
- Start generic-resource work with `ttai:v3_list_resource_types`. It defines
  available types, safe optional fields, common record fields, and the matching
  list/detail calls. It never returns credentials.
- Outbound SIP: `ttai:list_sip_trunks` first, then `ttai:create_sip_call` (or batch)
  with trunk + scenario + E.164. Some directory connectors omit write SIP
  tools — use a full MCP client when placing calls.
- Meeting bots join **Google Meet, Zoom, or Teams**.
- For Knowledge Base or Custom Function IDs, use the Resource Directory to
  discover safe records, then load the Scenario write schema. Never invent IDs
  or expect credentials to be readable.
- MCP does not expose the caller's profile. `ttai:v3_get_entitlements` reports
  effective capability limits, but no price, wallet, subscription, or
  payment-provider data.
- Resource meanings → [../../kb/entities/resources.md](../../kb/entities/resources.md).
- Channels / results → [../../kb/entities/scenario-engine.md](../../kb/entities/scenario-engine.md).

Complete install + FAQ: repo [MCP.md](../../../../MCP.md).

## Key Files

- [connect.md](connect.md) — authentication and scope conventions
- [../../kb/operating-model.md](../../kb/operating-model.md) — capability protocol
- [../../kb/entities/resources.md](../../kb/entities/resources.md) — resource meanings
