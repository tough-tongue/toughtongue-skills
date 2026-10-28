# Report Templates

Fill every bracket. Drop a section that has no data rather than padding it.
Scores are on the 0–10 scale of `final_score` and `report_card[].score`.

## Contents

- Team performance report
- Scenario health report
- Individual coaching report
- Pattern map (from call transcripts)
- Call outcomes (outcome Scenarios, no scores)

## Team Performance Report

```markdown
# [Scenario Name]: Team Performance

**Window**: [from] – [to] · **Sessions**: [N analyzed] ([M] excluded: no
evaluation yet) **Team**: [organization or group], [K] participants

## Score Summary

- Average: [x.x]/10 · Median: [x.x] · Range: [lo]–[hi]
- Trend: [improving / flat / declining] ([weekly averages])

## Skill Breakdown (report-card topics)

| Topic   | Avg score | Weight | Reading                   |
| ------- | --------- | ------ | ------------------------- |
| [topic] | [x.x]/10  | [w]%   | [one-line interpretation] |

## Top Improvement Areas

### 1. [Behavior, named concretely]

- Seen in [n]/[N] sessions
- Evidence: "[short transcript quote]" ([participant], [date], [analytics_url])
- Action: [specific practice drill or coaching move]

## Standouts

- [Most improved participant, or best session worth sharing, with analytics_url]

## Recommended Actions

1. [Action, owner, by when]
```

## Scenario Health Report

Use when the question is whether the Scenario itself works. If the verdict is a
Scenario fault, hand the diagnosis and quotes to the `ttai-agent` skill.

```markdown
# Scenario Health: [Scenario Name] ([scenario_id])

**Window**: [from] – [to] · **Sessions**: [N]

## Vital Signs

- Completed: [x]% ([completed] of [total]); terminated: [y]%
- Average duration: [m] min (from legacy `duration`, in seconds)
- Average score: [x.x]/10 · Analysis coverage: [x]% have an evaluation
- Learning flags: [n] sessions with `learning_results.status = issue`

## Symptoms

- **[Signal, e.g. 40% of sessions under 2 min]** — [data]. Suspected cause:
  [e.g. the AI ends the call early].
- **[Signal, e.g. one topic scores < 3 for everyone]** — [data]. Suspected
  cause: [e.g. the rubric asks for something the roleplay never allows].

## Transcript Evidence

- "[quote showing the failure]" ([analytics_url])
- Learning note: [learning_results.issue / suggestion, if present]

## Verdict

- [Scenario fault → hand to ttai-agent with this diagnosis]
- [User skill gap → coaching recommendation instead]
```

Signals that point at the Scenario rather than the people: most sessions end
early or as `terminated`; one topic scores low for everyone, including strong
performers; transcripts show the AI breaking character, repeating itself, or
ignoring answers; `learning_results` flags the same issue repeatedly.

## Individual Coaching Report

```markdown
# Coaching Report: [Name]

**Window**: [from] – [to] · **Sessions**: [N] on [Scenario(s)]

## Progress

- Score trend: [first] → [latest] ([direction])
- Strongest topic: [topic] ([x.x]/10) · Focus topic: [topic] ([x.x]/10)

## What's Working

- [strength, with an evidence quote]

## Focus Areas (max 3)

### 1. [Concrete behavior]

- What happens: [pattern from weaknesses and transcripts]
- Try this: [from improvement_results.action_items where available]
- Practice: re-run [Scenario name] focusing on [topic]

## Next Check-in

- [Date or session-count target]
```

Keep it to one page. Three focus areas at most; more is a list, not coaching.

## Pattern Map

Use when the user wants a practice Scenario built from real calls ("what do
prospects actually say?"). Pull transcripts for the population, fill this map,
and hand it to the `ttai-agent` skill, which turns it into objections, persona
pools, and rubric gates. A pattern makes the cut when it appears in at least
three sessions; quote the best line heard, marked tone only, never as a script.

```markdown
# [Scenario or call type]: Pattern Map

**Corpus**: [N] sessions, [from] – [to], [who the humans were]

## Stages

| # | Stage | What good looks like (session) | Common miss (session) |
| - | ----- | ------------------------------ | --------------------- |

## Counterpart moves (3+ sessions each)

1. **"[what they say]"** — [n] sessions ([ids]). Beat it by: [the move]. Best
   line heard (tone only): "[quote]" ([session])

## Red flags (cap the score when present)

| Pattern | Sessions | Example line |
| ------- | -------- | ------------ |

## Persona pools

- **Level 1 ([n] sessions)**: [who they are, what they ask, what stops them].
  Rotate: [2–3 profiles]. Moves pool: [numbers from the list above]
- **Level 2 ([n] sessions)**: [the researched, guarded version]

## Rubric seeds

- Score the human per stage; weight the stages where sessions were won or lost
- Red flags cap the score even when the outcome was good
- Facts the human states must match [source]; an unsourced one costs points
```

## Call Outcomes

Reach from the bots, outcomes from the connected calls' sessions. State each
denominator.

```markdown
# [Scenario Name]: Call Outcomes

**Window**: [from] – [to] [timezone] · **Outcome variable**: `[name]`

## Reach

- Dialled [N] · connected [c] ([x]%) · [success value] [s] ([x]% of connected)
- Not connected: [failure_reason] [n], [failure_reason] [n]

## Outcomes (connected calls)

| Outcome  | Calls | % of connected |
| -------- | ----- | -------------- |
| [value]  | [n]   | [x]%           |
| null     | [n]   | [x]%           |
| off-list | [n]   | [x]%           |

## Problem Outcomes

- [Outcome value], [n] calls: [theme] — "[short quote]" ([analytics_url])

## Actions

1. [Prompt or extraction fix, with the evidence] → `ttai-agent` skill
2. [Follow-ups due, e.g. callbacks booked]
```

Keep phone numbers and lead names out of a shared report.
