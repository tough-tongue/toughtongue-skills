# Scenario

A **scenario** is a Tough Tongue AI voice (or text) agent definition — the
primary authorable resource in the datastore.

## Contents

| File                                       | Load when                              |
| ------------------------------------------ | -------------------------------------- |
| [model-selection.md](model-selection.md)   | Stamping `ai_model_config` / voices    |
| [control.md](control.md)                   | Strategy, tools, appearance, flags     |
| [ai-instructions.md](ai-instructions.md)   | Writing or repairing `ai_instructions` |
| [rubrik.md](rubrik.md)                     | Scoring / evaluation text              |
| [processed-rubrik.md](processed-rubrik.md) | Structured criteria from rubrik        |

Situation playbooks live under
[../../scenario-recipes/](../../scenario-recipes/index.md) — not here.

## Create vs update (MCP)

| Op         | Rule                                                                   |
| ---------- | ---------------------------------------------------------------------- |
| Create     | `ttai:create_scenario` — **no** `id`; needs `name` + `ai_instructions` |
| Update     | `ttai:update_scenario` — **requires** `id`; partial                    |
| Draft help | `ttai:generate_scenario` — still create/update after                   |

`type` over MCP: `"default"` \| `"super"` \| `"composite"` only.

Edits apply to **new** sessions, not a call already in progress.

## Authoring order

1. Pick workspace ([../key-control-entities.md](../key-control-entities.md)).
2. Load [../../scenario-authoring.md](../../scenario-authoring.md).
3. Load [model-selection.md](model-selection.md) — always set `provider` + `model`.
4. Load [control.md](control.md) — strategy, tools, appearance.
5. Load [ai-instructions.md](ai-instructions.md) + one **recipe**.
6. Optional rubrik → [rubrik.md](rubrik.md).
7. Call MCP via [../../../features/mcp/](../../../features/mcp/index.md).

## Next

- Run across channels → [../scenario-engine.md](../scenario-engine.md)
- Sharing → [../auth-and-sharing.md](../auth-and-sharing.md)

## Key Files

- [../../scenario-authoring.md](../../scenario-authoring.md)
- [model-selection.md](model-selection.md)
- [control.md](control.md)
- [ai-instructions.md](ai-instructions.md)
- [rubrik.md](rubrik.md)
