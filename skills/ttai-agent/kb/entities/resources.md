# Resources

Everything in a Tough Tongue AI workspace that is not the control plane
([key-control-entities.md](key-control-entities.md)) is a **resource**.
Resources share one permission / sharing universe — see
[auth-and-sharing.md](auth-and-sharing.md).

MCP does not yet publish a JSON schema per resource. Until it does, load
the relevant `ttai` tool's input schema before writing. Do not invent
fields.

## Contents

- Primary authoring resources
- Run / results resources
- Operational attachments
- What MCP cannot set today

---

## Primary authoring resources

| Resource                | What it is                                        | MCP (high level)                                        |
| ----------------------- | ------------------------------------------------- | ------------------------------------------------------- |
| **Scenario**            | Voice (or text) agent definition                  | create / update / list / get / generate / access tokens |
| **Collection (course)** | Named group of scenarios — library or course path | list / get (create/update in Studio today)              |

Scenario deep-dive: [scenario/](scenario/index.md).

## Run / results resources

| Resource      | What it is                                           | MCP                                        |
| ------------- | ---------------------------------------------------- | ------------------------------------------ |
| **Session**   | One run of a scenario — transcript, scores, analysis | list / get / batch / create / post_process |
| **Analytics** | Aggregates over sessions                             | `get_analytics`                            |

How runs are **started** across channels: [scenario-engine.md](scenario-engine.md).

## Operational attachments

| Resource                                               | What it is                                        | MCP                            |
| ------------------------------------------------------ | ------------------------------------------------- | ------------------------------ |
| **SIP trunk**                                          | Phone line for outbound / inbound                 | list                           |
| **SIP call**                                           | One phone call (trunk + scenario + E.164)         | list / create / batch / delete |
| **Meeting bot**                                        | Joins Google Meet, Zoom, or Teams as the scenario | list / schedule / delete       |
| **Browser context**                                    | Logged-in browser for scripted product demos      | `authenticate_browser`         |
| **Access token (SAT)**                                 | Time-limited link to a private scenario           | create (scenario or self)      |
| **Static assets / custom functions / knowledge bases** | Studio-attached capabilities                      | **not** writable via MCP today |

## What MCP cannot set today

Attach in Scenario Studio (not invent ids over MCP):

- `knowledge_base_ids`
- `custom_function_ids`
- Some Studio-only scenario `type` values (e.g. meet-assist)

Voice-agent mid-session MCP servers use `mcp_server_ids` on the scenario
when the catalog id is known — do not invent ids.

## Next

- Permissions → [auth-and-sharing.md](auth-and-sharing.md)
- Scenario fields → [scenario/](scenario/index.md)
- Tool table → [../../features/mcp/tools-by-resource.md](../../features/mcp/tools-by-resource.md)

## Key Files

- [../operating-model.md](../operating-model.md) — resolve focus and scope
- [key-control-entities.md](key-control-entities.md) — workspace context
- [scenario-engine.md](scenario-engine.md) — runtime channels
- [../../features/mcp/tools-by-resource.md](../../features/mcp/tools-by-resource.md) — live actions
