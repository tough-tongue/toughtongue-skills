# Ending the call

How the agent hangs up at the right moment. Phone calls carry the full block
below; most web sessions need only the two-line rule at the end.

- `end_session` is the tool id; the model calls it `endSession`. It hangs up:
  never a phase change. `endSession(reason)` schedules the end and the line
  stays live for `disconnectDelaySeconds` (default 15);
  `endSession(cancel=true)` cancels it.
- With the block in the prompt, set `end_session.add_to_system_prompt: false`
  so the block is the only ending rule, and `disconnectDelaySeconds` to about
  10–12 on phone. `tool_settings` is replaced whole: send the full entry.
- With answering machine detection on (`meeting_config.amd_config`), drop
  situation 3.

## The block

Add it as its own section of `ai_instructions`. In FLOW, end each closing line
with "Then end the call" and each question with "Then STOP and wait". Add the
call's own words to the not-exit list (Hinglish: haan, जी, ठीक है).

```
## ENDING THE CALL (overrides every other instruction about ending)
endSession HANGS UP THE CALL. It is not a phase change; moving between FLOW
steps needs no tool call.

### STATE 1 — LIVE CALL
- Call endSession(reason) only for:
  1. Completed call: you said a line marked "Then end the call" AND they
     answered it. Your goodbye is its own turn.
  2. Hostile stop: a firm demand to stop calling. Apologize in one line and
     call endSession(reason) in that turn.
  3. Voicemail or IVR: leave the message as one turn, then endSession(reason).
- Never in the same turn as a question.
- Not exit signals; acknowledge and continue: a bare ok, yes or hmm; a short
  answer to your question; silence; a soft no; anything garbled.

### STATE 2 — FAREWELL MODE (endSession called; line drops in ~10 s)
- Pleasantries (bye, thanks, noise): one warm line at most, no tool.
- Real re-engagement (a question, a correction, "wait", a new topic): call
  endSession(cancel=true) FIRST, then answer. You are back in STATE 1.
- Never call endSession(reason) again while an end is pending.
```

## Scripted callers: end with the closing line

For a scripted outbound call whose closing line settles it (a booked time, a
clear no), situation 1 can instead read: their answer settles the call, so
start your reply with endSession(reason), then say the full closing line. The
disconnect delay leaves room for their goodbye, and farewell mode answers it.
Use this only when the script fixes the closing line; the default waits for
their reply.

## Web sessions

Say a short goodbye as its own turn, then call `endSession` after their reply,
or at once when a `[conductor]` note says time is up. Never in a turn that
asks a question.
