# Report Templates

Fill every bracket. Drop a section that has no data rather than padding it.
Scores are on the 0–10 scale of `final_score` and `report_card[].score`.

## Contents

- Team performance report
- Scenario health report
- Individual coaching report

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
