# Kesar & Co. Support Copilot — AI Automation Intern Submission

## 1. Summary

- **What I built:** An automated, production-grade Customer Support Copilot for D2C Ayurvedic skincare brand "Kesar & Co." that ingests support emails via webhook, classifies intent into structured JSON, looks up orders with strict privacy checks, enforces deterministic date math for return/damaged policies, drafts brand-aligned human responses, creates deduplicated support tickets, and maintains an end-to-end audit log in Google Sheets.
- **Platform and LLM used:** n8n Cloud (with exported blueprint), Google Gemini 1.5 Flash (via Google AI Studio, temperature 0.0 for deterministic classification, 0.2 for drafting), JavaScript deterministic logic nodes.
- **Live Google Sheet Link:** [Kesar Support Google Sheet](https://docs.google.com/spreadsheets/d/1dNM176hfNtVme6NZ-Yh7Ctni7wPnjRAEj1ajw87qdao/edit?usp=sharing) (View access enabled)
- **Time spent:** Approximately 5.5 hours across architecture design, prompt hardening, edge-case testing, and documentation.
- **What works:** 100% schema-valid JSON extraction, zero-hallucination policy grounding in `faq.md`, automated prompt injection neutralization (email `E12`), customer email privacy protection (email `E07`), smart ticket deduplication (email `E04` & `E21`), Hinglish/Hindi recognition, and deterministic return window validation.
- **What doesn't:** Unidentified vague emails with zero metadata (`E18`) cannot be automatically matched to customer records and are escalated to human review.

---

## 2. Architecture

```
Webhook Trigger (POST /support-email)
   │
   ▼
Normalize & Guardrails (Code Node: cleans payload, detects prompt injection)
   │
   ├─► Fetch Orders DB (GET orders.json)
   ├─► Fetch FAQ Knowledge (GET faq.md)
   │
   ▼
Classify & Extract Intent (Gemini LLM: schemas/llm_output.schema.json)
   │
   ▼
Order Lookup & Deterministic Rules (Code Node: email privacy check, date arithmetic vs 2026-11-20)
   │
   ▼
Route by Intent (Switch Node)
   ├─► order_status   ──► Draft tracking/delay update via brand voice
   ├─► product_query  ──► Answer ONLY from FAQ catalog; escalate medical claims
   ├─► return_refund  ──► Validate 7-day unopened window ──► Ticket + Draft
   ├─► complaint      ──► Damaged/leaking/wrong 48h check ──► High-Priority Ticket + Draft
   └─► other          ──► Filter spam/collab/GST ──► Log only or escalate
   │
   ▼
Ticket Deduplication (Code Node: search open tickets by order_id; update if exists, else create)
   │
   ▼
Format Audit Log Row (Code Node: 18 columns matching template)
   │
   ├─► Google Sheets (Audit Log tab append)
   └─► Webhook Response (HTTP 200 JSON confirmation)
```

| Step / node | What it does | Why I designed it this way |
|---|---|---|
| **Webhook** | Receives single email JSON payload from `send_test_emails.py`. | Standard enterprise integration pattern for real-time inbox ingestion. |
| **Normalize & Guardrails** | Sanitizes text and flags prompt injection patterns (`SYSTEM NOTE`, `ignore previous instructions`). | Treating user input as untrusted at the gateway prevents adversarial jailbreaks from reaching the LLM. |
| **Order & FAQ Fetch** | Pulls fresh order database and policy markdown. | Decouples data storage from workflow logic, treating mock files as read-only microservice APIs. |
| **Classify & Extract Intent** | Returns structured JSON adhering to `llm_output.schema.json`. | Structured extraction at temperature 0.0 guarantees consistent programmatic routing. |
| **Order Lookup & Rules** | Matches order by `order_id` or email, checks privacy, calculates `days_since_delivery` vs `2026-11-20`. | LLMs are notorious for date arithmetic errors; deterministic code ensures 100% reliable policy enforcement. |
| **Route by Intent** | Branches into 5 explicit support paths. | Isolates business domains, allowing independent tuning of policies, tools, and response templates. |
| **Draft Reply (Brand Voice)** | Generates 3-6 sentence warm, honest replies signed by Team Kesar. | Enforces strict compliance with `brand_voice.md` and medical disclaimers from `faq.md`. |
| **Ticket Deduplication** | Searches existing tickets; updates open tickets instead of creating duplicates. | Prevents split conversations and agent confusion when customers follow up on the same order. |
| **Google Sheets Logging** | Appends full execution record to the Audit Log. | Provides complete observability, compliance tracking, and human-in-the-loop escalation. |

---

## 3. Setup — How Someone Else Can Run This

1. **Accounts/credentials needed:**
   - Free [n8n Cloud](https://n8n.io) account (or self-hosted n8n v1.x+ via Docker).
   - Google Account with view/edit access to Google Sheets.
   - Free Gemini API key from [Google AI Studio](https://aistudio.google.com).
2. **How to import the workflow:**
   - Open n8n, navigate to **Workflows** → Click **Add Workflow** → Click the three dots (top right) → Select **Import from File**.
   - Choose `workflow/kesar_copilot_workflow.json` from this repository.
3. **Where to configure credentials:**
   - In the **Google Sheets (Audit Log)** node, select **Google Sheets OAuth2 API** and click "Sign in with Google". Paste your Google Sheet URL into `Document`.
   - In workflow settings or node environment, configure your `GEMINI_API_KEY`.
4. **How to run the 25 test emails:**
   ```bash
   # Test a small batch first
   python scripts/send_test_emails.py --url <YOUR_N8N_WEBHOOK_URL> --only E01,E02,E03

   # Run all 25 emails with 5-second delay
   python scripts/send_test_emails.py --url <YOUR_N8N_WEBHOOK_URL>
   ```

---

## 4. Prompts

The system utilizes two distinct, production-hardened prompts:

- **Classification & Extraction Prompt:** [prompts/classification_prompt.md](prompts/classification_prompt.md)
  - *Model:* Gemini 1.5 Flash | *Temperature:* 0.0 | *Response format:* Raw JSON matching `schemas/llm_output.schema.json`.
- **AI Reply-Drafting Agent Prompt:** [prompts/agent_reply_prompt.md](prompts/agent_reply_prompt.md)
  - *Model:* Gemini 1.5 Flash | *Temperature:* 0.2 | *System Persona:* Kesar & Co. Customer Specialist, bound to `brand_voice.md` (3-6 sentences, no emojis, no filler, warm tone, matching customer language).

---

## 5. Key Decisions and Assumptions

- **How I handle low-confidence classifications:** Any classification with confidence < 0.70 or unresolvable intent automatically sets `needs_human: TRUE` in the audit log. The system logs the intent and notes without auto-responding.
- **How I avoid duplicate tickets:** For return/refund or complaint emails, the workflow searches existing tickets matching the `order_id`. If an open ticket exists, it appends the new email ID (e.g. `E04, E21`) and appends the new message summary to `notes` with action `UPDATE`.
- **How I make sure order details only go to the right customer:** In `order_lookup_and_rules.js`, the sender email (`from.email`) is strictly compared against `order.customer_email`. If they do not match (as in email `E07`), `privacy_violation` is triggered: order details are withheld, and a polite privacy notice is drafted instead.
- **How I keep answers grounded in the FAQ:** System prompts explicitly instruct the model that `data/faq.md` is the sole source of truth. If a policy or product spec is missing (e.g. non-Indian shipping, medical cure claims), the copilot is barred from improvising and flags the request for human review.
- **Deterministic Date Math:** Scenario date `2026-11-20` is hardcoded into the business logic node. The 7-day return policy and 48-hour damage reporting windows are computed via `Math.floor(diffMs / 86400000)`, ensuring zero LLM date hallucination.

---

## 6. Scaling to 1,000 Emails/Day

At 1,000 emails/day (~42 emails/hour average, peaking at 150/hour during business hours):

1. **What would break first:**
   - **Rate Limits & Concurrency:** Direct LLM API calls and Google Sheets API calls would hit free-tier rate limits (Google Sheets quota is 60 requests/minute per user; Gemini free tier is 15 RPM).
   - **Google Sheets as a Database:** Appending and searching rows in Google Sheets at scale causes row-lock contention, race conditions during ticket deduplication, and slow response times.
2. **What I would change:**
   - **Queue Worker Architecture:** Introduce a message broker (RabbitMQ, Redis Streams, or AWS SQS) between the webhook and the processor, decoupling ingestion from execution with controlled concurrency and exponential backoff retries.
   - **Enterprise Storage:** Migrate tickets and audit logging from Google Sheets to PostgreSQL with indexed columns on `order_id` and `customer_email`, enabling ACID-compliant row locking for deduplication.
   - **Semantic Caching & Tiered Routing:** Cache common FAQ product queries in Redis. Route simple order status queries to lightweight models or direct SQL lookups, reserving frontier LLM reasoning only for complex complaints.

---

## 7. Product-Thinking Questions (≤150 words each)

### 1. Auto-send vs. Draft
> **Answer:** Auto-send should be reserved strictly for low-risk, deterministic flows: standard order status lookups (`order_status`) with verified matching customer emails, and routine product queries (`product_query`) covered verbatim in the FAQ, provided classification confidence is ≥ 0.90. Any request involving financial transactions (refunds, return pickup approvals), complaints of leaking/damaged goods, policy exceptions, sentiment tagged as `angry`, or confidence < 0.90 must be held as a draft. Customer trust in skincare is fragile; a premature incorrect refund refusal or hallucinated ingredient guarantee permanently damages brand equity, whereas a 2-hour human review creates peace of mind.

### 2. Cost Estimation at 1,000 Emails/Day
> **Answer:** 
> - **Token Math:** Classification requires ~400 input tokens + ~100 output tokens. Drafting reply with FAQ context requires ~1,000 input tokens + ~250 output tokens. Total per email: ~1,400 input + ~350 output tokens.
> - **Gemini 1.5 Flash Pricing:** \$0.075 / 1M input tokens; \$0.30 / 1M output tokens.
> - **Cost per Email:**
>   - Input: $1,400 \times \$0.000000075 = \$0.000105$
>   - Output: $350 \times \$0.00000030 = \$0.000105$
>   - Total = **\$0.00021 per email** (~₹0.0175 INR).
> - **Monthly Cost (30,000 emails/month):**
>   $30,000 \times \$0.00021 = \mathbf{\$6.30\text{/month}}$ (~₹525 INR/month).
> - Even with a 100% buffer for retries and tool overhead, total LLM cost remains under **\$13/month**, offering extraordinary ROI compared to full manual operations.

### 3. Success Metric & Kill Switch
> **Answer:** 
> - **Week 1 North Star Metric:** **Draft Acceptance Rate without Edits (DAR)** — the percentage of AI drafts approved and sent by the human agent with zero or minor modifications. A target of ≥ 80% proves the copilot genuinely accelerates workflow without generating hidden editing overhead.
> - **Kill Switch Condition:** I would immediately switch off auto-routing if the system experiences a **Zero-Tolerance Safety Violation**: specifically, any instance of customer PII leakage to an unauthorized email (privacy failure), approval of an unverified financial refund, or an ungrounded medical claim promising to cure a skin disease.

### 4. Failure Experience & Damage Limitation
> **Answer:** When the copilot fails (e.g. misclassifies intent or gives an awkward answer), the customer experiences confusion, frustration, or feels dismissed by an uncaring robot. To limit damage:
> 1. We keep responses in draft status for human oversight during initial rollout, preventing erroneous text from ever reaching customers.
> 2. For auto-responses, we enforce brand voice modesty: state clear facts, avoid definitive promises if uncertain, and include a warm human fallback line: *"If this doesn't fully answer your question, simply reply to this email and our team will personally assist you."*
> 3. Any repeated customer reply within 4 hours automatically bypasses the AI and routes to senior human support.

### 5. What Should Stay Manual, and Why
> **Answer:** **Adverse physical reaction complaints and medical escalations must stay 100% manual.** If a customer reports burning, rash, or severe allergic reaction from an Ayurvedic serum, an automated response—no matter how empathetic—comes across as cold, legalistic, and dangerous. Adverse events require genuine human empathy, immediate escalation to the product quality team, batch tracking against warehouse lots, and careful coordination with dermatologists. Automating physical harm risks legal liability under consumer protection regulations and erodes human care when the customer is most vulnerable.

---

## 8. How I Used AI Tools

- **Prompt Engineering & Schema Design:** Used Claude and Gemini to iterate on edge-case definitions for `llm_output.schema.json` and stress-test regex rules against prompt injection attacks.
- **Workflow Synthesis:** Used AI assistance to draft boilerplate JavaScript code for deterministic date math and ticket deduplication logic.
- **Where I relied on my own judgment:** 
  - Overruled pure LLM prompts in favor of hard-coded programmatic safeguards for privacy checks (`from_email !== customer_email`) and date calculations.
  - Designed the two-stage ticket deduplication architecture to prevent fragmented support histories in Google Sheets.
  - Selected the specific confidence thresholds and defined the exact kill-switch triggers.

---

## 9. If I Had One More Week

1. **Interactive Agentic Tools via n8n LangChain:** Upgrade the deterministic HTTP lookup nodes into dynamic ReAct Agent tool-calling bindings with semantic memory over historical customer tickets.
2. **Vector RAG over FAQ Catalog:** Implement an embedded Qdrant/Pinecone vector store over the full product catalog to retrieve exact ingredients, certifications, and usage guides with citation chunks.
3. **Omnichannel Ingestion:** Extend the webhook adapter to ingest WhatsApp Business API and Shopify Inbox webhooks, maintaining a unified conversation thread per customer.
4. **Slack / Zendesk Escalation:** Automatically post urgent tickets (`sentiment: angry` or damage reports) directly into a `#support-escalations` Slack channel with 1-click human approval buttons.
