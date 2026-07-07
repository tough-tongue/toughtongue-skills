---
name: scenario-creator
description: >
  Create ToughTongue AI practice scenarios (cold call, sales roleplay, coaching)
  via the ttai MCP server. Classifies the scenario type, applies type-specific
  authoring rules, gathers context from URLs, transcripts, or other connected
  tools, validates against a checklist, and creates the scenario with
  ttai:create_scenario. Use when the user says "create a scenario", "build a
  practice scenario", "make a roleplay for...", "I have a call in 30 minutes,
  help me rehearse", or provides a brief, company info, or call transcripts
  for scenario creation.
---

# Scenario Creator

Create production-ready ToughTongue AI scenarios and push them live through the
ttai MCP server. Classify → load rules → gather context → draft → validate →
`ttai:create_scenario` → return the practice link.

## Prerequisites

- The **ttai** MCP server must be connected. Tool references below use the
  `ttai:` server prefix (e.g. `ttai:create_scenario`); some agents surface
  these as `mcp__ttai__create_scenario`. If the tools are missing, tell the
  user to install the ToughTongue plugin or add the MCP server (see the repo
  README) with a `TTAI_PAT` token from
  <https://app.toughtongueai.com/developer>.

## Workflow

### Step 1: Establish account context

Call `ttai:list_organizations` first.

- If the user belongs to organizations and the scenario is for a team, pass the
  chosen `org_id` on every subsequent tool call.
- If no organizations, or the scenario is personal practice, omit `org_id`.
- If ambiguous, ask which context to create in.

### Step 2: Classify scenario type

| Type | AI plays | Reference file |
|------|----------|----------------|
| **Cold Call / SDR** | The outbound caller (user plays the lead) | [references/cold-call.md](references/cold-call.md) |
| **Sales Roleplay** | The prospect (user practices selling) | [references/sales-roleplay.md](references/sales-roleplay.md) |
| **Coaching** | The trainer/mentor (teaches via exercises) | [references/coaching.md](references/coaching.md) |
| **Demo** | The AI SDR / product demo agent | [references/demo.md](references/demo.md) |
| **Other** | Anything else (interview, support, negotiation) | [references/scenario-fields.md](references/scenario-fields.md) only |

Decision signals:

- "cold call", "outbound", "lead qualification", "AI calls the customer",
  "SDR call" → Cold Call / SDR
- "practice selling", "objection handling", "prospect roleplay", "pitch practice",
  "prep me for this meeting" → Sales Roleplay
- "coach", "train my team", "teach", "onboarding", "framework" → Coaching
- "demo my product", "AI SDR demo", "show prospects", "browser demo",
  "slide demo", "product walkthrough" → Demo

If ambiguous, ask ONE question: "Should the AI play the caller/seller, the
buyer/prospect, a coach/trainer, or a product demo agent?"

Read [references/scenario-fields.md](references/scenario-fields.md) (always)
plus the matching type reference.

### Step 3: Gather context

- **URLs provided** (company site, product page, LinkedIn): fetch them. Extract
  company name, product, target audience, key features, pricing model. Fold
  into the `ai_instructions` CONTEXT section and `user_friendly_description`.
- **Other connected tools**: if the user references meetings, CRM records, call
  transcripts, or documents available through other MCP servers (calendar,
  Gong, Notion, ...), pull the relevant details and use them as scenario
  context — real names, real objections, real positioning beat invented ones.
- **Pasted material** (transcripts, briefs, positioning docs): mine it for the
  persona, objections, and vocabulary the scenario should reproduce.

### Step 4: Clarifying questions (minimal)

Only ask when the answer is not obvious from the brief. Otherwise use defaults:

| Question | Ask when | Default |
|----------|----------|---------|
| Language & voice | Locale unclear from context | `en-US`, defaults from [references/scenario-fields.md](references/scenario-fields.md) |
| Call sub-type (cold call) | Warm/cold/follow-up unclear | Warm lead |
| Coaching pattern | Coaching type only | Pattern A (Situation-First) |
| Public or private | Team/enterprise use implied | `is_public: true` |

### Step 5: Draft the scenario payload

Build a JSON payload matching the `ttai:create_scenario` input schema (load
the tool schema before calling). Author these fields, in order of importance:

1. `name` — short, descriptive display title.
2. `ai_model_config` — set explicitly based on scenario type. See the "When
   to use which" table in [references/scenario-fields.md](references/scenario-fields.md).
   Cold call and slide-demo scenarios use Landmass/cascade-01 (requires TTS,
   STT, LLM fields). Sales roleplay uses Galaxy/medium. Coaching and
   browser-demo use Ocean/medium-stable.
3. `ai_instructions` — the core field, 500+ words, structured with `##`
   sections per the type reference. For Landmass/cascade scenarios, also load
   [references/cascade-tts.md](references/cascade-tts.md) and include the
   voice-pipeline blocks (output rules, transcription-error handling, natural
   speech style, SSML emotion tags if Cartesia).
4. `user_instructions` — what the human should know before starting:
   situation → what to expect → how to succeed → tips.
5. `rubrik` — evaluation criteria. CRITICAL: evaluate the correct party
   (cold call rubrics evaluate the LEAD; sales rubrics evaluate the REP;
   demo rubrics produce a buyer intelligence report).
6. `user_friendly_description` — 1-2 public-facing sentences.
7. `strategy`, `tools_config`, `session_analysis`, `appearance` — per the
   type reference and [references/scenario-fields.md](references/scenario-fields.md) defaults.
8. `is_recording: true` for voice scenarios; `is_public` per Step 4.

Do NOT set `id` — `ttai:create_scenario` rejects it (that is
`ttai:update_scenario`'s job).

### Step 6: Validate

Run the universal checklist, plus the type-specific checklist from the
reference file:

- [ ] `name`, `ai_instructions`, `user_friendly_description` present
- [ ] `ai_model_config` set explicitly per the "When to use which" table
- [ ] `ai_instructions` structured with `##` sections; no unresolved
      placeholders except intentional `{{ dynamic_vars }}`
- [ ] `tools_config.tools.end_session` enabled with `add_to_system_prompt: true`
- [ ] `session_analysis.is_auto_analysis: true` and `is_auto_submit: true`
- [ ] `rubrik` evaluates the correct party, categories with weights
- [ ] Cascade scenarios (Landmass): voice-pipeline blocks from
      [cascade-tts.md](references/cascade-tts.md), `strategy.welcome_instructions`
      (directive form, never quoted speech), conductor wrap-up message,
      `appearance.language_code` matches locale
- [ ] Every dynamic variable `{{ var }}` has a documented missing-value fallback

### Step 7: Create

Call `ttai:create_scenario` with the payload (and `org_id` if applicable). On
validation errors, fix the named field and retry — do not strip features to
force it through.

### Step 8: Return links

Report back with:

- **Practice link**: `https://app.toughtongueai.com/run/<scenario_id>`
- **Embed link** (if the user builds apps): `https://app.toughtongueai.com/embed/<scenario_id>`
- What was created (type, persona, evaluation focus) in 2-3 sentences.
- For private scenarios: mention `ttai:create_scenario_access_token` mints
  1-hour access tokens for sharing.

## Quick path: `ttai:generate_scenario`

For a fast draft without hand-authoring, the `ttai:generate_scenario` tool
generates `ai_instructions`, `user_instructions`, and a description
server-side from a name and context document. Use it when the user wants
speed over control, then review the output and create via
`ttai:create_scenario`. Prefer full authoring for anything the user will run
with a team.

## Pitfalls

- **Never stack questions** in voice-agent turns — one question per turn is the
  #1 authoring rule for natural calls.
- **Never quote the opening line** in `welcome_instructions` — use directive
  form ("Start with: ... Then STOP and wait."). Quoted text is delivered
  robotically and restarts on interruption.
- **Wrong rubric target** — a cold-call rubric that scores the AI caller
  instead of the lead produces useless reports.
- **Missing end_session guidance** — without explicit timing rules the agent
  either never hangs up or hangs up mid-conversation.
- The API token stays server-side; never embed `TTAI_PAT` in anything you
  generate for the user's app.
