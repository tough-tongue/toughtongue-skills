# Demo Scenario Patterns

Rules for scenarios where the **AI is a product demo agent** — walking prospects
through a product via live browser navigation or slides.

## Contents

- Two demo formats (browser vs slides)
- ai_instructions structure and demo flow
- Qualifying questions pattern
- Tools config (browser, slides, knowledge base)
- Rubric: Demo Intelligence Report
- Technical config and quality checklist

Key facts:

- Two formats: **browser-based** (live navigation) and **slide-based** (curated
  narrative)
- AI is proactive — drives the conversation, does not wait for the user
- Qualifying questions are woven between demo sections, not asked upfront
- Rubric produces a buyer intelligence report, not a performance review
- Browser demos: Ocean `medium-stable` (fall back to Galaxy if Ocean is
  unavailable).
- **Slide demos: Landmass `cascade` + Cartesia** by default for polished TTS
  narration — load [cascade-tts.md](cascade-tts.md). Use Galaxy/Ocean realtime
  only when native voice is enough and narration polish is secondary. See
  [../scenario/model-selection.md](../scenario/model-selection.md).

---

## Choosing the Format

- **Browser** — web app / SaaS product with a demo URL. Primary tool: `browser`
  (step, goto, act, observe, capture).
- **Slides** — curated narrative, no demo environment. Primary tool:
  `google_slides` (embedUrl).
- **Slides** — complex product requiring login/setup. Use a controlled story
  arc.
- **Both** — hybrid: slides for narrative, browser for a live "wow" moment.

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

The browser is pre-opened at the product URL in a live view the prospect sees.
The agent calls `runBrowserCommand`:

- `command: "step"` + `name` — replay a recorded flow, if the Scenario has
  recorded steps (exact selectors first; AI repair only when one fails)
- `command: "goto"` + `url` — navigate directly
- `command: "act"` + `instruction` — one unscripted interaction
- `command: "observe"` + `instruction` — check that controls exist on a stable
  page; not a required step before `act`
- `command: "capture"` — screenshot; costs an extra turn, so use it only at
  visual milestones

Decision order: `step` when a recorded flow matches, else `goto`, then `act`.
Commands run asynchronously: the agent keeps narrating and waits for completion
before a dependent command.

**Pre-recorded steps (deterministic demos):** for a scripted walkthrough that
always clicks the same things, record the flow into `tool_settings.steps` and
make `step` the primary command — hand off to the `ttai-browser-demo-builder`
skill for the recording workflow, selector rules, and instruction templates.

**Logins:** the Scenario keeps one saved browser profile
(`tool_settings.contextId`, set by the platform — never invent or hand-edit it).
For a product behind a login, enable the `browser` tool, then call
`ttai:authenticate_browser` and give the returned link to the user; they log in
once and the profile keeps the session. On update, resend the full
`browser.tool_settings` so `contextId` and `steps` survive.

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
[Pre-opened URL, key URLs map, which recorded step or page each phase uses]

## DEMO FLOW
### Phase 1: Introduction (Home Page)
### Phase 2: [Core Feature 1]
### Phase 3: [Core Feature 2]
### Phase 4: [Core Feature 3]
### Phase 5: Wrap-Up — summarize, ask what resonated, suggest next steps

## KNOWLEDGE BASE
Only if a Knowledge Base is attached (`knowledge_base_ids`): use
knowledge_base_search for deeper questions. Otherwise answer from CONTEXT and
do not mention a KB.
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
    "silence": {
      "silence_threshold": 6000,
      "end_session": false,
      "force_agent_to_speak": true
    },
    "conductor": {
      "enabled": true,
      "messages": [
        {
          "time_seconds": 600,
          "message": "Wrap up the demo: summarize and say goodbye. Wait for their reply before end_session.",
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
        "tool_settings": { "initialUrl": "[PRODUCT_URL]" }
      },
      "google_slides": {
        "should_register": false,
        "add_to_system_prompt": false
      },
      "knowledge_base_search": {
        "should_register": false,
        "add_to_system_prompt": false
      },
      "end_session": {
        "should_register": true,
        "add_to_system_prompt": true,
        "tool_settings": { "disconnectDelaySeconds": 5 }
      },
      "emoji_reaction": {
        "should_register": true,
        "add_to_system_prompt": false
      }
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
    "silence": {
      "silence_threshold": 6000,
      "end_session": false,
      "force_agent_to_speak": true
    }
  },
  "tools_config": {
    "tools": {
      "google_slides": {
        "should_register": true,
        "add_to_system_prompt": false,
        "tool_settings": { "embedUrl": "[GOOGLE_SLIDES_EMBED_URL]" }
      },
      "browser": { "should_register": false, "add_to_system_prompt": false },
      "knowledge_base_search": {
        "should_register": false,
        "add_to_system_prompt": false
      },
      "end_session": {
        "should_register": true,
        "add_to_system_prompt": true,
        "tool_settings": { "disconnectDelaySeconds": 5 }
      },
      "emoji_reaction": {
        "should_register": true,
        "add_to_system_prompt": false
      }
    }
  },
  "session_analysis": { "is_auto_analysis": true, "is_auto_submit": true }
}
```

Use Landmass Cascade when the demo needs a selected external voice or full
STT/LLM/TTS control. Load [cascade-tts.md](cascade-tts.md) only for that
pipeline. Realtime Galaxy or Ocean is appropriate when native voice and a faster
interactive exchange matter more.

---

## Qualifying Questions

Embed 3-5 qualifying questions that the agent weaves in naturally between demo
sections — not as an interrogation. Frame as conversation:

- "How are you currently handling [problem]?" (situation)
- "What's the biggest challenge with that approach?" (problem)
- "How large is the team involved?" (sizing)
- "What's your timeline for making a change?" (urgency)

---

## Rubric: Demo Intelligence Report

Demo rubrics are NOT performance reviews — they produce buyer intelligence for
the sales team.

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

Each line: anti-pattern → instead.

- Feature tour (showing everything) → focus on 3-5 sections mapped to the
  prospect's problem.
- Reading slides verbatim → use slides as visual anchors; add context beyond the
  screen.
- Waiting for the user to drive → be proactive — transition between sections.
- Interrogating with qualifying questions upfront → weave questions naturally
  between demo sections.
- Enabling `knowledge_base_search` with no docs → discover the approved
  Knowledge Base ID, attach it through Scenario authoring, then enable the tool.
- Browser demo without `multimodal_analysis` → enable it when the plan allows —
  visual navigation is part of the analysis.

---

## Quality Checklist

- [ ] Format chosen: browser or slide (or hybrid)
- [ ] 3-5 core sections defined with clear flow
- [ ] Qualifying questions woven between sections
- [ ] Knowledge base: enable the tool only when `knowledge_base_ids` attaches
      one
- [ ] `ai_model_config` matches the selected browser or slide delivery path
- [ ] Cascade voice-pipeline blocks only if the model is Landmass Cascade
- [ ] `conductor` timeout at ~600s (10 min)
- [ ] `end_session` enabled
- [ ] Rubric produces a Demo Intelligence Report (not a performance review)
- [ ] Browser demo: `initialUrl` set; login done via `ttai:authenticate_browser`
      if needed; `multimodal_analysis: true` when the plan allows
- [ ] Slide demo: `embedUrl` set in `google_slides` tool settings

## Key Files

- [../scenario/authoring.md](../scenario/authoring.md) — durable authoring
  principles
- [../scenario/model-selection.md](../scenario/model-selection.md) — pipeline
  stamps
- [../scenario/control.md](../scenario/control.md) — strategy / tools
- [../scenario/ai-instructions.md](../scenario/ai-instructions.md) — prompt
  shape
- [../scenario/workflow.md](../scenario/workflow.md) — create/update procedure
