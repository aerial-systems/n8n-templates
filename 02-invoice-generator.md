# Invoice Generator from Google Sheets

**Price: $22 on Gumroad**  
**Category: Finance / Operations**  
**Complexity: Intermediate (9 nodes)**  
**Runtime: ~2 seconds per invoice**

---

## What This Workflow Does

This workflow turns a Google Sheet of unbilled line items into professional PDF invoices and emails them to clients — all automated. You maintain a spreadsheet; n8n handles the rest.

1. **Reads a Google Sheet** where you track billable work (client, hours, rate, description, status)
2. **Filters** rows where `status = "unbilled"` — only processes new items
3. **Groups line items by client** — one invoice per client, even if they have multiple rows
4. **Calculates totals** (subtotal per line item, invoice total, tax if applicable)
5. **Generates a PDF invoice** using an HTML template with your logo, line items, and payment terms
6. **Emails the PDF** to the client via Gmail SMTP (or any SMTP provider)
7. **Updates the sheet** — marks rows as `status = "invoiced"` and adds invoice number + date
8. **Saves a copy** of the PDF to Google Drive for your records

---

## Nodes Used (in execution order)

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | **Schedule Trigger** | `n8n-nodes-base.scheduleTrigger` | Runs daily at 9 AM or manually via "Execute Workflow" |
| 2 | **Google Sheets Read** | `n8n-nodes-base.googleSheets` | Reads all rows from `Invoices` sheet |
| 3 | **Filter Unbilled** | `n8n-nodes-base.if` | Keeps only rows where `status == "unbilled"` |
| 4 | **Item Lists** | `n8n-nodes-base.itemLists` | Groups rows by `client_email` so each client gets one invoice |
| 5 | **Code (Calculate)** | `n8n-nodes-base.code` | Pure JavaScript: loops through grouped items, calculates subtotals, tax, grand total |
| 6 | **HTML Template** | `n8n-nodes-base.html` | Renders the invoice HTML with dynamic data using n8n's template engine |
| 7 | **Convert to PDF** | `n8n-nodes-base.convertToFile` | Converts HTML output to a PDF file (uses headless Chrome) |
| 8 | **Gmail Send** | `n8n-nodes-base.gmail` | Sends the PDF as an email attachment to the client |
| 9 | **Google Sheets Update** | `n8n-nodes-base.googleSheets` | Writes back `status = "invoiced"`, invoice number, and sent date |

---

## Required Credentials

| Service | Credential Name in n8n | Setup Time |
|---------|----------------------|------------|
| Google Sheets | Google Sheets OAuth2 | 5 min |
| Gmail | Gmail OAuth2 | 3 min |
| Google Drive (optional) | Google Drive OAuth2 | 2 min |

---

## Spreadsheet Structure

Your Google Sheet must have a tab named `Invoices` with these columns:

| Column | Header | Example | Notes |
|--------|--------|---------|-------|
| A | `client_name` | Acme Corp | Company or contact name |
| B | `client_email` | billing@acme.com | Where invoice PDFs go |
| C | `description` | Website redesign - Phase 1 | Line item description |
| D | `quantity` | 12.5 | Hours or units |
| E | `rate` | 95.00 | Hourly rate or unit price |
| F | `status` | unbilled | unbilled / invoiced / paid |
| G | `invoice_number` | INV-2024-0042 | Leave blank; auto-filled |
| H | `invoice_date` | 2024-08-15 | Leave blank; auto-filled |
| I | `due_date` | 2024-09-14 | Optional: Net-30 default |
| J | `notes` | Priority project | Optional: appears on invoice |

---

## Setup Instructions

### Step 1: Create the Google Sheet
1. Create a new Google Sheet called `Invoice Tracker`
2. Add a tab called `Invoices` with the columns listed above
3. Add 2-3 test rows with `status = "unbilled"`
4. Copy the Sheet ID from the URL: `https://docs.google.com/spreadsheets/d/{SHEET_ID}/edit`

### Step 2: Import & Configure
1. Import `02-invoice-generator.json` into n8n
2. In the **Google Sheets** nodes, paste your Sheet ID
3. In the **Schedule Trigger**, set your preferred time (default: 9 AM daily)

### Step 3: Customize the HTML Template
1. Open the **HTML Template** node
2. Replace `YOUR_LOGO_URL` with your company logo (hosted image URL)
3. Replace `YOUR_COMPANY_NAME`, `YOUR_ADDRESS`, payment terms, and bank details
4. Adjust colors in the `<style>` block to match your brand

### Step 4: Test
1. Click **Execute Workflow** manually
2. Check the target email inbox for the test invoice PDF
3. Verify the Google Sheet rows were updated to `invoiced`

---

## Error Handling

- **Empty sheet / no unbilled rows**: The IF node's false branch exits silently. No emails sent, no errors.
- **Invalid email**: Gmail node will fail. Add an **Error Trigger** → **Slack** node to notify you of bounce-backs
- **PDF generation fails**: `convertToFile` needs headless Chrome available on your n8n instance. If you're on n8n cloud, this works out of the box. On self-hosted Docker, ensure Chrome is installed
- **Sheet structure mismatch**: If columns are renamed, the Filter and Update nodes will fail. Keep the column mapping consistent
- **Rate limiting**: Gmail SMTP has a 500 emails/day limit on free Google accounts. For higher volume, use SendGrid or Postmark

---

## Customization Ideas

- **Add Stripe Payment Link**: After generating the invoice, add a **Stripe** node to create a Payment Link and include it in the email
- **Tax calculation**: Modify the **Code** node to look up tax rates by client country/state
- **Recurring invoices**: Change the Schedule Trigger to run weekly/monthly for retainer clients
- **Multi-currency**: Add a currency column and format amounts with proper symbols in the HTML template
- **Client Portal**: Instead of emailing, upload the PDF to a shared Google Drive folder and notify via Slack

---

## HTML Invoice Template (Preview)

The included template produces a clean, print-ready invoice with:
- Your logo and company info at top
- "INVOICE" header with number and dates
- Line item table (description, qty, rate, amount)
- Subtotal / Tax / Total section
- Payment terms and bank/wire details in footer
- Professional fonts (Inter, system fallbacks)

---

## FAQ

**Q: Can I use this for non-hourly billing?**  
A: Yes — set `quantity = 1` and `rate = fixed_price`. The math still works.

**Q: How do I handle clients with different tax rates?**  
A: Add a `tax_rate` column to your sheet and reference it in the Code node: `item.tax_rate || 0`.

**Q: What if Gmail blocks "less secure apps"?**  
A: Use OAuth2 (not app passwords). The Gmail node in n8n uses OAuth2 by default — it won't be blocked.
