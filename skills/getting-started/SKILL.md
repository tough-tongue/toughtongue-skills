---
name: getting-started
description: >
  Onboard a user to the Tough Tongue AI plugin: verify the MCP connection and
  authentication, look at what's in their account, and route them into their
  first workflow. This skill should be used when the user says "get started with
  Tough Tongue AI", "set up Tough Tongue AI", "is my Tough Tongue AI MCP working",
  "test my Tough Tongue AI connection", "what can I do with Tough Tongue AI", or has
  just installed the plugin. Not for creating, refining, or analyzing
  scenarios directly — hand off to scenario-creator, scenario-refiner, or
  session-analyst for those.
---

# Getting Started with Tough Tongue AI

Verify the setup, orient around the user's account, and launch their first
journey. Keep each step short — this is a welcome mat, not a manual.

## Step 1: Verify the connection

Call the `ttai` MCP tool `list_organizations`.

- **Tools not found** — the MCP server is not registered. Ask how they
  installed:
  - Plugin install: restart the agent app fully and start a new thread; the
    plugin registers the server via its bundled `.mcp.json`.
  - Skills-only install (skills.sh / Cursor): register manually —
    `codex mcp add ttai --url https://api.toughtongueai.com/api/public/mcp`
    then `codex mcp login ttai` (Codex), or
    `claude mcp add --transport http ttai
    https://api.toughtongueai.com/api/public/mcp` then `/mcp` to
    authenticate (Claude Code).
- **401 / auth error** — the OAuth login hasn't been completed for this
  client. Walk through the login: sign in at
  <https://app.toughtongueai.com>, then run `/mcp` and authenticate
  (Claude Code), `codex mcp login ttai` (Codex), or open Cursor Settings >
  MCP and log in on the ttai server (Cursor) — a browser consent page opens;
  approve it. If the user is on a manual PAT config (headless/CI), instead
  check `TTAI_PAT` is visible to the agent process: re-export in the shell
  profile, on macOS also `launchctl setenv TTAI_PAT "$TTAI_PAT"`, then fully
  quit and reopen the agent app. NEVER ask the user to paste a token into
  the chat.
- **Success** — report what came back: personal account only, or
  organizations (name them). Explain that org work needs `org_id` passed to
  tools, and this happens automatically in the other skills.

## Step 2: Orient around the account

Call `list_scenarios`, and `list_sessions` with a small limit.

Summarize in 2-3 sentences what exists: how many scenarios, whether sessions
have been run, whether analyses are present. This decides the recommended
first journey below.

## Step 3: Launch the first journey

Offer the paths that fit what Step 2 found, then invoke the matching skill:

| Account state | Recommend | Skill |
|---|---|---|
| Empty (no scenarios) | Create a first practice scenario from a brief, URL, or call transcript | **scenario-creator** |
| Scenarios but few sessions | Share a practice link; or create a scenario for an upcoming meeting | **scenario-creator** |
| Sessions with analyses | Team/scenario performance report | **session-analyst** |
| Low-scoring or complained-about scenario | Diagnose and fix it from real transcripts | **scenario-refiner** |

Close by showing 2-3 of these prompts as things to try next (pick the ones
matching their account state):

- "Pull the last 3 calls from Gong where we lost on pricing and create a
  practice scenario for that objection."
- "For my discovery-call scenario, pull the last 50 sessions — what are the
  top 5 improvement areas?"
- "Pull the 5 lowest-scoring sessions for our onboarding scenario and fix
  the scenario."
- "I have a call with [name] from [company] in 30 minutes — create a quick
  practice scenario so I can rehearse."
