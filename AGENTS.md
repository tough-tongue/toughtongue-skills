# toughtongue-skills — Agent Guide

Skills + plugin manifests for the ToughTongue AI MCP server. This repo is
public and is installed directly into end users' agents — every word ships.

## Rules

- **MCP tool names are a contract.** Skills may only reference tools that the
  hosted MCP server exposes (see README "MCP Server" section). If the server
  catalog changes, skills change in the same PR.
- **Skills are MCP-first.** Every workflow ends in an MCP tool call
  (`create_scenario`, `update_scenario`, `list_sessions`, ...) — never in
  "write a file to disk" or references to internal ToughTongue repos, paths,
  or CLIs.
- **No secrets.** The only credential is the `TTAI_PAT` environment variable,
  referenced by name only. Never hardcode tokens, user IDs, or org IDs.
- **Token discipline in SKILL.md files.** Skills are loaded into agent
  context; keep SKILL.md under ~250 lines and push depth into `references/`
  files that agents load on demand.
- **Public bar.** No internal jargon, no unfinished sections, no hallucinated
  features. Terminology: "ToughTongue AI" (one word "ToughTongue"), not
  "Tough Tongue AI".

## Structure

- `skills/<name>/SKILL.md` — frontmatter (`name`, `description`) + workflow.
  The description doubles as the trigger; include the phrases users actually
  say.
- `skills/<name>/references/` — depth files the skill tells agents to load.
- `.claude-plugin/`, `.codex-plugin/`, `.cursor-plugin/`, `.agents/plugins/` —
  platform manifests. Keep `version` in sync across all of them when releasing.
- `.mcp.json` / `mcp.json` — identical content; both exist for platform
  compatibility. Change both or neither.

## Versioning

Bump the version in all plugin manifests together (`.claude-plugin/plugin.json`,
`.codex-plugin/plugin.json`, `.cursor-plugin/plugin.json`) on any user-visible
change.
