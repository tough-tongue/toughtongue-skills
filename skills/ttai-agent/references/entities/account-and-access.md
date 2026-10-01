# Account and access

Who you are, which workspace you act in, what your plan allows, and who can see
and run a Scenario. MCP selects workspaces and reads account facts; it does not
provision users, organizations, members, or plans.

## Contents

- User and workspaces
- Plan, balance, and entitlements
- Scenario visibility
- Scenario Access Tokens (SAT)
- Agent checklist

## User and workspaces

The signed-in human behind OAuth (or a `TTAI_PAT` in headless setups) owns a
**personal** workspace and may belong to **organizations**. MCP exposes no
profile tool: never infer a user ID, email, or plan from successful login.

- **Personal** — omit `org_id`.
- **Organization** — pass the opaque `id` from `ttai:list_organizations` as
  `org_id`.

Notes:

- Scenarios, sessions, calls, and bots created with `org_id` belong to that
  organization. No MCP tool moves resources between workspaces.
- `ttai:list_organizations` returns each organization's `id`, `name`, `slug`,
  and the caller's `role`. Never send the slug as `org_id`.
- Inviting members, changing roles, creating organizations, and plan changes
  happen in the web app at
  [app.toughtongueai.com](https://app.toughtongueai.com).
- A `403` means the caller lacks access to that organization or resource: switch
  workspace or have the user fix membership in the app.

## Plan, balance, and entitlements

- **Minutes balance** — `ttai:get_balance`. Personal balance, or the org's
  shared balance plus the caller's quota when quotas are on.
- **Effective plan and limits** — `ttai:v3_get_entitlements`. Role (in an org),
  tier identity/status, limits; no price or billing history.

Notes:

- Query balance only when the operation may consume it; it is informational.
- Plans gate some features (for example Scenario count, Ocean models, multimodal
  analysis). Check `ttai:v3_get_entitlements` or let the server answer; never
  guess a tier from a balance or a Scenario.

## Scenario visibility

- `is_public: true` (default) — anyone with the run link can start a session.
- `is_public: false` — needs a SAT or an in-app share to open.
- `passcode` — extra gate before start.
- `starts_at` / `ends_at` — availability window.
- `access` — `is_one_time_access`, `max_access_count`.
- `analysis_access` — `default` (run page only), `always` (everywhere), `never`
  (owner share links only).

Run link: `https://app.toughtongueai.com/run/<scenario_id>`. Embeds and iframes
use the same Scenario ID and visibility rules.

## Scenario Access Tokens (SAT)

Short-lived bearer tokens for a private Scenario or an embed. `valid_for_hours`
is 1–168 (default 1). The response carries `access_token`, `expires_at`, and a
sample `iframe_src`.

Each tool and who runs the Scenario with its token:

- `ttai:create_scenario_access_token` — no `email`: an anonymous runner. With
  `email`: a specific runner (see below).
- `ttai:create_self_scenario_access_token` — the caller; usage bills to the
  caller.

Notes:

- Confirm the audience and duration immediately before minting.
- An email-targeted SAT is a consequential write: in an organization it creates
  or reuses a dormant organization user (the email must follow the
  organization's managed-address format the server requires); in a reseller
  context it creates or reuses a real user. Confirm the intended identity.
- Return the link to the user; never store tokens in long-lived notes.

## Agent checklist

1. Resolve workspace (`ttai:list_organizations` → `org_id` or personal).
2. Ambiguous team vs personal before a write → ask.
3. Before sharing a private Scenario, mint a SAT or confirm `is_public`.
4. Never invent organization IDs, tokens, or plan facts.

## Key Files

- [resources.md](resources.md) — what lives in a workspace
- [scenario-engine.md](scenario-engine.md) — run channels
- [../mcp/tools.md](../mcp/tools.md) — tool conventions
