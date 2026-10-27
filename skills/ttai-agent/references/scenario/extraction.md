# Outcome extraction

Voice agents that take conversations for a business (outbound callers,
screeners, intake, KYC) produce outcomes, not scores: the fields a CRM or the
next human acts on. Extraction reads the finished transcript and returns one
value per variable in `session_analysis.extraction_vars`.

## Contents

- Extraction, a rubric, or both
- Settings
- Writing variables
- Worked example
- In-session capture with `collect_data`
- Reading and using results
- Updating variables
- Checklist

## Extraction, a rubric, or both

- **Nobody is scored** — the agent takes the conversation for the business.
  Leave `rubrik` empty, set `is_auto_analysis: false`, and turn extraction on.
  Sessions then carry `extraction_results` and no report card.
- **A person is scored** — practice, coaching, interviews. The rubric scores
  them ([rubric.md](rubric.md)). Add extraction only for fields someone will
  filter or export.
- **Both** — a practice Scenario can run both: the rubric scores the learner;
  extraction records what the session produced (the agreed price, who does
  what, `full` / `provisional` / `impasse`) and teaching signals (who named
  the first figure, whether a hidden fact surfaced).

## Settings

```json
{
  "session_analysis": {
    "is_auto_analysis": false,
    "enable_extraction": true,
    "extraction_vars": [
      {
        "name": "call_result",
        "type": "text",
        "description": "Return exactly one of: …"
      }
    ]
  }
}
```

- `enable_extraction: true` — the gate. Extraction runs after the session
  ends, with or without analysis.
- `extraction_vars[]` — `name` (the key in the results; snake_case),
  `description` (the extractor's instructions for that variable), `type`
  (`text` by default, `number`, `boolean`, `list`, `date`).
- One model call reads the whole transcript and fills every variable. Keep the
  list to fields someone uses.

## Writing variables

The description is the whole instruction for that variable. Write it like a
prompt.

**Outcome — one variable, exactly one value**

- "Return exactly one of: …", then one line per value saying when it applies.
- Values for calls that never became a conversation: no live person
  (`'Voicemail'`: voicemail, IVR, a carrier message) and the wrong person
  (`'Wrong Number'`).
- A catch-all (`'Ambiguous'`): a dropped call, unclear replies, details
  confirmed but no outcome reached.
- "Judge the FINAL state of the call, not intermediate moments."
- A DISAMBIGUATION line per confusable pair: "A busy customer who still gave a
  time is 'Qualified', not 'Call Back Later'."
- Use the outcomes the FLOW closes on, so the agent and the extractor share one
  definition.

**Context** — open every long description with a CONTEXT paragraph: who the
agent and the person are, why they spoke, and that the transcript is
speech-to-text of phone audio (name the languages when calls are code-mixed),
so words may be garbled: infer intent from the whole conversation.

**Formats**

- Times: one strict format with a named timezone, for example
  `YYYY-MM-DD HH:MM AM/PM` in IST, "no other text and no timezone suffix".
- Special values for answers that are not a time: `SHORTLY` (call right away),
  `ANYTIME`, the date alone when only a date was agreed. Use `text` with a
  strict format, not `date`, when a field has special values.
- Absent → `null`. Say so in every description ("Null if not discussed").
- Counts and amounts: `type: number`, "as a plain number". Return `0` only when
  the person confirmed none.
- Other fixed choices: list the values, optionally "followed by the reason in
  a few words" (`'Cancelled: price too high'`).
- Not asked differs from asked and unknown: "'Does not remember' if asked and
  they could not recall. Null if not asked."
- Booleans by rule: "True only if <every condition>. Otherwise false."
- Lists: `type: list`, "one per item", and "only what the participant said or
  confirmed".

**Latest value wins**

- The person's latest clear correction wins.
- The person's words beat the agent's confirmation line: when they countered
  ("no, make it eight") and the agent confirmed its own proposal anyway, return
  their time, taking the day and AM/PM from the proposal it answered.
- Resolve "today", "tomorrow" and weekday names from the session date; a
  weekday means its next occurrence.
- Map vague parts of the day to fixed times inside the team's hours
  ("evening" = 06:00 PM).

**Handoff summary** — one `text` variable for the human who follows up: 2 to 4
sentences in plain English, written for that person. Cover the agreed next
step in the customer's own words, what they asked and what the agent answered,
objections, and anything to raise on the next call. If the call never
connected or had no usable content, one sentence saying so. Quote words from
another script in Latin transliteration.

**Language** — on multilingual calls, a variable with fixed values
(`'English'`, `'Hinglish'`, `'Hindi'`) and a marker for an explicit request
("append ' (asked for Hindi)'"), so the follow-up goes to someone who speaks
it.

## Worked example

An outbound caller for Acme Workspaces calls people who enquired about office
space and books a callback with the sales lead. Four variables, each one
`extraction_vars` entry (`name (type)`, then its `description`):

```text
call_result (text)
Call outcome for the CRM. Return exactly one of: 'Qualified', 'Call Back
Later', 'Not Interested', 'Wrong Number', 'Voicemail', 'Ambiguous'.
CONTEXT: a phone call between Maya (Acme's AI caller) and a person who
enquired about office space. The transcript is speech-to-text of phone audio,
so words may be garbled; infer intent. Judge the FINAL state of the call.
- 'Qualified': agreed to a callback from the sales lead, at any stated time.
- 'Call Back Later': busy or asked to be called later; no callback agreed.
- 'Not Interested': clearly declined, or asked not to be called.
- 'Wrong Number': not the person who enquired, or never enquired.
- 'Voicemail': no live person: voicemail, IVR or a carrier message only.
- 'Ambiguous': anything else, such as a dropped call or unclear replies.
DISAMBIGUATION: a callback agreed at the end beats earlier hesitation. A busy
person who still gave a time is 'Qualified'.

callback_time (text)
The callback time agreed with the sales lead, in US Eastern Time. STRICT
FORMAT: YYYY-MM-DD HH:MM AM/PM, e.g. '2026-11-03 02:00 PM'; no other text.
'SHORTLY' if Maya accepted an immediate call; 'YYYY-MM-DD' alone if only a
date was agreed. Null if no callback was agreed.
[CONTEXT paragraph, as in call_result]
Use the LATEST time the person agreed to; their words beat Maya's
confirmation line. Resolve today, tomorrow and weekdays from the session date.
Morning = 10:00 AM, afternoon = 02:00 PM, evening = 05:00 PM.

team_size (number)
Desks needed, as a plain number, from the latest figure the person stated or
confirmed. Null if not discussed.

summary_for_sales (text)
A 2 to 4 sentence brief in plain English for the sales lead who calls back:
team size and move-in timing, the agreed callback time in the person's own
words, questions they asked and what Maya answered, and any hesitation. If the
call did not connect or had no usable content, say so in one sentence.
```

## In-session capture with `collect_data`

When the agent reads values the transcript will not hold, such as a document
held up to the webcam, have it record them during the session. Register
`collect_data` (`should_register: true`); the agent calls it as `collectData`.

- In `ai_instructions`: after the last check, call `collectData` once,
  silently, with every field in one payload (`key: value`, one per line).
  Never after a single answer. Write `not captured` for anything unread or
  unanswered; never invent a value; never speak the tool name.
- Use the same field names in the `collectData` instruction and in
  `extraction_vars`.
- Start each description with "Prefer the collectData note." and end it with
  "Null if never readable or marked not captured."
- A timed conductor message before the close can force the `collectData` call
  when sessions run long.

## Reading and using results

- Read `extraction_results` with `include_fields: ["extraction"]`: one session
  with `ttai:get_resource(sessions)`, many with `ttai:list_resources(sessions)`
  and `filters: {scenario_ids: […]}`. Each variable `name` is a key.
- Any value can be `null`. A call with no usable transcript can return `null`
  for every variable, the catch-all included.
- `post_session_function_id` attaches an existing post-session Custom Function
  (find it with `ttai:list_resources(custom-functions)`). It receives the
  session's `extracted_variables` with its id and analysis results: the way to
  push outcomes into a CRM. MCP attaches Custom Functions; it does not create
  them.
- Calls held elsewhere (a dialer, Gong, Zoom): `ttai:upload_session` runs the
  same extraction on an uploaded transcript or recording.
- Outcome reports across many sessions are the `ttai-session-analyst` skill's
  job.

## Updating variables

- `ttai:update_scenario` merges `session_analysis` one level: each key you send
  replaces that key. `extraction_vars` is one key, so the list is replaced
  whole. Read the Scenario (`ttai:get_resource(scenarios, id)`), edit the
  list, and send every variable; sending one deletes the rest.
- Keys you leave out keep their values. Send `is_auto_analysis` explicitly
  when the Scenario stops or starts scoring someone.
- Treat `name` as an interface: a hook or CRM mapping reads it. Rename a
  variable only with whoever owns that mapping.
- Confirm before `ttai:update_scenario`. Changes apply to sessions processed
  afterwards; check a changed variable on a fresh session.

## Checklist

- [ ] Nobody scored: `rubrik` empty, `is_auto_analysis: false`. A scored
      person: the rubric, plus only the fields someone uses
- [ ] `enable_extraction: true`
- [ ] One outcome variable: exactly-one-of values, a definition each,
      no-live-person and wrong-person values, a catch-all, the final state,
      disambiguation; values match the FLOW's outcomes
- [ ] A CONTEXT paragraph in every long description
- [ ] Times: one strict format, a named timezone, special values, a `null` rule
- [ ] Counts as `number`; `0` only when confirmed none
- [ ] Latest correction wins; the person's words beat the agent's confirmation;
      relative days from the session date
- [ ] A handoff summary for the human who follows up
- [ ] A language variable on multilingual calls
- [ ] Values seen mid-session: one `collectData` call; descriptions prefer its
      note
- [ ] The update sent the full `extraction_vars` list; a re-fetch shows every
      variable
- [ ] A fresh session's `extraction_results` read as intended

## Key Files

- [rubric.md](rubric.md) — when a person is scored
- [control.md](control.md) — other `session_analysis` settings
- [ai-instructions.md](ai-instructions.md) — FLOW outcomes the outcome
  variable mirrors
