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

- **Scenario** — one voice (or text) agent definition, versioned. MCP:
  `list_resources` / `get_resource` (types `scenarios`, `scenario-versions`);
  `ttai:create_scenario` / `ttai:update_scenario`.
- **Collection (course)** — ordered group of Scenarios. MCP: read only (type
  `collections`); create and edit in the web app.
- **FileBase** — a folder of files: an Agent Desktop app (kind `app`) or
  agentic knowledge (kind `knowledge`). MCP: type `filebases`;
  `ttai:create_filebase` / `ttai:write_filebase`. Apps: the `ttai-agent-apps`
  skill. Knowledge: attach by ID in `knowledge_filebase_ids`; see
  [../mcp/tools.md](../mcp/tools.md#knowledge-filebases).

## Runs and results

- **Session** — one run: transcript, evaluation, extraction, recording. MCP:
  type `sessions` (legacy filters for person and date); `ttai:upload_session`
  (ingest); `ttai:analyze_session`.
- **Analytics** — aggregates over sessions: usage, durations, top Scenarios,
  members. MCP: `ttai:get_workspace_info(sections: [usage])`.

How runs start: [scenario-engine.md](scenario-engine.md).

## Operational attachments

- **SIP trunk** — customer-owned phone line, inbound or outbound. MCP:
  `ttai:list_resources(sip-trunks)`.
- **Bot** — a phone call (`kind: phone`: trunk + Scenario + E.164 numbers) or a
  meeting bot (`kind: meeting`: Google Meet, Zoom, or Teams). MCP: type `bots`;
  `ttai:schedule_bot`; `ttai:cancel_bot` before it starts.
- **Browser profile** — the Scenario's saved browser for live demos; it keeps a
  human's login. MCP: `ttai:authenticate_browser` returns a link for a human
  login.
- **Access token (SAT)** — time-limited link to a private Scenario. MCP:
  `ttai:create_token` (`scenario_access`), for a runner or for yourself.
- **Custom Function** — HTTP function a Scenario can call. MCP: type
  `custom-functions`; attach by ID.
- **Knowledge Base** — legacy RAG: uploaded documents the agent searches with
  `knowledge_base_search`. MCP: type `knowledge-bases`; attach by ID in
  `knowledge_base_ids`. Not a knowledge FileBase.
- **Voice-agent MCP server** — external tools the agent can call mid-session.
  MCP: attach existing IDs via `mcp_server_ids`.
- **Avatar** — system face: static, hybrid, or live. MCP: type `avatar`.

Generic discovery uses `ttai:list_resources` / `ttai:get_resource` with types
such as `avatar`, `custom-functions`, `knowledge-bases`, and `sip-trunks`.
`ttai:read_guide` lists each type's filters.

## Sensitive boundary

Directory types (`sip-trunks`, `knowledge-bases`, `custom-functions`,
`avatar`) return safe metadata plus requested `include_fields`; secret values
come back masked, never document chunks. MCP cannot create Knowledge Bases,
Custom Functions, SIP trunks, or MCP servers. Attach only IDs returned by a trusted workspace call, after loading the
Scenario schema (`ttai:get_schema("scenario")`)
([../scenario/control.md](../scenario/control.md#linked-resources)).

## Key Files

- [account-and-access.md](account-and-access.md) — workspaces and sharing
- [scenario-engine.md](scenario-engine.md) — runtime channels and results
- [../mcp/tools.md](../mcp/tools.md) — tool catalog
