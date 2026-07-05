# Database Schema — Google Sheet "Support Tickets"

Storage backend: **Google Sheets** (single tab). Row 1 must contain the exact
column headers below — the n8n "Create Ticket in Google Sheets" node uses
**Map Automatically**, which matches incoming JSON fields to sheet columns by
name, so header spelling must match exactly.

| Column Name | Type (as used) | Description |
|---|---|---|
| `customer_name` | Text | Sender's name, extracted from the email's `From` header |
| `company` | Text | Organization name (left blank — not extracted from email in this build) |
| `sender_email` | Text | Sender's email address, extracted from the `From` header |
| `issue_summary` | Text | First 100 characters of the email subject |
| `detailed_description` | Text | Full email body |
| `category` | Text | One of: Technical Support / Billing / Sales Inquiry / Feature Request / Bug Report / Account Access / Refund Request / General Inquiry |
| `priority` | Text | One of: Critical / High / Medium / Low |
| `sentiment` | Text | One of: Positive / Neutral / Negative |
| `product_service` | Text | Related product/service (left blank — not extracted in this build) |
| `suggested_department` | Text | Team assignment derived from category via fixed lookup table |
| `suggested_tags` | Array/Text | Keyword-based tags (e.g. mobile, web, payment, general) |
| `confidence_score` | Number | Fixed at 0.75 (rule-based classification, not probabilistic) |
| `received_at` | Date/time (ISO string) | Timestamp the email was received, from Gmail's `Date` header |

## Notes

- **Audit trail**: Google Sheets' built-in **Version History** (File → Version
  history → See version history) satisfies the audit trail requirement without
  extra engineering.
- **Department mapping** is implemented in code (see `Rule Based Ticket Classify`
  node in `workflow.json`) as a plain JavaScript object (`departmentMap`), making
  it easy to edit without touching classification logic:
  ```js
  const departmentMap = {
    'Billing': 'Finance',
    'Refund Request': 'Finance',
    'Sales Inquiry': 'Sales',
    'Feature Request': 'Product Team',
    'Bug Report': 'Technical Support',
    'Account Access': 'Customer Success',
    'Technical Support': 'Technical Support',
    'General Inquiry': 'Customer Success'
  };
  ```
- **Status / lifecycle field**: not included as an automated column in this
  build. Manual agent review and status changes are done by editing the sheet
  directly (add a `status` column yourself if you want to track Open / In
  Progress / Resolved manually).
- **Ticket ID**: not generated as a separate field — each spreadsheet row
  functions as the unique ticket record (row number can serve as an informal ID).
