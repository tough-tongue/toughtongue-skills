# ToughTongue Skills

Agent skills and MCP server for [ToughTongue AI](https://app.toughtongueai.com)
— the voice-agent platform for high-stakes conversation practice: interviews,
sales, coaching, negotiations.

Install once, then work with ToughTongue from Claude Code, Codex, Cursor, or
any agent that supports the Agent Skills format:

- *"Pull the last 3 lost deals from our call notes and create a practice
  scenario for the pricing objection."*
- *"For the Enterprise Discovery Call scenario, pull the last 50 sessions —
  what are the top 5 improvement areas?"*
- *"The onboarding agent keeps ending calls too early. Pull the low-scoring
  sessions, find out why, and fix the scenario."*

## Install

### skills.sh (Cursor, Copilot, Gemini CLI, Cline, and more)

```bash
npx skills add tough-tongue/toughtongue-skills
```

### Claude Code

```text
/plugin marketplace add tough-tongue/toughtongue-skills
/plugin install toughtongue@toughtongue-skills
```

### Codex

```bash
git clone https://github.com/tough-tongue/toughtongue-skills
cd toughtongue-skills
codex plugin marketplace add .
codex plugin add toughtongue@toughtongue
```

## Authenticate

1. Create a Personal Access Token at
   [app.toughtongueai.com/developer](https://app.toughtongueai.com/developer)
   — it looks like `ttai_pat_...`.
2. Export it in the environment your agent runs in:

```bash
export TTAI_PAT="ttai_pat_..."
```

On macOS, GUI apps need the variable at the launchd level too:

```bash
launchctl setenv TTAI_PAT "$TTAI_PAT"
```

Then fully restart your agent app and start a new thread. Verify with:

```text
Call the ttai MCP tool list_organizations and show me the result.
```

Never commit the PAT or paste it into config files — the plugin references it
only through the `TTAI_PAT` environment variable.

## Skills

| Skill | What it does |
|---|---|
| [scenario-creator](skills/scenario-creator) | Create production-ready scenarios from a brief, URL, or call transcript. Classifies the type (cold call, sales roleplay, coaching), applies proven authoring patterns, validates, and creates via MCP. |
| [scenario-refiner](skills/scenario-refiner) | Fix a live scenario from real evidence. Pulls the scenario and low-scoring session transcripts, diagnoses the root cause, and applies the smallest possible edit via MCP. |
| [session-analyst](skills/session-analyst) | Turn session data into answers. Aggregates scores, report cards, and weaknesses across a team or scenario into structured reports — ready to hand off to slides or email tools. |

## MCP Server

The plugin registers ToughTongue's hosted MCP server at
`https://api.toughtongueai.com/api/public/mcp` (streamable HTTP),
authenticated with your `TTAI_PAT` as a bearer header. It exposes 26 tools
over the public API, including:

- **Scenarios** — `create_scenario`, `update_scenario`, `get_scenario`,
  `list_scenarios`, `generate_scenario`
- **Sessions** — `list_sessions`, `get_sessions_batch`, `create_session`,
  `post_process_session`
- **Analytics & account** — `get_analytics`, `list_organizations`,
  `get_balance`, `list_subscriptions`
- **Phone & meetings** — `create_sip_call`, `create_sip_batch`,
  `schedule_meeting_bot`, `list_meeting_bots`
- **Access** — `create_scenario_access_token`,
  `create_self_scenario_access_token`

Every tool accepts an optional `org_id` (from `list_organizations`) for
organization-scoped work.

### MCP without the plugin

If you only want the tools (no skills):

```bash
# Codex
codex mcp add ttai --url https://api.toughtongueai.com/api/public/mcp \
  --bearer-token-env-var TTAI_PAT

# Claude Code
claude mcp add --transport http ttai https://api.toughtongueai.com/api/public/mcp \
  --header "Authorization: Bearer ${TTAI_PAT}"
```

## Troubleshooting

- **Fewer than 26 ttai tools listed** — your agent trimmed or cached tool
  discovery. Start a fresh thread; if it persists, remove and re-add the MCP
  server.
- **401 / authentication errors** — `TTAI_PAT` is not visible to the agent
  process. Re-export, `launchctl setenv` on macOS, fully restart the app.
- **Scenario edits not taking effect in a running call** — scenario changes
  apply to new sessions only; sessions compile their prompt at start.

## Repository Structure

```
toughtongue-skills/
├── .claude-plugin/        # Claude Code plugin + marketplace manifests
├── .codex-plugin/         # Codex plugin manifest
├── .cursor-plugin/        # Cursor plugin manifest
├── .agents/plugins/       # Codex plugin-marketplace entry
├── .mcp.json              # Hosted MCP server registration (TTAI_PAT bearer)
└── skills/
    ├── scenario-creator/  # SKILL.md + type-specific authoring references
    ├── scenario-refiner/  # SKILL.md + runtime behavior reference
    └── session-analyst/   # SKILL.md + report templates
```

## Related

- [voice-ai-quickstart](https://github.com/tough-tongue/voice-ai-quickstart) —
  starter templates for building apps on ToughTongue AI (Next.js, Flask,
  co-navigation demo, scenario-as-code CLI) and the `toughtongue-ai`
  integration skill for developers embedding the platform.
- [Platform docs](https://app.toughtongueai.com/docs) ·
  [llms-full.txt](https://app.toughtongueai.com/llms-full.txt) (AI-readable
  API reference)

## License

MIT
