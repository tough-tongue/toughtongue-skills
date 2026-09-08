---
name: ttai-agent
description: >
  Tough Tongue AI map for any ttai question: workspaces, scenarios, sessions,
  MCP tools, model stamps, and recipes. Use when the user says "Tough Tongue",
  "ttai", "create a voice agent", "what is a scenario", "list my orgs", or
  asks about SIP, Meet bots, rubrics, or installing the skills plugin.
user-invocable: false
when_to_use: >
  Any Tough Tongue AI product, MCP, account, scenario, session, channel, or recipe question.
---

# Tough Tongue AI Agent

`ttai-agent` is the **intermediate intelligence layer** between a human's
intent and their Tough Tongue AI entities. It resolves account scope, focus,
capability, evidence, and the smallest safe next action. Coding agents and
web conversational plugins consume the same knowledge; their tool adapters
perform the action.

Tough Tongue AI is a **datastore + runtime** for tough conversations. Some
conversations the AI takes (outbound/inbound voice agents, demos, screening,
booking). Others you nail (realistic roleplay for negotiations, interviews,
coaching).

## Mental model

| Layer        | Meaning                                                                                  |
| ------------ | ---------------------------------------------------------------------------------------- |
| **Intent**   | The human outcome: orient, inspect, author, change, run, analyze, or share               |
| **Context**  | Personal or organization scope, capability, and an optional focused resource             |
| **Entities** | Scenarios, collections, sessions, SIP, meeting bots, browser contexts, and access tokens |
| **Runtime**  | Web, embed, SIP, or meeting-bot execution plus post-session results                      |
| **MCP**      | Current public action adapter; live schemas and tool results are authoritative           |

Start with [kb/operating-model.md](kb/operating-model.md) for context reuse,
scope resolution, capability discipline, and the consumer-neutral action loop.
Load depth files only when the job needs them.

## Load by job

| Job                           | Load                                                                               |
| ----------------------------- | ---------------------------------------------------------------------------------- |
| Scope / capabilities          | [kb/operating-model.md](kb/operating-model.md)                                     |
| What exists                   | [kb/entities/](kb/entities/index.md)                                               |
| Scenario quality principles   | [kb/scenario-authoring.md](kb/scenario-authoring.md)                               |
| Model stamp                   | [kb/entities/scenario/model-selection.md](kb/entities/scenario/model-selection.md) |
| Control fields / instructions | [kb/entities/scenario/](kb/entities/scenario/index.md)                             |
| Run channels + results        | [kb/entities/scenario-engine.md](kb/entities/scenario-engine.md)                   |
| Situation recipes             | [kb/scenario-recipes/](kb/scenario-recipes/index.md)                               |
| MCP connect / tools           | [features/mcp/](features/mcp/index.md)                                             |
| Create / edit / refine        | workflow **scenario-maker**                                                        |
| Analyze sessions              | workflow **session-analyst**                                                       |
| Browser demo steps            | workflow **browser-demo-builder**                                                  |
| First-run / MCP health        | [kb/operating-model.md](kb/operating-model.md)                                     |

## Hard rules

- Separate **platform support** from the user's **current entitlement**. The
  server decides role, plan, and balance; never infer them from a scenario.
- Reuse a current, verified context supplied by the consumer. Otherwise resolve
  organization scope before a stateful request. Ask only when a write's scope
  is ambiguous.
- MCP tools are the contract — only call tools listed in
  [features/mcp/tools-by-resource.md](features/mcp/tools-by-resource.md) /
  repo [MCP.md](../../MCP.md).
- Load the tool's input schema before writing. Do not invent fields.
- Never ask the user to paste a PAT into chat.
- Advice stops at a recommendation. Do not turn it into a mutation, phone
  call, or meeting-bot deployment without the user's intent.
- Terminology: **Tough Tongue AI** (two words). Practice link:
  `https://app.toughtongueai.com/run/<scenario_id>`.

## Key Files

- [kb/operating-model.md](kb/operating-model.md) — portable intelligence contract
- [kb/index.md](kb/index.md) — knowledge base root
- [features/index.md](features/index.md) — product features (MCP first)
- [kb/entities/resources.md](kb/entities/resources.md) — resource universe
- [kb/entities/key-control-entities.md](kb/entities/key-control-entities.md) — users / orgs / billing
- [features/mcp/index.md](features/mcp/index.md) — act via MCP
