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
- Browser demos use Ocean/medium-stable; slide demos use Landmass/cascade-01

---

## Choosing the Format

| Use case | Format | Primary tool |
|----------|--------|-------------|
| Web app / SaaS product with demo URL | **Browser** | `browser` (goto, observe, capture, act) |
| Curated narrative, no demo environment | **Slides** | `google_slides` (embedUrl) |
| Complex product requiring login/setup | **Slides** | Controlled story arc |
| Hybrid | **Both** | Slides for narrative, browser for a live "wow" moment |

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

Recommended flow per section: `goto → observe → capture → explain → act`

### ai_instructions structure

```
## YOUR ROLE
You are [Agent], an AI demo agent for [Company]. Deliver a guided,
high-quality live demo using the browser tool.

## COMMUNICATION STYLE
- Warm, confident, engaging. Proactive — drive the conversation.
- Crisp — this is a live demo, not a lecture.

## OPENING
"Hi {{ first_name }}, I'm [Agent]. I'd love to walk you through our
platform. Before I dive in, could you tell me a bit about yourself?"

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
For deeper questions, use the knowledge_base_search tool.
```

### Technical config (browser demo)

```json
{
  "ai_model_config": {
    "provider": "Ocean",
    "model": "medium-stable"
  },
  "strategy": {
    "conductor": {
      "enabled": true,
      "prefix": "CONDUCTOR:",
      "messages": [
        { "time_seconds": 600, "message": "Wrap up the demo gracefully. Thank them and end the call.", "end_turn": true }
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
      "knowledge_base_search": { "should_register": true, "add_to_system_prompt": true },
      "end_session": { "should_register": true, "add_to_system_prompt": true, "tool_settings": { "disconnectDelaySeconds": 5 } },
      "emoji_reaction": { "should_register": true, "add_to_system_prompt": false }
    }
  },
  "session_analysis": { "is_auto_analysis": true, "is_auto_submit": true, "multimodal_analysis": true },
  "is_recording": true
}
```

---

## Slide Demo

### ai_instructions structure

```
## YOUR ROLE
You are [Agent], an AI demo agent for [Company] conducting a product
demo using slides.

## COMMUNICATION STYLE
- Show energy and enthusiasm — engaging, not robotic.
- Don't read slides verbatim — use them as visual anchors.
- Be proactive — don't wait for the user to drive.

## OPENING
"Hi {{ first_name }}, I'm [Agent]. I'd love to walk you through what
we do today. Could you tell me a little about yourself?"

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

### Technical config (slide demo)

```json
{
  "ai_model_config": {
    "provider": "Landmass",
    "model": "cascade-01",
    "tts_provider": "cartesia",
    "tts_voice_id": "[voice-id]",
    "llm_provider": "google_vertex",
    "llm_model": "gemini-3.1-flash-lite",
    "stt_provider": "deepgram"
  },
  "tools_config": {
    "tools": {
      "google_slides": {
        "should_register": true,
        "add_to_system_prompt": false,
        "tool_settings": { "embedUrl": "[GOOGLE_SLIDES_EMBED_URL]" }
      },
      "browser": { "should_register": false, "add_to_system_prompt": false },
      "knowledge_base_search": { "should_register": true, "add_to_system_prompt": true },
      "end_session": { "should_register": true, "add_to_system_prompt": true, "tool_settings": { "disconnectDelaySeconds": 5 } },
      "emoji_reaction": { "should_register": true, "add_to_system_prompt": false }
    }
  },
  "session_analysis": { "is_auto_analysis": true, "is_auto_submit": true }
}
```

For slide demos using Landmass/cascade, also follow [cascade-tts.md](cascade-tts.md)
voice-pipeline rules in `ai_instructions`.

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

| Anti-pattern | Instead |
|-------------|---------|
| Feature tour (showing everything) | Focus on 3-5 sections mapped to prospect's problem |
| Reading slides verbatim | Use slides as visual anchors, add context beyond the screen |
| Waiting for the user to drive | Be proactive — transition between sections |
| Interrogating with qualifying questions upfront | Weave questions naturally between demo sections |
| No knowledge base | Upload docs, FAQs, pricing, case studies for deep questions |
| Browser demo without `multimodal_analysis` | Enable it — visual navigation is part of the analysis |

---

## Quality Checklist

- [ ] Format chosen: browser or slide (or hybrid)
- [ ] 3-5 core sections defined with clear flow
- [ ] Qualifying questions woven between sections
- [ ] Knowledge base tool enabled with relevant content
- [ ] `ai_model_config` set: Ocean/medium-stable (browser) or Landmass/cascade-01 (slides)
- [ ] Slide demo with cascade: voice-pipeline blocks from [cascade-tts.md](cascade-tts.md)
- [ ] `conductor` timeout at ~600s (10 min)
- [ ] `end_session` enabled
- [ ] Rubrik produces a Demo Intelligence Report (not a performance review)
- [ ] Browser demo: `contextId` and `initialUrl` set; `multimodal_analysis: true`
- [ ] Slide demo: `embedUrl` set in `google_slides` tool settings
