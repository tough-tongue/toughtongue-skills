# Built-in tools

What each `tools_config.tools` entry does, the function the model calls, and
the prompt rules that make it behave. Field mechanics (`should_register`,
`add_to_system_prompt`, merging on update) are in
[../scenario/control.md](../scenario/control.md#tools_config); how tools
differ between web and phone sessions is in
[../scenario/runtime.md](../scenario/runtime.md).

## Contents

- Catalog
- Prompting any tool
- Timed answers: card + timer
- Document capture: webcam_snapshot + collect_data
- Shared notes and a design board: notepad + architecture_board
- Other tool ids

## Catalog

Web means the run link and embeds. Phone (SIP) runs only the server-side tools
listed in [../scenario/control.md](../scenario/control.md#tools_config).
"Model calls" is the name to use in `ai_instructions`; a dash means the
runtime's own guidance names it, so keep `add_to_system_prompt: true` and
describe the action in plain words.

| Tool id                 | What it does                                    | Model calls                                                                         | Runs on    | Settings                                          |
| ----------------------- | ----------------------------------------------- | ----------------------------------------------------------------------------------- | ---------- | ------------------------------------------------- |
| `end_session`           | Ends the session after the goodbye              | `endSession`                                                                        | web, phone | `disconnectDelaySeconds` (default 15)             |
| `card`                  | Shows a titled text card                        | `runCardCommand` (`command: publish`, `title`, `description`)                       | web        | —                                                 |
| `mcq`                   | Multiple-choice knowledge check                 | —                                                                                   | web        | —                                                 |
| `emoji_reaction`        | Reacts with an emoji                            | —                                                                                   | web        | —                                                 |
| `image_generation`      | Generates a teaching image                      | —                                                                                   | web        | —                                                 |
| `slide_generation`      | Generates a teaching slide                      | —                                                                                   | web        | —                                                 |
| `google_slides`         | Shows one slide of a published deck             | `runGoogleSlidesCommand` (`command: show`, `slideNumber`)                           | web        | `embedUrl`                                        |
| `browser`               | Drives a live product demo                      | `runBrowserCommand`                                                                 | web        | `initialUrl`, `steps` (platform sets `contextId`) |
| `timer`                 | Starts a countdown and reports when it ends     | `setTimer(durationInSeconds, message)`                                              | web        | —                                                 |
| `webcam_snapshot`       | Captures one camera frame for the model to read | `captureWebcam` (`command: capture`)                                                | web        | —                                                 |
| `collect_data`          | Records answers as one structured note          | `collectData`                                                                       | web, phone | —                                                 |
| `notepad`               | Shared notes the agent writes                   | commands `show`, `overwrite`, `read`                                                | web        | —                                                 |
| `architecture_board`    | The user's diagram canvas                       | commands `show`, `read`, `clear`; sends `[board]` events                            | web        | —                                                 |
| `agent_desktop`         | Runs Agent Desktop apps beside the call         | `desktop_launch`, `desktop_info`, `desktop_read`, `desktop_write`, `desktop_delete` | web        | `apps: {"<app name>": true}`                      |
| `knowledge_base_search` | Searches a legacy Knowledge Base                | —                                                                                   | web, phone | needs `knowledge_base_ids`                        |
| `cold_transfer`         | Transfers the call                              | —                                                                                   | phone      | `transfer_to`                                     |
| `ask_conductor`         | Fetches a fresh directive on demand             | `ask_conductor`                                                                     | web, phone | registered by an on-demand conductor message      |

Where each one goes next:

- `google_slides` — SLIDE MAP and link rules in
  [../scenario/control.md](../scenario/control.md#google-slides-decks).
- `browser` — [../recipes/demo.md](../recipes/demo.md); recorded steps belong
  to the `ttai-browser-demo-builder` skill.
- `agent_desktop` — building and attaching apps belongs to the
  `ttai-agent-apps` skill. App events reach the agent as
  `[desktop:<app>] <name> <json>`.
- Knowledge FileBases need no entry: `knowledge_filebase_ids` gives the agent
  `getKnowledgeBite` and `listKnowledgeBites`.

## Prompting any tool

- Register only the tools the FLOW uses (`should_register: false` for any
  other tool already in the config), and send both booleans for each:
  `add_to_system_prompt` defaults to `false` when omitted.
- `add_to_system_prompt: true` adds the runtime's usage guidance. Keep it on
  unless `ai_instructions` carries complete rules for that tool (a deck with a
  full SLIDE MAP can run with it off). When the prompt's rule differs from the
  runtime guidance, say in the prompt that its rule overrides it.
- Cue each call in the FLOW step that needs it, with the function name and
  arguments from the catalog.
- Never speak tool or function names: list them in the GUARDRAILS NEVER line.
- Never read card or slide text word for word: explain it in your own words,
  or let it stand (a timed question card). An MCQ is the exception: show it,
  then read the question ([../recipes/coaching.md](../recipes/coaching.md)).
- Bracketed messages (`[conductor:timer]`, `[board]`, `[desktop:<app>]`) come
  from the runtime, not the user. Say what each one means and that it is not
  speech.
- Never prescribe a web-only tool in a Scenario that runs on phone.

## Timed answers: card + timer

For video-essay practice, timed interview answers, and think-then-answer
drills. Web only.

- `setTimer` starts the clock. Its result ("Timer set for N seconds") means
  the clock started, not that time is up.
- Time is up only when `[conductor:timer] Timer complete - "<message>"`
  arrives. Give each timer a distinct `message` ("think time", "answer time")
  so the agent knows which one ended.
- A forced turn often arrives before that (the tool result itself, or a
  silence nudge). The prompt must say what to say then, and that it must not
  advance.
- A new `setTimer` cancels the running one: starting the answer timer early
  throws away the rest of the think time.
- `runCardCommand` shows the question. The agent never reads it aloud.

Copy-ready prompt sections (adapt the counts and times):

```
## TIMER WAIT (overrides the generic timer instructions)
The setTimer result means the clock STARTED. Time is up only when
"[conductor:timer] Timer complete" arrives with that timer's message.
- Think time: if anything else arrives first, say only one short line that
  they can keep thinking. Do not start the answer. Do not call setTimer.
- Answer time: if anything else arrives first, say only one short line that
  they can continue. Do not move to the next question.
- A new setTimer cancels the running one.

## FLOW — for each question N of <M>
a. Call runCardCommand with command publish, title "Question N of <M>",
   description the exact question. Never read the card aloud.
b. Say they have <30> seconds to think. Call setTimer with
   durationInSeconds <30> and message "think time". Say their time starts now.
c. Only after "[conductor:timer]" for think time: invite the answer and call
   setTimer with durationInSeconds <60> and message "answer time".
d. If they clearly finish early, thank them and go to the next question. If
   answer time ends while they are speaking, thank them and move on.
```

To score how they come across on camera:

- `session_analysis.multimodal_analysis: true` (plan-gated; the write can
  return `403`), `is_recording: true`, and `evaluation_target` naming the
  person on camera.
- A presence criterion (eye line, energy, setting) in `rubrik`, and the same
  criterion in `processed_rubrik` with `requires_video: true`; its prompt says
  how to score when no recording exists. `processed_rubrik` is replaced whole,
  so send every criterion ([../scenario/rubric.md](../scenario/rubric.md)).

## Document capture: webcam_snapshot + collect_data

For video KYC, ID checks, or any document read on camera. Web only.

- `captureWebcam` takes one frame. The image arrives in the agent's next
  message: it waits for it and never comments on a document it has not seen.
- It is a read, not a pose check. Name the fields to read per document; the
  step passes only when those fields are readable.
- Unreadable (cut off, glare, blur, too far, a field covered): one coaching
  line, then capture again. Cap the retries (four is a good default), then
  mark the document not captured and move on. Without a cap the agent keeps
  asking for another try.
- Never invent a number, name, or date. Never speak a full ID number; confirm
  the last four characters only.
- `collectData` runs once, after the last check, with every field in one
  payload. Write "not captured" for anything unread or unanswered. A pass or
  "verified" claim follows only from fields actually read.
- With extraction on, start each `extraction_vars` description with "Prefer
  the collectData note." so results come from what the agent recorded.

Copy-ready prompt sections:

```
## WEBCAM — DOCUMENT READ
When they say the <document> is in front of the camera, call captureWebcam
with command capture. The image arrives in your NEXT message. Wait for it.
Never describe a document you have not seen.
Read: <field 1>, <field 2>, <field 3>. Pass this step only when those
fields are readable.
If it is cut off, blurred, has glare, or a field is covered: one coaching
line (closer, farther, hold steady, tilt away from the light), then capture
again. After four failed captures, mark it not captured and move on.
Never invent a number, name or date. Never say a full ID number aloud;
confirm the last four characters only.
If the camera is off or the capture fails, ask them to turn it on, then
capture again.

## COLLECT DATA
After the last check, call collectData exactly once with every field below,
key: value, one per line. Write not captured for anything unread or
unanswered. Never call it after a single answer.
- <field_1>: …
```

## Shared notes and a design board: notepad + architecture_board

For system-design interviews, workshops, and any session where the agent keeps
notes while the user builds a diagram. Web only.

- `notepad` — shared notes the agent controls; the user can type in them too.
  `read` before writing so their edits survive; `overwrite` (markdown) sets the
  whole page.
- `architecture_board` — the user's canvas: they drag typed components and
  wire them. The agent watches `[board]` events. The board is the ground
  truth of their design: `read` it before any summary or critique, ask when
  speech and board disagree, and never `clear` it without their agreement.
- One surface is on screen at a time, and every notepad command pulls the
  screen off the board. A surface shown once stays in the panel's arrow
  navigation, so `show` the board once at the start, then bring the notepad
  forward.
- Steer with the board's own component names (Service, Worker, Queue, Blob
  Store, CDN, Load Balancer) so a hint maps to something they can drag.
- React to board events like an interviewer glancing at a whiteboard; do not
  narrate each one.

Copy-ready TOOLS section:

```
## TOOLS
Notepad: shared notes. You control it; they can type in it too. read it
before you write.
Board: their design canvas. They control it. Watch the [board] events.
read it before any summary or critique. If what they say and the board
disagree, ask about it. Never clear it without their agreement.
Only one is on screen at a time; every notepad command pulls the screen off
the board. At the start, show the board once so it stays in the panel's
arrows, then show the notepad.
```

## Other tool ids

`whiteboard`, `mermaid`, `pdf_upload`, `screen_snapshot`, `code`, `form`,
`memory_search`, `small_talk`, `weather`, `arcade_gmail`, and
`arcade_calendar` are valid ids with no usage rules here. Enable one only when
the user asks for it, keep `add_to_system_prompt: true` so the runtime
supplies its guidance, and test it in one session before relying on it.

Stored configs can also hold `custom_function` and `mcp` keys; leave them as
stored. Custom Functions and MCP servers attach through `custom_function_ids`
and `mcp_server_ids`
([../scenario/control.md](../scenario/control.md#linked-resources)).

## Key Files

- [../scenario/control.md](../scenario/control.md) — `tools_config` fields
  and Google Slides decks
- [../scenario/runtime.md](../scenario/runtime.md) — tool registration and
  phone sessions
- [../scenario/rubric.md](../scenario/rubric.md) — `processed_rubrik` and
  `requires_video`
- [../scenario/ai-instructions.md](../scenario/ai-instructions.md) — TOOLS
  and GUARDRAILS sections
