# Skill Evaluations

Evaluation scenarios for each skill, in the format from Anthropic's
[skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices#build-evaluations-first).
There is no built-in runner — run each `query` against an agent with the
plugin installed and an authenticated ttai MCP connection (OAuth login, or a
`TTAI_PAT` bearer header on headless setups), then grade against
`expected_behavior`.

| File                                                             | Skill under test          |
| ---------------------------------------------------------------- | ------------------------- |
| [ttai-agent.json](ttai-agent.json)                               | ttai-agent                |
| [ttai-session-analyst.json](ttai-session-analyst.json)           | ttai-session-analyst      |
| [ttai-browser-demo-builder.json](ttai-browser-demo-builder.json) | ttai-browser-demo-builder |
| [ttai-agent-apps.json](ttai-agent-apps.json)                     | ttai-agent-apps           |

Grading notes:

- Evals hit the live Tough Tongue AI API — run them in a test organization, and
  clean up created scenarios afterward.
- A pass requires every `expected_behavior` item to be observed, not a
  majority.
- Re-run all evals before any release that touches a SKILL.md or reference
  file.
