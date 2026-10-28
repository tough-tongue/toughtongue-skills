# Built-in tools

Prompt rules for the built-in tools that misbehave without them. Field
mechanics (`should_register`, `add_to_system_prompt`, merging on update) are in
[control.md](../scenario/control.md#tools_config). Every tool here is web only.

## Any tool

- Cue each call in the FLOW step that needs it, by its function name.
- Never speak tool or function names; list them in the GUARDRAILS NEVER line.
- Bracketed messages (`[conductor:timer]`, `[board]`, `[desktop:<app>]`) come
  from the runtime, not the user. Say what each one means.

## Timed answers: card + timer

For video essays, timed interview answers, and think-then-answer drills.

- `setTimer(durationInSeconds, message)` starts a countdown. Its result means
  the clock started, not that time is up. Time is up only when
  `[conductor:timer] Timer complete - "<message>"` arrives, so give each timer
  a distinct `message`.
- A forced turn (the tool result, a silence nudge) often comes first: the
  prompt says what to say then, and not to advance.
- A new `setTimer` cancels the running one.
- Show the question with `runCardCommand` (`command: publish`, `title`,
  `description`); never read it aloud.

```
## TIMER WAIT (overrides the generic timer instructions)
The setTimer result means the clock STARTED. Time is up only when
"[conductor:timer] Timer complete" arrives with that timer's message.
If anything else arrives first, say one short line that they can keep going.
Do not advance and do not call setTimer again: a new timer cancels this one.

## FLOW — for each question N of <M>
a. runCardCommand publish, title "Question N of <M>", description the exact
   question. Never read the card aloud.
b. Say they have <30> seconds to think; setTimer <30>, message "think time".
c. Only after "[conductor:timer]" for think time: invite the answer; setTimer
   <60>, message "answer time".
d. When answer time ends or they clearly finish, thank them and move on.
```

## Document capture: webcam_snapshot + collect_data

For video KYC and ID checks.

- `captureWebcam` (`command: capture`) takes one frame; the image arrives in
  the agent's next message. It waits, and never describes a document it has
  not seen.
- Name the fields to read; the step passes only when they are readable.
  Unreadable: one coaching line (closer, steady, away from the light), then
  capture again, up to four tries, then mark it not captured.
- Never invent a number, name or date. Say only the last four characters of an
  ID number aloud.
- `collectData` runs once, after the last check, with every field
  (`key: value`, one per line) and `not captured` for gaps. With extraction on,
  start each variable's description with "Prefer the collectData note."

## Notes and a design board: notepad + architecture_board

For system-design interviews and workshops.

- `notepad`: notes the agent writes (`show`, `read`, `overwrite` in markdown).
  `read` before writing so the user's edits survive.
- `architecture_board`: the user's canvas; the agent gets `[board]` events.
  `read` it before any summary or critique, ask when speech and board
  disagree, and never `clear` it without their agreement.
- One surface is on screen at a time, and each notepad command pulls the board
  off. `show` the board once at the start so it stays in the panel's arrows.
- Hint with the board's component names (Service, Queue, Blob Store, CDN,
  Load Balancer).
