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

These compose with the rest of your MCP ecosystem — Gong, Notion, calendar,
slides, email. See [What you can do](#what-you-can-do) for the full journeys.

## Install

### Claude Code

Inside Claude Code:

```text
/plugin marketplace add tough-tongue/toughtongue-skills
/plugin install toughtongue@toughtongue-skills
```

Or from the terminal:

```bash
claude plugin marketplace add tough-tongue/toughtongue-skills
claude plugin install toughtongue@toughtongue-skills
```

The plugin bundles the skills and registers the ToughTongue MCP server
(`.mcp.json`) in one install — no separate `claude mcp add` step needed.
Just set `TTAI_PAT` (see Authenticate below).

Then run `/toughtongue:getting-started` — it verifies the connection, looks
at your account, and starts your first workflow.

Skills are namespaced after install: invoke them as
`/toughtongue:scenario-creator`, `/toughtongue:scenario-refiner`,
`/toughtongue:session-analyst` — or just describe the task and Claude picks
the right skill automatically.

### Codex

```bash
codex plugin marketplace add tough-tongue/toughtongue-skills
codex plugin add toughtongue@toughtongue
```

Then restart Codex and start a new thread. The plugin bundles the skills and
registers the ToughTongue MCP server (`.mcp.json`) in one install. Say
"get me started with ToughTongue" to verify the setup and start your first
workflow.

### Cursor / Copilot / Gemini CLI / Cline (via skills.sh)

```bash
npx skills add tough-tongue/toughtongue-skills
```

Installs the skills; add the MCP server separately (see
"MCP without the plugin" below).

### Local development / testing

```bash
# Claude Code — load the plugin without installing:
claude --plugin-dir /path/to/toughtongue-skills
# after edits, run /reload-plugins inside the session

# Codex — register the checkout as a local marketplace:
codex plugin marketplace add /path/to/toughtongue-skills
codex plugin add toughtongue@toughtongue
```

## Authenticate

1. Create a Personal Access Token (PAT) at
   [app.toughtongueai.com/developer](https://app.toughtongueai.com/developer).
2. Export it in the environment your agent runs in:

```bash
export TTAI_PAT="<your-token>"
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

## What you can do

Coding agents are where work happens now. These journeys show ToughTongue
composing with the other tools already connected to your agent — copy any
prompt to start.

### 1. Practice this sales call

A sales manager spots an AE struggling with pricing objections in real calls.

> Pull the last 3 calls from Gong where we lost on pricing. Create a
> ToughTongue scenario to practice handling that objection, using our
> positioning doc from Notion. Give me the shareable practice link.

Gong MCP (`search_calls` / `list_calls` → `get_call_transcript`) + Notion MCP
→ **scenario-creator** builds a sales roleplay from the real objections →
`create_scenario` → shareable practice link.

### 2. How is my team doing?

A VP of Sales wants the team's skill gaps, not raw transcripts.

> For the "Enterprise Discovery Call" scenario, pull the last 50 sessions.
> What are the top 5 improvement areas? Build me a 3-slide deck.

**session-analyst** → `list_sessions` (scores, strengths, weaknesses per
session) → aggregates report-card topics and weakness themes → slides MCP for
the deck.

### 3. Refine from real conversations

A CS manager notices low scores on the onboarding scenario.

> Pull the 5 lowest-scoring sessions for our onboarding scenario, figure out
> what went wrong, and fix the scenario.

`list_sessions` (sorted by score) → `get_sessions_batch` (full transcripts +
evaluations) → **scenario-refiner** diagnoses the root cause → surgical
`update_scenario` — live for the next session.

### 4. Prep me for this meeting

A sales rep has a discovery call in 30 minutes.

> I have a call with Sarah Chen from Acme Corp in 30 minutes. Create a quick
> practice scenario so I can rehearse.

Calendar MCP + web search for attendee/company context → **scenario-creator**
→ `create_scenario` → start practicing in minutes.

### 5. Automated post-call coaching

An engineering team wires coaching into their call pipeline.

> Every time a call ends in Gong, analyze the transcript and email the rep a
> coaching report.

Webhook script → `create_session` (ingest transcript against a coaching
scenario) + `post_process_session` → poll `get_sessions_batch` until analysis
completes → email MCP sends the report. Full recipe in
[session-analyst's report templates](skills/session-analyst/references/report-templates.md).

## Skills

| Skill | What it does |
|---|---|
| [getting-started](skills/getting-started) | Post-install onboarding. Verifies the MCP connection and PAT, looks at what's in your account, and routes you into your first workflow. |
| [scenario-creator](skills/scenario-creator) | Create production-ready scenarios from a brief, URL, or call transcript. Classifies the type (cold call, sales roleplay, coaching), applies proven authoring patterns, validates, and creates via MCP. |
| [scenario-refiner](skills/scenario-refiner) | Fix a live scenario from real evidence. Pulls the scenario and low-scoring session transcripts, diagnoses the root cause, and applies the smallest possible edit via MCP. |
| [session-analyst](skills/session-analyst) | Turn session data into answers. Aggregates scores, report cards, and weaknesses across a team or scenario into structured reports — ready to hand off to slides or email tools. |

## MCP Server

The plugin registers ToughTongue's hosted MCP server at
`https://api.toughtongueai.com/api/public/mcp` (Streamable HTTP,
`TTAI_PAT` bearer auth). It exposes 26 tools over the public API: scenarios
(create, update, generate, access tokens), sessions (list with evaluations,
batch fetch, ingest, post-process), analytics and organizations, SIP phone
calls, meeting bots, and collections.

**See [MCP.md](MCP.md)** for the full tool catalog, per-client setup
(Claude Code, Codex, Cursor, Copilot, Windsurf, Gemini CLI), and
troubleshooting.

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

More clients in [MCP.md](MCP.md).

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
├── MCP.md                 # MCP server docs: setup per client, tool catalog
├── skill-evals/           # Evaluation scenarios per skill
└── skills/                # Each: SKILL.md + references/ + agents/openai.yaml
    ├── getting-started/   # Post-install onboarding: verify, orient, first journey
    ├── scenario-creator/  # Type-specific authoring references
    ├── scenario-refiner/  # Runtime behavior reference
    └── session-analyst/   # Report templates
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
