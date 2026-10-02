# Scenario runtime behavior

What a scenario _actually_ becomes at runtime — system prompt assembly, tool
registration, conductor/silence mechanics, and the difference between browser
sessions and phone (SIP) sessions. Read this before diagnosing a behavior
failure or changing a live Scenario.

## Contents

- System prompt assembly (variables, templates, new-sessions-only)
- Tool system (two-axis control, end_session, browser vs SIP surfaces)
- Conductor (timed mid-call messages)
- Silence / nudge handling
- Strategy quick-reference
- Common failure modes → fix locations

---

## 1. System Prompt Assembly

The `ai_instructions` field is never sent to the LLM as-is. At session start it
is compiled:

1. **Dynamic variable substitution** — `{{ var }}` placeholders are filled from
   the run's variables (`?t_company=Acme` on the run link, `dynamic_vars` on
   calls and bots). An unset variable renders as empty text, so the sentence
   around it breaks — every variable needs a documented missing-value fallback
   in the instructions.
2. **Template wrapping** — controlled by
   `strategy.system_instructions_template`:
   - unset (default) → **STANDARD** template: about 550 tokens of identity
     framing, behavioral rules, and interaction guidelines around
     `ai_instructions`.
   - `"minimal"` → **MINIMAL** template: about 300 tokens (date, language,
     memory, and text-message handling). Most newer scenarios use minimal.
   - Do NOT switch templates as a side effect of a fix — it changes overall
     agent behavior, not just the issue at hand.
3. **Tool instructions appended** — every tool with `add_to_system_prompt: true`
   injects its usage guidance into the compiled prompt.

The compiled prompt is fixed at session start. **Edits to a scenario only affect
sessions started after the update.**

**Greeting:** if `skip_auto_start` is false, the worker asks the model to "Greet
the user with your opening line." That line must exist in FLOW. There is no
`strategy.welcome_instructions` field.

---

## 2. Tool System

### Two-axis control

Every entry in `tools_config.tools.<tool_id>` has two booleans:

```json
{
  "end_session": {
    "should_register": true,
    "add_to_system_prompt": true,
    "tool_settings": { "disconnectDelaySeconds": 12 }
  }
}
```

- `should_register: true` → the function declaration is sent to the model. If
  `false`, the AI literally cannot call the tool.
- `add_to_system_prompt: true` → the tool's usage instructions are injected into
  the system prompt, so the AI knows WHEN and HOW to call it.
- `true` + `false` → the model can call it but has no guidance. Valid pattern
  when timing is handled entirely inside `ai_instructions`.
- A Scenario created without `tools_config` gets a broad default set (most tools
  registered, only `end_session` prompted). Check what is actually registered
  before blaming the prompt.

### `end_session` specifics

- Reads `tool_settings.disconnectDelaySeconds` (default 15): the agent's goodbye
  keeps playing for that many seconds before disconnect.
- `endSession(reason)` schedules the end (the reason is logged, never spoken);
  `endSession(cancel=true)` cancels a pending end when the user substantively
  re-opens the conversation. Thanks, goodbyes, and noise do not cancel.
- The goodbye is its own turn; call the tool only after the user's reply.
  Premature calls are the #1 source of "the agent hung up on me" complaints.

### Session-surface differences

- **Tool set** — browser: full catalog (card, mcq, browser, slides, ...). Phone
  (SIP): server-side only — `end_session`, `knowledge_base_search`,
  `collect_data`, `cold_transfer`, `ask_conductor` if an on-demand conductor
  message is set, and attached Custom Functions and MCP servers.
- **Visual tools (card/mcq/slides)** — browser: work. Phone (SIP): silently
  unavailable — don't prescribe them for phone scenarios.

Filler words and backchannel depend on the pipeline, not the surface: they need
Landmass `cascade`.

If a scenario is used over SIP and its instructions prescribe visual tools, the
agent will narrate actions it cannot perform. Fix the instructions, not the
config.

**Pipeline:** do not infer the pipeline from SIP alone. For outbound cold calls,
start with the situation recipe: Cascade is preferred for controllable external
TTS, while Ocean realtime is the documented fallback when Cascade/Cartesia is
unavailable.

---

## 3. Conductor (Timed Mid-Call Messages)

```json
{
  "conductor": {
    "enabled": true,
    "messages": [
      {
        "time_seconds": 300,
        "message": "Wrap up the call. Confirm next step, end warmly.",
        "end_turn": true
      }
    ]
  }
}
```

- Messages are sorted by `time_seconds` and fired sequentially after the
  greeting completes. They are injected as internal system directives — the
  agent is told never to mention them aloud. `prefix` is deprecated and ignored.
- `end_turn: true` interrupts current agent speech first; `false` queues the
  directive for the next turn boundary.
- `trigger: "on_demand"` (at most one message) registers `ask_conductor`: the
  agent calls it to get a fresh directive built from the live transcript.
- If wrap-up fires mid-conversation, the timer is too low for the real call
  length distribution — check average `duration` across recent sessions (legacy
  `ttai:list_sessions` returns `duration`; `ttai:get_analytics` gives
  aggregates) before picking a new value.
- `strategy.max_duration_seconds` also flows through the conductor: a wrap-up
  directive 30 s before the cap, then a hard disconnect.

---

## 4. Silence / Nudge Handling

```json
{
  "silence": {
    "silence_threshold": 6000,
    "end_session": false,
    "force_agent_to_speak": true
  }
}
```

Two modes after `silence_threshold` ms of nobody speaking:

- **Extreme** (`end_session: true`) — disconnect immediately. No nudge. Only for
  flows where silence genuinely means the user left.
- **Talkative** (`end_session: false`) — keep the call alive:
  - `force_agent_to_speak: true` → inject a static "check in with the user and
    continue" directive.
  - `force_agent_to_speak: false` → a lightweight LLM decides whether to
    interject and what to say (pushes the conversation forward, checks if the
    user is still there after repeated nudges).

If `silence` is omitted, the platform default is **120000 ms then hang up**
(`end_session: true`). Always set talkative mode for practice.

Minimum threshold: 3000 ms. Typical practice threshold: 5000-8000 ms. "Long
silence then the call dropped" almost always means the default hung up,
`end_session: true` where a nudge was wanted, or a threshold that's too low.

---

## 5. Strategy Quick-Reference

Field meanings and defaults live in [control.md](control.md#strategy). Runtime
trap: if a scenario sets `filler_words` or `backchannel` but
`ai_model_config.model` is not `cascade`, that config is silently ignored. Flag
it if you spot it. Never write `cascade-01` — use `cascade`.

---

## 6. Common Failure Modes → Fix Locations

Each symptom lists its owner (where to fix), then the mechanism.

- **"end_session never called"** — `ai_instructions` end-of-call block: timing
  instruction unclear or `add_to_system_prompt: false`.
- **"end_session called mid-conversation"** — `ai_instructions`: add "NEVER call
  before closing line + customer farewell".
- **"Wrap-up fires too early"** — `strategy.conductor.messages[].time_seconds`:
  bump to match real call length distribution.
- **"Long silence then call drops"** — `strategy.silence`: omitted (120s
  hangup), or `end_session: true` when a nudge was wanted.
- **"Agent monologues at start"** — `ai_instructions` FLOW opening: directive
  Beat 1 + "Then STOP and wait"; `skip_auto_start: false`.
- **"Restarts its opening when interrupted"** — `ai_instructions` FLOW: quoted
  opening — rewrite as a directive.
- **"Opening breaks when a name is missing"** — `ai_instructions` CONTEXT block:
  add "If `firstname` is blank, open without a name."
- **"Wrong accent / language drift"** — `appearance.language_code` +
  `transcribe_config`: both must match the locale.
- **"AI sounds robotic"** — `ai_instructions` style rules: add ONE concrete
  varied-acknowledgment example, not a paragraph.
- **"AI says 'end_session' / 'tool' aloud"** — GUARDRAILS section: single NEVER
  bullet.
- **"Agent doesn't speak first"** — `strategy.skip_auto_start`: should be
  `false` for AI-led calls.

## Key Files

- [workflow.md](workflow.md) — refine procedure
- [model-selection.md](model-selection.md) — pipeline choice
- [control.md](control.md) — behavior controls
- [ai-instructions.md](ai-instructions.md) — prompt refinements
