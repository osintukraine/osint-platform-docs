# Create a Custom RSS Feed

**Time: ~10 minutes**

In this tutorial, you'll learn how to transform your intelligence platform into a personal RSS feed machine. By the end, you'll be able to subscribe to any search query in your favorite RSS reader - whether that's Feedly, Inoreader, Thunderbird, or any other RSS application.

---

## What You'll Learn

After this tutorial, you will be able to:

1. Build search queries with filters (keywords, channels, importance level)
2. Generate RSS feed URLs for any search
3. Subscribe to feeds in popular RSS readers
4. Understand importance filtering for high-value feeds
5. Share feeds with team members securely

---

## Prerequisites

Before starting, make sure you have:

- The OSINT Intelligence Platform API running (`docker-compose up -d api`)
- At least 10 messages indexed in the platform (from your monitored channels)
- An RSS reader installed or account created (Feedly, Inoreader, Thunderbird, etc.) - optional for step 7
- Basic understanding of query parameters (the query string part of a URL)

**Time Check:** This tutorial takes about 10 minutes to complete.

---

## Understanding Dynamic RSS Feeds

The OSINT platform treats **every search as an RSS feed**. Unlike traditional RSS feeds that are pre-defined by publishers, this platform generates RSS feeds dynamically based on your search parameters.

### How It Works

```
Your Search Query
    ↓
API receives request
    ↓
Searches message database
    ↓
Generates RSS XML
    ↓
Your RSS reader displays items
```

### Key Concepts

- **Base URL**: `http://localhost:8000/rss/search` (on local machine) or `https://your-domain.com/rss/search` (production)
- **Query Parameters**: URL parameters that define what gets included in the feed
- **Feed Updates**: The feed updates automatically as new messages matching your criteria are added
- **Caching**: Popular feeds are cached for 5 minutes to reduce server load

---

## Step 1: Build Your First Query

Let's create a simple RSS feed that captures all high-importance messages from your monitored channels.

### Understanding Query Parameters

The RSS feed endpoint accepts these common filters:

| Parameter | Example | Meaning |
|---|---|---|
| `q` | `Bakhmut` | Search keyword |
| `importance_level` | `high` | Message importance (high, medium, low) |
| `days` | `7` | Only messages from last N days |
| `channel_id` | `123` | Filter to specific channel |
| `verified` | `true` | Only from verified channels |

### Build Your Query

Let's start simple. We'll create a feed for **all high-importance messages from the last 7 days**.

Open your terminal and build the URL:

```bash
# On local machine, the base URL is:
BASE_URL="http://localhost:8000"

# Build the feed URL with parameters:
FEED_URL="${BASE_URL}/rss/search?importance_level=high&days=7"

echo $FEED_URL
```

**Expected Output:**

```
http://localhost:8000/rss/search?importance_level=high&days=7
```

---

## Step 2: Test Your Feed in the Browser

Before subscribing in an RSS reader, let's test it directly in your browser.

1. Copy your feed URL from Step 1
2. Open it in your web browser
3. You should see XML feed output (RSS format)

**Expected Output:**

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<rss version="2.0">
<channel>
  <title>OSINT Platform Search: importance_level=high days=7</title>
  <description>Dynamic RSS feed of search results</description>
  <link>http://localhost:8000</link>
  <lastBuildDate>Tue, 09 Dec 2025 15:30:45 GMT</lastBuildDate>
  <item>
    <title>Russian Forces Advance Near Bakhmut</title>
    <description>Combat report...</description>
    <pubDate>Tue, 09 Dec 2025 14:22:10 GMT</pubDate>
    <link>http://localhost:8000/messages/12345</link>
  </item>
  <!-- More items here -->
</channel>
</rss>
```

**If you see this:** Your feed is working! Move to Step 3.

**If you see an error:** Check the troubleshooting section below.

---

## Step 3: Create More Specific Feeds

Now that you understand the basic format, let's create a few useful feeds for different intelligence needs.

### Feed 1: Combat-Related Messages from Ukraine

```bash
FEED_URL="http://localhost:8000/rss/search?q=combat&days=7"
echo $FEED_URL
```

This searches for messages containing "combat" from the last 7 days.

### Feed 2: Equipment Losses (High Importance)

```bash
FEED_URL="http://localhost:8000/rss/search?q=equipment%20losses&importance_level=high&days=30"
echo $FEED_URL
```

Note: `%20` is the URL-encoded space character.

### Feed 3: Civilian Impact (Latest 100 items)

```bash
FEED_URL="http://localhost:8000/rss/search?q=civilian&limit=100"
echo $FEED_URL
```

### Feed 4: Channel-Specific Feed

To filter to a specific channel, you need its ID. Let's find it first:

```bash
# List monitored channels
docker-compose exec -T postgres psql -U osint_user -d osint_platform -c \
  "SELECT id, name FROM channels LIMIT 10;"
```

**Expected Output:**

```
 id |      name
----+-----------------
  1 | News Channel
  2 | Analysis Channel
  3 | Breaking News
```

Now create a feed for channel ID 1:

```bash
FEED_URL="http://localhost:8000/rss/search?channel_id=1&importance_level=high"
echo $FEED_URL
```

---

## Step 4: Understanding Importance Levels

The platform classifies messages into three importance tiers:

| Level | Description | Use Case |
|---|---|---|
| `high` | Critical intelligence: verified military activity, combat reports, equipment movements | Alert feeds, priority inbox |
| `medium` | Moderate value: general updates, political statements, regional developments | Awareness feeds, background research |
| `low` | Background information: propaganda, opinions, routine updates | Archive feeds, historical reference |

### Example Feeds by Importance

**Alert Feed** (only critical):
```bash
http://localhost:8000/rss/search?importance_level=high&days=1
```

**Daily Brief** (all message levels):
```bash
http://localhost:8000/rss/search?days=1
```

**Weekly Archive** (everything from last week):
```bash
http://localhost:8000/rss/search?days=7
```

---

## Step 5: Add Feed to Feedly

Now let's subscribe to your feed in Feedly (the most popular RSS reader).

1. Go to [Feedly.com](https://feedly.com) and sign in (or create account)
2. Click the **"+"** button in the top left, or use the **"Add Content"** button
3. Paste your feed URL from Step 1 into the search box:
   ```
   http://localhost:8000/rss/search?importance_level=high&days=7
   ```
4. Feedly should detect it as an RSS feed
5. Click **"Subscribe"**
6. Add to a folder (e.g., "OSINT - High Priority") or create a new one
7. Click **"Subscribe"**

**Expected Result:** Your feed appears in Feedly's sidebar. You should see recent items from the last 7 days.

---

## Step 6: Add Feed to Inoreader

For Inoreader users:

1. Go to [Inoreader.com](https://www.inoreader.com) and sign in
2. Click **"Add a subscription"** in the left sidebar (or use the **"+"** button)
3. Paste your feed URL:
   ```
   http://localhost:8000/rss/search?importance_level=high&days=7
   ```
4. Click **"Subscribe"**
5. Choose a folder (e.g., "Intelligence Feeds")
6. Click **"Add"**

**Expected Result:** Items appear in your Inoreader inbox within seconds.

---

## Step 7: Add Feed to Thunderbird (Email + RSS Reader)

Mozilla Thunderbird has built-in RSS support:

1. Open Thunderbird
2. Click **"Add Other Accounts..."** or use **Manage** → **Subscribe to Newsfeeds**
3. Paste your feed URL
4. Thunderbird will auto-detect it as RSS
5. Choose a local folder to store items
6. Click **"Subscribe"**

**Expected Result:** Feed items appear as "messages" in your Thunderbird inbox.

---

## Step 8: Create a Feed for Your Team

To share a feed with team members securely:

### Option 1: Simple Share (Public URL)

If your feed contains non-sensitive intelligence:

```bash
# Share this URL with team:
https://your-domain.com/rss/search?importance_level=high&q=intelligence
```

Team members can paste this into any RSS reader and subscribe.

### Option 2: Authenticated Feed (with API Token)

For sensitive feeds, add authentication:

```bash
# Get your API token (or create one)
API_TOKEN="your_api_token_here"

# Build authenticated feed URL
FEED_URL="http://localhost:8000/rss/search?importance_level=high&q=sensitive-topic&token=${API_TOKEN}"

echo $FEED_URL
```

Share the URL only with authorized team members.

---

## Common Feed Recipes

Here are ready-to-use feed URLs for different intelligence needs:

### Recipe 1: Daily Intelligence Briefing

All messages from today, sorted by importance:

```bash
http://localhost:8000/rss/search?days=1
```

### Recipe 2: Critical Alerts Only

Only high-importance messages from last 24 hours:

```bash
http://localhost:8000/rss/search?importance_level=high&days=1
```

### Recipe 3: Weekly Archive Feed

Everything from last 7 days (for weekly digests):

```bash
http://localhost:8000/rss/search?days=7&limit=100
```

### Recipe 4: Keyword-Focused Feed

All messages mentioning a specific term:

```bash
http://localhost:8000/rss/search?q=Bakhmut&days=30
```

### Recipe 5: High-Value Military Intel

High-importance messages mentioning military topics:

```bash
http://localhost:8000/rss/search?q=military&importance_level=high&days=7
```

### Recipe 6: Specific Channel Only

All messages from one monitored channel:

```bash
# Find channel ID first, then:
http://localhost:8000/rss/search?channel_id=3
```

---

## Advanced: Building Complex Queries

You can combine multiple parameters for sophisticated feeds:

### Multi-Keyword Feed

Search for multiple keywords (use `q` parameter):

```bash
# Messages containing either "equipment" OR "losses"
http://localhost:8000/rss/search?q=equipment%20losses&days=7
```

### Date-Range Feed

Combine multiple parameters:

```bash
# High-importance messages from last 30 days in Channel 5
http://localhost:8000/rss/search?channel_id=5&importance_level=high&days=30&limit=100
```

### Maximum Item Feed

Get more items in your feed:

```bash
# 250 most recent messages (default is 50)
http://localhost:8000/rss/search?days=30&limit=250
```

---

## Troubleshooting

### "Feed is empty but I have messages"

**Problem:** Your feed URL returns no items.

**Solutions:**

1. **Check message count:**
   ```bash
   docker-compose exec -T postgres psql -U osint_user -d osint_platform -c \
     "SELECT COUNT(*) FROM messages WHERE created_at > NOW() - INTERVAL '7 days';"
   ```

2. **If count is 0:** You don't have messages from the last 7 days
   - Extend the date range: change `days=7` to `days=30`
   - Add a public channel to monitoring to get test messages

3. **If count > 0:** API might not be responding
   ```bash
   curl -I http://localhost:8000/rss/search?days=7
   ```
   Should return HTTP 200

4. **Check API logs:**
   ```bash
   docker-compose logs -f api | grep -i "rss\|search"
   ```

### "RSS reader says feed is invalid"

**Problem:** The URL produces an error instead of RSS XML.

**Solutions:**

1. **Test in browser:** Paste URL directly in browser
   ```
   http://localhost:8000/rss/search?importance_level=high&days=7
   ```
   You should see XML starting with `<?xml version="1.0"...`

2. **Check URL syntax:** Make sure:
   - URL starts with `http://` or `https://`
   - Parameters are separated by `&` (not `;`)
   - Spaces are encoded as `%20`
   - No trailing spaces or newlines

3. **Test with curl:**
   ```bash
   curl -I http://localhost:8000/rss/search?importance_level=high
   ```
   Should return `200 OK`

### "RSS reader shows items but they're not updating"

**Problem:** Feed subscribed but no new items appear.

**Solutions:**

1. **Check update frequency:** Different RSS readers have different refresh rates
   - Feedly: typically 1-4 hours for public feeds
   - Inoreader: can be set to 15 minutes
   - Thunderbird: check settings

2. **Force manual refresh:** Most readers have a refresh button

3. **Verify new messages are being added:**
   ```bash
   docker-compose exec -T postgres psql -U osint_user -d osint_platform -c \
     "SELECT COUNT(*) FROM messages WHERE created_at > NOW() - INTERVAL '1 hour';"
   ```

---

## What You Learned

Congratulations! You now understand:

1. **RSS feed URLs** - How to build feed URLs with query parameters
2. **Query parameters** - What `importance_level`, `days`, `q`, etc. do
3. **RSS reader integration** - How to subscribe in Feedly, Inoreader, Thunderbird
4. **Feed filtering** - How to create targeted feeds for different needs
5. **Feed sharing** - How to share feeds securely with team members

---

## Next Steps

Now that you're producing RSS feeds, you can:

1. **Setup Discord Alerts** - [Tutorial: Setup Discord Alerts](setup-discord-alerts.md)
   - Turn RSS feeds into Discord notifications
   - Alert your team in real-time

2. **Create Multiple Feeds** - Build feeds for different intelligence areas
   - One for military updates
   - One for political developments
   - One for equipment tracking

3. **Subscribe in Your Reader** - Add all your feeds to Feedly or Inoreader
   - Organize by folders/collections
   - Set up smart filters

4. **Share with Team** - Give authenticated feed URLs to team members
   - Each person can subscribe in their reader
   - Feeds update automatically

---

## Key Takeaways

| Concept | Key Point |
|---------|-----------|
| **Dynamic Feeds** | Every search is a feed - no pre-configuration needed |
| **Query Building** | Use URL parameters to filter: importance_level, days, q, channel_id |
| **Reader Agnostic** | Works with any RSS reader (Feedly, Inoreader, Thunderbird, etc.) |
| **Real-Time Updates** | Feeds auto-update as new messages arrive |
| **Secure Sharing** | Use API tokens for authenticated feeds |

---

**Next: Set up real-time Discord alerts for your feeds? Check out [Setup Discord Alerts](setup-discord-alerts.md)**
