# Account and access

Who you are, which workspace you act in, what your plan allows, and who can see
and run a Scenario. MCP reads the workspace with `ttai:get_workspace_info`; it
does not provision users, organizations, members, quotas, or plans.

## Contents

- User and workspaces
- Plan, balance, and entitlements
- Scenario visibility
- Scenario Access Tokens (SAT)
- Agent checklist

## User and workspaces

The signed-in human behind OAuth (or a `TTAI_PAT` in headless setups) owns a
**personal** workspace and may belong to **organizations**. Read both with
`ttai:get_workspace_info` (default sections `me`, `organizations`, `plan`,
`balance`); never infer a user ID, email, or plan from successful login.

- **Personal** — omit `org_id`.
- **Organization** — pass an opaque `organizations[].id` from
  `ttai:get_workspace_info` as `org_id`.

Notes:

- Scenarios, sessions, calls, and bots created with `org_id` belong to that
  organization. No MCP tool moves resources between workspaces.
- `organizations` lists each membership's `id`, `name`, `slug`, the caller's
  `role`, and `quota_enabled`. Never send the slug as `org_id`.
- Roles: `view` < `edit` (Admin in the app) < `owner`; `no_access` = end users.
- `members` (org only, `query` filters name/email) lists people, roles, and
  `quota_available`.
- Inviting members, changing roles, creating organizations, and plan changes
  happen in the web app at
  [app.toughtongueai.com](https://app.toughtongueai.com).
- A `403` means the caller lacks access to that organization or resource: switch
  workspace or have the user fix membership in the app.

## Plan, balance, and entitlements

- **Minutes balance** — `get_workspace_info` section `balance`. Personal wallet,
  or the org wallet plus the caller's quota when quotas are on.
- **Effective plan and limits** — section `plan`. Tier identity/status, limits;
  no price or billing history.
- **Usage** — section `usage` (`start_date` / `end_date`); org editors also get
  a daily series and per-member totals.
- **Quotas** — on: an org session needs minutes in the org wallet AND the
  member's quota. Off: org wallet only. Owners toggle quotas and grant quota
  minutes in the web app; no MCP tool changes them.

Notes:

- Query balance only when the operation may consume it; it is informational.
- Plans gate some features (for example Scenario count, Ocean models, multimodal
  analysis). Check `get_workspace_info` section `plan` or let the server answer; never
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
is 1–168 (default 1). The response carries `type`, `token`, `expires_at`, and
`url` (an iframe embed).

`ttai:create_token` takes one `request` with `type: "scenario_access"`
(`scenario_id`, `valid_for_hours`, optional `email`) and decides who runs the
Scenario with the token. (`type: "filebase_view"` instead links a FileBase; see
the MCP tools reference.)

- No `email`: an anonymous runner. With `email`: a specific runner (see below).
- `self_billed: true` (no `email`): the caller runs it; usage bills to the
  caller.

Notes:

- Confirm the audience and duration immediately before minting.
- An email-targeted SAT is a consequential write: in an organization it creates
  or reuses a dormant organization user (the email must follow the
  organization's managed-address format the server requires); in a reseller
  context it creates or reuses a real user. Confirm the intended identity.
- Return the link to the user; never store tokens in long-lived notes.

## Agent checklist

1. Resolve workspace (`ttai:get_workspace_info` → `organizations[].id` as
   `org_id`, or personal).
2. Ambiguous team vs personal before a write → ask.
3. Before sharing a private Scenario, mint a SAT or confirm `is_public`.
4. Never invent organization IDs, tokens, or plan facts.

## Key Files

- [resources.md](resources.md) — what lives in a workspace
- [scenario-engine.md](scenario-engine.md) — run channels
- [../mcp/tools.md](../mcp/tools.md) — tool conventions
