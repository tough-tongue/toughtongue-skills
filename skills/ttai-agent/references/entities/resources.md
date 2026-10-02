# Resources

Everything in a Tough Tongue AI workspace beyond the account itself
([account-and-access.md](account-and-access.md)). Resources share one permission
and sharing model.

## Contents

- Authoring resources
- Runs and results
- Operational attachments
- Sensitive boundary

## Authoring resources

Each resource: what it is, then what MCP can do with it.

- **Scenario** — one voice (or text) agent definition, versioned. MCP: V3 list /
  detail / versions; create / update.
- **Collection (course)** — named group of Scenarios, a library or course. MCP:
  list / get only; create and edit in the web app.

## Runs and results

- **Session** — one run: transcript, evaluation, extraction, recording. MCP: V3
  list; legacy list / get / batch; create (ingest); post-process.
- **Analytics** — aggregates over sessions: usage, durations, top Scenarios,
  members. MCP: `ttai:get_analytics`.

How runs start: [scenario-engine.md](scenario-engine.md).

## Operational attachments

- **SIP trunk** — customer-owned phone line, inbound or outbound. MCP:
  `ttai:list_sip_trunks`; safe metadata in the Resource Directory.
- **SIP call** — one phone call (trunk + Scenario + E.164 number). MCP: list /
  create / batch / delete before start.
- **Meeting bot** — joins Google Meet, Zoom, or Teams as the Scenario. MCP: list
  / schedule / delete before join.
- **Browser profile** — the Scenario's saved logins for live browser demos. MCP:
  `ttai:authenticate_browser` returns a link for a human login.
- **Access token (SAT)** — time-limited link to a private Scenario. MCP: create
  (for a runner, or for yourself).
- **Custom Function** — HTTP function a Scenario can call. MCP: discover in the
  Resource Directory; attach by ID.
- **Knowledge Base** — processed documents a Scenario can search. MCP: discover
  in the Resource Directory; attach by ID.
- **Voice-agent MCP server** — external tools the agent can call mid-session.
  MCP: attach existing IDs via `mcp_server_ids`.
- **Avatar** — system face: static, hybrid, or live. MCP: Resource Directory
  type `avatar`.

Generic discovery starts with `ttai:v3_list_resource_types`: it lists types
(`avatar`, Custom Functions, Knowledge Bases, SIP trunks, user preferences),
their safe optional fields, and the matching `ttai:v3_list_resources` /
`ttai:v3_get_resource` calls.

## Sensitive boundary

The Resource Directory returns purpose-built safe metadata, never credentials,
endpoint URLs, document chunks, or arbitrary model fields. It is not generic
CRUD: MCP cannot create Knowledge Bases, Custom Functions, SIP trunks, or MCP
servers. Attach only IDs returned by a trusted workspace call, after loading the
`ttai:create_scenario` / `ttai:update_scenario` schema
([../scenario/control.md](../scenario/control.md#linked-resources)).

## Key Files

- [account-and-access.md](account-and-access.md) — workspaces and sharing
- [scenario-engine.md](scenario-engine.md) — runtime channels and results
- [../mcp/tools.md](../mcp/tools.md) — tool catalog
