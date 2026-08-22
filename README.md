# AI Lead Triage & Routing System

An n8n automation that ingests inbound leads, uses a free LLM (via Groq) to analyze and score them, logs every lead to Google Sheets, and routes Hot leads to instant Slack + drafted email follow-up, while Cold leads go to a separate nurture list.

## Why this project

Manually triaging inbound leads doesn't scale — someone has to read every message, guess intent and budget, and decide who gets a call today versus who gets a "maybe later." This workflow automates that first-pass judgment call, so a sales team's time goes to the leads most likely to convert, while nothing falls through the cracks.

## Architecture

```
Webhook (trigger)
   ↓
Groq LLM — sentiment / budget / urgency / lead_score analysis
   ↓
Code node — parses AI's JSON string into structured fields
   ↓
Google Sheets — logs every lead + AI analysis (master log)
   ↓
Switch node — routes by lead_score
   ├── Hot  → Slack alert + Gmail draft (instant human follow-up)
   ├── Warm → (currently logged only; earmarked for a 3-day follow-up reminder)
   └── Cold → Google Sheets (separate "Nurture List" tab)
```

## Key design decisions

**Webhook over Form Trigger.**
A Form Trigger is easier to demo but n8n-native. A Webhook mirrors how real lead sources — website forms, ad platforms, CRMs — actually integrate with automation tools, so it's the more representative choice for a portfolio piece.

**Groq instead of OpenAI.**
n8n's built-in OpenAI node supports a custom Base URL, so it can be pointed at any OpenAI-compatible API. Groq's free tier runs open-weight models (`openai/gpt-oss-20b`) on their own inference hardware at very low latency, at no cost — a good fit for a workflow that needs to score leads in real time without per-call billing.

**Structured JSON output via prompt constraints.**
Rather than letting the model return free text, the prompt asks for a strict JSON schema (sentiment, budget_mentioned, estimated_budget, urgency, lead_score, reasoning) with explicit scoring rules for Hot/Warm/Cold. This makes the output deterministic enough for downstream automation to consume reliably, instead of parsing arbitrary prose.

**A Code node to parse the AI's response.**
The AI returns a JSON *string*, not a native object n8n can map field-by-field. A small JavaScript node parses that string and re-attaches the original webhook fields (name, email, message, source), producing one flat object that both Google Sheets and the Switch node can consume directly.

**Switch node over IF.**
An IF node only branches two ways. Since leads fall into three tiers (Hot/Warm/Cold), a Switch node keeps the routing logic explicit and makes it trivial to add a distinct Warm path later, rather than nesting IF nodes.

**Draft email, not auto-send.**
For Hot leads, the workflow creates a Gmail *draft* rather than sending automatically. AI-scored "Hot" is a strong signal, not a certainty — a human should review a personalized reply before it goes out. This also avoids the reputational and correctness risk of an LLM sending unreviewed messages on someone's behalf.

**Separate sheet tab for Cold leads.**
Every lead is logged to a master sheet regardless of score, so nothing is lost. Cold leads are *also* appended to a separate "Nurture List" tab — giving sales a filtered view of who to revisit later, without cluttering the primary hot-lead log.

## Setup

1. **Groq API key** — create one free at [console.groq.com](https://console.groq.com), no card required.
2. **n8n OpenAI-node credential** — set the Base URL to `https://api.groq.com/openai/v1` and paste the Groq key as the API key.
3. **Model** — `openai/gpt-oss-20b` (Groq deprecated the older Llama chat models; check [Groq's model docs](https://console.groq.com/docs/models) for the current recommended list).
4. **Google Sheets** — one spreadsheet, two tabs: a main log and a `Nurture List`, both with headers: `name, email, message, source, sentiment, budget_mentioned, estimated_budget, urgency, lead_score, reasoning`.
5. **Slack / Gmail** — connect via OAuth in their respective n8n node credentials.

## Known issues worth knowing about (and how they were fixed)

- **`store` parameter rejected by Groq** — n8n's OpenAI node sends `store: true` by default; Groq only accepts `false`/`null`. Fixed by adding `Store: false` under the node's Options.
- **Stale column mapping after editing sheet headers** — n8n caches column names at node-setup time; editing headers afterward requires refreshing the "Column to Match On" field in the Sheets node.
- **Hidden whitespace in sheet headers** — a trailing/leading space in a column name (e.g. `"MESSAGE "`) silently breaks simple `{{ $json.fieldName }}` expressions. Bracket notation (`{{ $json["MESSAGE "] }}`) works around it, but the real fix is retyping the header cleanly.

## Possible next steps

- Give the Warm branch its own action (e.g., a delayed follow-up reminder).
- Switch the Webhook from Test URL to Production URL for live use.
- Add a header-based shared secret to the Webhook for basic authentication.
  ___
  # AI Customer Support Ticket Categorizer & Auto-Responder

An end-to-end automation workflow built in **n8n** that ingests customer support tickets, classifies them using an LLM, logs them for tracking, and triggers real-time alerts for urgent cases.

## What it does

1. **Receives** a support ticket (name, email, message) via a webhook — simulating a form, helpdesk, or CRM submission.
2. **Classifies** the ticket using an LLM (Groq's `openai/gpt-oss-20b` model, accessed through Groq's free, OpenAI-compatible API) into:
   - **Category**: Billing, Technical, or General
   - **Urgency**: High, Medium, or Low
3. **Generates** a draft customer-facing reply automatically, so a human agent can review and send it rather than write from scratch.
4. **Logs** every ticket — name, email, message, category, urgency, and draft reply — to a Google Sheet for tracking and reporting.
5. **Routes** High-urgency tickets down a separate path that immediately emails a real-time alert (via Gmail), so nothing urgent sits unnoticed in a queue.

## Architecture

```
Webhook (POST) 
   → LLM Node (Groq API — classification + draft reply) 
      → Code Node (parses AI's JSON response into structured fields) 
         → Google Sheets (logs every ticket) 
         → IF Node (urgency == High?) 
               → True: Gmail (sends real-time alert)
               → False: (ticket already logged, no further action)
```

## Tech stack

- **n8n** — workflow orchestration (webhook, logic, integrations)
- **Groq API** (`openai/gpt-oss-20b`) — fast, free-tier LLM inference for classification and draft generation, called via n8n's OpenAI-compatible AI node
- **JavaScript (n8n Code node)** — parses and reshapes the LLM's raw JSON output into clean fields for downstream use
- **Google Sheets API** — persistent ticket log
- **Gmail API** — real-time urgent-ticket alerting

## Example flow

**Input (POST body):**
```json
{
  "name": "Jane Smith",
  "email": "jane.smith@example.com",
  "message": "I have been locked out of my account for hours and I am losing money every minute, please help immediately!"
}
```

**AI classification output:**
```json
{
  "category": "Technical",
  "urgency": "High",
  "draft_response": "Hi Jane, I'm sorry to hear you're locked out of your account. I'm escalating this immediately — could you confirm the email or username on the account so we can restore access as quickly as possible?"
}
```

**Result:** Row logged to Google Sheets, and an alert email is sent within seconds containing the customer's details, original message, and the AI-drafted reply — ready for a support agent to approve and send.

## What I learned building this

- Wiring together a webhook trigger, an LLM classification step, data transformation, and multi-branch conditional logic into a single production-style automation.
- Working with **OpenAI-compatible third-party APIs** (Groq) inside a tool built primarily around OpenAI's own API — including handling provider-specific quirks (unsupported parameters, model deprecations) that don't show up in the happy-path tutorials.
- Parsing unstructured LLM text output into structured, reliable fields using a Code node, rather than trusting the model's raw output downstream.
- Debugging real n8n error classes: expression syntax errors, field name mismatches (case sensitivity, spaces in keys), and conditional-branch routing logic.
- The importance of **verifying with real, live test data** — not just trusting a "success" status without checking the actual downstream systems (inbox, spreadsheet) it claims to have affected.

## Potential next steps

- Add Header Auth (a shared secret key) to the webhook to prevent unauthorized/public submissions.
- Support multi-channel intake (email parsing, Slack, or a simple web form) instead of only raw webhook POSTs.
- Add a Slack alert branch alongside Gmail for teams that triage in Slack.
- Store the AI's draft reply back into a "ready to send" queue with one-click approval, rather than just logging it.
___
# RSS to AI Summary Email — n8n Workflow

An automated n8n workflow that pulls the latest articles from an RSS feed, summarizes each one using Google Gemini AI, and emails the summaries via Gmail — fully hands-off, on a daily schedule.

## What It Does

Every day at a scheduled time, this workflow:

1. **Wakes up automatically** via a Schedule Trigger
2. **Fetches articles** from a chosen RSS feed
3. **Limits the batch** to the 5 most recent articles (to stay within free AI API rate limits)
4. **Summarizes each article** into 3 short, plain-English bullet points using Google Gemini
5. **Emails the summary** to a chosen inbox via Gmail, with the real article title in the subject line

## Workflow Structure

```
Schedule Trigger → RSS Read → Limit (5) → Message a model (Gemini) → Send a message (Gmail)
```

| Node | Type | Purpose |
|---|---|---|
| Schedule Trigger | Trigger | Runs the workflow automatically at a set time each day |
| RSS Read | RSS Feed Read | Pulls articles (title, link, content, publish date) from the feed URL |
| Limit | Limit | Caps the batch to 5 items so the AI step doesn't hit rate limits |
| Message a model | Google Gemini | Sends each article's content to Gemini and returns a 3-bullet-point summary |
| Send a message | Gmail | Sends one email per article with the summary as the body |

## Setup

### 1. RSS Feed
- Node: **RSS Read**
- Field: **URL** — set to your feed of choice (e.g. `https://www.nasa.gov/feed/`)

### 2. Google Gemini (free tier)
- Get a free API key at [aistudio.google.com/apikey](https://aistudio.google.com/apikey)
- In the **Message a model** node, create a credential and paste the key in
- Model used: `models/gemini-3-flash-preview`
- Prompt used:
  ```
  Summarize the following news article in 3 short bullet points, written in plain simple English:

  {{ $json["content:encoded"] }}
  ```

### 3. Gmail
- In the **Send a message** node, connect your Gmail account (sign in with Google)
- Fields:
  - **To**: your recipient email address
  - **Subject**: `Article Summary: {{ $('RSS Read').item.json.title }}` *(set this field to "Expression" mode, not "Fixed")*
  - **Message**: `{{ $json.content.parts[0].text }}` *(pulls the AI-generated summary)*

### 4. Schedule
- Node: **Schedule Trigger**
- Set **Trigger Interval** to `Days`, and set the **Hour** (and optional Minute) for when you want it to run daily
- Connect this node's output into **RSS Read**

### 5. Activate
- Save the workflow
- Toggle the workflow to **Active** (top-right of the n8n editor)

## Notes & Gotchas

- **Free-tier rate limits**: Google Gemini's free tier allows a limited number of requests per minute (around 5 for the flash model). Sending too many articles at once will trigger a "too many requests" error. The **Limit** node keeps this in check.
- **Expression vs. Fixed fields**: Any field using `{{ }}` syntax to pull in dynamic data (like the article title in the subject line) must be switched to **Expression** mode in n8n, not left as **Fixed** (plain text) — otherwise it will literally show the raw code instead of the real value.
- **One email per article**: Since Limit is set to 5, this sends 5 separate emails per run. To combine them into a single daily digest instead, add a **Merge**/aggregation step before the Send a message node.

## Possible Next Steps

- Combine all summaries into a single digest email instead of 5 separate ones
- Pull from multiple RSS feeds at once
- Swap in a different topic/feed by just changing the URL in **RSS Read**
- Add an error-notification workflow so you're alerted if a run fails
