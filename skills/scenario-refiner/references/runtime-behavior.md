# Scenario Runtime Behavior

What a scenario *actually* becomes at runtime — system prompt assembly, tool
registration, conductor/silence mechanics, and the difference between browser
sessions and phone (SIP) sessions. Read this before diagnosing anything
architectural.

---

## 1. System Prompt Assembly

The `ai_instructions` field is never sent to the LLM as-is. At session start
it is compiled:

1. **Dynamic variable substitution** — `{{ var }}` placeholders are filled
   from URL parameters (`?t_company=Acme`). Unfilled placeholders remain as
   literal text, which the agent may speak aloud — every variable needs a
   documented missing-value fallback in the instructions.
2. **Template wrapping** — controlled by `strategy.system_instructions_template`:
   - unset (default) → **STANDARD** template: adds ~600-800 tokens of identity
     framing, role enforcement, and behavioral rules around `ai_instructions`.
   - `"minimal"` → **MINIMAL** template: adds only ~50 tokens (memory /
     language / context glue). Most newer scenarios use minimal.
   - Do NOT switch templates as a side effect of a fix — it changes overall
     agent behavior, not just the issue at hand.
3. **Tool instructions appended** — every tool with `add_to_system_prompt:
   true` injects its usage guidance into the compiled prompt.

The compiled prompt is fixed at session start. **Edits to a scenario only
affect sessions started after the update.**

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

- `should_register: true` → the function declaration is sent to the model.
  If `false`, the AI literally cannot call the tool.
- `add_to_system_prompt: true` → the tool's usage instructions are injected
  into the system prompt, so the AI knows WHEN and HOW to call it.
- `true` + `false` → the model can call it but has no guidance. Valid pattern
  when timing is handled entirely inside `ai_instructions`.
- When `tools_config` is missing entirely, `end_session` and `memory_search`
  default to on.

### `end_session` specifics

- Reads `tool_settings.disconnectDelaySeconds` (default 15): the agent's
  goodbye keeps playing for that many seconds before disconnect.
- Signature: `endSession(reason)` — the reason is logged, never user-visible.
- Always instruct the AI to say its goodbye BEFORE calling the tool.
  Premature calls are the #1 source of "the agent hung up on me" complaints.

### Session-surface differences

| Aspect | Browser session | Phone (SIP) session |
|---|---|---|
| Tool set | Full catalog (card, mcq, browser, slides, ...) | Server tools only: `end_session`, `knowledge_base_search`, `collect_data`, `cold_transfer` |
| Visual tools (card/mcq/slides) | Work | Silently unavailable — don't prescribe them for phone scenarios |
| Filler words / backchannel | Not available | Cascade voice pipeline only |
| Ambient sound | Not available | Available |

If a scenario is used over SIP and its instructions prescribe visual tools,
the agent will narrate actions it cannot perform. Fix the instructions, not
the config.

---

## 3. Conductor (Timed Mid-Call Messages)

```json
{
  "conductor": {
    "enabled": true,
    "prefix": "CONDUCTOR:",
    "messages": [
      { "time_seconds": 300, "message": "Wrap up the call. Confirm next step, end warmly.", "end_turn": true }
    ]
  }
}
```

- Messages are sorted by `time_seconds` and fired sequentially after the
  greeting completes. They are injected as internal system directives — the
  agent is told never to mention them aloud.
- `end_turn: true` interrupts current agent speech first; `false` queues the
  directive for the next turn boundary.
- If wrap-up fires mid-conversation, the timer is too low for the real call
  length distribution — check average `duration` across recent sessions
  (`list_sessions`) before picking a new value.
- Max-duration enforcement also flows through the conductor: warn → short
  grace period → hard disconnect.

---

## 4. Silence / Nudge Handling

```json
{
  "silence": {
    "silence_threshold": 4000,
    "end_session": false,
    "force_agent_to_speak": true
  }
}
```

Two modes after `silence_threshold` ms of nobody speaking:

- **Extreme** (`end_session: true`) — disconnect immediately. No nudge. Only
  for flows where silence genuinely means the user left.
- **Talkative** (`end_session: false`) — keep the call alive:
  - `force_agent_to_speak: true` → inject a static "check in with the user
    and continue" directive.
  - `force_agent_to_speak: false` → a lightweight LLM decides whether to
    interject and what to say (pushes the conversation forward, checks if
    the user is still there after repeated nudges).

Typical threshold: 4000-8000 ms. "Long silence then the call dropped"
almost always means `end_session: true` where talkative mode was wanted, or
a threshold that's too low.

---

## 5. Strategy Quick-Reference

| Field | Effect | Notes |
|---|---|---|
| `skip_auto_start` | Who speaks first | `false` = AI; `true` = user |
| `welcome_instructions` | Exact first beat | Directive form, never quoted text |
| `silence.silence_threshold` | ms before silence action | 4000-8000 typical |
| `silence.end_session` | Disconnect vs nudge | See section 4 |
| `conductor.messages[]` | Timed directives | See section 3 |
| `filler_words` | Comma-separated TTS phrases | Cascade voice pipeline only — silently ignored on realtime models |
| `system_instructions_template` | Prompt wrapper | `"minimal"` or unset (standard) |
| `disable_transcription` | Turn off STT | Rare |

If a scenario sets `filler_words` but `ai_model_config.model` is not
`cascade-01`, the filler config is silently ignored. Flag it if you spot it.

---

## 6. Common Failure Modes → Fix Locations

| Symptom | Owner | Mechanism |
|---|---|---|
| "end_session never called" | `ai_instructions` end-of-call block | Timing instruction unclear or `add_to_system_prompt: false` |
| "end_session called mid-conversation" | `ai_instructions` | Add "NEVER call before closing line + customer farewell" |
| "Wrap-up fires too early" | `strategy.conductor.messages[].time_seconds` | Bump to match real call length distribution |
| "Long silence then call drops" | `strategy.silence` | Threshold too low, or `end_session: true` when nudge was wanted |
| "Agent monologues at start" | `strategy.welcome_instructions` or opening flow | Add "Then STOP and wait" |
| "Agent doesn't speak first" | `strategy.skip_auto_start` | Should be `false` for AI-led calls |
| "Placeholder `{{ firstname }}` spoken literally" | `ai_instructions` CONTEXT block | Add "If `firstname` blank, open without a name." |
| "Wrong accent / language drift" | `appearance.language_code` + `transcribe_config` | Both must match the locale |
| "AI sounds robotic" | `ai_instructions` style rules | Add ONE concrete varied-acknowledgment example, not a paragraph |
| "AI says 'end_session' / 'tool' aloud" | GUARDRAILS section | Single NEVER bullet |
| "Restarts its opening when interrupted" | `strategy.welcome_instructions` | Quoted speech — rewrite as directive |
