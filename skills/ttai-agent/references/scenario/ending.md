# Ending the call

How a Scenario's agent hangs up at the right moment and stays polite while the
line closes. Read this when writing the end of `ai_instructions`, or when users
report hang-ups on a short reply, a goodbye followed by silence, or an agent
that ends mid-flow. Phone calls need the full block; most web sessions need
only two lines.

## Contents

- What `end_session` does
- Two states
- The ENDING THE CALL block
- When the tool fires on a completed call
- Settings
- Silence, voicemail, and the conductor
- Failure modes and fixes

---

## What `end_session` does

- `end_session` is the tool id in `tools_config.tools`. The model calls it as
  `endSession`; use that name inside `ai_instructions`.
- It hangs up. It is not a phase change, a progress note, or a "section done"
  marker: moving between FLOW steps needs no tool call.
- `endSession(reason)` schedules the end. The reason is logged, never spoken.
  The line stays live for `tool_settings.disconnectDelaySeconds` (default 15),
  then drops.
- `endSession(cancel=true)` cancels a pending end. A second
  `endSession(reason)` while an end is pending does nothing.

## Two states

The agent is always in one of two states:

- **Live call** — from the start, or after a cancel. The agent ends the call
  only in a short list of situations; everything else is not an exit.
- **Farewell mode** — `endSession(reason)` has been called and the line is
  still live until the delay runs out. The agent answers pleasantries without
  a tool and cancels first when the person really re-engages.

## The ENDING THE CALL block

Add this as its own section of `ai_instructions` and fill the bracketed parts.
In FLOW, end every closing line with "Then end the call" and every question
with "Then STOP and wait", so the block's trigger is unambiguous. The words in
the lists are what the other person says, not lines for the agent; add the
call's own language (for Hinglish: haan, जी, ठीक है).

```
## ENDING THE CALL (overrides every other instruction about ending)

`endSession` HANGS UP THE CALL. It is not a phase change or a "section done"
marker; moving between FLOW steps needs no tool call. `endSession(reason)`
schedules the end; `endSession(cancel=true)` cancels it. You are always in
one of two states.

### STATE 1 — LIVE CALL (you have not called endSession)
- Call `endSession(reason)` in exactly these situations, nothing else:
  1. Completed call: you said a line this prompt marks "Then end the call"
     AND they answered it (a goodbye, thanks, okay). Your goodbye is its own
     turn; wait for their reply before the tool.
  2. Hostile stop: a firm, explicit demand to stop calling. Apologize in one
     short line and call `endSession(reason)` in that turn.
  3. [Only when this prompt handles voicemail] Voicemail or IVR: leave the
     message as one turn, then call `endSession(reason)` without waiting.
- Never in the same turn as a question. If your last line asked anything,
  stop and wait for the reply.
- These are NOT exit signals; acknowledge and continue: a bare ok, yes or
  hmm; a short answer to your own question; silence or low energy; a single
  soft no; a refusal to share one detail; anything garbled. Phone audio
  garbles short replies, so you are far more likely mishearing than hearing
  a real exit. Ending too early is the worst mistake you can make.

### STATE 2 — FAREWELL MODE (endSession called; line drops in ~[10] s)
- The line is still live. Stay present until it drops.
- Pleasantries (bye, thank you, okay, background noise, another voice,
  anything garbled): reply in one warm line if it merits one. Call no tool.
- Real re-engagement (a substantive question, a correction, wait, one more
  thing, a new topic, an objection to ending): call `endSession(cancel=true)`
  FIRST, then answer. You are back in STATE 1.
- Never call `endSession(reason)` again while an end is pending; it does
  nothing. Never use cancel=true to end a call.
```

Drop situation 3 when answering machine detection is on (see below). Keep the
number in the STATE 2 heading equal to `disconnectDelaySeconds`.

## When the tool fires on a completed call

Both variants keep farewell mode; they differ only in situation 1.

**Wait for their goodbye (default).** The closing line is its own turn; the
agent calls `endSession(reason)` after the other person answers it. Use it for
conversations, interviews, research calls, roleplays, and web sessions: the
person always gets a turn before the end is scheduled. Its cost: after the
goodbye the line waits on them, and if they stay silent or the agent forgets
the tool, the call sits open in silence.

**Tool with the closing line (scripted outbound calls).** For a scripted caller
whose closing line settles the call (a confirmed callback time, a clear no, a
wrong number), replace situation 1 with:

```
  1. Completed call: their answer settles the call (a confirmed time, a
     clear no, a postponement), so your reply is the closing turn. Start that
     reply with the `endSession(reason)` call, then say the full closing
     line. The line drops a few seconds after the tool call, so the closing
     line comes right after it. Never end the turn with only the line.
```

The disconnect delay is the window for their goodbye, and farewell mode
answers it. Gains: no silent wait after the goodbye, and the agent cannot
forget to hang up. Cost: the close is committed before they speak, so a late
question depends on the agent cancelling in farewell mode. Pick it only when
the script fixes the closing line and the person rarely has more to say.

**Web sessions** need only the default in two lines: say a short goodbye as
its own turn, then call `endSession` after their reply, or right away when a
`[conductor]` note says time is up. Never in a turn that asks a question.

## Settings

- `tools_config.tools.end_session`: `should_register: true`. Set
  `add_to_system_prompt: false` when `ai_instructions` carries the block, so
  the block is the only ending rule. A web Scenario with the two-line rule, or
  no ending section, can keep `true`.
- `tool_settings.disconnectDelaySeconds` is the farewell window: about 10–12 on
  phone calls (room for their goodbye and one warm reply), 5–8 on the web.
- `tool_settings` is replaced whole on update: read the current `end_session`
  entry, merge, and send the full entry with `ttai:update_scenario`.

## Silence, voicemail, and the conductor

- **Silence.** `strategy.silence.end_session: true` hangs up after
  `silence_threshold` with no goodbye and no farewell mode; it bypasses the
  block. Keep it for flows where silence means the person left. Calls that
  carry the block use talkative silence (`end_session: false`). An omitted
  `silence` means 120 s, then hang up ([runtime.md](runtime.md)).
- **Voicemail.** When `meeting_config.amd_config.enabled` is true on outbound
  calls, answering machine detection handles voicemail and IVR
  (`voicemail.action` `hangup`, or `leave_message` with `voicemail.message`;
  see [control.md](control.md)). The prompt then needs no voicemail rule:
  drop situation 3. With detection off, keep situation 3 and a voicemail
  section that gives the message.
- **Conductor.** With the block in place, a wrap-up message points at it
  (close as ENDING THE CALL says) instead of restating timing. A final
  time-is-up message may tell the agent to say its goodbye and call
  `endSession` in the same turn.

## Failure modes and fixes

Read the transcripts first (`ttai:list_resources(sessions)` filtered by
`scenario_ids`, with `include_fields: ["transcript"]`): see whether the call
ended after a short reply, after the agent's own line, mid-flow, or after a
long silence. Fix the one cause the evidence shows.

- **Hangs up on ok, haan, or a short answer** — a soft reply read as an exit.
  Add the not-exit-signals list with the call's own words.
- **Says goodbye, then sits in silence** — the tool never fired after the
  reply. Check `should_register: true`, mark closing lines "Then end the
  call", and make situation 1 explicit. For a scripted caller, offer the
  tool-with-closing-line variant.
- **Ends the call right after a FLOW step** — `endSession` used as a phase
  marker. Add the "hangs up, not a phase marker" line; remove FLOW wording
  that ties finishing a step to a tool call.
- **Repeats the goodbye or calls the tool again** — no farewell mode. Add
  STATE 2.
- **Reopens the call on thanks or bye** — cancels on pleasantries. STATE 2:
  pleasantries get one warm line and no tool.
- **Line drops while answering a late question** — no cancel rule. STATE 2:
  real re-engagement cancels first, then answers.
- **Drops after a long pause with no goodbye** — the silence setting, not the
  tool. Set `strategy.silence.end_session: false`.
- **Never hangs up** — the tool is not registered, or
  `add_to_system_prompt: false` with no ending section in the prompt. Register
  it and add the block.

Edits affect new sessions only.

## Key Files

- [ai-instructions.md](ai-instructions.md) — prompt skeleton
- [runtime.md](runtime.md) — tool registration and silence
- [control.md](control.md) — `tools_config` and `meeting_config`
