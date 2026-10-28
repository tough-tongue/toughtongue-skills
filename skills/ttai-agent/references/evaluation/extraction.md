# Outcome extraction

Agents that take conversations for a business (outbound callers, screeners,
intake) produce outcomes, not scores: the fields a CRM or the next human acts
on. Extraction reads the finished transcript and returns one value per
variable in `session_analysis.extraction_vars`.

## Settings

- Nobody is scored: leave `rubrik` empty and set `is_auto_analysis: false`.
  When a person is scored ([rubric.md](../scenario/rubric.md)), add extraction
  only for fields someone will filter or export.
- `enable_extraction: true`, and `extraction_vars[]` entries of `name` (the
  result key, snake_case), `type` (`text` default, `number`, `boolean`,
  `list`, `date`) and `description` (the extractor's whole instruction).
- One model call fills every variable. Keep the list to fields someone uses.

## Writing variables

- **Outcome:** one variable, "Return exactly one of: …", one line per value
  saying when it applies. Include values for no live person (`'Voicemail'`)
  and a catch-all (`'Ambiguous'`). Add "Judge the FINAL state of the call" and
  a DISAMBIGUATION line for each confusable pair. Use the outcomes the FLOW
  closes on.
- **Context:** open long descriptions with who spoke, why, and that the
  transcript is speech-to-text of phone audio, so infer intent.
- **Times:** one strict format with a named timezone; `text` (not `date`) when
  the field has special values such as `SHORTLY`. Resolve "today" and weekdays
  from the session date. The person's latest clear correction wins, and their
  words beat the agent's confirmation line.
- **Absent:** say "Null if not discussed" in every description. Counts are
  `number`; `0` only when the person confirmed none.
- **Handoff summary:** one `text` variable, 2 to 4 sentences for the human who
  follows up: next step in the customer's words, questions and answers,
  objections. One sentence if the call never connected.

```text
call_result (text)
Return exactly one of: 'Qualified', 'Call Back Later', 'Not Interested',
'Wrong Number', 'Voicemail', 'Ambiguous'. CONTEXT: a phone call between Maya
(Acme's AI caller) and a person who enquired about office space; the
transcript is speech-to-text, so infer intent. Judge the FINAL state.
- 'Qualified': agreed to a callback from the sales lead, at any stated time.
- 'Call Back Later': busy or asked to be called later; no callback agreed.
- 'Not Interested': clearly declined, or asked not to be called.
- 'Wrong Number': not the person who enquired.
- 'Voicemail': no live person: voicemail, IVR or a carrier message only.
- 'Ambiguous': anything else, such as a dropped call.
DISAMBIGUATION: a busy person who still gave a time is 'Qualified'.

callback_time (text)
Agreed callback in US Eastern Time, STRICT FORMAT 'YYYY-MM-DD HH:MM AM/PM'.
'SHORTLY' if Maya accepted an immediate call. Null if none was agreed.
```

## Results and updates

- Read `extraction_results` with `include_fields: ["extraction"]` on
  `ttai:get_resource(sessions)` or `ttai:list_resources(sessions)`. Any value
  can be `null`.
- To push outcomes into a CRM, attach an existing post-session Custom Function
  with `post_session_function_id` (find one with
  `ttai:list_resources(custom-functions)`); it receives `extracted_variables`.
  MCP does not create Custom Functions.
- `ttai:update_scenario` replaces `extraction_vars` whole: read the Scenario,
  edit the list, send every variable. Treat `name` as an interface a CRM
  mapping reads. Confirm first; changes apply to sessions processed afterwards.
