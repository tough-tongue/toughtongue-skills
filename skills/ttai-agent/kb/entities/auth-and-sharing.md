# Auth and sharing

How Tough Tongue AI decides **who can see and run** a resource. Applies to
scenarios and collections; sessions inherit the scenario's workspace.

## Contents

- Workspaces (personal vs org)
- Scenario visibility (`is_public`, passcode)
- Scenario access tokens (SAT)
- Org membership
- Agent checklist

---

## Workspaces

| Workspace    | How MCP scopes it                            |
| ------------ | -------------------------------------------- |
| Personal     | Omit `org_id`                                |
| Organization | Pass `org_id` from `ttai:list_organizations` |

Resources created in an org are owned by that org. Do not move resources
across orgs via MCP (no such tool).

## Scenario visibility

| Field                       | Effect                                                                  |
| --------------------------- | ----------------------------------------------------------------------- |
| `is_public: true` (default) | Anyone with the run link can start a session                            |
| `is_public: false`          | Needs a SAT (or Studio share flow) to open                              |
| `passcode`                  | Optional extra gate before start                                        |
| `analysis_access`           | `"default"` \| `"always"` \| `"never"` — who sees post-session analysis |

Practice URL: `https://app.toughtongueai.com/run/<scenario_id>`.

## Scenario access tokens (SAT)

Short-lived (about 1 hour) tokens for private or embed use.

| Tool                                     | Use                                       |
| ---------------------------------------- | ----------------------------------------- |
| `ttai:create_scenario_access_token`      | Mint a link for a scenario you can access |
| `ttai:create_self_scenario_access_token` | Self-serve variant where supported        |

Return the token/URL to the user; do not log secrets in long-lived notes.

## Org membership

MCP assumes the OAuth/PAT identity. There is no MCP tool to invite users or
change roles — that stays in the web app. If a tool returns 403, the actor
lacks access to that org or resource; switch workspace or ask the user to
fix membership in the app.

## Agent checklist

1. Resolve workspace (`list_organizations` → `org_id` or personal).
2. Before sharing a private scenario, mint a SAT or confirm `is_public`.
3. Never invent access tokens or org ids.
4. Embed / iframe runners still use the same scenario id + visibility rules
   — see [scenario-engine.md](scenario-engine.md).

## Key Files

- [../operating-model.md](../operating-model.md) — selected-scope protocol
- [key-control-entities.md](key-control-entities.md) — workspace memberships
- [scenario-engine.md](scenario-engine.md) — run channels
- [../../features/mcp/connect.md](../../features/mcp/connect.md) — SAT tool use
