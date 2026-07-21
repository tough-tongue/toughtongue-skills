# ToughTongue AI MCP Server

[![MCP Registry](https://img.shields.io/badge/MCP_Registry-com.toughtongueai%2Fmcp-blue)](https://registry.modelcontextprotocol.io/v0.1/servers?search=com.toughtongueai/mcp)

Connect your AI agent to the ToughTongue AI platform. Create and refine
voice-agent scenarios, pull session results and analytics, place SIP phone
calls, and schedule meeting bots — directly from any MCP client like Claude
Code, Codex, Cursor, or Copilot.

ToughTongue hosts the MCP server at:

```
https://api.toughtongueai.com/api/public/mcp
```

It uses Streamable HTTP. There is nothing to install and no local process to
run. Clients authenticate with a Personal Access Token (PAT) bearer token
(CLI and coding agents) or OAuth 2.1 (claude.ai web, ChatGPT web).

## Contents

- Get a token
- Connect your client (Claude Code, Codex, Cursor, Copilot, Windsurf, Gemini CLI)
- Clients without remote MCP support (mcp-remote)
- Web connectors over OAuth (claude.ai, ChatGPT)
- Conventions
- Tools (full catalog)
- Verify and troubleshoot
- FAQ
- Pair with the skills

## Get a token

1. Create a PAT at
   [app.toughtongueai.com/developer](https://app.toughtongueai.com/developer)
   — copy it when shown.
2. Export it in the environment your agent runs in:

```bash
export TTAI_PAT="<your-token>"
```

On macOS, GUI-launched apps also need the variable at the launchd level:

```bash
launchctl setenv TTAI_PAT "$TTAI_PAT"
```

Never commit the token or paste it into shared config files — reference it
via the environment variable.

## Connect your client

Replace `${TTAI_PAT}` with the environment variable (preferred) or your
token, depending on what your client's config supports.

### Claude Code

```bash
claude mcp add --transport http ttai https://api.toughtongueai.com/api/public/mcp \
  --header "Authorization: Bearer ${TTAI_PAT}"
```

### Codex

Configuration is shared between the Codex CLI and IDE extension.

```bash
codex mcp add ttai --url https://api.toughtongueai.com/api/public/mcp \
  --bearer-token-env-var TTAI_PAT
```

Or configure directly in `~/.codex/config.toml`:

```toml
[mcp_servers.ttai]
url = "https://api.toughtongueai.com/api/public/mcp"
bearer_token_env_var = "TTAI_PAT"
```

If this is your first HTTP MCP server in Codex, you may need to enable the
rmcp client in `~/.codex/config.toml`:

```toml
[features]
experimental_use_rmcp_client = true
```

### Cursor

[**Install in Cursor**](cursor://anysphere.cursor-deeplink/mcp/install?name=ttai&config=eyJ1cmwiOiJodHRwczovL2FwaS50b3VnaHRvbmd1ZWFpLmNvbS9hcGkvcHVibGljL21jcCIsImhlYWRlcnMiOnsiQXV0aG9yaXphdGlvbiI6IkJlYXJlciAke1RUQUlfUEFUfSJ9fQ==)
— one click adds the server; make sure `TTAI_PAT` is exported first.

Or manually: open the command palette and choose "Cursor Settings" > "MCP" >
"Add new global MCP server":

```json
{
  "mcpServers": {
    "ttai": {
      "url": "https://api.toughtongueai.com/api/public/mcp",
      "headers": {
        "Authorization": "Bearer ${TTAI_PAT}"
      }
    }
  }
}
```

### GitHub Copilot (VS Code)

Add to your `settings.json`:

```json
{
  "mcp": {
    "servers": {
      "ttai": {
        "type": "http",
        "url": "https://api.toughtongueai.com/api/public/mcp",
        "headers": {
          "Authorization": "Bearer ${TTAI_PAT}"
        }
      }
    }
  }
}
```

### Windsurf

```json
{
  "mcpServers": {
    "ttai": {
      "serverUrl": "https://api.toughtongueai.com/api/public/mcp",
      "headers": {
        "Authorization": "Bearer ${TTAI_PAT}"
      }
    }
  }
}
```

### Gemini CLI

```json
{
  "mcpServers": {
    "ttai": {
      "httpUrl": "https://api.toughtongueai.com/api/public/mcp",
      "headers": {
        "Authorization": "Bearer ${TTAI_PAT}"
      }
    }
  }
}
```

## Clients without remote MCP support

For clients that only run local stdio MCP servers (Claude Desktop, Zed, older
VS Code), bridge to the hosted server with
[mcp-remote](https://github.com/geelen/mcp-remote):

```bash
npx -y mcp-remote https://api.toughtongueai.com/api/public/mcp \
  --header "Authorization: Bearer ${TTAI_PAT}"
```

### Claude Desktop

Open Claude Desktop settings > "Developer" tab > "Edit Config":

```json
{
  "mcpServers": {
    "ttai": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.toughtongueai.com/api/public/mcp",
        "--header",
        "Authorization: Bearer ${TTAI_PAT}"
      ]
    }
  }
}
```

### Any other stdio client

- **Command**: `npx`
- **Arguments**: `-y mcp-remote https://api.toughtongueai.com/api/public/mcp --header "Authorization: Bearer ${TTAI_PAT}"`
- **Environment**: `TTAI_PAT` must be set

## Web connectors (OAuth)

Browser clients connect over OAuth 2.1 with dynamic client registration — no
PAT and no config file.

### claude.ai (web)

1. In claude.ai, go to **Settings > Connectors > Add custom connector**.
2. Set **Name** to `ToughTongue AI` and **URL** to
   `https://api.toughtongueai.com/api/public/mcp`. Leave the OAuth **Client
   ID** and **Secret** empty — dynamic client registration handles that.
3. Click **Add**. Approve on the ToughTongue consent page in the OAuth popup.
   You must be signed in at
   [toughtongueai.com](https://app.toughtongueai.com) first, or the consent
   step won't appear. The connector then shows **Connected**.
4. Verify: start a new chat, enable the connector, and ask "List my
   ToughTongue organizations."

The OAuth-issued access token is a PAT under the hood — manage or revoke it at
[app.toughtongueai.com/developer](https://app.toughtongueai.com/developer).

### ChatGPT (web)

Web only (no mobile). Availability depends on your plan: **Business,
Enterprise, and Edu** get full MCP including write tools (beta); **Pro** is
read/fetch only; **Free, Plus, and Go** can't add custom apps.

1. Enable **Developer mode**: **Settings > Apps > Advanced settings**. On
   Business, only admins/owners can toggle it; on Enterprise/Edu an admin
   grants access via **Permissions & Roles > Connected Data** (RBAC).
2. Go to **Settings > Apps > Create**. Set the **MCP Server URL** to
   `https://api.toughtongueai.com/api/public/mcp`, choose **OAuth**
   authentication, click **Scan Tools**, complete the OAuth consent, then
   **Create**.
3. The app appears as a **Draft** with a "Dev" label. Test it in chat; admins
   publish it workspace-wide from **Workspace Settings > Apps > Drafts >
   Publish**.

Caveats: ChatGPT freezes the tool snapshot at approval — tool changes need an
admin refresh or republish. No refresh tokens are issued, so you
re-authenticate after the access token expires (advertised at ~1 year). Write
actions prompt for confirmation, and deep research uses read-only tools.

## Conventions

- **Organizations**: every tool accepts an optional `org_id`. Call
  `list_organizations` first and pass the chosen ID for team/organization
  work; omit it for personal context.
- **Scenario create vs update**: `create_scenario` rejects an `id`;
  `update_scenario` requires one and applies partial updates (send only the
  fields you change).
- **Async analysis**: `post_process_session` returns immediately; poll the
  session until `evaluation_results` appears.
- **Token safety**: the PAT authorizes your whole account — keep it
  server-side, never in client code or generated apps.

## Tools

26 tools over the ToughTongue public API. Annotations: R = read-only,
W = write, D = destructive.

### Scenarios

| Tool | | Description |
|---|---|---|
| `list_scenarios` | R | List scenarios you own or can access |
| `get_scenario` | R | Full scenario detail including `ai_instructions`, `strategy`, `tools_config` |
| `create_scenario` | W | Create a scenario (full field schema; omit `id`) |
| `update_scenario` | W | Partial update of an existing scenario (`id` required) |
| `generate_scenario` | W | Server-side generation of scenario content from a name/context |
| `create_scenario_access_token` | W | Mint a 1-hour Scenario Access Token (SAT) for private scenarios |
| `create_self_scenario_access_token` | W | Mint a self-billed SAT |

### Sessions

| Tool | | Description |
|---|---|---|
| `list_sessions` | R | List sessions with evaluation and improvement results; filters: `scenario_id`, `user_email`, `from_date`/`to_date`, `is_org`, pagination |
| `get_sessions_batch` | R | Fetch specific sessions by ID (fast path for deep dives) |
| `create_session` | W | Create a session, e.g. ingest an external transcript for analysis |
| `post_process_session` | W | Trigger analysis/extraction in the background (also retries failed runs) |

### Analytics & account

| Tool | | Description |
|---|---|---|
| `get_analytics` | R | Unified personal or org-wide analytics (stats, usage, member breakdown) |
| `list_organizations` | R | Organizations the token can act in — call this first |
| `get_balance` | R | Wallet balance |
| `list_subscriptions` | R | Active subscriptions |

### Phone (SIP)

| Tool | | Description |
|---|---|---|
| `list_sip_trunks` | R | Configured SIP trunks |
| `list_sip_calls` | R | Past and scheduled SIP calls |
| `create_sip_call` | W | Place a single outbound call (E.164 number + scenario) |
| `create_sip_batch` | W | Place a batch of outbound calls |
| `delete_sip_call` | D | Cancel a SIP call |

### Meeting bots

| Tool | | Description |
|---|---|---|
| `list_meeting_bots` | R | Scheduled meeting bots |
| `schedule_meeting_bot` | W | Send a bot to a Google Meet meeting |
| `delete_meeting_bot` | D | Cancel a scheduled bot |

### Collections

| Tool | | Description |
|---|---|---|
| `list_collections` | R | List scenario collections |
| `get_collection` | R | Collection detail |

### Browser

| Tool | | Description |
|---|---|---|
| `authenticate_browser` | W | Create an authenticated browser context for browser-tool demo scenarios |

## Verify and troubleshoot

Smoke test after connecting — ask your agent:

```text
Call the ttai MCP tool list_organizations and show me the result.
```

| Symptom | Fix |
|---|---|
| Fewer than 26 ttai tools listed | Client trimmed or cached tool discovery. Start a fresh thread; if it persists, remove and re-add the server. |
| 401 / authentication errors | `TTAI_PAT` is not visible to the agent process. Re-export, `launchctl setenv` on macOS, fully restart the app. |
| Scenario edit not reflected in a running call | Scenario changes apply to new sessions only — sessions compile their prompt at start. |
| Tool works personally but not for team data | Pass `org_id` (from `list_organizations`) and `is_org: true` where the tool supports it. |

## FAQ

<details>
<summary>Does the server support Streamable HTTP?</summary>

Yes — Streamable HTTP is the only transport, at
`https://api.toughtongueai.com/api/public/mcp`. There is no SSE endpoint and
no local package to run.
</details>

<details>
<summary>Can I authenticate with OAuth instead of a token?</summary>

Yes. The server supports OAuth 2.1 with dynamic client registration — used by
claude.ai web and ChatGPT web (see [Web connectors](#web-connectors-oauth)).
The token it issues is a PAT under the hood; revoke it at
[Developer settings](https://app.toughtongueai.com/developer). PAT bearer
headers keep working for servers, CI, and headless agents.
</details>

<details>
<summary>mcp-remote shows an internal or auth error when connecting</summary>

Clear its cached auth state with `rm -rf ~/.mcp-auth` and reconnect. If the
error persists, update Node to a current LTS version.
</details>

<details>
<summary>Is there a rate limit?</summary>

Yes — tool calls are rate-limited per token (30 calls/minute). Agents doing
large session pulls should paginate with larger `limit` values instead of
many small calls.
</details>

## Pair with the skills

This repo also ships three skills — scenario-creator, scenario-refiner, and
session-analyst — that encode proven workflows on top of these tools.
Installing the plugin (see [README.md](README.md)) registers this MCP server
and the skills in one step.

## API reference

The MCP tools wrap the ToughTongue public API. For endpoint-level detail:
[app.toughtongueai.com/docs](https://app.toughtongueai.com/docs) ·
[llms-full.txt](https://app.toughtongueai.com/llms-full.txt) (AI-readable).
