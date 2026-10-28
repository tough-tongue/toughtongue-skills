# Dynamic variables and branching

`ai_instructions` is a Liquid template. Per-run values fill `{{ var }}`, and the
same engine runs `{% if %}`, `{% case %}`, `{% assign %}` and filters, so one
Scenario can carry its own variants instead of being cloned per persona,
language, or campaign. This file is the authoring guide; the shape of the prompt
itself is in [ai-instructions.md](ai-instructions.md).

## Contents

- Where values come from
- What runs where
- Syntax that works
- Patterns
- Worked example
- Pitfalls
- Checklist

## Where values come from

- **Run link or embed** — `?t_<name>=<value>` on
  `https://app.toughtongueai.com/run/<scenario_id>` or an embed URL. The `t_`
  prefix is stripped, so `?t_level=L2` fills `{{ level }}`.
- **Web run page** — the app lists every variable the template references
  (inside `if` and `case` blocks too; `assign` locals excluded) and asks for the
  ones the link did not supply. The user may start with blanks; a blank is
  empty text and counts as `blank`.
- **Reserved on web** — `first_name` and `full_name` auto-fill from the
  signed-in user when the template references them.
- **Calls and bots** — `dynamic_vars` on `ttai:schedule_bot` (`kind: "phone"`
  or `"meeting"`). The same values reach post-session hooks.
- All values arrive as **strings**.
- **Platform (reserved)** — filled by Tough Tongue AI, never by the user:
  `tt_session_type` (`web`, `meeting_bot`, `sip_inbound`, `sip_outbound`) and
  `tt_call_time["+05:30"]` (session start at a fixed UTC offset, no daylight
  saving; a phone call gets the clock only when the prompt references it).
  Never ask for, pass, or invent a `tt_*` value.

## What runs where

| Surface                                                                                      | Engine                                   | Notes                                             |
| -------------------------------------------------------------------------------------------- | ---------------------------------------- | ------------------------------------------------- |
| Web sessions (Galaxy, Ocean)                                                                 | LiquidJS in the browser                  | strict filters; undefined variables render empty  |
| Phone (SIP), LiveKit rooms, Landmass models                                                  | python-liquid on the server              | same policy                                       |
| `super` Scenario stage fields (`name`, `instructions`, `common_instructions`, `goal`, `role`) | same fill as `ai_instructions`           |                                                   |
| `user_instructions`, `rubrik`                                                                | not filled                               | write them plain                                  |
| Conductor messages                                                                           | bare `{{ transcript }}` and `{{ ai_instructions }}` only | no filters, no other variables    |
| `[[ key ]]`                                                                                  | auto-update placeholders ([control.md](control.md)) | Liquid ignores square brackets; filled after the Liquid pass |

## Syntax that works

Standard Liquid, supported by both engines:

- Output and defaults: `{{ lead_name }}`, `{{ lead_name | default: "there" }}`
  (`default:` also covers an empty string).
- Branches: `{% if x and x != blank %} … {% elsif %} … {% else %} … {% endif %}`,
  `{% unless %}`, `{% case x %}{% when "a" %} … {% else %} … {% endcase %}`.
- Locals: `{% assign ok = "a,b,c" | split: "," %}`,
  `{% capture greet %} … {% endcapture %}`.
- Tests and loops: `contains` on strings and arrays; `{% for p in parts %}`
  over an assigned array.
- Filters: `downcase`, `upcase`, `strip`, `replace`, `truncate`, `size`,
  `split`, `plus`, `minus`, `date: "%A"` or `"%H%M"` (parses a date string).
- Whitespace control `{%- … -%}`, `{% comment %}`, `{% raw %}`.

## Patterns

Each one replaces a copy of the Scenario.

1. **One Scenario, several personas.** Choose the level per run
   (`?t_level=L2`).

   ```
   {% assign level = level | default: "L1" %}
   {% case level %}
   {% when "L2" %}You researched the product, catch loose claims, and have a decision-maker off the call.
   {% else %}You are a trusting first-timer; money and family are the only brakes.
   {% endcase %}
   ```

2. **Skip or add a phase on a blank value.**

   ```
   {% if lead_name != blank %}Start with: ask whether you are speaking with {{ lead_name }}. Then STOP and wait.
   {% else %}No name on file: start with the reason-for-call line. Then STOP and wait.{% endif %}
   ```

3. **Allow-lists.** Decide in the template, not in the model's head.

   ```
   {% assign served = "bali,vietnam,thailand" | split: "," %}
   {% assign d = destination | downcase | strip %}
   {% unless served contains d %}We do not serve {{ destination }}: say so once, offer the served list, and continue only if they pick one.{% endunless %}
   ```

4. **Language per lead.** `{% case preferred_language %}` with a default and
   one block per language. Never announce the choice aloud.

5. **Clock-aware rules.** Reference `tt_call_time` at the callee's offset and
   state the rule in words.

   ```
   It is now {{ tt_call_time["+05:30"] }} for the customer.
   Experts call back only between eleven AM and seven thirty PM India time. Offer a slot inside those hours; if today's are over, offer tomorrow from eleven.
   ```

6. **Campaign text.**
   `{% if current_offer != blank %}Mention {{ current_offer }} once, in your own words, after the first question.{% endif %}`

7. **One Scenario for web and phone.** Branch screen-only steps with
   `{% if tt_session_type == "web" %}` instead of keeping a phone copy.

## Worked example

A travel pre-sales caller confirms a trip enquiry and books an expert callback.
One Scenario serves first calls and follow-ups, with or without a name on file:

```
## CONTEXT
- Customer: {{ lead_name | default: "not on file" }}
- Destination: {{ destination | default: "not on file" }}
- Travellers: {{ number_travellers | default: "not on file" }}
Never speak "not on file"; FLOW says what to do instead.

## FLOW
{% if previous_call_summary != blank %}
Previous call: {{ previous_call_summary }}. Open as a follow-up: say you called before, then confirm what is still missing.
{% else %}
{% if lead_name != blank %}Start with: ask whether you are speaking with {{ lead_name }}. Then STOP and wait.
{% else %}Start with: say who you are and that they enquired about a trip, and ask if that is right. Then STOP and wait.{% endif %}
{% endif %}
{% if destination == blank %}Next, ask which destination they are considering.{% endif %}
{% if number_travellers == blank %}Then ask how many adults and children are travelling.{% endif %}
Then ask what time an expert should call back. One question per turn.
```

## Pitfalls

- **Strings everywhere.** `{% if n > 2 %}` with `n` set to `"8"` raises a type
  error on the server engine; the browser engine coerces and passes. A branch
  can work on the run page and break on a phone call. Convert first:
  `{% assign ni = n | plus: 0 %}`.
- **Any error falls back to the raw template.** An unknown filter, a syntax
  error, or a type error makes the server use the original text, tags and
  placeholders included, with only a log warning. Test every branch from the
  run link before a phone campaign.
- **`for` needs an assigned array.** `{% for p in csv | split: "," %}` is a
  syntax error; `assign` first, then loop.
- **Values are HTML-escaped today.** `&` arrives as `&amp;` and an apostrophe
  as `&#39;`. Until the platform changes this, keep `&` and quotes out of
  values (write "and").
- **Referenced means asked.** The web run page prompts for every variable the
  template mentions, including inside branches. Keep optional variables
  optional with `!= blank` guards so a blank is a valid answer.
- **Never speak a placeholder.** Guard every `{{ var }}` with `default:` or a
  branch; never let "not on file", "null", or a raw `{{ }}` reach the voice.
- **Not everywhere.** `user_instructions` and the rubric are plain text;
  conductor messages take bare `{{ transcript }}` only.

## Checklist

- [ ] Every `{{ var }}` has a `default:` or a `!= blank` branch
- [ ] Numeric comparisons go through `plus: 0`
- [ ] No `tt_*` value requested from the user, passed in `dynamic_vars`, or
      invented; `tt_call_time` uses the offset of the people on the call
- [ ] Each `case` or `if` branch tested from a run link with and without the
      value
- [ ] No `&` or quotes in values the caller passes
- [ ] Variants that differ only by persona, language, or offer live in one
      Scenario

## Key Files

- [ai-instructions.md](ai-instructions.md) — prompt shape and spoken-line rules
- [control.md](control.md) — `auto_update_config` placeholders, strategy
- [../recipes/cold-call.md](../recipes/cold-call.md) — outbound caller recipe
