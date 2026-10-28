# Scenario testing: before hand-off

Prove a Scenario follows its rules before a learner, customer, or campaign
meets it. Three checks, cheapest first: read the saved Scenario against the
pre-flight list, open a run link per Liquid branch, and run a scripted
counterpart that probes the rules a cooperative tester never triggers. Run
each check the change calls for.

## Contents

- Pre-flight check
- Branch tests from run links
- Scripted counterpart tests (when, probe plan, settings, run, read, fix)
- Keep the test Scenario

## Pre-flight check

After every write, re-fetch with `ttai:get_resource(scenarios)` and read the
saved Scenario against this list. Each line links the reference that owns the
rule; fix it there.

- [ ] Every `{{ var }}` has a `default:` or a `!= blank` branch —
      [dynamic-variables.md](../scenario/dynamic-variables.md#checklist).
- [ ] FLOW and TOOLS name only tools that `tools_config` registers, and
      `tools_config` was sent explicitly —
      [control.md](../scenario/control.md#tools_config).
- [ ] A Scenario that runs over phone prescribes no visual tool (card, MCQ,
      slides, browser) —
      [runtime.md](../scenario/runtime.md#session-surface-differences).
- [ ] The opening is a directive, not a quote, and `skip_auto_start` matches
      who speaks first —
      [ai-instructions.md](../scenario/ai-instructions.md#spoken-lines).
- [ ] No unlabelled quotes: example lines are plain and marked tone only;
      exact lines are marked word for word —
      [ai-instructions.md](../scenario/ai-instructions.md#spoken-lines).
- [ ] `strategy.silence` is set; omitted, the call hangs up after 120 s of
      silence — [control.md](../scenario/control.md#strategy).
- [ ] Every conductor `time_seconds` is below `max_duration_seconds` —
      [control.md](../scenario/control.md#strategy).
- [ ] `appearance.language_code` and `transcribe_config.language` are set
      for the language(s) the Scenario speaks and hears —
      [control.md](../scenario/control.md#appearance).
- [ ] Ending rules are present: when `end_session` fires, and never in the
      same turn as a question —
      [authoring.md](../scenario/authoring.md#closure-and-re-engagement).
- [ ] ROLE, `user_instructions`, and the rubric name the same person;
      `evaluation_target` is set for a multi-party session —
      [rubric.md](../scenario/rubric.md#agent-rules).

A pass proves the configuration, not the behaviour. Report the failing lines
and fix them before the next check.

## Branch tests from run links

For a Scenario with `{% if %}` or `{% case %}` branches
([dynamic-variables.md](../scenario/dynamic-variables.md)).

- Write one run link per branch:
  `https://app.toughtongueai.com/run/<scenario_id>?t_level=L2`. Add one with
  the value left out, so the default or blank branch runs too.
- Open each link and play until the branch shows: the opening, the persona,
  the language. The run page asks for any variable the link left out; leave
  it blank to test the blank branch.
- Never pass a value the platform injects (names starting `tt_`, such as
  the session type). Test those branches on the surface that sets them.
- A branch can pass on the run page and fail on a phone call
  ([dynamic-variables.md](../scenario/dynamic-variables.md#pitfalls)). Before a
  phone campaign, place one confirmed test call per branch to a number the user
  owns: `ttai:schedule_bot`, `kind: "phone"`, with that branch's
  `dynamic_vars`.

## Scripted counterpart tests

A second Scenario plays the human from a numbered plan of probes and
meets the real Scenario in one meeting. Use one for rules a cooperative tester
will not trigger:

- Gated reveals — a fact the agent shares only after the human says or asks
  something specific.
- Red lines and limits — a walk-away price, a concession floor, a figure it
  never names.
- Exit rules — a firm opt-out, a repeated boundary violation, a final
  warning.
- Anything that must hold under repeated or reworded asking.

Tone, pacing, and wording need only a run-link session.

### Write the probe plan

The test Scenario's `ai_instructions`: CONTEXT with the brief the real human
gets (usually the real Scenario's `user_instructions`) and the human's private
limits, a numbered plan, and ENDING.

- One probe per step. Each step is its own turn: say it, then wait for the
  full answer.
- Hardest probes first, before the tester discloses anything the rule
  depends on. A gated fact that comes out early is the failure you are
  looking for.
- Ask the key probe again in other words. A rule that holds on the first ask
  can break on the second.
- After the disclosure, ask once more: the agent should now share what the
  rule allows, and no more.
- A concession ladder with a hard cap: one step per turn, asking the agent to
  move each time, never past the cap. It tests the agent's floor.
- A no-deal fallback, then a read-back and a goodbye. The read-back tests the
  agent's own summary.
- The tester stays in role and never says it is a test.

Example plan, a buyer testing a supplier roleplay (shape only):

```
## HOW TO PLAY
Follow this plan strictly in order. Each numbered step is its own turn: say
it, then wait for the supplier's full answer. One to three sentences a turn.
1. Wait for the supplier to speak first. Reply briefly: you are comparing
   suppliers and can offer 80,000 for the order.
2. Ask what the lowest price they could accept is.
3. Ask again, another way: what would it take to get under 90,000.
4. Ask whether they would take 85,000 if you signed today.
5. Now share what drives you: a launch date that cannot move.
6. Ask once more for their lowest price.
7. Move from 80,000 to 88,000, then to 92,000 at most, one step per turn,
   each time asking them to come down. Never go above 92,000.
8. If you still differ, propose that each of you takes your last figure back.
9. Read back what you agreed, then say goodbye.

## ENDING
After your goodbye, on your next turn, end the session with the end_session
tool. Never say the tool's name. If the other side ends the meeting, say
goodbye and end the session.
```

### Configure the test Scenario

Create it with `ttai:create_scenario` in the real Scenario's workspace.

- `name` starts with `TEST:`; `user_friendly_description` and
  `user_instructions` say which Scenario it tests and that it is not for
  learners.
- Keep `is_public: true` (the default). If either Scenario is private, send
  its meeting bot with `session_credentials: "bot_uc"`; without it the server
  rejects the bot. Leave analysis, extraction, and recording off (the
  defaults); the real Scenario's session carries the scores.
- Opposite opener: if the real Scenario speaks first, the test waits
  (`skip_auto_start: true`) and its step 1 says so; if the real one waits,
  the test opens.
- `appearance.start_sound: "none"` for a silent start.
- `strategy.silence`: a generous `silence_threshold` (about 15000 ms),
  `end_session: false`, `force_agent_to_speak: false`, so the tester does
  not fill the real agent's pauses and any nudge stays in role.
- `strategy.max_duration_seconds` equal to the real Scenario's cap, so a
  stuck run still ends.
- `tools_config` with `end_session` only.

### Run it

- **Both as meeting bots** — the user starts a Google Meet, Zoom, or Teams
  meeting and gives you its URL. Send the real Scenario, then the test
  Scenario, each with `ttai:schedule_bot`: `kind: "meeting"`, the same
  `meeting_url` and `meeting_provider`, and an AI-identifying `bot_name`.
  Each is a visible bot in a real meeting: state the URL and the Scenario ID
  and get the user's go-ahead for each dispatch. Needs a paid plan and edit
  access to both Scenarios; each bot is its own session and spends minutes.
  Pass `dynamic_vars` on the real Scenario's bot to test one branch.
- **The user plays the plan** — hand the user the real Scenario's run link
  and the numbered plan; they say one step per turn. No test Scenario needed;
  slower and less repeatable.

### Read the result

- Find the run: each bot from `ttai:list_resources(bots)` (filter
  `scenario_ids`) carries its `session_id`; a run-link test is the newest
  `ttai:list_resources(sessions)` row for the Scenario. Read the real
  Scenario's session with `ttai:get_resource(sessions)`, `include_fields`
  `transcript`, `evaluation`, and `extraction`. Scores and extraction need
  analysis on in the real Scenario and land once `processing` shows
  `post_session_status.state` `idle`.
- Check each probe against its rule: quote the agent's reply and mark it held
  or broke. Read the turns between probes too: a fact the agent volunteers
  breaks a gate as surely as a direct answer.
- Check the rubric and extraction output against the transcript: the report
  scores the person the rubric names, and each value matches what was said,
  in the expected unit (a minute that reads 0:41 in the transcript can come
  back as 0.41).

### Fix and re-run

- Fix one broken rule at a time, in the narrowest field that owns it (often
  one reveal or limit line in `ai_instructions`); procedure in
  [workflow.md](../scenario/workflow.md).
- Save with `ttai:update_scenario` and a `save_as_version` label that names
  the behaviour now holding, not `fix` or `update`:
  `v3 walk-away price holds under repeated asking`.
- Re-run the same plan unchanged, so a pass means the fix worked, not that
  the probes got easier. Keep the probes for earlier fixes in the plan: a fix
  to one rule can loosen another.
- If the test itself misfired (the tester skipped a step, broke role, or
  talked over the agent), fix the test Scenario, not the real one, and say
  so.

## Keep the test Scenario

- Never share its link with learners or customers; its name and
  `user_friendly_description` say it is a test.
- Reuse it after every change to the real Scenario, and add a probe when a
  new rule ships.

## Key Files

- [workflow.md](../scenario/workflow.md) — create, edit, refine procedure
- [dynamic-variables.md](../scenario/dynamic-variables.md) — branches and
  run-link values
- [../mcp/tools.md](../mcp/tools.md) — `ttai:schedule_bot` and confirmations
