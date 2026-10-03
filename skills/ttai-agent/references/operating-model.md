# Operating model

How to turn a person's request into a scoped, evidenced action on their Tough
Tongue AI workspace. The same protocol serves a coding agent calling `ttai:`
tools and a web chat host that injects verified context and delegates tool calls
to its own adapter. Never assume a terminal, browser, filesystem, or a
particular model provider.

## Contents

- Consumer contract and context packet
- Context and capability protocol
- Account facts available through MCP
- First-run orientation
- Scope and focus
- Operation loop

## Consumer contract and context packet

Each input maps to one output:

- **Human request** → intent: orient, inspect, author, change, run, analyze, or
  share.
- **Verified account / workspace context** → the selected personal or
  organization scope.
- **Named resource or active screen** → the focused entity and its ID.
- **Consumer capabilities** → a safe tool/action plan.
- **Live tool result** → an evidence-backed next step or a precise blocker.

A host may hand over only verified, relevant facts in this optional shape. It is
a handoff convention, not a Tough Tongue AI resource or an authorization:

```json
{
  "intent": "author | inspect | run | analyze",
  "scope": { "kind": "organization", "org_id": "verified-id" },
  "focus": { "kind": "scenario", "id": "verified-id" },
  "account": { "source": "authenticated-host", "observed_at": "ISO-8601" },
  "consumer": { "can_call_ttai": true, "can_render_actions": true }
}
```

Personal scope has no `org_id`. Focus may be `scenario`, `session`, or `none`.
Omit unknown fields. A web host may add a verified profile or entitlement
summary under `account`; an MCP-only consumer leaves it out. Never reuse a
packet across users, accounts, or organizations.

## Context and capability protocol

Reuse verified context from the same user and scope; do not re-fetch each turn.
At the first Tough Tongue AI action in a conversation, call
`ttai:read_guide` unless its guide is already in
context.

1. **Resolve scope.** No current scope → `ttai:list_resources(organizations)`. Pass the
   returned opaque `id` as `org_id` (never a slug); omit it for personal. Ask
   only when personal vs organization is materially ambiguous for a write.
2. **Resolve focus.** A Scenario ID is authoritative. A title → one
   `ttai:list_resources(scenarios, query: "<title>")`, then `ttai:get_resource(scenarios)`.
   Account-level requests have no focus.
3. **Separate four facts.**
   - **Supported** — Tough Tongue AI has the entity or operation.
   - **Available** — this consumer can call the required tool.
   - **Allowed** — the user's role, plan, and balance permit it.
   - **Requested** — the user actually asked for it.
4. **Load evidence.** Read the target entity, the relevant reference, and the
   live tool schema before a mutation.
5. **Act and explain.** State scope, changed entity, result, and any limitation,
   without exposing private fields or credentials.

The server is authoritative for entitlement. A `403` or balance error is a real
blocker, not a cue to retry with invented parameters.

## Account facts available through MCP

- **Organization memberships** — `ttai:list_resources(organizations)`. Do not infer
  permissions beyond the returned role.
- **Minutes balance** — `ttai:get_resource(account)`. Do not infer whether a balance
  grants a feature.
- **Effective capabilities** — `ttai:get_resource(account)`. Do not infer price,
  wallet, payment provider, or billing history.
- **User profile** — not exposed by MCP. Do not infer user ID, email, or other
  identity data.

If `ttai:get_resource(account)` is absent from `tools/list`, say the deployment
does not publish that fact. Never fabricate account data in an MCP-only client.

## First-run orientation

For "get started", "is MCP working?", or "what can I do?", stay read-only:

1. Call `ttai:read_guide`. Missing tools or `401` →
   [mcp/clients.md](mcp/clients.md); never ask for a token in chat.
2. Call `ttai:list_resources(organizations)`.
3. Take a light inventory: `ttai:list_resources(scenarios)` and `ttai:list_resources(sessions)`
   with small limits (`include_total: true` only when the count matters). Report
   only what the responses show.
4. Route to the next job:
   - New goal or no Scenario → create
     ([scenario/workflow.md](scenario/workflow.md)).
   - Existing Scenario to change or fix → edit or refine (same file).
   - Session evidence across a team → the `ttai-session-analyst` skill.
   - Browser walkthrough to record → the `ttai-browser-demo-builder` skill.

If the person already named a job, skip inventory. Never create, update, call,
schedule, or share during orientation without a separate request.

## Scope and focus

A **workspace** is personal or an organization. A **focus** is the resource
under discussion — a conversation concept, not a stored resource.

Each focus lists its first read, then the usual next action:

- **Scenario** — known ID or one title search → create, update, share, or run.
- **Sessions for a Scenario** — `ttai:list_resources(sessions)` → read evidence or
  analyze.
- **SIP call** — `ttai:list_resources(sip-trunks)`, `ttai:list_resources(bots)` → place or
  inspect a call.
- **Meeting bot** — `ttai:list_resources(bots)` → schedule or inspect.
- **Account / workspace** — organizations and small typed lists → recommend a
  workflow.

Do not change a focused Scenario while the request is only about learning from
it. Session evidence can justify a later, separate refinement.

## Operation loop

1. **Understand** — restate the outcome and the operation type.
2. **Ground** — resolve scope, focus, permissions, and facts from live data.
3. **Choose** — the narrowest resource and channel that meets the goal.
4. **Prepare** — load the relevant reference and the live tool schema.
5. **Execute** — the smallest requested change; confirm external or destructive
   actions first ([mcp/tools.md](mcp/tools.md#confirm-before-acting)).
6. **Verify** — re-read what changed. Sessions and post-processing are async.
7. **Hand back** — separate facts, recommendation, action taken, and open
   decisions.

Advice-only requests stop after **Choose**.

## Key Files

- [mcp/tools.md](mcp/tools.md) — tool catalog, read policy, confirmations
- [entities/account-and-access.md](entities/account-and-access.md) — workspaces,
  plans, sharing
- [scenario/workflow.md](scenario/workflow.md) — Scenario writes
