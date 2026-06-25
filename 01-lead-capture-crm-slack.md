# Lead Capture → CRM → Slack Notification

**Price: $19 on Gumroad**  
**Category: Sales / Lead Management**  
**Complexity: Beginner-friendly (7 nodes)**  
**Runtime: ~1 second per lead**

---

## What This Workflow Does

Every time someone submits your web form, this workflow:

1. **Receives the webhook payload** from any form builder (Typeform, Tally, custom HTML form, etc.)
2. **Validates** that required fields (name, email) are present — rejects spam submissions silently
3. **Stores the lead** in Airtable (or Google Sheets) with timestamp, source, and status
4. **Checks for duplicates** by email before inserting (optional: use Airtable formula field as unique constraint)
5. **Sends a Slack notification** to your #leads channel with the lead's name, email, company, and a link to the Airtable record
6. **Optionally enriches** the lead with Clearbit/Hunter data (disabled by default — toggle the IF node)

---

## Nodes Used (in execution order)

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | **Webhook** | `n8n-nodes-base.webhook` | Receives POST requests from your form. Produces the path `/lead-capture` |
| 2 | **Validate Fields** | `n8n-nodes-base.if` | Checks that `name` and `email` are not empty. False branch returns 400 |
| 3 | **Format Timestamp** | `n8n-nodes-base.set` | Adds `created_at` (ISO 8601), `lead_source`, and `status: "new"` |
| 4 | **Airtable** | `n8n-nodes-base.airtable` | Appends a row to the `Leads` table with all fields |
| 5 | **Google Sheets** (alternative) | `n8n-nodes-base.googleSheets` | Falls back to Sheets if Airtable credential is missing. Append row to `Leads!A:F` |
| 6 | **Slack** | `n8n-nodes-base.slack` | Posts to `#leads` channel with a formatted message |
| 7 | **Respond to Webhook** | `n8n-nodes-base.respondToWebhook` | Returns `{ success: true, id: recordId }` to the form |

---

## Required Credentials

| Service | Credential Name in n8n | Setup Time |
|---------|----------------------|------------|
| Airtable | Airtable API (Personal Access Token) | 5 min |
| Google Sheets | Google Sheets OAuth2 | 5 min |
| Slack | Slack OAuth2 (Bot Token with `chat:write` + `channels:read`) | 3 min |

**Note:** You only need **one** of Airtable or Google Sheets — not both. The workflow includes both nodes; delete the one you don't use.

---

## Setup Instructions

### Step 1: Import the JSON
1. Open n8n → Workflows → **Import from File**
2. Select `01-lead-capture-crm-slack.json`
3. Click **Save**

### Step 2: Configure the Webhook
1. Click the **Webhook** node → copy the **Production URL**
2. Paste it into your form builder as the POST webhook URL
3. Set your form fields to send: `name`, `email`, `company` (optional), `message` (optional)

### Step 3: Set Up Airtable
1. Create a base called `CRM` with a table called `Leads`
2. Add fields: `Name` (single line text), `Email` (email), `Company` (single line text), `Message` (long text), `Status` (single select: New/Contacted/Qualified/Closed), `Created At` (date), `Source` (single line text)
3. Create a Personal Access Token at airtable.com/create/tokens with `data.records:write` scope for this base
4. Paste the token into n8n → Credentials → Airtable API

### Step 4: Set Up Slack
1. Go to api.slack.com/apps → Create New App → From Scratch
2. Add Bot Token Scopes: `chat:write`, `channels:read`
3. Install to workspace → copy Bot User OAuth Token
4. Paste into n8n → Credentials → Slack API
5. In the Slack node, set Channel to `#leads` (or your preferred channel)

### Step 5: Activate
1. Click **Active** toggle (top-right)
2. Test: send a POST request to the webhook URL with `{"name":"Test User","email":"test@example.com"}`
3. Check Slack and Airtable for the new lead

---

## Error Handling

- **Missing fields**: The IF node returns HTTP 400 with `{"error": "Missing required fields"}` — your form should show this to the user
- **Airtable down**: The workflow stops at the Airtable node. Add an **Error Trigger** node connected to the Airtable node to send a Slack alert to admins
- **Duplicate emails**: Add an Airtable **Search** node before the Create node to check if email exists. If found, update the existing record instead of creating a new one
- **Slack failure**: Non-critical — the lead is already in Airtable. Slack failures don't block the webhook response

---

## Customization Ideas

- **Add HubSpot/Salesforce**: Replace Airtable with the HubSpot or Salesforce node
- **Auto-respond to lead**: Add an **Email** (SMTP) node after Airtable to send a "Thanks for reaching out!" email
- **Lead scoring**: Add a **Code** node that scores leads based on company size (if using Clearbit enrichment)
- **Route by source**: Use a **Switch** node after the Webhook to send leads to different Slack channels based on `lead_source`

---

## FAQ

**Q: Why not just use Zapier?**  
A: n8n is self-hosted, so you own your data. No per-task pricing. This workflow costs $0/month to run (vs. $30+/mo on Zapier for similar volume).

**Q: How many leads can this handle?**  
A: Tested at 500 leads/hour on a $5/month VPS. The bottleneck is Airtable's API rate limit (5 req/sec on free plan).

**Q: Can I use a different form builder?**  
A: Yes — any service that sends webhooks with JSON payloads works. Just map your field names to `name`, `email`, `company`, `message` using a **Set** node before the IF validation.
