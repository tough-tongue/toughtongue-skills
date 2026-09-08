# Scenario recipes

Situation playbooks. Each file fills **content** on top of the scenario
skeleton in [../entities/scenario/](../entities/scenario/index.md).

Load **one** recipe per job. Always also load control + ai-instructions.

## Contents

| Situation            | AI plays                           | File                                                                       |
| -------------------- | ---------------------------------- | -------------------------------------------------------------------------- |
| Cold call / SDR      | Outbound caller (user is the lead) | [cold-call.md](cold-call.md)                                               |
| Sales roleplay       | Prospect (user practices selling)  | [sales-roleplay.md](sales-roleplay.md)                                     |
| Coaching             | Trainer / mentor                   | [coaching.md](coaching.md)                                                 |
| Demo                 | Product demo agent                 | [demo.md](demo.md)                                                         |
| Interview            | Interviewer (user is candidate)    | [facilitated/interview.md](facilitated/interview.md)                       |
| Meeting facilitation | Facilitator / observer             | [facilitated/meeting-facilitation.md](facilitated/meeting-facilitation.md) |
| Cascade TTS addendum | Landmass `cascade` speech rules    | [cascade-tts.md](cascade-tts.md)                                           |
| Other                | Support, negotiation, …            | scenario-authoring + control + ai-instructions                             |

## Pipeline reminder

Start from the **situation → stamp** table in
[model-selection.md](../entities/scenario/model-selection.md):

| Situation | Stamp |
| --- | --- |
| Cold call / slides | Landmass `cascade` + Cartesia |
| Sales roleplay | Galaxy `medium-stable` |
| Coaching / browser demo | Ocean `medium-stable` |

Cascade `ai_instructions` also need [cascade-tts.md](cascade-tts.md).

Details: [../entities/scenario/model-selection.md](../entities/scenario/model-selection.md).

## Key Files

- [../scenario-authoring.md](../scenario-authoring.md) — general quality principles
- [../entities/scenario/model-selection.md](../entities/scenario/model-selection.md) — pipeline stamps
- [../entities/scenario/ai-instructions.md](../entities/scenario/ai-instructions.md) — prompt shape
