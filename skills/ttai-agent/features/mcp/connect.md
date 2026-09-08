# MCP connect

## Tools appear as

`ttai:…` (some clients: `mcp__ttai__…`). If they are missing, the server is
not registered — point the user at the repo README / [MCP.md](../../../../MCP.md);
do not invent REST calls.

## OAuth (default)

First call opens a browser consent. Claude Code: `/mcp`. Codex:
`codex mcp login ttai`. Cursor: Settings → MCP → log in.

**401.** Login not completed for this client. Walk through OAuth. For
headless/CI only, `TTAI_PAT` must be visible to the process — never ask the
user to paste a token into chat.

**PAT** is a last resort (`Authorization: Bearer ${TTAI_PAT}`).

## Organizations

Reuse a current, verified workspace context if the consumer has one. Otherwise
call `ttai:list_organizations` first.

- Team work: pass `org_id` on every later call (`is_org: true` where the
  tool has it).
- Personal practice: omit `org_id`.
- Ambiguous: ask which workspace.
- The public MCP does not expose the caller's profile, effective plan, or
  feature gates. `ttai:list_subscriptions` lists customers subscribed to
  paid scenarios or collections—it is not a plan lookup.

Details: [../../kb/entities/key-control-entities.md](../../kb/entities/key-control-entities.md).

## Conventions

- **Load the tool schema** before calling. Do not guess field names.
- `create_scenario` **rejects** `id`. `update_scenario` **requires** `id`
  and is partial — send only changed fields. Create requires `name` +
  `ai_instructions`. `type` is `default` | `super` | `composite` only.
- Scenario edits apply to **new** sessions, not a call already in progress.
- `post_process_session` is async; poll `get_session` until results appear.
- Prefer larger `limit` over many small lists. Respect any live rate-limit
  response rather than assuming a fixed limit.

## Managing a scenario (tool sequence)

1. Author payload from **kb** (control + `ai_instructions` + recipe).
2. Load `create_scenario` / `update_scenario` schema.
3. Create: no `id`. Update: `id` + changed fields only.
4. On validation errors, fix the named field and retry.
5. Return `https://app.toughtongueai.com/run/<id>`. Private: mint a SAT
   ([../../kb/entities/auth-and-sharing.md](../../kb/entities/auth-and-sharing.md)).

`generate_scenario` drafts prose server-side — use for speed, then create
or update.

## Key Files

- [../../kb/operating-model.md](../../kb/operating-model.md) — context and capability protocol
- [tools-by-resource.md](tools-by-resource.md) — action catalog
- [../../kb/entities/auth-and-sharing.md](../../kb/entities/auth-and-sharing.md) — sharing rules
