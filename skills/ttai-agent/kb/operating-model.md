# TTAI Agent operating model

`ttai-agent` is an **intermediate intelligence layer** between a person's
intent and their Tough Tongue AI entities. It turns an ambiguous request into
a scoped, evidenced action plan; an MCP client, coding agent, or web
conversation host performs the action.

It is knowledge, not a second API client. Never assume a terminal, browser,
filesystem, or a particular model/provider.

## Contents

- Consumer contract
- Portable context packet
- Context and capability protocol
- Scope and focus
- Operation loop
- Static handbook

---

## Consumer contract

Every consumer provides intent plus any verified context it already has. The
layer returns a decision that a consumer can explain and execute.

| Input | Output |
| --- | --- |
| Human request | Intent: orient, inspect, author, change, run, analyze, or share |
| Verified account / workspace context | Selected personal or organization scope |
| Named resource or active screen | Focused entity and its ID |
| Consumer capabilities | A safe tool/action plan |
| Live tool result | Evidence-backed next step or a precise blocker |

**Coding agent:** reads the linked knowledge and calls `ttai:` tools.
**Web conversational agent:** injects the same verified context, renders the
answer, and delegates tool calls to its server-side adapter.

Neither consumer should duplicate entity rules in its system prompt. Load the
smallest relevant handbook page instead.

## Portable context packet

Pass only verified, relevant facts. This is an optional handoff shape between a
web host, coding agent, and later turns; it is not a Tough Tongue AI resource
or a replacement for live authorization.

```json
{
  "intent": "author | inspect | run | analyze",
  "scope": { "kind": "organization", "org_id": "verified-id" },
  "focus": { "kind": "scenario", "id": "verified-id" },
  "account": { "source": "authenticated-host", "observed_at": "ISO-8601" },
  "consumer": { "can_call_ttai": true, "can_render_actions": true }
}
```

Use `personal` scope without an `org_id`; a focus may be a `scenario`,
`session`, or `none`. Omit unknown fields rather than guessing them. A web
host may add a verified profile or entitlement summary inside `account`; an
MCP-only consumer should leave it absent. Never reuse this packet across
users, accounts, or organizations.

## Context and capability protocol

Start any stateful request with this sequence. Reuse a verified context packet
from the same user and scope; do not re-fetch it on every turn.

1. **Resolve scope.** If no current scope is known, call
   `ttai:list_organizations`. Omit `org_id` for the personal workspace.
   Before a write, ask only if personal versus organization is materially
   ambiguous.
2. **Resolve focus.** A scenario ID is authoritative. A supplied name requires
   `ttai:list_scenarios` before a read or write. An account-level request has
   no focused scenario.
3. **Separate support from entitlement.**
   - **Supported**: Tough Tongue AI has the entity or operation.
   - **Available**: this consumer can call the required tool.
   - **Allowed**: the user's role, plan, and balance permit it.
   - **Requested**: the user has actually asked to do it.
4. **Load evidence.** Read the target entity, relevant handbook page, and the
   live tool schema before a mutation.
5. **Act and explain.** State the selected scope, changed entity, result, and
   any limitation without exposing private fields or credentials.

The server is authoritative for entitlement. Never claim that a feature is
available merely because the platform supports it. A `403` or balance error
is a real blocker, not a cue to retry with invented parameters.

### Current public-MCP account facts

| Need | Current source | Do not infer |
| --- | --- | --- |
| Organization memberships | `ttai:list_organizations` | An organization ID or role |
| Wallet minutes | `ttai:get_balance` | An organization wallet balance |
| Product subscribers | `ttai:list_subscriptions` | The caller's plan or feature gate |
| Effective plan / feature gates | Not published by the public MCP today | Ocean, SIP, meeting-bot, or multimodal entitlement |
| User profile | Not published by the public MCP today | User ID, email, or other identity data |

`list_subscriptions` lists people subscribed to scenarios or collections
created by the caller. It does **not** return the caller's platform
subscription tier. If a web host has an authenticated account/entitlement
object, it may pass that verified data into its own context packet; do not
fabricate equivalent data in an MCP-only client.

## First-run orientation

When someone says "get started", "is MCP working?", or "what can I do?",
perform a short read-only orientation:

1. **Verify connection.** Call `ttai:list_organizations`. On a missing tool or
   `401`, load [../features/mcp/connect.md](../features/mcp/connect.md); never
   ask for a token in chat.
2. **Take a lightweight inventory.** If connected, list scenarios and recent
   sessions with schema-supported small limits. State only what the live
   response establishes: personal versus organization scope, whether scenarios
   exist, and whether there is session evidence to inspect.
3. **Route to the smallest next workflow.**
   - No scenario or a named new goal → `scenario-maker`.
   - Existing scenario to refine or run → `scenario-maker`.
   - Existing session evidence to interpret → `session-analyst`.
   - Browser walkthrough to record → `browser-demo-builder`.

If the person already named a job, skip broad inventory and load the relevant
workflow. Never create, update, call, schedule, or share anything during
orientation without a separate explicit request.

## Scope and focus

A **workspace** is personal or organization-scoped. A **focus** is the
resource the human is discussing now. Focus is a conversation concept, not a
new Tough Tongue AI resource.

| Focus | First read | Usual next action |
| --- | --- | --- |
| Scenario | `ttai:get_scenario` | Create, update, share, or run |
| Sessions for a scenario | `ttai:list_sessions` | Read evidence or analyze |
| SIP call | `ttai:list_sip_trunks` / calls | Place or inspect a call |
| Meeting bot | scenario + meeting details | Schedule or inspect |
| Account / workspace | Organizations and relevant lists | Recommend a workflow |

Do not change a focused scenario while the request is only about learning
from it. Existing session evidence can justify a later, separate refinement.

## Operation loop

Use the same loop in a coding-agent chat and a web conversational surface:

1. **Understand** — restate the outcome and identify the operation type.
2. **Ground** — resolve scope, focus, permissions, and facts from live data.
3. **Choose** — select the narrowest resource and channel that meets the goal.
4. **Prepare** — load the relevant knowledge page and live tool schema.
5. **Execute** — make the smallest requested change; destructive or
   external-facing operations need explicit intent.
6. **Verify** — re-read mutable resources where practical. Sessions and
   post-processing can be asynchronous.
7. **Hand back** — distinguish facts, recommendation, action taken, and
   remaining user decisions.

Advice-only requests stop after **Choose**. The intelligence layer does not
turn an explanation into a write, phone call, or meeting-bot deployment.

## Static handbook

Use these files as the durable product handbook. Live schemas and live tool
results win whenever they differ.

| Question | Load |
| --- | --- |
| What entities and scopes exist? | [entities/](entities/index.md) |
| How should an agent be authored? | [scenario-authoring.md](scenario-authoring.md) |
| Which scenario fields mean what? | [entities/scenario/](entities/scenario/index.md) |
| Which channel runs a scenario? | [entities/scenario-engine.md](entities/scenario-engine.md) |
| Which MCP action is available? | [../features/mcp/](../features/mcp/index.md) |
| Which interaction pattern fits? | [scenario-recipes/](scenario-recipes/index.md) |

## Key Files

- [../SKILL.md](../SKILL.md) — entry point and loading map
- [entities/key-control-entities.md](entities/key-control-entities.md) — account facts
- [scenario-authoring.md](scenario-authoring.md) — durable authoring principles
- [../features/mcp/connect.md](../features/mcp/connect.md) — live MCP conventions
