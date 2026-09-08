# MCP feature

How to **act** on the Tough Tongue AI datastore through the `ttai` MCP
server. What to put in a scenario lives under [../../kb/](../../kb/index.md).

Server: `https://api.toughtongueai.com/api/public/mcp` (Streamable HTTP).
Full client install matrix: repo [MCP.md](../../../../MCP.md).
Context, focus, and entitlement discipline:
[kb/operating-model.md](../../kb/operating-model.md).

## Contents

| File                                         | Topic                                    |
| -------------------------------------------- | ---------------------------------------- |
| [connect.md](connect.md)                     | OAuth vs PAT, organizations, conventions |
| [tools-by-resource.md](tools-by-resource.md) | Tool catalog grouped by resource         |

## Do not put here

- `ai_instructions` skeletons → [../../kb/entities/scenario/ai-instructions.md](../../kb/entities/scenario/ai-instructions.md)
- Model stamps → [../../kb/entities/scenario/model-selection.md](../../kb/entities/scenario/model-selection.md)
- Strategy / tools → [../../kb/entities/scenario/control.md](../../kb/entities/scenario/control.md)
- Situation playbooks → [../../kb/scenario-recipes/](../../kb/scenario-recipes/index.md)

Those files are knowledge. This folder only teaches connection and tool use.

## MCP resource: agent guide

The hosted server also publishes a markdown resource:

- URI: `ttai://guide/mcp-agent` (name **mcp-agent-guide**)
- Content: skills repo, plugins, global vs repo install scope, how this MCP
  relates to the plugin

Prefer reading that resource (or the skills README) when the user asks how
to install skills. Some clients still ignore MCP resources — fall back to
linking <https://github.com/tough-tongue/toughtongue-skills>.

## Smoke test

```text
Call ttai:list_organizations and show the result.
```

## Key Files

- [connect.md](connect.md) — OAuth, scope, and action conventions
- [tools-by-resource.md](tools-by-resource.md) — current tool catalog
- [../../kb/operating-model.md](../../kb/operating-model.md) — consumer-neutral protocol
