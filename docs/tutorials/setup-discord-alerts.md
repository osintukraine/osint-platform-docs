# Setup Discord Alerts

**Time: ~15 minutes**

In this tutorial, you'll set up real-time Discord notifications for critical intelligence using n8n workflows and RSS feeds. Your team will receive instant alerts in Discord channels whenever high-value intelligence is detected.

---

## What You'll Learn

After this tutorial, you will be able to:

1. Create a Discord webhook for your server
2. Set up an n8n workflow to process RSS feeds
3. Filter alerts by importance level
4. Route different content to different Discord channels
5. Test the alert pipeline end-to-end

---

## Prerequisites

Before starting, make sure you have:

- A Discord server with admin permissions (or ability to manage webhooks)
- The OSINT platform running with API and RSS endpoints active
- At least one RSS feed created (see [Create Custom RSS Feed](create-custom-rss-feed.md) tutorial)
- n8n service running (`docker-compose up -d n8n`)
- Basic understanding of Discord webhook URLs
- ~15 minutes of uninterrupted time

**Time Check:** This tutorial takes about 15 minutes to complete.

---

## How The Alert Pipeline Works

Here's the complete flow:

```
RSS Feed (high-importance messages)
    ↓
n8n Workflow (polls RSS, formats alerts)
    ↓
Discord Webhook (sends to your channel)
    ↓
Discord Channel (team sees alerts)
```

The workflow automatically polls your RSS feed every few minutes and sends formatted messages to Discord when new items appear.

---

## Step 1: Create a Discord Webhook

First, we'll create a Discord webhook that n8n can send messages to.

### Create a Test Channel

1. Open your Discord server
2. Create a new channel called `#osint-alerts` (or similar)
3. Write down the channel name

### Create the Webhook

1. In Discord, right-click on the `#osint-alerts` channel
2. Select **Edit Channel**
3. Go to the **Integrations** tab in the left sidebar
4. Click **Webhooks**
5. Click **New Webhook** (or **Create Webhook**)
6. Name it `OSINT Bot` or similar
7. Click **Create**
8. Click **Copy Webhook URL**

**Expected Result:** You now have a webhook URL that looks like:
```
https://discord.com/api/webhooks/1234567890/abcdefghijklmnopqrst
```

!!! warning "Keep Your Webhook URL Private"
    This URL lets anyone post to your Discord channel. Do NOT share it publicly or commit it to git.

---

## Step 2: Test the Webhook Manually

Before setting up n8n, let's verify the webhook works:

```bash
# Save your webhook URL
WEBHOOK_URL="https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN"

# Send a test message
curl -X POST "${WEBHOOK_URL}" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Test alert from OSINT platform!",
    "username": "OSINT Bot"
  }'
```

**Expected Result:** A message appears in your Discord `#osint-alerts` channel saying "Test alert from OSINT platform!"

If it doesn't work, check:
- You have the correct webhook URL
- The channel still exists
- The webhook still exists in Discord

---

## Step 3: Access n8n

Now we'll create a workflow in n8n to automatically poll your RSS feed and send alerts.

1. Open n8n at `http://localhost:5678`
2. Sign in (default: email: test@n8n.com, password: password)
3. Click **Create Workflow** or **New Workflow**
4. Give it a name: `OSINT Discord Alerts`
5. Click **Create**

**Expected Result:** You're now in the n8n workflow editor with a blank workflow.

---

## Step 4: Add an RSS Feed Node

We'll add a node that reads your RSS feed:

1. Click the **+** button to add a node
2. Search for **RSS Feed**
3. Click **RSS Feed** (from the list)
4. Configure the node:
   - **URL**: Paste your RSS feed URL from the [Create Custom RSS Feed](create-custom-rss-feed.md) tutorial
     ```
     http://localhost:8000/rss/search?importance_level=high&days=1
     ```
   - **Poll Time**: Set to `5m` (checks feed every 5 minutes)

5. Click the node to expand it and see options
6. Under **Options**, set:
   - **Include Content as Attachment**: Yes (this includes full message content)

**Expected Result:** The RSS Feed node is configured and ready to fetch items.

---

## Step 5: Add a Discord Notification Node

Now we'll add the node that sends messages to Discord:

1. Click the **+** button to add another node
2. Search for **Discord Webhook**
3. Click **Discord Webhook** (from the HTTP node or use HTTP node)
4. Configure the node:
   - **URL**: Paste your Discord webhook URL:
     ```
     https://discord.com/api/webhooks/1234567890/abcdefghijklmnopqrst
     ```
   - **Method**: POST
   - **Headers**: Add header
     - Key: `Content-Type`
     - Value: `application/json`

5. In **Body** (request body), add JSON:
   ```json
   {
     "content": "New Alert: {{ $json.title }}",
     "username": "OSINT Bot",
     "embeds": [
       {
         "title": "{{ $json.title }}",
         "description": "{{ $json.contentSnippet }}",
         "color": 16711680,
         "url": "{{ $json.link }}"
       }
     ]
   }
   ```

**Expected Result:** The Discord node is configured to format and send alerts.

---

## Step 6: Connect the Nodes

Now we'll wire the RSS node to the Discord node:

1. Click the small circle on the right side of the **RSS Feed** node
2. Drag it to the left side of the **Discord Webhook** node
3. Release to connect them

**Expected Result:** There's now a line connecting RSS Feed → Discord Webhook.

---

## Step 7: Test the Workflow

Let's test that everything works:

1. Click the **Execute Workflow** button (play icon) in the top right
2. Watch the nodes execute
3. Check your Discord channel for a test message

**Expected Output in Discord:**
```
New Alert: Russian Forces Advance Near Bakhmut

New Alert: Russian Forces Advance Near Bakhmut
Combat report from military sources. Significant tactical movement detected.

[View Message](http://localhost:8000/messages/12345)
```

**If the test worked:** Jump to Step 8.

**If nothing appeared:** See the Troubleshooting section.

---

## Step 8: Save and Activate the Workflow

Now let's save and activate this workflow so it runs automatically:

1. Click **Save** in the top right
2. Confirm the workflow name: `OSINT Discord Alerts`
3. Click **Save Workflow**
4. Click the **Activate** toggle (top right)
5. Confirm you want to activate it

**Expected Result:**
- The toggle turns blue/green
- n8n shows "Workflow is now active"
- The RSS feed will now be checked every 5 minutes
- Matching items will be sent to Discord automatically

---

## Step 9: Route Alerts to Different Channels

Now let's enhance the workflow to route different types of alerts to different channels.

### Create Multiple Discord Channels

In your Discord server, create additional channels:
- `#osint-military` (for military intelligence)
- `#osint-political` (for political updates)
- `#osint-critical` (for critical alerts only)

### Create Multiple Workflows

Create separate workflows for different feed types:

**Workflow 1: Military Intelligence**
- RSS URL: `http://localhost:8000/rss/search?q=military&importance_level=high&days=1`
- Webhook: Point to `#osint-military` channel
- Polling: Every 5 minutes

**Workflow 2: Political Updates**
- RSS URL: `http://localhost:8000/rss/search?q=political&importance_level=high&days=1`
- Webhook: Point to `#osint-political` channel
- Polling: Every 5 minutes

**Workflow 3: Critical Only**
- RSS URL: `http://localhost:8000/rss/search?q=critical&importance_level=high&days=1`
- Webhook: Point to `#osint-critical` channel
- Polling: Every 2 minutes (faster for critical)

For each workflow:
1. Click **+** at the top left to create a new workflow
2. Copy the configuration from Step 4-6 above
3. Change the RSS URL to filter different content
4. Change the Discord webhook to point to the different channel
5. Activate the workflow

---

## Step 10: Customize Message Formatting

You can customize how alerts look in Discord using embeds:

### Color-Coded Alerts

Modify the Discord node's JSON body to use different colors:

**Red (Critical):**
```json
{
  "embeds": [
    {
      "title": "{{ $json.title }}",
      "color": 16711680
    }
  ]
}
```

**Orange (High):**
```json
{
  "embeds": [
    {
      "title": "{{ $json.title }}",
      "color": 16744448
    }
  ]
}
```

**Blue (Medium):**
```json
{
  "embeds": [
    {
      "title": "{{ $json.title }}",
      "color": 255
    }
  ]
}
```

### Add Channel Information

Include the original channel in the alert:

```json
{
  "embeds": [
    {
      "title": "{{ $json.title }}",
      "fields": [
        {
          "name": "Source Channel",
          "value": "Your Channel Name",
          "inline": true
        },
        {
          "name": "Importance",
          "value": "High",
          "inline": true
        }
      ]
    }
  ]
}
```

---

## Step 11: Add Role Mentions for Critical Alerts

For critical alerts, you can mention roles to get their attention:

Modify the Discord webhook body:

```json
{
  "content": "<@&ROLE_ID> New Critical Alert",
  "embeds": [
    {
      "title": "{{ $json.title }}",
      "description": "{{ $json.contentSnippet }}",
      "color": 16711680
    }
  ]
}
```

**Find Role ID:**
1. In Discord, go to Server Settings → Roles
2. Right-click the role
3. Select "Copy Role ID"
4. Replace `ROLE_ID` in the workflow

---

## Step 12: Monitor Alert Delivery

Check that alerts are being delivered:

### In n8n

1. Open your workflow
2. Click the **Execution History** tab
3. You should see rows for each time the workflow ran
4. Green checkmarks = successful execution
5. Red X = failed execution
6. Click a row to see details

### In Discord

1. Check your alert channels
2. Look for incoming messages
3. Verify they have the correct formatting
4. Check timestamps to verify polling frequency

### Common Issues

**No executions appearing:**
- Workflow isn't activated
- No matching RSS items exist
- n8n service stopped

**Executions failing:**
- Discord webhook URL incorrect
- Discord channel deleted
- Network issue

---

## Step 13: Adjust Polling Frequency

Depending on your needs, adjust how often n8n checks the RSS feed:

| Frequency | Best For |
|---|---|
| 1 minute | Very critical feeds, quick response needed |
| 5 minutes | Normal alerts, balanced approach |
| 15 minutes | Low-priority feeds, reduce API load |
| 1 hour | Archive feeds, historical data |

To change:
1. Open your workflow
2. Click the **RSS Feed** node
3. Change **Poll Time** to desired interval
4. Click **Save**
5. Workflow continues running with new frequency

---

## Troubleshooting

### "No alerts appear in Discord"

**Problem:** Workflow is activated but messages don't appear in Discord.

**Solutions:**

1. **Verify webhook URL:**
   ```bash
   # Test webhook manually
   WEBHOOK_URL="https://discord.com/api/webhooks/YOUR_ID/YOUR_TOKEN"
   curl -X POST "${WEBHOOK_URL}" \
     -H "Content-Type: application/json" \
     -d '{"content": "Test"}'
   ```

2. **Check n8n execution history:**
   - Open workflow
   - Click "Execution History"
   - Look for red X marks (failures)
   - Click failed execution to see error

3. **Verify RSS feed has items:**
   ```bash
   curl http://localhost:8000/rss/search?importance_level=high&days=1
   ```
   Should return items with `<item>` tags

4. **Check Discord permissions:**
   - Server Settings → Webhooks
   - Verify webhook still exists
   - Check channel exists and isn't deleted

### "Workflow executes but sends empty messages"

**Problem:** Messages appear but are blank.

**Solutions:**

1. **Check RSS field names:** The JSON template might use wrong field names
   ```bash
   # Get actual RSS fields
   curl http://localhost:8000/rss/search?importance_level=high&days=1 | grep -i title
   ```

2. **Update template with correct fields:**
   - Replace `{{ $json.title }}` with actual field from RSS
   - Replace `{{ $json.contentSnippet }}` with correct field

3. **Test with simple message:**
   ```json
   {
     "content": "Alert test"
   }
   ```
   If this works, the RSS field names are wrong.

### "Too many alerts, Discord being spammed"

**Problem:** Receiving excessive alert messages.

**Solutions:**

1. **Increase polling interval:**
   - Change RSS node Poll Time to `15m` or `1h`
   - Fewer checks = fewer alerts

2. **Filter RSS feed:**
   - Change RSS URL to be more specific:
     ```
     http://localhost:8000/rss/search?importance_level=high&q=specific-keyword&days=1
     ```

3. **Create separate workflows:**
   - One for critical (fast polling)
   - One for routine (slow polling)
   - Prevents overwhelming one channel

### "n8n service isn't running"

**Problem:** Can't access n8n at http://localhost:5678

**Solutions:**

1. **Start n8n:**
   ```bash
   docker-compose up -d n8n
   ```

2. **Verify it's running:**
   ```bash
   docker-compose ps n8n
   ```
   Should show "Up"

3. **Check logs:**
   ```bash
   docker-compose logs n8n | tail -50
   ```

### "Discord webhook URL is invalid"

**Problem:** Workflow fails with webhook error.

**Solutions:**

1. **Verify webhook exists:**
   - In Discord, go to Server Settings → Webhooks
   - Check webhook is still there
   - If missing, create a new one

2. **Get correct URL:**
   - Server Settings → Webhooks
   - Click the webhook
   - Click "Copy Webhook URL"
   - Paste in n8n workflow

3. **Test webhook:**
   ```bash
   WEBHOOK_URL="..."
   curl -X POST "${WEBHOOK_URL}" \
     -H "Content-Type: application/json" \
     -d '{"content": "test"}'
   ```

---

## What You Learned

Congratulations! You now understand:

1. **Discord webhooks** - How to create and test webhook URLs
2. **n8n workflows** - How to build automation workflows
3. **RSS polling** - How to periodically check RSS feeds for updates
4. **Alert formatting** - How to customize Discord message format
5. **Alert routing** - How to send different alerts to different channels
6. **Troubleshooting** - How to debug common issues

---

## Next Steps

Now that you have Discord alerts working, you can:

1. **Create Advanced Filters** - Set up more RSS feeds with different queries
   - Military intelligence feed
   - Political developments feed
   - Equipment losses feed

2. **Add More Channels** - Create dedicated Discord channels for different intelligence types
   - #osint-military
   - #osint-political
   - #osint-critical

3. **Integrate with Telegram** - Use n8n to also send alerts to a Telegram bot
   - Same workflow, add a Telegram node
   - Alert your team in multiple places

4. **Set Up Alerts for Specific Entities** - Create feeds that track specific people/organizations
   - Use `q=Wagner%20Group` to track mentions
   - Route to dedicated team member channel

5. **Scale to Production** - Deploy this to your production server
   - Update RSS URLs to point to production domain
   - Create Discord webhooks on production server
   - Set up monitoring/alerting on workflow failures

---

## Key Takeaways

| Concept | Key Point |
|---------|-----------|
| **Webhooks** | One-way integration - n8n sends data to Discord |
| **Polling** | n8n checks RSS feed on a schedule (every 5 min, etc.) |
| **Automation** | No manual work - alerts sent automatically |
| **Routing** | Different feeds go to different Discord channels |
| **Customization** | Change message format, colors, mentions as needed |

---

**All done! Your team is now receiving real-time intelligence alerts in Discord.**
