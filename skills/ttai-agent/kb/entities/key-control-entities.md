# Account, workspace, and capacity

The identity and commercial plane of the Tough Tongue AI datastore. MCP
exposes only part of it: select a workspace and inspect limited account facts;
do not provision users, organizations, or plans through `ttai` tools.

## Contents

- User
- Organization
- Billing and capability visibility
- How agents should treat them

---

## User

The signed-in human behind OAuth or a PAT. Owns a **personal** workspace and
may belong to one or more **organizations**.

MCP does not expose a `get_user` tool. Do not infer an ID, email, plan, or
other profile fields from successful authentication. A web consumer may supply
verified profile data from its authenticated application session; an MCP-only
consumer cannot fetch it today.

## Organization

A shared workspace for a team. Scenarios, sessions, SIP trunks, and
collections created with `org_id` live on that org.

| Action             | Tool / rule                                              |
| ------------------ | -------------------------------------------------------- |
| List memberships   | `ttai:list_organizations`                                |
| Scope later calls  | Pass `org_id` (and `is_org: true` where the tool has it) |
| Personal workspace | Omit `org_id`                                            |

Ambiguous team vs personal? Ask which workspace before writes.

## Billing and capability visibility

| Resource                       | Tool                      | Notes                                                                                        |
| ------------------------------ | ------------------------- | -------------------------------------------------------------------------------------------- |
| Personal wallet                | `ttai:get_balance`        | Available usage minutes; not an organization wallet                                          |
| Product subscribers            | `ttai:list_subscriptions` | Subscribers to the caller's paid scenarios / collections; **not** the caller's platform plan |
| Effective plan / feature gates | Not exposed               | Do not infer model, SIP, meeting-bot, or analysis entitlement                                |

Plan changes and top-ups happen in the web app at
[app.toughtongueai.com](https://app.toughtongueai.com), not via MCP.
The live server is authoritative when an operation is denied for role, plan,
or balance.

## Agent rules

1. Reuse a current, verified consumer context when it already includes scope.
2. Otherwise call `ttai:list_organizations` early in a stateful thread.
3. Do not invent org ids — only use ids returned by tools.
4. Query balance only when the requested operation may consume it; treat it as
   informational and never claim you changed a plan.
5. Treat a `403` or balance error as a precise entitlement blocker, not proof
   of the user's tier.
6. Sharing and who can run a scenario → [auth-and-sharing.md](auth-and-sharing.md).

## Key Files

- [../operating-model.md](../operating-model.md) — context and capability protocol
- [resources.md](resources.md) — workspace resources
- [auth-and-sharing.md](auth-and-sharing.md) — permissions and sharing
