# Connect a client

Per-client registration of the hosted `ttai` MCP server. Plugin installs do this
automatically — use this file for manual, headless, and web setups, and for auth
errors. Tools appear as `ttai:…` (some clients: `mcp__ttai__…`). If they are
missing, the server is not registered: fix registration here; never fall back to
invented REST calls.

Server: `https://api.toughtongueai.com/api/public/mcp` (Streamable HTTP; no SSE,
no local process).

## Contents

- Auth: OAuth first, PAT for headless
- Coding agents (Claude Code, Codex, Cursor, Copilot, Windsurf, Gemini CLI)
- Stdio-only clients (mcp-remote)
- Web connectors (claude.ai, ChatGPT)
- Troubleshooting and FAQ

## Auth

- **OAuth 2.1 (default)** — register the URL with no credentials; the client
  opens a browser consent page on first use (sign in at
  [app.toughtongueai.com](https://app.toughtongueai.com) first). The issued
  token is a PAT underneath; revoke it at
  [app.toughtongueai.com/developer](https://app.toughtongueai.com/developer).
- **PAT bearer (headless / CI)** — create a PAT at the developer portal,
  `export TTAI_PAT="<token>"`, and reference it by name only. On macOS, GUI apps
  also need `launchctl setenv TTAI_PAT "$TTAI_PAT"`. Never paste the token into
  config files or chat.

## Coding agents

<details>
<summary>Claude Code</summary>

```bash
claude mcp add --transport http ttai https://api.toughtongueai.com/api/public/mcp
```

Run `/mcp` in a session and finish the browser login. PAT:

```bash
claude mcp add --transport http ttai https://api.toughtongueai.com/api/public/mcp \
  --header "Authorization: Bearer ${TTAI_PAT}"
```

</details>

<details>
<summary>Codex</summary>

```bash
codex mcp add ttai --url https://api.toughtongueai.com/api/public/mcp
codex mcp login ttai
```

Or in `~/.codex/config.toml`:

```toml
[mcp_servers.ttai]
url = "https://api.toughtongueai.com/api/public/mcp"
# PAT: bearer_token_env_var = "TTAI_PAT"
```

PAT via CLI: add `--bearer-token-env-var TTAI_PAT` to `codex mcp add`. If this
is your first HTTP MCP server, enable
`[features] experimental_use_rmcp_client = true`.

</details>

<details>
<summary>Cursor</summary>

[**Install in Cursor**](cursor://anysphere.cursor-deeplink/mcp/install?name=ttai&config=eyJ1cmwiOiJodHRwczovL2FwaS50b3VnaHRvbmd1ZWFpLmNvbS9hcGkvcHVibGljL21jcCJ9)
adds the server; Cursor prompts for OAuth on first use. Manual: **Cursor
Settings > MCP > Add new global MCP server**:

```json
{
  "mcpServers": {
    "ttai": { "url": "https://api.toughtongueai.com/api/public/mcp" }
  }
}
```

PAT: add `"headers": { "Authorization": "Bearer ${env:TTAI_PAT}" }`.

</details>

<details>
<summary>GitHub Copilot (VS Code)</summary>

**MCP: Open User Configuration** (or `.vscode/mcp.json` per workspace):

```json
{
  "servers": {
    "ttai": {
      "type": "http",
      "url": "https://api.toughtongueai.com/api/public/mcp"
    }
  }
}
```

PAT: add `"headers": { "Authorization": "Bearer ${env:TTAI_PAT}" }`.

</details>

<details>
<summary>Windsurf</summary>

Omit `headers` if your version prompts for OAuth:

```json
{
  "mcpServers": {
    "ttai": {
      "serverUrl": "https://api.toughtongueai.com/api/public/mcp",
      "headers": { "Authorization": "Bearer ${env:TTAI_PAT}" }
    }
  }
}
```

</details>

<details>
<summary>Gemini CLI</summary>

Omit `headers` if your version prompts for OAuth:

```json
{
  "mcpServers": {
    "ttai": {
      "httpUrl": "https://api.toughtongueai.com/api/public/mcp",
      "headers": { "Authorization": "Bearer ${TTAI_PAT}" }
    }
  }
}
```

</details>

## Stdio-only clients

Claude Desktop, Zed, and older VS Code bridge through
[mcp-remote](https://github.com/geelen/mcp-remote), which runs the OAuth flow
itself:

```json
{
  "mcpServers": {
    "ttai": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.toughtongueai.com/api/public/mcp"
      ]
    }
  }
}
```

Claude Desktop: **Settings > Developer > Edit Config**. Headless: append
`"--header", "Authorization: Bearer ${TTAI_PAT}"` to `args`.

## Web connectors

Same OAuth flow, no PAT and no config file.

<details>
<summary>claude.ai</summary>

1. **Settings > Connectors > Add custom connector**.
2. Name `Tough Tongue AI`, URL above. Leave OAuth Client ID and Secret empty
   (dynamic client registration).
3. **Add**, approve on the consent page (signed in at the app first). The
   connector shows **Connected**.
4. New chat, enable the connector, ask "List my Tough Tongue AI organizations."

</details>

<details>
<summary>ChatGPT (web only)</summary>

Simplest: install the official Tough Tongue AI app from the ChatGPT app
directory (no Developer mode). The steps below publish your own custom app.

Business, Enterprise, and Edu get full MCP including writes (beta); Pro is
read/fetch only.

1. Enable **Developer mode** (**Settings > Apps > Advanced settings**; on
   Business only admins/owners, on Enterprise/Edu via **Permissions & Roles >
   Connected Data**).
2. **Settings > Apps > Create**: MCP Server URL above, **OAuth**, **Scan
   Tools**, complete consent, **Create**.
3. The app is a **Draft**; admins publish via **Workspace Settings > Apps >
   Drafts > Publish**.

ChatGPT freezes the tool snapshot at approval; tool changes need an admin
refresh. Writes prompt for confirmation; deep research is read-only.

</details>

## Troubleshooting

- **Expected tool missing** — the client cached discovery. Start a new thread;
  else remove and re-add the server.
- **401 / auth error** — finish OAuth: Claude Code `/mcp`, Codex
  `codex mcp login ttai`, Cursor Settings > MCP. PAT: fix `TTAI_PAT`.
- **Edit not live in a call** — Scenario edits apply to new sessions only.
- **Works personally, not by org** — pass `org_id` from
  `ttai:get_workspace_info` (`organizations[].id`).
- **mcp-remote internal error** — `rm -rf ~/.mcp-auth`, reconnect, update Node
  to current LTS.
- **Rate limited** — 30 calls/minute per token; wait, then retry; request only
  the page you need.

Smoke test: `ttai:read_guide`, then
`ttai:get_workspace_info`.

Never ask the user to paste a token into chat. On `401`, finish OAuth for that
client; `TTAI_PAT` is only for headless or CI processes.

## Key Files

- [tools.md](tools.md) — tool catalog and call conventions
- [../operating-model.md](../operating-model.md) — first-run orientation
