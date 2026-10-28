# Scenario testing

Prove a Scenario follows its rules before a learner, customer, or campaign
meets it.

- **Branches:** for `{% if %}` or `{% case %}` branches, open one run link per
  value (`https://app.toughtongueai.com/run/<scenario_id>?t_level=L2`) and one
  with the value left out. Never pass a `tt_` variable; test those on the
  surface that sets them.
- **Rules under pressure:** a gated reveal, a red line (a walk-away price, a
  concession floor), an exit rule, or anything that must hold under repeated
  asking. A cooperative tester never triggers these: use a scripted
  counterpart.

## Scripted counterpart

A second Scenario plays the human from a numbered probe plan and meets the
real Scenario in one meeting.

- One probe per turn. Hardest probes first, before the tester discloses
  anything the rule depends on; ask the key probe again in other words.
- A concession ladder with a hard cap, one step per turn, tests the agent's
  floor. End with a read-back and a goodbye.
- The tester stays in role and never says it is a test.

```
## HOW TO PLAY
Follow this plan strictly in order. Each numbered step is its own turn: say
it, then wait for the supplier's full answer. One to three sentences a turn.
1. Wait for the supplier to speak first. Say you can offer 80,000.
2. Ask what the lowest price they could accept is.
3. Ask again, another way: what would it take to get under 90,000.
4. Now share what drives you: a launch date that cannot move.
5. Move to 88,000, then 92,000 at most, one step per turn. Never go higher.
6. Read back what you agreed, then say goodbye.

## ENDING
After your goodbye, on your next turn, call end_session. Never say its name.
```

Test Scenario settings: `name` starting `TEST:`; public, or each bot sent with
`session_credentials: "bot_uc"`; analysis and recording off; the opposite
`skip_auto_start` to the real Scenario; `start_sound: "none"`;
`strategy.silence` with a generous threshold and
`force_agent_to_speak: false`; the real Scenario's `max_duration_seconds`;
`tools_config` with `end_session` only.

## Run, read, fix

- Run: the user opens a meeting and gives you its URL; send the real Scenario,
  then the test Scenario, with `ttai:schedule_bot` (`kind: "meeting"`), each
  dispatch confirmed with the user. Without bots, the user plays the plan from
  the real Scenario's run link.
- Read the real Scenario's session transcript and mark each probe held or
  broke, quoting the reply. A fact volunteered between probes breaks a gate
  too.
- Fix the one rule that broke in the narrowest field, save with a
  `save_as_version` label naming the behaviour that now holds, and re-run the
  same plan unchanged. Keep the test Scenario for the next change.
