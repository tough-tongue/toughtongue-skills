# Demo Scenario Patterns

Rules for scenarios where the **AI is a product demo agent** — walking
prospects through a product via live browser navigation or slides.

## Contents

- Two demo formats (browser vs slides)
- ai_instructions structure and demo flow
- Qualifying questions pattern
- Tools config (browser, slides, knowledge base)
- Rubrik: Demo Intelligence Report
- Technical config and quality checklist

Key facts:

- Two formats: **browser-based** (live navigation) and **slide-based** (curated narrative)
- AI is proactive — drives the conversation, does not wait for the user
- Qualifying questions are woven between demo sections, not asked upfront
- Rubrik produces a buyer intelligence report, not a performance review
- Browser demos: Ocean `medium-stable` (fall back to Galaxy if Ocean is
  unavailable).
- **Slide demos: Landmass `cascade` + Cartesia** by default for polished TTS
  narration — load [cascade-tts.md](cascade-tts.md). Use Galaxy/Ocean realtime
  only when native voice is enough and narration polish is secondary.
  See [../entities/scenario/model-selection.md](../entities/scenario/model-selection.md).

---

## Choosing the Format

| Use case                               | Format      | Primary tool                                          |
| -------------------------------------- | ----------- | ----------------------------------------------------- |
| Web app / SaaS product with demo URL   | **Browser** | `browser` (goto, observe, capture, act)               |
| Curated narrative, no demo environment | **Slides**  | `google_slides` (embedUrl)                            |
| Complex product requiring login/setup  | **Slides**  | Controlled story arc                                  |
| Hybrid                                 | **Both**    | Slides for narrative, browser for a live "wow" moment |

---

## Demo Flow Strategy

Every demo follows a **beginning → middle → end** arc:

1. **Opening** — Introduce the agent, ask about the prospect's background
2. **Core sections (3-5)** — Show features mapped to their stated problem
3. **Wrap-up** — Summarize, ask what resonated, suggest next steps

### Storyline patterns

- **Problem → Solution → Proof**: Start with prospect's pain, show how the
  product solves it, end with case study / results
- **Day in the Life**: Walk through a typical workflow using the product
- **Build It Together**: Create something live with the prospect directing
- **Three Pillars**: Frame around 3 core capabilities, qualify between each

---

## Browser Demo

### Tool workflow

The browser is pre-opened at the product URL. Use `runBrowserCommand`:

- `command: "goto"` + `url` — navigate to a page
- `command: "observe"` + `instruction` — scan page elements
- `command: "act"` + `instruction` — interact with elements
- `command: "capture"` — take a screenshot to see the page
- `command: "step"` + `name` — replay a pre-recorded batch of actions
  instantly (zero AI processing), if the scenario has recorded steps

Recommended flow per section: `goto → observe → capture → explain → act`

**Pre-recorded steps (deterministic demos):** for a scripted walkthrough
that always clicks the same things, record the flow into
`tool_settings.steps` and make `step` the primary command — hand off to the
**browser-demo-builder** skill for the recording workflow, selector rules,
and instruction templates. That skill also covers
`ttai:authenticate_browser`, which produces the persistent authenticated
`[BROWSER_CONTEXT_ID]` used below.

### ai_instructions structure

```
## ROLE
You are [Agent], an AI demo agent for [Company]. Deliver a guided,
high-quality live demo using the browser tool.

## COMMUNICATION STYLE
- Warm, confident, engaging. Proactive — drive the conversation.
- Crisp — this is a live demo, not a lecture.

## FLOW
Start with: Hi {{ first_name }}, I'm [Agent]. I'd love to walk you through
our platform. Ask them to tell you a bit about themselves. Then STOP and
wait. If first_name is blank, skip the name.

## QUALIFYING QUESTIONS (weave in naturally)
- [Question 1] / [Question 2] / [Question 3]

## BROWSER TOOL INSTRUCTIONS
[Pre-opened URL, key URLs map, goto/observe/capture/act workflow]

## DEMO FLOW
### Phase 1: Introduction (Home Page)
### Phase 2: [Core Feature 1]
### Phase 3: [Core Feature 2]
### Phase 4: [Core Feature 3]
### Phase 5: Wrap-Up — summarize, ask what resonated, suggest next steps

## KNOWLEDGE BASE
Only if docs are already attached in Studio: use knowledge_base_search for
deeper questions. Otherwise answer from CONTEXT and do not mention a KB.
```

### Technical config (browser demo)

```json
{
  "ai_model_config": {
    "provider": "Ocean",
    "model": "medium-stable"
  },
  "strategy": {
    "skip_auto_start": false,
    "system_instructions_template": "minimal",
    "silence": { "silence_threshold": 6000, "end_session": false, "force_agent_to_speak": true },
    "conductor": {
      "enabled": true,
      "messages": [
        {
          "time_seconds": 600,
          "message": "Wrap up the demo gracefully. Thank them and end the call.",
          "end_turn": true
        }
      ]
    }
  },
  "tools_config": {
    "tools": {
      "browser": {
        "should_register": true,
        "add_to_system_prompt": true,
        "tool_settings": { "contextId": "[BROWSER_CONTEXT_ID]", "initialUrl": "[PRODUCT_URL]" }
      },
      "google_slides": { "should_register": false, "add_to_system_prompt": false },
      "knowledge_base_search": { "should_register": false, "add_to_system_prompt": false },
      "end_session": {
        "should_register": true,
        "add_to_system_prompt": true,
        "tool_settings": { "disconnectDelaySeconds": 5 }
      },
      "emoji_reaction": { "should_register": true, "add_to_system_prompt": false }
    }
  },
  "session_analysis": {
    "is_auto_analysis": true,
    "is_auto_submit": true,
    "multimodal_analysis": true
  },
  "is_recording": true
}
```

---

## Slide Demo

### ai_instructions structure

```
## ROLE
You are [Agent], an AI demo agent for [Company] conducting a product
demo using slides.

## COMMUNICATION STYLE
- Show energy and enthusiasm — engaging, not robotic.
- Don't read slides verbatim — use them as visual anchors.
- Be proactive — don't wait for the user to drive.

## FLOW
Start with: Hi {{ first_name }}, I'm [Agent]. I'd love to walk you through
what we do today. Ask them to tell you a little about themselves. Then STOP
and wait. If first_name is blank, skip the name.

## QUALIFYING QUESTIONS (weave in naturally)
- [Question 1] / [Question 2] / [Question 3]

## SLIDE SUMMARY
- Slide 1 (Title): [Company overview, core value prop]
- Slide 2: [Key feature / problem]
- Slide 3: [Differentiator]
- Slide 4: [Results, case studies, proof]
- Slide 5: [Pricing, integrations]
- Slide 6: [Thank you, next steps]

## DEMO FLOW STRATEGY
1. Listen to background → 2. Transition to slides →
3. Weave in their context → 4. Qualify between slides → 5. Close
```

### Technical config (slide demo: Cascade option)

```json
{
  "ai_model_config": {
    "provider": "Landmass",
    "model": "cascade",
    "tts_provider": "cartesia",
    "tts_voice_id": "[SELECTED_VOICE_ID]",
    "llm_provider": "google_vertex",
    "llm_model": "gemini-3.1-flash-lite",
    "stt_provider": "deepgram"
  },
  "strategy": {
    "skip_auto_start": false,
    "system_instructions_template": "minimal",
    "silence": { "silence_threshold": 6000, "end_session": false, "force_agent_to_speak": true }
  },
  "tools_config": {
    "tools": {
      "google_slides": {
        "should_register": true,
        "add_to_system_prompt": false,
        "tool_settings": { "embedUrl": "[GOOGLE_SLIDES_EMBED_URL]" }
      },
      "browser": { "should_register": false, "add_to_system_prompt": false },
      "knowledge_base_search": { "should_register": false, "add_to_system_prompt": false },
      "end_session": {
        "should_register": true,
        "add_to_system_prompt": true,
        "tool_settings": { "disconnectDelaySeconds": 5 }
      },
      "emoji_reaction": { "should_register": true, "add_to_system_prompt": false }
    }
  },
  "session_analysis": { "is_auto_analysis": true, "is_auto_submit": true }
}
```

Use Landmass Cascade when the demo needs a selected external voice or full
STT/LLM/TTS control. Load [cascade-tts.md](cascade-tts.md) only for that
pipeline. Realtime Galaxy or Ocean is appropriate when native voice and a
faster interactive exchange matter more.

---

## Qualifying Questions

Embed 3-5 qualifying questions that the agent weaves in naturally between
demo sections — not as an interrogation. Frame as conversation:

- "How are you currently handling [problem]?" (situation)
- "What's the biggest challenge with that approach?" (problem)
- "How large is the team involved?" (sizing)
- "What's your timeline for making a change?" (urgency)

---

## Rubrik: Demo Intelligence Report

Demo rubrics are NOT performance reviews — they produce buyer intelligence
for the sales team.

```
# Demo Intelligence Report

## 1. Buyer Interest Signals
- Which features/sections generated engagement?
- Follow-up questions or verbal interest indicators

## 2. Objections and Concerns
- What did the prospect push back on?
- Topics they seemed disengaged from

## 3. Use Case Identification
- What problem is the prospect trying to solve?
- Industry/role context and team size

## 4. Conversion Likelihood
- Score: [Low / Medium / High]

## 5. Follow-Up Recommendations
- Next steps for human rep
- Materials to send and optimal timing
```

---

## user_instructions Structure

```
Welcome! I'm [Agent], and I'll be demoing [Company] for you today.

I'll walk you through:
- [Key area 1]
- [Key area 2]
- [Key area 3]

Feel free to ask questions at any point!
```

---

## Anti-Patterns

| Anti-pattern                                    | Instead                                                                            |
| ----------------------------------------------- | ---------------------------------------------------------------------------------- |
| Feature tour (showing everything)               | Focus on 3-5 sections mapped to prospect's problem                                 |
| Reading slides verbatim                         | Use slides as visual anchors, add context beyond the screen                        |
| Waiting for the user to drive                   | Be proactive — transition between sections                                         |
| Interrogating with qualifying questions upfront | Weave questions naturally between demo sections                                    |
| Enabling `knowledge_base_search` with no docs   | Attach FAQs/pricing in Scenario Studio first — MCP cannot set `knowledge_base_ids` |
| Browser demo without `multimodal_analysis`      | Enable it — visual navigation is part of the analysis                              |

---

## Quality Checklist

- [ ] Format chosen: browser or slide (or hybrid)
- [ ] 3-5 core sections defined with clear flow
- [ ] Qualifying questions woven between sections
- [ ] Knowledge base: enable the tool only if docs are already attached in Studio
- [ ] `ai_model_config` matches the selected browser or slide delivery path
- [ ] Cascade voice-pipeline blocks only if the model is Landmass Cascade
- [ ] `conductor` timeout at ~600s (10 min)
- [ ] `end_session` enabled
- [ ] Rubrik produces a Demo Intelligence Report (not a performance review)
- [ ] Browser demo: `contextId` and `initialUrl` set; `multimodal_analysis: true`
- [ ] Slide demo: `embedUrl` set in `google_slides` tool settings

## Key Files

- [../scenario-authoring.md](../scenario-authoring.md) — durable authoring principles
- [../entities/scenario/model-selection.md](../entities/scenario/model-selection.md) — pipeline stamps
- [../entities/scenario/control.md](../entities/scenario/control.md) — strategy / tools
- [../entities/scenario/ai-instructions.md](../entities/scenario/ai-instructions.md) — prompt shape
- [../../../browser-demo-builder/SKILL.md](../../../browser-demo-builder/SKILL.md) — deterministic browser steps
