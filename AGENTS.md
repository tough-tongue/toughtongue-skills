# toughtongue-skills — Agent Guide

Skills + plugin manifests for the Tough Tongue AI MCP server. This repo is
public and is installed directly into end users' agents — every word ships.

## Rules

- **MCP tool names are a contract.** Skills may only reference tools that the
  hosted MCP server exposes (catalog in
  `skills/ttai-agent/references/mcp/tools.md`). If the server
  catalog changes, that file and the skills change in the same PR.
- **Skills are MCP-first.** Every workflow ends in an MCP tool call
  (`create_scenario`, `update_scenario`, `list_sessions`, ...) — never in
  "write a file to disk" or references to internal Tough Tongue AI repos, paths,
  or CLIs.
- **No secrets.** Authentication is OAuth-first: the shipped MCP configs
  (`.mcp.json` / `mcp.json`) carry no credentials and clients run the browser
  OAuth flow. The only credential ever named is the `TTAI_PAT` environment
  variable (headless/CI fallback), referenced by name only. Never hardcode
  tokens, user IDs, or org IDs.
- **Token discipline in SKILL.md files.** Skills are loaded into agent
  context; keep SKILL.md under ~250 lines and push depth into `references/`
  files that agents load on demand.
- **Public bar.** No internal jargon, no unfinished sections, no hallucinated
  features. Terminology: "Tough Tongue AI" (two words, matching the public
  brand at toughtongueai.com), not "ToughTongue AI". Positioning: the platform
  for tough conversations, with a dual storyline — some the AI takes (voice
  agents that call, demo, screen, book), others you nail (realistic roleplay
  for negotiations, interviews, coaching). Do not describe the platform as
  only roleplay/training, and do not drop the practice half either — lead
  with the automate + practice pairing.

## Structure

Every skill name starts with `ttai-`. One main skill plus specialists; each
skill is self-contained and follows the Agent Skills standard
(<https://agentskills.io/specification>).

- `skills/ttai-agent/` — **main skill**. Turns intent into a scoped,
  capability-aware action: workspace scope, entities, Scenario
  create/edit/repair, situation recipes, and the MCP tool map. Consumer-neutral:
  coding agents and web conversational plugins share it.
- `skills/ttai-session-analyst/` — session evidence → reports; transcript
  ingestion.
- `skills/ttai-browser-demo-builder/` — deterministic browser demo steps on a
  Scenario's browser tool.
- New skill only for a distinct job with its own vocabulary and triggers;
  otherwise add a reference to `ttai-agent`.
- **No cross-skill file links.** Never link `../<other-skill>/…` — a skill
  installed alone (skills.sh, `npx skills add --skill`) gets dead links. Name
  the sibling skill instead and inline the few facts the job needs.

- `skills/<name>/SKILL.md` — frontmatter (`name`, `description`, `when_to_use`) + workflow.
  `name` + `description` + `when_to_use` are the Claude Code meta-prompt markers
  (combined trigger text capped at 1,536 characters). The description doubles as
  the trigger; front-load the key use case in the first sentence (Codex truncates
  descriptions under context pressure) and include the phrases users actually say.
  Descriptions are third person and precise: name the objects and tools the
  skill acts on, the exact user phrases, and what it does NOT cover (with the
  sibling skill that does). Broad descriptions make skills collide.
- Depth files — `references/` (subfolders by domain allowed, ≤7 files each).
  **Every reference file is linked directly from SKILL.md** — one level deep,
  no `index.md` hub chains (agents only partially read chained files). Files
  over 100 lines carry a Contents block at the top.
- `skills/<name>/agents/openai.yaml` — Codex UI metadata + the ttai MCP
  dependency declaration (omit `dependencies` only on a skill that never
  calls tools).
  Keep the MCP URL in sync with `.mcp.json`.
- `skill-evals/` — 3 evaluation scenarios per skill; re-run before releases
  that touch a SKILL.md or reference file.
- MCP tool references in skills use the qualified `ttai:tool_name` form.
- `plugin.json` (repo root) — Agent Plugins 1.0.0 manifest
  (<https://agent-plugins.org>); the portable format read natively by Cursor,
  Codex, GitHub Copilot, Kiro, and VS Code. `$schema` and `name` are required.
  [README.md](README.md) has the mermaid of how the skills compose.
- `.claude-plugin/`, `.codex-plugin/`, `.cursor-plugin/`, `.agents/plugins/` —
  platform manifests. Keep `version` in sync across all of them when releasing.
- `.mcp.json` — Claude-native MCP config (`"type": "http"`).
- `mcp.json` — Agent Plugins strict MCP config (`$schema` +
  `"type": "streamable-http"`). The two files carry the same server URL but
  deliberately different formats; when the MCP URL changes, update both (plus
  `skills/*/agents/openai.yaml`).
- `.plugin/marketplace.json` — read by the `npx plugins` CLI
  (vercel-labs/plugins) before `.claude-plugin/marketplace.json`. Its plugin
  `source` MUST stay the string `"./"` — the CLI treats non-string sources as
  remote and refuses to install them.
- `.claude-plugin/marketplace.json` plugin `source` must be the object form
  `{ "source": "github", "repo": "tough-tongue/toughtongue-skills" }`, not the
  string shorthand `"."`. Claude Code CLI accepts both; Claude Desktop/Cowork
  remote sync rejects the string form with "Marketplace sync failed."
- Never create a root-level `marketplace.json`: the `npx plugins` CLI prefers
  it when copying a marketplace into `~/.claude/plugins/`, which would leak a
  string source back into Claude tooling (the Cowork sync bug above).

## Descriptions

Exactly two product descriptions exist. Change both together; never fork a
third.

- **Coding agents** — one string in every plugin and marketplace manifest:
  `plugin.json`, `.claude-plugin/*`, `.codex-plugin/plugin.json`
  (`description` and `interface.longDescription`), `.cursor-plugin/*`,
  `.plugin/*`. Codex `interface.shortDescription` is the only short form.
- **Web agents** — `server.json` (MCP registry, read by web connectors).

## Versioning

Bump the version in all plugin manifests together (`plugin.json`,
`.claude-plugin/plugin.json`, `.codex-plugin/plugin.json`,
`.cursor-plugin/plugin.json`) on any user-visible change. The explicit `version` field controls when installed users receive
updates — without a bump, Claude Code users on marketplace installs do not
update.

## Releasing & Publishing

### Pre-release checklist (every release)

1. `claude plugin validate .` — must pass; the community-marketplace review
   pipeline runs the same check on submission.
2. Local smoke test, Claude Code: `claude --plugin-dir .` then invoke
   `/toughtongue:ttai-agent` (and `/reload-plugins` after edits).
3. Local smoke test, Codex: `codex plugin marketplace add <checkout-path>`,
   `codex plugin add toughtongue@toughtongue`, restart, verify the ttai MCP
   tools and all the skills appear.
4. Verify auth flows: OAuth first — complete the browser login (Claude Code:
   `/mcp`; Codex: `codex mcp login ttai`), then "Call the ttai MCP tool
   list_organizations" must succeed in both agents. Also spot-check the PAT
   fallback: a manual server config with the `TTAI_PAT` bearer header must
   still work.
5. Local smoke test, `npx plugins` CLI: `npx plugins discover .` must show 1
   local plugin with all skills + MCP; `npx plugins add . -t claude-code -s local -y`
   must install cleanly.
6. After push: in Claude Desktop/Cowork, Add marketplace → Sync must still
   succeed (regression check for the string-source sync bug).
7. Bump `version` in all four plugin manifests; tag the release.

### Submission assets

Reused across every channel below — do not go hunting for these again:

| Asset                  | URL                                             |
| ---------------------- | ----------------------------------------------- |
| Privacy policy         | <https://app.toughtongueai.com/privacy-policy/> |
| Terms of service       | <https://app.toughtongueai.com/terms/>          |
| Public MCP docs        | <https://app.toughtongueai.com/docs/mcp>        |
| PAT / developer portal | <https://app.toughtongueai.com/developer>       |

The paths are `/privacy-policy/` and `/terms/`; `/privacy` and `/tos` 404.

### Distribution channels

| Channel                                                  | Mechanism                                                                                                                                                                                                                                                                                                                                                                                               | Status                   |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| Claude Code (self-hosted marketplace)                    | This repo's `.claude-plugin/marketplace.json`; users run `/plugin marketplace add tough-tongue/toughtongue-skills`                                                                                                                                                                                                                                                                                      | Live on push             |
| Claude community marketplace (`@claude-community`)       | Submit at <https://platform.claude.com/plugins/submit> (Console, works for individual authors) or <https://claude.ai/admin-settings/directory/submissions/plugins/new> (Team/Enterprise). Review pins a commit SHA in `anthropics/claude-plugins-community`; CI auto-bumps on new pushes; catalog syncs nightly                                                                                         | Submit once              |
| Codex (GitHub marketplace)                               | This repo's `.agents/plugins/marketplace.json`; users run `codex plugin marketplace add tough-tongue/toughtongue-skills`. Codex also reads `.claude-plugin/marketplace.json` for compatibility                                                                                                                                                                                                          | Live on push             |
| Claude Connectors Directory (the in-app connectors list) | Submits the _hosted MCP server_ (`https://api.toughtongueai.com/api/public/mcp`), not this repo, via <https://claude.ai/admin-settings/directory/submissions/new>. Requires a Team or Enterprise Claude org with directory-management access. Server already meets the technical bar: streamable HTTP, OAuth 2.1 with dynamic client registration, PKCE S256, correct 401 `resource_metadata` discovery | Not submitted            |
| Codex official Plugin Directory                          | Publishing is "coming soon" per OpenAI docs — no self-serve yet. Interim: share to a ChatGPT workspace via Codex app → Plugins → Created by you → Share                                                                                                                                                                                                                                                 | Watch docs               |
| `npx plugins add` (cross-agent installer)                | The `plugins` npm CLI (vercel-labs/plugins) reads this repo's `.plugin/marketplace.json` and installs into every detected agent: Claude Code, Cursor, Codex, Grok Build, Kimi Code, GitHub Copilot CLI, VS Code. Users run `npx plugins add tough-tongue/toughtongue-skills`                                                                                                                            | Live on push             |
| Agent Plugins 1.0.0 (native clients)                     | Root `plugin.json` + `skills/` + `mcp.json` per <https://agent-plugins.org>; loaded directly by clients that support the standard (Cursor, Codex, GitHub Copilot, Kiro, VS Code)                                                                                                                                                                                                                        | Live on push             |
| skills.sh                                                | Indexes public GitHub repos with `skills/`                                                                                                                                                                                                                                                                                                                                                              | Live once repo is public |
