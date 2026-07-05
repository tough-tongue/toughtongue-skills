# toughtongue-skills — Agent Guide

Skills + plugin manifests for the ToughTongue AI MCP server. This repo is
public and is installed directly into end users' agents — every word ships.

## Rules

- **MCP tool names are a contract.** Skills may only reference tools that the
  hosted MCP server exposes (full catalog in [MCP.md](MCP.md)). If the server
  catalog changes, MCP.md and the skills change in the same PR.
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
  The description doubles as the trigger; front-load the key use case in the
  first sentence (Codex truncates descriptions under context pressure) and
  include the phrases users actually say.
- `skills/<name>/references/` — depth files the skill tells agents to load.
  Files over 100 lines carry a Contents block at the top.
- `skills/<name>/agents/openai.yaml` — Codex UI metadata + the ttai MCP
  dependency declaration. Keep the MCP URL in sync with `.mcp.json`.
- `skill-evals/` — 3 evaluation scenarios per skill; re-run before releases
  that touch a SKILL.md or reference file.
- MCP tool references in skills use the qualified `ttai:tool_name` form.
- `.claude-plugin/`, `.codex-plugin/`, `.cursor-plugin/`, `.agents/plugins/` —
  platform manifests. Keep `version` in sync across all of them when releasing.
- `.mcp.json` / `mcp.json` — identical content; both exist for platform
  compatibility. Change both or neither.

## Versioning

Bump the version in all plugin manifests together (`.claude-plugin/plugin.json`,
`.codex-plugin/plugin.json`, `.cursor-plugin/plugin.json`) on any user-visible
change. The explicit `version` field controls when installed users receive
updates — without a bump, Claude Code users on marketplace installs do not
update.

## Releasing & Publishing

### Pre-release checklist (every release)

1. `claude plugin validate .` — must pass; the community-marketplace review
   pipeline runs the same check on submission.
2. Local smoke test, Claude Code: `claude --plugin-dir .` then invoke
   `/toughtongue:scenario-creator` (and `/reload-plugins` after edits).
3. Local smoke test, Codex: `codex plugin marketplace add <checkout-path>`,
   `codex plugin add toughtongue@toughtongue`, restart, verify the ttai MCP
   tools and all three skills appear.
4. Verify `TTAI_PAT` flows: with the env var set, "Call the ttai MCP tool
   list_organizations" must succeed in both agents.
5. Bump `version` in all three plugin manifests; tag the release.

### Distribution channels

| Channel | Mechanism | Status |
|---|---|---|
| Claude Code (self-hosted marketplace) | This repo's `.claude-plugin/marketplace.json`; users run `/plugin marketplace add tough-tongue/toughtongue-skills` | Live on push |
| Claude community marketplace (`@claude-community`) | Submit at <https://platform.claude.com/plugins/submit> (Console, works for individual authors) or <https://claude.ai/admin-settings/directory/submissions/plugins/new> (Team/Enterprise). Review pins a commit SHA in `anthropics/claude-plugins-community`; CI auto-bumps on new pushes; catalog syncs nightly | Submit once |
| Codex (GitHub marketplace) | This repo's `.agents/plugins/marketplace.json`; users run `codex plugin marketplace add tough-tongue/toughtongue-skills`. Codex also reads `.claude-plugin/marketplace.json` for compatibility | Live on push |
| Codex official Plugin Directory | Publishing is "coming soon" per OpenAI docs — no self-serve yet. Interim: share to a ChatGPT workspace via Codex app → Plugins → Created by you → Share | Watch docs |
| skills.sh | Indexes public GitHub repos with `skills/` | Live once repo is public |
