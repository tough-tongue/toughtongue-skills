# Resources

Everything in a Tough Tongue AI workspace that is not the control plane
([key-control-entities.md](key-control-entities.md)) is a **resource**.
Resources share one permission / sharing universe — see
[auth-and-sharing.md](auth-and-sharing.md).

For bounded generic discovery, first call `ttai:v3_list_resource_types`. Its
catalog advertises safe resource fields and the matching
`ttai:v3_list_resources` / `ttai:v3_get_resource` calls. Load the relevant
`ttai` tool's input schema before writing; do not invent fields.

## Contents

- Primary authoring resources
- Run / results resources
- Operational attachments
- Sensitive boundary

---

## Primary authoring resources

| Resource                | What it is                                        | MCP (high level)                                                       |
| ----------------------- | ------------------------------------------------- | ---------------------------------------------------------------------- |
| **Scenario**            | Voice (or text) agent definition                  | V3 compact list/version discovery; legacy create / update / detail    |
| **Collection (course)** | Named group of scenarios — library or course path | list / get (create/update in Studio today)                             |

Scenario deep-dive: [scenario/](scenario/index.md).

## Run / results resources

| Resource      | What it is                                           | MCP                                        |
| ------------- | ---------------------------------------------------- | ------------------------------------------ |
| **Session**   | One run of a scenario — transcript, scores, analysis | list / get / batch / create / post_process |
| **Analytics** | Aggregates over sessions                             | `get_analytics`                            |

How runs are **started** across channels: [scenario-engine.md](scenario-engine.md).

## Operational attachments

| Resource                | What it is                                        | MCP                                                                 |
| ----------------------- | ------------------------------------------------- | ------------------------------------------------------------------- |
| **SIP trunk**           | Phone line for outbound / inbound                 | list; safe metadata in the Resource Directory                     |
| **SIP call**            | One phone call (trunk + scenario + E.164)         | list / create / batch / delete                                    |
| **Meeting bot**         | Joins Google Meet, Zoom, or Teams as the scenario | list / schedule / delete                                          |
| **Browser context**     | Logged-in browser for scripted product demos      | `authenticate_browser`                                             |
| **Access token (SAT)**  | Time-limited link to a private scenario           | create (scenario or self)                                         |
| **Custom Function**     | Scenario-invoked HTTP function                    | discover safely, then attach by ID through Scenario authoring      |
| **Knowledge Base**      | Processed sources a Scenario can use              | discover safely, then attach by ID through Scenario authoring      |

## Sensitive boundary

The generic Resource Directory exposes purpose-built safe metadata, never
credentials, URLs, source chunks, or arbitrary model fields. It is not generic
CRUD. Use IDs returned by a trusted workspace call and validate the
`create_scenario` or `update_scenario` tool schema before attaching a Knowledge
Base, Custom Function, or voice-agent MCP server.

## Next

- Permissions → [auth-and-sharing.md](auth-and-sharing.md)
- Scenario fields → [scenario/](scenario/index.md)
- Tool table → [../../features/mcp/tools-by-resource.md](../../features/mcp/tools-by-resource.md)

## Key Files

- [../operating-model.md](../operating-model.md) — resolve focus and scope
- [key-control-entities.md](key-control-entities.md) — workspace context
- [scenario-engine.md](scenario-engine.md) — runtime channels
- [../../features/mcp/tools-by-resource.md](../../features/mcp/tools-by-resource.md) — live actions
