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

- **Scenario** — one voice (or text) agent definition, versioned. MCP: list / get /
  versions (`resource_*`); create / update (`scenario_*`).
- **Collection (course)** — named group of Scenarios, a library or course. MCP:
  list / get only; create and edit in the web app.

## Runs and results

- **Session** — one run: transcript, evaluation, extraction, recording. MCP: list / get
  (`resource_*`, legacy filters for person and date); create (ingest); analyze.
- **Analytics** — aggregates over sessions: usage, durations, top Scenarios,
  members. MCP: `ttai:get_resource(usage)`.

How runs start: [scenario-engine.md](scenario-engine.md).

## Operational attachments

- **SIP trunk** — customer-owned phone line, inbound or outbound. MCP:
  `ttai:list_resources(sip-trunks)`; safe metadata in the Resource Directory.
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

Generic discovery uses `ttai:list_resources` / `ttai:get_resource` with types
such as `avatar`, `custom-functions`, `knowledge-bases`, `sip-trunks`, and
`user-preferences`. `ttai:read_guide` lists each type's filters and safe
optional fields.

## Sensitive boundary

The Resource Directory returns purpose-built safe metadata, never credentials,
endpoint URLs, document chunks, or arbitrary model fields. It is not generic
CRUD: MCP cannot create Knowledge Bases, Custom Functions, SIP trunks, or MCP
servers. Attach only IDs returned by a trusted workspace call, after loading the
Scenario schema (`ttai:get_schema("scenario")`)
([../scenario/control.md](../scenario/control.md#linked-resources)).

## Key Files

- [account-and-access.md](account-and-access.md) — workspaces and sharing
- [scenario-engine.md](scenario-engine.md) — runtime channels and results
- [../mcp/tools.md](../mcp/tools.md) — tool catalog
