# Task 4 — AI Customer Support Ticket Automation

n8n workflow that monitors a Gmail support inbox, classifies incoming emails using
rule-based logic, logs them as tickets in Google Sheets, and sends the customer an
automatic acknowledgment email.

## Actual architecture used (and why)

| Spec suggestion | What this submission uses | Reason |
|---|---|---|
| Airtable / Notion / NocoDB | **Google Sheets** | Same Google account already used for Gmail — zero extra signup, still satisfies "structured storage" |
| AI/LLM classification (OpenAI/Claude/Gemini) | **Rule-based classification (JavaScript Code node)** | Gemini and OpenRouter free-tier keys hit persistent quota/rate-limit errors during the 72-hour window; rule-based keyword matching gives 100% reliable, reproducible results with zero API cost or downtime risk. Swapping in a real LLM later is a single-node change (see "Upgrading to a real LLM" below). |

## Workflow

```
Gmail Trigger (poll every 1 min)
        │
        ▼
Extract Email Metadata   (Code node — parses sender name/email, subject, body)
        │
        ▼
Rule Based Ticket Classify   (Code node — category, priority, sentiment, department, tags)
        │
        ▼
Create Ticket in Google Sheets   (Append Row)
        │
        ▼
Send Acknowledgment Email   (Gmail)
```

## Files in this submission

- `workflow.json` — importable n8n workflow (5 nodes)
- `DATABASE_SCHEMA.md` — Google Sheet column definitions
- `sample_emails.json` — 10 test emails covering every category + 2 edge cases
- `README.md` — this file

## Setup instructions

### 1. Create the Google Sheet
1. Go to https://sheets.new
2. Name it **Support Tickets** (or anything — you'll select it in n8n by picker, not by name)
3. In row 1, add these exact column headers (see `DATABASE_SCHEMA.md`):
   ```
   customer_name | company | sender_email | issue_summary | detailed_description | category | priority | sentiment | product_service | suggested_department | suggested_tags | confidence_score | received_at
   ```

### 2. Import the workflow
1. In n8n: **Workflows → Add Workflow → Import from File**
2. Upload `workflow.json`
3. Two nodes will show credential warnings (Gmail nodes, Google Sheets node) — expected

### 3. Connect Gmail credential
1. Open **Gmail Trigger - Monitor Support Inbox** → Credential → Create New → "Sign in with Google"
2. Choose the Gmail account you want to use as the support inbox → Allow
3. Open **Send Acknowledgment Email** → select the same Gmail credential

### 4. Connect Google Sheets credential
1. Open **Create Ticket in Google Sheets** → Credential → Create New → sign in with the **same** Google account
2. Set **Document** to your "Support Tickets" spreadsheet
3. Set **Sheet** to the correct tab (usually the first/default tab)
4. Mapping Mode is already set to **"Map Automatically"** — it matches columns by name, so as long as your sheet headers match the schema exactly, no manual mapping is needed

### 5. Activate and test
1. Toggle the workflow **Active**
2. Send any email from `sample_emails.json` to your support Gmail address
3. Within ~1 minute, check:
   - n8n **Executions** tab — all 5 nodes should show green
   - The Google Sheet — a new row should appear with correct category/priority/sentiment
   - The sender's inbox — an acknowledgment email should arrive

## AI / classification logic

The `Rule Based Ticket Classify` node uses keyword pattern matching against the
email subject + body to determine:

- **Category** (Technical Support, Billing, Sales Inquiry, Feature Request, Bug
  Report, Account Access, Refund Request, General Inquiry) — checked in priority
  order, first match wins
- **Priority** (Critical, High, Medium, Low) — based on urgency keywords
  (e.g. "urgent", "asap", "immediately" → Critical)
- **Sentiment** (Positive, Neutral, Negative) — based on emotional language
- **Suggested department** — a fixed lookup table mapping each category to a team
  (e.g. Billing → Finance, Bug Report → Technical Support)
- **Tags** — simple keyword-based tagging (mobile, web, payment, or general)
- **Confidence score** — fixed at 0.75 since this is deterministic rule-based logic,
  not a probabilistic model

The full matching logic is visible directly in the `Rule Based Ticket Classify`
node's JavaScript code inside `workflow.json`.

## Manual agent review

As specified, manual review/status changes are done directly in the Google Sheet
(no automation needed here, same as the spec's Airtable-based examples would work).
To demonstrate this: open the sheet, and manually add notes or change values on
any ticket row after it's created.

## Error handling

- If sender name/email can't be parsed from the `From` header, the code falls back
  to `"Unknown"` rather than throwing an error
- If subject/body is empty, safe fallback strings (`"(no subject)"`, empty string)
  are used
- Since classification is rule-based (not an external AI call), there is no
  possibility of malformed-JSON parsing failures or API timeouts — this was a
  deliberate reliability trade-off given the free-tier AI API quota issues
  encountered during development

## Upgrading to a real LLM (future work)

To swap in a real AI model instead of rule-based logic:
1. Replace the `Rule Based Ticket Classify` Code node with an HTTP Request node
   calling an LLM provider (OpenRouter, Gemini, OpenAI, Claude, etc.)
2. Keep the same output field names (`category`, `priority`, `sentiment`,
   `suggested_department`, `suggested_tags`, `confidence_score`) so the rest of
   the workflow (Google Sheets, acknowledgment email) requires no changes
3. Add a validation/fallback Code node after the AI call to handle malformed
   JSON responses and rate-limit errors gracefully

## Assumptions

- One Google account is used for both Gmail (trigger + send) and Google Sheets
  (storage), for simplicity
- Attachments are not parsed/stored — the spec only requires classification and
  ticketing based on email content
- Ticket IDs are implicit (the Google Sheets row itself); no separate ticket ID
  field is generated
- Status/lifecycle tracking (Open → In Progress → Resolved) is handled manually
  by editing the sheet directly, not automated
