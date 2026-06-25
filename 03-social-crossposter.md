# Social Media Cross-Poster

**Price: $25 on Gumroad**  
**Category: Marketing / Content Distribution**  
**Complexity: Intermediate (11 nodes)**  
**Runtime: ~3 seconds per post (depends on API latency)**

---

## What This Workflow Does

Write once, publish everywhere. This workflow watches your blog's RSS feed, detects new posts, and cross-posts them to X (Twitter), LinkedIn, and Telegram — with platform-specific formatting and optional AI-generated summaries.

1. **Monitors your RSS feed** (WordPress, Ghost, Medium, Substack, or any RSS source) on a schedule
2. **Compares against a Google Sheet** of previously posted URLs — never double-posts
3. **Extracts** title, URL, featured image, and first 280 characters
4. **Formats platform-specific messages** — short for X, professional for LinkedIn, casual for Telegram
5. **Posts to X (Twitter)** via API with link card
6. **Posts to LinkedIn** as a personal or company page update
7. **Posts to Telegram** channel with rich preview
8. **Logs the post** to a Google Sheet with timestamp and platform status
9. **Sends a Slack summary** confirming all platforms posted successfully (or which failed)

---

## Nodes Used (in execution order)

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | **Schedule Trigger** | `n8n-nodes-base.scheduleTrigger` | Runs every 15 minutes (or hourly) |
| 2 | **RSS Feed Read** | `n8n-nodes-base.rssFeedRead` | Fetches latest posts from your blog's RSS URL |
| 3 | **Google Sheets Read** | `n8n-nodes-base.googleSheets` | Reads the Posted URLs log to avoid duplicates |
| 4 | **Filter New Posts** | `n8n-nodes-base.if` | Compares each RSS item URL against the posted log |
| 5 | **Code (Extract)** | `n8n-nodes-base.code` | Strips HTML tags, truncates description, extracts image |
| 6 | **Set (X Format)** | `n8n-nodes-base.set` | Formats the tweet: `Title + link + hashtags` under 280 chars |
| 7 | **X (Twitter) Post** | `n8n-nodes-base.twitter` | Sends the tweet with OAuth 1.0a |
| 8 | **LinkedIn Post** | `n8n-nodes-base.linkedIn` | Creates a share/update on your LinkedIn profile or page |
| 9 | **Telegram Send** | `n8n-nodes-base.telegram` | Sends a message to your Telegram channel with link preview |
| 10 | **Google Sheets Append** | `n8n-nodes-base.googleSheets` | Logs the posted URL + timestamp + platform statuses |
| 11 | **Slack Summary** | `n8n-nodes-base.slack` | Posts a digest: "📢 Cross-posted: [Title] → X ✅ | LinkedIn ✅ | Telegram ✅" |

---

## Required Credentials

| Service | Credential Name in n8n | Setup Time | Difficulty |
|---------|----------------------|------------|------------|
| X (Twitter) | Twitter OAuth1 | 10 min | Medium — requires Developer account + API keys |
| LinkedIn | LinkedIn OAuth2 | 8 min | Medium — requires LinkedIn App + OAuth scopes |
| Telegram | Telegram Bot Token | 2 min | Easy — create bot via @BotFather |
| Google Sheets | Google Sheets OAuth2 | 5 min | Easy |
| Slack | Slack OAuth2 Bot Token | 3 min | Easy |

**Note:** You can disable any platform by toggling its node off. The workflow still runs for enabled platforms.

---

## Setup Instructions

### Step 1: RSS Feed
1. Find your blog's RSS feed URL (usually `yoursite.com/feed` or `yoursite.com/rss`)
2. Paste it into the **RSS Feed Read** node

### Step 2: Posted URLs Log Sheet
1. Create a Google Sheet called `Cross-Post Log`
2. Add headers: `Post URL | Title | Date Posted | X Status | LinkedIn Status | Telegram Status`
3. Paste the Sheet ID into both Google Sheets nodes

### Step 3: Platform Credentials

**X (Twitter):**
1. Go to developer.x.com → create a Project → create an App
2. Get API Key, API Secret, Access Token, Access Token Secret
3. Create n8n Twitter OAuth1 credential with all 4 values
4. **Important:** Use Free tier (1,500 tweets/month). Paid Basic is $100/mo — skip it.

**LinkedIn:**
1. Go to linkedin.com/developers → Create App
2. Add Products: "Share on LinkedIn" and "Sign In with LinkedIn"
3. Get Client ID and Client Secret
4. In OAuth2 settings, add redirect URI: `https://your-n8n-instance.com/rest/oauth2-credential/callback`
5. Create n8n LinkedIn OAuth2 credential → authenticate

**Telegram:**
1. Message @BotFather on Telegram → `/newbot` → name it → get token
2. Add your bot to your channel as **Administrator** (needs post permission)
3. Get your channel ID: forward a channel message to @userinfobot
4. Create n8n Telegram credential with the bot token

### Step 4: Customize Message Formats
- **X node**: Edit the tweet template. Max 280 chars. Include 2-3 hashtags.
- **LinkedIn node**: Edit for professional tone. Add a personal commentary line.
- **Telegram node**: Edit for your community voice. Telegram supports MarkdownV2.

### Step 5: Activate & Test
1. Click **Active** → **Execute Workflow** manually
2. Check each platform for the test post
3. Verify the Google Sheet logs the post
4. Check Slack for the summary

---

## Anti-Duplication Logic

The workflow stores all previously posted URLs in Google Sheets. Before posting, it:
1. Reads the entire log sheet
2. Compares each new RSS item's URL against the log
3. If URL exists → skip (false branch does nothing)
4. If URL is new → proceed to post on all platforms

This means you can delete and re-add items to your RSS feed without triggering re-posts.

---

## Error Handling

- **X rate limit**: Twitter Free API limits to 1,500 tweets/month (~50/day). If you hit the limit, the X node fails. Add an **Error Trigger** → **Slack** alert
- **LinkedIn token expiry**: LinkedIn OAuth tokens expire after 60 days. Set a calendar reminder to re-authenticate. The LinkedIn node will return a 401 error
- **Telegram bot removed from channel**: Returns `403 Forbidden`. Re-add bot as admin
- **RSS feed down**: The RSS node returns empty. Workflow exits silently — no error
- **Partial failure**: If one platform fails, the others still post. The Slack summary shows which succeeded and which failed

---

## Customization Ideas

- **AI Summary**: Add an **OpenAI** node before the Set nodes to generate a custom social caption from the article content
- **Threads support**: Use the **HTTP Request** node to call Threads API (Meta is slowly opening it)
- **Post at optimal times**: Replace the Schedule Trigger with a **Cron** node set to Tue/Thu 10 AM EST
- **Hashtag library**: Add a **Code** node that picks 3 random hashtags from a curated list based on post category
- **Mastodon/Bluesky**: Add **HTTP Request** nodes calling their APIs — n8n doesn't have native nodes for these yet

---

## FAQ

**Q: Will this get my Twitter account suspended?**  
A: No — this posts original links to your own content. It's not spam or automation of engagement (likes/follows). Just don't exceed 50 posts/day on the Free tier.

**Q: Can I post to LinkedIn Company Pages?**  
A: Yes — the LinkedIn node supports both personal profiles and organization pages. Select "Organization" in the node settings and enter your Company Page ID.

**Q: What if my RSS feed includes full article HTML?**  
A: The **Code (Extract)** node strips HTML tags and truncates to the first 200 characters. Adjust `maxLength` in the JS code to change the excerpt length.
