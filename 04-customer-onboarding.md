# Customer Onboarding Automation

**Price: $25 on Gumroad**  
**Category: Operations / Customer Success**  
**Complexity: Intermediate (12 nodes)**  
**Runtime: ~5 seconds per new signup**

---

## What This Workflow Does

When a new customer signs up (via webhook from your app, Stripe checkout, or a form), this workflow kicks off a complete onboarding sequence:

1. **Receives the signup webhook** with customer name, email, plan, and company
2. **Creates a Notion database entry** in your "Customers" database with all signup data
3. **Creates onboarding tasks** in Notion (or Todoist/Asana) for your team: "Send welcome kit", "Schedule kickoff call", "Set up account"
4. **Sends a personalized welcome email** via Gmail with next steps and resource links
5. **Adds the customer to your Mailchimp list** (or any email marketing platform) for the onboarding drip sequence
6. **Schedules a follow-up reminder** — creates a Google Calendar event for Day 3 check-in
7. **Sends a Slack notification** to #customer-success with customer details and a link to their Notion page
8. **Waits 3 days** and sends an automated follow-up email checking in on their progress

---

## Nodes Used (in execution order)

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | **Webhook** | `n8n-nodes-base.webhook` | Receives signup event from your app or Stripe |
| 2 | **Validate Signup** | `n8n-nodes-base.if` | Ensures email + plan are present |
| 3 | **Format Customer Data** | `n8n-nodes-base.set` | Normalizes fields, adds timestamp, assigns onboarding rep |
| 4 | **Notion Create Page** | `n8n-nodes-base.notion` | Creates a new page in your Customers database |
| 5 | **Notion Create Tasks** | `n8n-nodes-base.notion` | Creates 3 tasks in your Tasks database linked to the customer |
| 6 | **Gmail Send Welcome** | `n8n-nodes-base.gmail` | Sends HTML welcome email with getting-started guide |
| 7 | **Mailchimp Add Subscriber** | `n8n-nodes-base.mailchimp` | Adds customer to "Onboarding" audience/tag |
| 8 | **Google Calendar Event** | `n8n-nodes-base.googleCalendar` | Creates "Day 3 Check-in" event for the account rep |
| 9 | **Slack Notification** | `n8n-nodes-base.slack` | Posts to #customer-success with customer summary |
| 10 | **Wait** | `n8n-nodes-base.wait` | Pauses workflow for 3 days |
| 11 | **Gmail Send Follow-up** | `n8n-nodes-base.gmail` | Sends automated check-in email |
| 12 | **Respond to Webhook** | `n8n-nodes-base.respondToWebhook` | Returns success confirmation to your app |

---

## Required Credentials

| Service | Credential Name in n8n | Setup Time | Notes |
|---------|----------------------|------------|-------|
| Notion | Notion API (Internal Integration) | 10 min | Requires creating integration + sharing DBs |
| Gmail | Gmail OAuth2 | 3 min | For sending emails |
| Mailchimp | Mailchimp API | 3 min | API key from Account → Extras → API keys |
| Google Calendar | Google Calendar OAuth2 | 3 min | Same OAuth as Gmail usually |
| Slack | Slack OAuth2 Bot Token | 3 min | For notifications |

---

## Setup Instructions

### Step 1: Notion Setup
1. Create a **Customers** database in Notion with these properties:
   - Name (title), Email (email), Company (text), Plan (select: Starter/Pro/Enterprise), Status (select: Onboarding/Active/Churned), Onboarding Rep (text), Signup Date (date), Notion Page ID (text)
2. Create a **Tasks** database with:
   - Task Name (title), Assigned To (person), Customer (relation → Customers DB), Due Date (date), Status (select: To Do/In Progress/Done), Type (select: Welcome Kit/Kickoff Call/Account Setup)
3. Create a Notion Internal Integration at notion.so/my-integrations
4. Share BOTH databases with the integration (click "..." → Connections → Add integration)
5. Copy the Integration Token → create n8n Notion credential

### Step 2: Mailchimp Setup
1. In Mailchimp, create an Audience for customers
2. Create a Tag called "Onboarding" (this triggers your onboarding automation/drip)
3. Get API key from Account → Extras → API keys
4. Find your Audience ID from Audience → Settings → Audience name & defaults

### Step 3: Configure the Workflow
1. Import `04-customer-onboarding.json`
2. Set the Webhook path (default: `/customer-signup`)
3. Paste your Notion database IDs into both Notion nodes
4. Configure the welcome email template (HTML node)
5. Set the Mailchimp Audience ID
6. Set the Calendar event duration and default attendee (your team's calendar)

### Step 4: Webhook Payload Format
Your app should POST to the webhook URL with this JSON:
```json
{
  "name": "Jane Smith",
  "email": "jane@acmecorp.com",
  "company": "Acme Corp",
  "plan": "Pro",
  "signup_source": "website",
  "onboarding_rep": "alex@yourcompany.com"
}
```

---

## Error Handling

- **Notion API rate limit**: 3 requests/second. The workflow has built-in spacing between the two Notion nodes. If you hit the limit, add a **Wait** node (3 seconds) between them
- **Mailchimp already subscribed**: The Mailchimp node returns an error for existing subscribers. Add an **Error Trigger** branch that checks for "Member Exists" and continues gracefully
- **Google Calendar conflict**: If the onboarding rep has a conflict at the default time, the event is still created. Consider adding a **Code** node that finds the next available slot
- **3-day wait timeout**: The **Wait** node holds the execution for exactly 72 hours. If your n8n instance restarts during this period, the workflow resumes correctly (execution state is persisted)
- **Gmail sending limits**: 500 emails/day on free Google Workspace. If you exceed this, queue the remaining emails

---

## Customization Ideas

- **Replace Notion with HubSpot**: Swap Notion nodes for HubSpot Deal + Task nodes
- **Add Intercom conversation**: Use the **HTTP Request** node to create an Intercom conversation for high-value plans
- **Dynamic onboarding by plan**: Use a **Switch** node after validation — different task sets for Starter vs. Enterprise
- **Add Calendly link**: Include the rep's Calendly link in the welcome email for self-scheduling
- **SMS welcome**: Add a **Twilio** node to send a welcome SMS (especially good for B2C)
- **Loom video**: Include a personalized Loom video link in the welcome email (pre-recorded by plan)
- **Stripe webhook**: Connect directly to Stripe's `checkout.session.completed` webhook instead of a custom endpoint

---

## FAQ

**Q: Why Notion instead of a CRM?**  
A: Notion is free for small teams and very flexible. This workflow works as a lightweight CRM. Swap Notion for HubSpot/Salesforce/Pipedrive if you already use one.

**Q: What happens if the customer signs up on a weekend?**  
A: The Day 3 check-in still fires. To skip weekends, add a **Code** node before the Wait node that calculates the next business day.

**Q: Can I add more follow-up emails?**  
A: Yes — duplicate the Wait + Gmail pair. Day 7, Day 14, Day 30 are common check-in points.
