# SaaS Metrics Dashboard

**Price: $25 on Gumroad**  
**Category: Analytics / SaaS Operations**  
**Complexity: Advanced (14 nodes)**  
**Runtime: ~10 seconds per weekly run**

---

## What This Workflow Does

Every Monday morning, this workflow pulls raw data from Stripe, calculates your key SaaS metrics, pushes them to a Google Sheets dashboard, and sends a Slack summary to your team. No more spreadsheets, no more manual Stripe exports.

1. **Pulls all active subscriptions** from Stripe API
2. **Pulls all invoices** from the last 30 days (for churn calculation)
3. **Calculates MRR** (Monthly Recurring Revenue) — sums active subscriptions × amount, normalized to monthly
4. **Calculates Net MRR Change** — new MRR + expansion MRR − churned MRR − contraction MRR
5. **Calculates Churn Rate** — (canceled subscriptions / starting subscriptions) × 100
6. **Calculates ARPU** — MRR ÷ active customer count
7. **Estimates LTV** — ARPU ÷ monthly churn rate (simple formula)
8. **Calculates Customer Count** — total active, new this week, churned this week
9. **Identifies top accounts** by MRR (for Slack highlight)
10. **Updates a Google Sheets dashboard** with all metrics, timestamped
11. **Sends a Slack summary** with key metrics, trends, and alerts
12. **Optionally sends email report** to stakeholders

---

## Nodes Used (in execution order)

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | **Schedule Trigger** | `n8n-nodes-base.scheduleTrigger` | Runs every Monday at 8 AM |
| 2 | **Stripe Get All** | `n8n-nodes-base.stripe` | Fetches all active subscriptions (paginated) |
| 3 | **Stripe Get All** | `n8n-nodes-base.stripe` | Fetches all invoices from last 30 days |
| 4 | **Code: Calculate MRR** | `n8n-nodes-base.code` | Sums subscription amounts, normalizes to monthly |
| 5 | **Code: Calculate Churn** | `n8n-nodes-base.code` | Compares invoices for cancellations, computes churn rate |
| 6 | **Code: Calculate LTV & ARPU** | `n8n-nodes-base.code` | Derives ARPU and estimated LTV |
| 7 | **Merge** | `n8n-nodes-base.merge` | Combines all calculated metrics into one data object |
| 8 | **Code: Format Dashboard Row** | `n8n-nodes-base.code` | Creates a single row for the Google Sheets dashboard |
| 9 | **Google Sheets Append** | `n8n-nodes-base.googleSheets` | Appends the metrics row to a running dashboard sheet |
| 10 | **Code: Delta Calculation** | `n8n-nodes-base.code` | Compares current week vs. previous week (trend arrows) |
| 11 | **Slack Message** | `n8n-nodes-base.slack` | Sends formatted summary with metrics + emoji trends |
| 12 | **IF: Alert Check** | `n8n-nodes-base.if` | Checks if churn > 5% or MRR dropped — sends alert |
| 13 | **Slack Alert** | `n8n-nodes-base.slack` | Sends urgent alert to #leadership channel |
| 14 | **Gmail Report** | `n8n-nodes-base.gmail` | Optional: emails the dashboard to stakeholders |

---

## Metrics Calculated

| Metric | Formula | Source |
|--------|---------|--------|
| **MRR** | Σ(active subscriptions × monthly_amount) | Stripe subscriptions |
| **Net New MRR** | New + Expansion − Churned − Contraction | Stripe invoices (this month vs last) |
| **Customer Count** | Count of unique active subscription customers | Stripe subscriptions |
| **New Customers** | Customers with first invoice this week | Stripe invoices |
| **Churned Customers** | Customers with canceled subscriptions this week | Stripe subscriptions (status=canceled) |
| **Monthly Churn Rate** | Churned ÷ Starting Count × 100 | Calculated |
| **ARPU** | MRR ÷ Active Customers | Calculated |
| **Estimated LTV** | ARPU ÷ Monthly Churn Rate | Calculated (naive) |
| **Top 5 Accounts** | Sorted by MRR, descending | Stripe subscriptions |

---

## Required Credentials

| Service | Credential Name in n8n | Setup Time | Notes |
|---------|----------------------|------------|-------|
| Stripe | Stripe API (Secret Key) | 2 min | Use **test mode** key first, then switch to live |
| Google Sheets | Google Sheets OAuth2 | 5 min | For the dashboard spreadsheet |
| Slack | Slack OAuth2 Bot Token | 3 min | For weekly summary + alerts |
| Gmail (optional) | Gmail OAuth2 | 3 min | For emailing the report |

---

## Google Sheets Dashboard Structure

The workflow appends one row per week to a sheet called `SaaS Metrics`. Headers:

| A | B | C | D | E | F | G | H | I | J | K | L |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Date | MRR | Net New MRR | Customers | New | Churned | Churn Rate % | ARPU | Est. LTV | Top Account | 2nd | 3rd |

The sheet grows over time — each week is a new row. Use this data to create charts (MRR over time, churn trend, customer growth).

---

## Setup Instructions

### Step 1: Stripe API Key
1. Go to dashboard.stripe.com → Developers → API keys
2. Copy your **Secret key** (starts with `sk_live_` or `sk_test_`)
3. Create n8n Stripe credential with the secret key
4. **Test with test mode first** — the workflow processes real data

### Step 2: Dashboard Spreadsheet
1. Create a Google Sheet called `SaaS Metrics Dashboard`
2. Add the headers listed above in Row 1
3. Create a second sheet called `Top Accounts` (for weekly top-5 list)
4. Paste the Sheet ID into all Google Sheets nodes

### Step 3: Configure Slack Channels
- **Weekly Summary**: Channel `#metrics` or `#general`
- **Alerts**: Channel `#leadership` (for high-churn or MRR-drop alerts)
- Adjust the thresholds in the IF node (default: churn > 5% OR MRR drop > 3%)

### Step 4: Test Run
1. Click **Execute Workflow** manually
2. Check the Google Sheet for the new row
3. Check Slack for the summary message
4. Verify the numbers match your Stripe dashboard

---

## Error Handling

- **Stripe API rate limit**: 100 requests/second in live mode, 25/sec in test. This workflow makes 2 API calls — well within limits
- **Empty subscription data**: If you have 0 active subscriptions (new business), the Code nodes handle division by zero gracefully — returns 0 instead of NaN
- **Large subscription volume**: The Stripe node auto-paginates. If you have 10,000+ subscriptions, consider adding a date filter to the Stripe query
- **Google Sheets API quota**: 60 requests/minute per user. One append per week is negligible
- **Missing previous week data**: On first run, there's no "previous week" to compare. The delta Code node returns `null` for trends — Slack message shows "—" instead of arrows

---

## Customization Ideas

- **Add Stripe MRR chart**: Use **QuickChart** node to generate a chart image and attach it to the Slack message
- **Segment by plan/product**: Add grouping logic in the Code nodes to report MRR by plan tier
- **Add cohort analysis**: Track MRR retention by signup month — more advanced but high-value
- **Push to Datadog/Grafana**: Replace Google Sheets with **HTTP Request** to Datadog API for real-time dashboards
- **Predictive alerts**: Add a simple linear regression in the Code node to forecast next month's MRR
- **ProfitWell/Baremetrics replacement**: This workflow + historical data = your own lightweight Baremetrics

---

## Important Notes

- **LTV calculation is simplified**: The formula `ARPU ÷ churn_rate` assumes constant churn. For more accuracy, use cohort-based LTV (beyond scope of this template)
- **Annual subscriptions**: The MRR Code node normalizes annual plans to monthly (`amount / 12`). Verify the `interval` field in your subscription data matches
- **Test mode vs. Live**: Stripe test data may have unrealistic churn patterns. Switch to live key only after verifying the workflow logic
- **Currency**: Assumes USD. For multi-currency, add FX conversion in the Code node

---

## FAQ

**Q: Can I run this daily instead of weekly?**  
A: Yes — change the Schedule Trigger to daily. The dashboard will have daily granularity. Change Slack message to reference "yesterday" instead of "this week."

**Q: Does this work with Stripe's new pricing model (usage-based)?**  
A: Partially. The MRR calculation uses `subscription.items.data[].price.unit_amount`. For usage-based pricing, you'll need to pull invoice line items and sum actual charges. This is a common customization.

**Q: What about refunds?**  
A: Refunds are captured in invoice data (negative amounts). The Net MRR Code node accounts for negative invoice totals as contraction MRR.

**Q: Can I add Paddle/Lemon Squeezy/Braintree?**  
A: Yes — replace the Stripe nodes with the appropriate payment provider's node or HTTP Request node. The calculation Code nodes work with any data shape (you'll need to adjust field mappings).
