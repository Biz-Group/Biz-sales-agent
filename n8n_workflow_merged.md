# N8N WORKFLOW — BIZ GROUP AI SALES AGENT PIPELINE (MERGED ARCHITECTURE)

## OVERVIEW

This workflow combines a two-agent architecture (live Sales Agent + post-call Analysis Agent) with automated routing, CRM logging, and action execution. It is based on an existing n8n workflow with enhancements for compliance, routing, and follow-through.

---

## ARCHITECTURE

```
                                    ┌─────────────────────┐
                                    │   DNC Pre-Check      │──[MATCHED]──▶ Log DNC ──▶ STOP
                                    └──────────┬──────────┘
                                               │ [CLEAN]
┌──────────────────┐    ┌──────────────────┐   │   ┌─────────────────────────┐
│ Incoming Call     │───▶│ Workflow          │───┘──▶│ ElevenLabs Voice        │
│ Trigger (POST)   │    │ Configuration     │       │ Interaction             │
└──────────────────┘    └──────────────────┘       │ (speechToSpeech)        │
                                                    └────────────┬────────────┘
                                                                 │
                                                    ┌────────────▼────────────┐
                                                    │ Sales Call AI Agent      │
                                                    │ (Claude Chat Model)     │
                                                    │   + Conversation Memory │
                                                    │   + Structure Call Data │
                                                    │     (Output Parser)     │
                                                    └────────────┬────────────┘
                                                                 │
                                                    ┌────────────▼────────────┐
                                                    │ Prepare Data for        │
                                                    │ Analysis (Code Node)    │
                                                    └────────────┬────────────┘
                                                                 │
                                                    ┌────────────▼────────────┐
                                                    │ Analysis AI Agent       │
                                                    │ (Claude Chat Model)     │
                                                    │   + Structure Analysis  │
                                                    │     Output (Parser)     │
                                                    └────────────┬────────────┘
                                                                 │
                                              ┌──────────────────▼──────────────────┐
                                              │        SWITCH NODE                   │
                                              │   (Route on outcome field)           │
                                              └──┬──────┬──────┬──────┬──────┬──────┘
                                                 │      │      │      │      │
                                    ┌────────────▼─┐ ┌──▼────────┐ ┌──▼──────┐ ┌──▼──────────┐ ┌──▼──────┐
                                    │BOOK MEETING  │ │SEND       │ │FOLLOW   │ │HUMAN        │ │TERMINAL │
                                    │              │ │PROPOSAL   │ │UP       │ │HANDOFF      │ │OUTCOMES │
                                    │Calendar +    │ │Format +   │ │Email +  │ │Slack +      │ │DNC /    │
                                    │Confirm Email │ │Email +    │ │CRM Tag +│ │CRM Flag +   │ │NOT_INT /│
                                    │              │ │Follow-Up  │ │Schedule │ │Assign       │ │CONSENT  │
                                    └──────┬───────┘ └─────┬─────┘ └────┬────┘ └──────┬──────┘ └────┬────┘
                                           │               │            │             │             │
                                           └───────────────┴────────────┴─────────────┴─────────────┘
                                                                        │
                                                           ┌────────────▼────────────┐
                                                           │  LOG TO CRM / SHEETS    │
                                                           │  (ALL outcomes)          │
                                                           └─────────────────────────┘
```

---

## NODE-BY-NODE SPECIFICATION

---

### NODE 1: Incoming Call Trigger

**Type:** Webhook (POST)

**Purpose:** Receives the inbound trigger when an outbound call should be placed. Could come from a CRM, a scheduler, or a manual trigger.

**Webhook URL:** `https://your-n8n-instance.com/webhook/incoming-call`

**Expected payload:**
```json
{
  "lead_id": "row_123",
  "name": "Ahmed Al Maktoum",
  "company": "Emirates Group",
  "role": "VP Human Resources",
  "phone": "+971501234567",
  "email": "ahmed@emiratesgroup.com",
  "source_notes": "Downloaded Axonify whitepaper, attended Feb webinar"
}
```

**Alternative triggers:**
- **Google Sheets Trigger** — watches for new rows or status changes in a leads sheet.
- **Schedule Trigger** — runs every 30 min during business hours (9am–5pm Gulf time, Sun–Thu), pulls leads with `status = READY_TO_CALL`.
- **CRM Webhook** — fires when a lead enters a specific pipeline stage in HubSpot/Pipedrive.

---

### NODE 1.5 (NEW): DNC Pre-Check

**Type:** IF Node + Google Sheets Lookup (or HTTP Request to DNC API)

**Purpose:** Before making any call, check if the phone number is on the Do-Not-Call list.

**Logic:**
```
Lookup phone number in DNC Sheet/database
  → IF matched: Skip call → Log as DNC_PRECHECKED → STOP
  → IF clean: Proceed to Workflow Configuration
```

**Implementation:**
```javascript
// Code node: Check DNC list
const phone = $input.first().json.phone;

// Option 1: Google Sheets lookup
// Option 2: Airtable lookup
// Option 3: Internal DNC API

const isDNC = false; // Replace with actual lookup

return [{ json: { ...($input.first().json), dnc_check: isDNC ? "BLOCKED" : "CLEAR" } }];
```

Add an IF node after: `$json.dnc_check === "CLEAR"` → proceed. Otherwise → log and stop.

---

### NODE 2: Workflow Configuration

**Type:** Set Node

**Purpose:** Prepare all context variables the Sales Agent needs.

**Fields to set:**
```javascript
{
  // Lead context
  LEAD_NAME: {{ $json.name }},
  COMPANY: {{ $json.company }},
  ROLE: {{ $json.role }},
  PHONE: {{ $json.phone }},
  EMAIL: {{ $json.email }},
  LEAD_SOURCE_NOTES: {{ $json.source_notes }},
  LEAD_ID: {{ $json.lead_id }},

  // System context
  MEETING_LINK: "https://calendly.com/bizgroup-sales/30min",
  SYSTEM_TIME_UTC: {{ $now.toISO() }},

  // Prompts (reference the separate prompt files)
  SALES_AGENT_PROMPT: "<Paste full Sales Call AI Agent prompt here>",
  ANALYSIS_AGENT_PROMPT: "<Paste full Analysis AI Agent prompt here>"
}
```

---

### NODE 3: ElevenLabs Voice Interaction

**Type:** ElevenLabs Node (speechToSpeech)

**Purpose:** Conducts the actual voice call.

**Configuration:**
- **Agent ID:** Your ElevenLabs Conversational AI agent
- **Mode:** Speech-to-speech
- **Voice:** Select a professional, warm voice (e.g., "Rachel" or a custom clone)
- **First message:** Injected from Workflow Configuration
- **System prompt:** Passed from `SALES_AGENT_PROMPT`

**This node connects to the Sales Call AI Agent for conversation logic.**

---

### NODE 4: Sales Call AI Agent

**Type:** AI Agent Node (Claude Chat Model + Memory + Output Parser)

**Sub-nodes:**

#### 4A: Claude Chat Model (Sales Agent)
- **Model:** claude-sonnet-4-5-20250929 (or claude-sonnet-4-5-20250929 for cost optimization)
- **System prompt:** The Sales Call AI Agent prompt (from separate file)
- **Temperature:** 0.7 (natural conversation, not too creative)
- **Max tokens:** 500 per turn (keeps responses concise for voice)

#### 4B: Conversation Memory
- **Type:** Window Buffer Memory
- **Window size:** 20 messages (enough for a full discovery call)
- **Purpose:** Maintains conversation context across turns so the agent remembers what was already discussed

#### 4C: Structure Call Data (Output Parser)
- **Type:** Structured Output Parser
- **Purpose:** Extracts basic call metadata from the conversation agent
- **Schema:** Lightweight — just enough for the handoff to analysis:
```json
{
  "call_ended": true,
  "consent_given": true,
  "recording_consented": true,
  "prospect_email_captured": "string or null",
  "general_impression": "string — brief note on how the call went"
}
```
- **Note:** This parser is intentionally lightweight. The heavy extraction happens in the Analysis Agent. Don't burden the conversation agent with complex JSON output.

---

### NODE 5: Prepare Data for Analysis

**Type:** Code Node (JavaScript)

**Purpose:** Package the full conversation transcript + all metadata into a clean input for the Analysis Agent.

```javascript
const conversationHistory = $node['Sales Call AI Agent'].json;
const workflowConfig = $node['Workflow Configuration'].json;
const callData = $node['Structure Call Data'].json;
const elevenLabsOutput = $node['ElevenLabs Voice Interaction'].json;

const analysisInput = {
  transcript: elevenLabsOutput.transcript || conversationHistory.output || "",
  call_metadata: {
    lead_id: workflowConfig.LEAD_ID,
    lead_name: workflowConfig.LEAD_NAME,
    lead_company: workflowConfig.COMPANY,
    lead_role: workflowConfig.ROLE,
    lead_phone: workflowConfig.PHONE,
    lead_email: workflowConfig.EMAIL,
    lead_source_notes: workflowConfig.LEAD_SOURCE_NOTES,
    call_duration_seconds: elevenLabsOutput.duration || null,
    recording_url: elevenLabsOutput.recording_url || null,
    recording_consented: callData.recording_consented || null,
    ai_disclosed: true,
    prospect_email_captured: callData.prospect_email_captured || null
  }
};

return [{ json: analysisInput }];
```

---

### NODE 6: Analysis AI Agent

**Type:** AI Agent Node (Claude Chat Model + Output Parser)

**Sub-nodes:**

#### 6A: Claude Chat Model (Analysis)
- **Model:** claude-sonnet-4-5-20250929
- **System prompt:** The Analysis AI Agent prompt (from separate file)
- **Temperature:** 0.1 (deterministic — we want consistent, accurate extraction)
- **Max tokens:** 4000 (needs room for full structured output)

#### 6B: Structure Analysis Output (Output Parser)
- **Type:** Structured Output Parser
- **Purpose:** Validates and enforces the full output schema
- **Schema:** The complete JSON schema from the Analysis Agent prompt (outcome, consent, lead, category_fit, qualification, proposal_inputs, etc.)

**Input message to Analysis Agent:**
```
Analyze the following sales call transcript and metadata. Produce structured JSON output following your schema.

TRANSCRIPT:
{{ $json.transcript }}

CALL METADATA:
{{ JSON.stringify($json.call_metadata) }}
```

---

### NODE 7: Switch Node — Route by Outcome

**Type:** Switch Node

**Field to evaluate:** `{{ $json.outcome }}`

**Routes:**

| Output | Condition | Destination |
|--------|-----------|-------------|
| 0 | `outcome` = `BOOK_MEETING` | → Book Meeting Branch |
| 1 | `outcome` = `SEND_PROPOSAL` | → Send Proposal Branch |
| 2 | `outcome` = `FOLLOW_UP` | → Follow-Up Branch |
| 3 | `outcome` = `HUMAN_HANDOFF` | → Human Handoff Branch |
| 4 | `outcome` IN (`DNC`, `NOT_INTERESTED`, `CONSENT_DENIED`, `NO_ANSWER`, `CALL_FAILED`) | → Terminal Outcomes (log only) |
| Fallback | Anything else | → Human Handoff (safe default) |

---

### NODE 7A: Book Meeting Branch

**Step 1 — Google Calendar Node:**
```
Event Title: "Biz Group Discovery — {{ $json.lead.name }} ({{ $json.lead.company }})"
Duration: {{ $json.next_step.meeting_duration_preference || 30 }} minutes
Attendees: {{ $json.next_step.email_confirmed }}, sales-team@bizgroup.ae
Description: |
  Category: {{ $json.category_fit }}
  Temperature: {{ $json.lead_temperature }}
  Summary: {{ $json.summary_bullets.join('\n') }}
  Pains: {{ $json.pains.join(', ') }}
```

**Step 2 — Gmail Node (Confirmation):**
```
To: {{ $json.next_step.email_confirmed || $json.lead.email }}
Subject: "Your meeting with Biz Group is confirmed"
Body:
  Hi {{ $json.lead.name }},

  Thanks for your time today. As discussed, we've scheduled a short session
  with one of our specialists to explore how Biz Group can help with
  {{ $json.desired_outcomes[0] || "your learning and engagement goals" }}.

  Meeting details will arrive via calendar invite shortly.

  Looking forward to it.

  Best,
  Biz Group Team
```

---

### NODE 7B: Send Proposal Branch

**Step 1 — Format Final Proposal (your existing node):**

This is where your existing "Format Final Proposal" node fits. It takes `proposal_inputs` from the Analysis Agent and generates a proposal document.

**If using Claude to generate the proposal:**
```
System: You are a proposal writer for Biz Group. Using the inputs below,
generate a professional 1-2 page proposal. Include: executive summary,
proposed solution, scope, timeline, indicative investment, and next steps.
Use the prospect's own language for their pains and goals. Never fabricate
capabilities.

Input: {{ JSON.stringify($json.proposal_inputs) }}
Pains: {{ JSON.stringify($json.pains) }}
Outcomes: {{ JSON.stringify($json.desired_outcomes) }}
```

**Step 2 — Return Proposal (your existing node) + Email:**
```
To: {{ $json.next_step.email_confirmed || $json.lead.email }}
Subject: "Biz Group — Tailored Proposal for {{ $json.lead.company }}"
Body: Professional cover email + proposal attached or inline
```

**Step 3 — Schedule Proposal Follow-Up:**
- Wait 3 business days
- Send follow-up email: "Hi {{name}}, just checking in on the proposal..."
- Create CRM task for sales team to follow up

---

### NODE 7C: Follow-Up / Nurture Branch

**Step 1 — Select & Send One-Pager (IF node for category):**

| category_fit | Attachment |
|-------------|------------|
| LEARNING_TECH | learning-platform-overview.pdf |
| TRAINING_COURSES | training-catalogue.pdf |
| TEAM_BUILDING | team-building-brochure.pdf |
| HYBRID / UNCLEAR | bizgroup-general-overview.pdf |

**Gmail Node:**
```
To: {{ $json.lead.email }}
Subject: "Quick overview from Biz Group"
Body:
  Hi {{ $json.lead.name }},

  Great speaking with you today. As promised, here's a quick overview
  of how we help organizations with {{ category-relevant description }}.

  No pressure at all — have a look when it suits you, and feel free
  to reach out if anything sparks a question.

  Best,
  Biz Group Team
```

**Step 2 — Schedule Follow-Up Task:**
- Create CRM/Sheet entry with `follow_up_date` (or default: 5 business days)
- Tag lead as `NURTURING`

---

### NODE 7D: Human Handoff Branch

**Step 1 — Slack Notification:**
```
Channel: #sales-escalations

🚨 *Human Handoff Required*

*Lead:* {{ $json.lead.name }} ({{ $json.lead.company }})
*Role:* {{ $json.lead.title }}
*Contact:* {{ $json.lead.email }} / {{ $json.lead.phone }}
*Category:* {{ $json.category_fit }}
*Temperature:* {{ $json.lead_temperature }}

*Reason:* {{ $json.handoff_reason }}

*Summary:*
{{ $json.summary_bullets.map(b => '• ' + b).join('\n') }}

*Pains:* {{ $json.pains.join(', ') }}
*Objections:* {{ $json.objections.map(o => o.objection).join(', ') }}
*Budget:* {{ $json.qualification.budget_range }}
*Timeline:* {{ $json.qualification.timeline }}

*Recording:* {{ recording_url || 'N/A' }}
```

**Step 2 — CRM Task:**
- Create task assigned to sales manager
- Priority: HIGH
- Due: Today
- Include full analysis output as note

---

### NODE 8: Log to CRM / Master Sheet (ALL outcomes)

**Type:** Google Sheets Node (Append Row)

**Purpose:** Every single call gets logged, regardless of outcome.

**Sheet columns:**
```
timestamp | lead_id | name | company | role | phone | email | outcome |
consent_recording | category_fit | lead_temperature | summary |
pains | desired_outcomes | current_solution | learners_count |
audience | stakeholders | timeline | budget_range | authority_level |
objections | next_step_type | next_step_details | handoff_reason |
call_duration | recording_url | follow_up_date | call_quality_notes
```

**Also update the lead's status in the source system:**

| Outcome | New Status |
|---------|-----------|
| BOOK_MEETING | MEETING_SCHEDULED |
| SEND_PROPOSAL | PROPOSAL_SENT |
| FOLLOW_UP | NURTURING |
| HUMAN_HANDOFF | ESCALATED |
| NOT_INTERESTED | CLOSED_LOST |
| DNC | DO_NOT_CONTACT |
| CONSENT_DENIED | CONSENT_DENIED |
| NO_ANSWER | RETRY_QUEUE |
| CALL_FAILED | RETRY_QUEUE |

---

### NODE 9: Error Handler

**Type:** Error Trigger Node

**Purpose:** Catch failures at any point.

**Actions:**
1. Slack notification to `#workflow-errors` with error details + lead info
2. Update lead status to `CALL_FAILED`
3. Add to retry queue (max 2 retries per lead, then escalate to human)

---

## ENHANCEMENTS

### A. Business Hours Gate
After the trigger, add a Code node:
```javascript
const now = new Date();
const hour = now.getUTCHours() + 4; // Gulf UTC+4
const day = now.getUTCDay(); // 0=Sun

// Business hours: Sun-Thu, 9am-5pm Gulf time
const isBusinessDay = day >= 0 && day <= 4; // Sun=0 to Thu=4
const isBusinessHour = hour >= 9 && hour < 17;

if (!isBusinessDay || !isBusinessHour) {
  // Queue for next business window
  return [{ json: { ...$json, queued: true, queue_reason: "Outside business hours" } }];
}

return [{ json: { ...$json, queued: false } }];
```

### B. Rate Limiting
Add a **Wait node** (60 seconds) between calls to avoid API overload and comply with telecom regs.

### C. Retry Logic
For `NO_ANSWER` and `CALL_FAILED`:
- Check retry count (stored in CRM/sheet)
- If < 2 retries → queue for retry (different time of day)
- If >= 2 retries → mark as `EXHAUSTED` and stop

### D. Follow-Up Nurture Sequence
For `FOLLOW_UP` outcomes, build a sub-workflow:
- **Day 0:** Send one-pager
- **Day 3:** Send case study or ROI calculator
- **Day 7:** AI follow-up call (shorter prompt, references first call)
- **Day 14:** Final email + close loop

### E. Call Quality Dashboard
Pipe the `call_quality_notes` and `compliance_flags` into a separate sheet for QA review. Flag calls where:
- AI disclosure was missed
- Consent wasn't obtained
- Agent named Axonify prematurely
- Call was under 30 seconds (likely a failure)

---

## CREDENTIALS REQUIRED

| Credential | Used By |
|-----------|---------|
| ElevenLabs API Key | Voice call node |
| Anthropic API Key (Claude) | Both AI Agent nodes + proposal generation |
| Google Sheets OAuth | Lead source, logging, DNC check |
| Gmail OAuth | All email sends |
| Google Calendar OAuth | Meeting booking |
| Slack OAuth | Human handoff + error alerts |
| Calendly API Key (optional) | Meeting scheduling alternative |
| CRM API Key (optional) | HubSpot / Pipedrive integration |

---

## BUILD ORDER (RECOMMENDED)

1. **Phase 1 — Core loop:** Trigger → Config → ElevenLabs → Sales Agent → Prepare → Analysis Agent → Log. Get this working end-to-end with a test phone number.
2. **Phase 2 — Routing:** Add Switch node + the four branches (meeting, proposal, follow-up, handoff). Test each branch independently.
3. **Phase 3 — Compliance:** Add DNC pre-check, business hours gate, error handler.
4. **Phase 4 — Polish:** Add retry logic, nurture sequence, quality dashboard.

---

## TESTING CHECKLIST

- [ ] Webhook trigger receives and parses lead data correctly
- [ ] DNC check blocks known numbers
- [ ] Business hours gate queues off-hours calls
- [ ] ElevenLabs initiates call and streams audio
- [ ] Sales Agent conducts natural conversation with memory
- [ ] Prepare Data node packages transcript + metadata cleanly
- [ ] Analysis Agent produces valid JSON matching schema
- [ ] Switch node routes each outcome type correctly
- [ ] BOOK_MEETING creates calendar event + sends email
- [ ] SEND_PROPOSAL generates proposal + sends email
- [ ] FOLLOW_UP sends correct one-pager + creates task
- [ ] HUMAN_HANDOFF fires Slack with full context
- [ ] ALL outcomes log to master sheet
- [ ] Lead status updates in source system
- [ ] Error handler catches and reports failures
- [ ] Rate limiting prevents API overload
- [ ] Retry logic handles NO_ANSWER correctly
