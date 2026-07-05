# Report Templates

Structures for the three standard reports, plus the webhook-driven coaching
recipe. Fill every bracket; drop sections that have no data rather than
padding them.

## Contents

- Team performance report
- Scenario health report
- Individual coaching report
- Recipe: webhook-driven post-call coaching

---

## Team Performance Report

```markdown
# [Scenario Name] — Team Performance
**Window**: [from] – [to] · **Sessions**: [N analyzed] ([M] excluded: not yet analyzed/incomplete)
**Team**: [org / group name], [K] participants

## Score Summary
- Average: [x.x]/10 · Median: [x.x] · Range: [lo]–[hi]
- Trend: [improving / flat / declining] ([evidence: week-over-week numbers])

## Skill Breakdown (report-card topics, weighted)
| Topic | Avg score | Weight | Reading |
|---|---|---|---|
| [Topic] | [x.x] | [%] | [one-line interpretation] |

## Top Improvement Areas
### 1. [Behavioral theme, named concretely]
- Seen in [n]/[N] sessions
- Evidence: "[short transcript quote]" — [user], [date] ([analytics_url])
- Action: [specific practice or scenario change]

### 2. ...

## Standouts
- [Who improved most / best session worth sharing, with analytics_url]

## Recommended Actions
1. [Action, owner, by-when]
```

---

## Scenario Health Report

Use when the question is "is the scenario itself working?" — feeds the
scenario-refiner skill.

```markdown
# Scenario Health: [Scenario Name] ([scenario_id])
**Window**: [from] – [to] · **Sessions**: [N]

## Vital Signs
- Completion rate: [x]% ([completed] of [started])
- Average duration: [m] min (expected: [m] min)
- Average score: [x.x]/10 · Analysis coverage: [x]% of sessions analyzed

## Symptoms
| Signal | Observation | Suspected cause |
|---|---|---|
| [e.g. 40% of sessions end < 2 min] | [data] | [e.g. agent ends call prematurely] |

## Transcript Evidence
- "[quote showing the failure]" ([analytics_url])

## Verdict
- [Scenario issue → hand to scenario-refiner with diagnosis]
- [User skill issue → coaching recommendation instead]
```

---

## Individual Coaching Report

```markdown
# Coaching Report: [Name]
**Window**: [from] – [to] · **Sessions**: [N] on [scenario(s)]

## Progress
- Score trend: [first] → [latest] ([direction])
- Strongest topic: [topic] ([x.x]) · Focus topic: [topic] ([x.x])

## What's Working
- [strength, with evidence quote]

## Focus Areas (max 3)
### 1. [Concrete behavior]
- What happens: [pattern from weaknesses/transcripts]
- Try this: [action item — drawn from improvement_results where available]
- Practice: [specific scenario link to re-run]

## Next Check-in
- [Date / session count target]
```

Keep it to one page. Three focus areas maximum — more is a list, not coaching.

---

## Recipe: Webhook-Driven Post-Call Coaching

Automate "every real call gets a coaching report" (e.g. from a call-recording
platform's webhook):

1. **Ingest** — when the external webhook fires with a transcript, call
   `create_session` with the transcript content against a dedicated coaching
   scenario (one whose rubric evaluates the rep's performance on real calls).
2. **Trigger analysis** — call `post_process_session` for the new session.
   It returns immediately; processing is async.
3. **Poll** — re-fetch the session (`get_sessions_batch`) every ~30s until
   `evaluation_results` is present (or a `failed` state appears — retry once
   via `post_process_session`).
4. **Format** — build the Individual Coaching Report from
   `evaluation_results` + `improvement_results`.
5. **Deliver** — send via the team's email/Slack tooling; include the
   `analytics_url` for the full breakdown.

Notes:

- The coaching scenario is created once (use the scenario-creator skill);
  every webhook call reuses its ID.
- Keep the `TTAI_PAT` in the automation's server-side environment — never in
  client code or the webhook payload.
