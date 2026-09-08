# Entities

Tough Tongue AI is a **datastore**. Everything you create or observe is an
entity under a workspace (personal account or organization).
Start with [../operating-model.md](../operating-model.md) to resolve that
workspace and the request's focused entity.

## Contents

| File                                               | Topic                                                    |
| -------------------------------------------------- | -------------------------------------------------------- |
| [key-control-entities.md](key-control-entities.md) | Users, organizations, billing (mostly read-only via MCP) |
| [resources.md](resources.md)                       | Resource universe — scenarios, courses, SIP, bots, …     |
| [auth-and-sharing.md](auth-and-sharing.md)         | Visibility, SATs, org membership, permissions            |
| [scenario/](scenario/index.md)                     | Scenario resource (model, control, prose)                |
| [scenario-engine.md](scenario-engine.md)           | Orchestration channels + fetching results                |

## Mental split

- **Control entities** — identity and money. You rarely "create" these via
  MCP; you **select** a workspace (`org_id` or personal) and read balance /
  subscriptions.
- **Resources** — everything else. Same auth/sharing rules. Scenarios and
  courses (collections) are the primary authoring surfaces; sessions are
  runs; SIP trunks, meeting bots, browser contexts, and tokens are
  operational attachments.

## Key Files

- [../operating-model.md](../operating-model.md) — scope and focus protocol
- [key-control-entities.md](key-control-entities.md) — identity and capacity facts
- [resources.md](resources.md) — resource universe
- [scenario/index.md](scenario/index.md) — scenario authoring model
