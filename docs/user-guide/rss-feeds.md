# RSS Feed Management

Create and manage custom RSS feeds from any search query to monitor intelligence in your favorite feed reader.

## Overview

The platform's "Subscribe to any search" feature transforms any filtered search into an RSS, Atom, or JSON feed. This enables continuous monitoring without manual checks.

### Key Features

- **Live Filters** - Any search/filter combination becomes a feed
- **Three Formats** - RSS 2.0, Atom 1.0, JSON Feed
- **Authenticated Feeds** - Optional token-based authentication
- **URL Signing** - Cryptographically signed feed URLs
- **Universal Support** - Works with any feed reader

## Creating a Feed

### Step 1: Configure Your Search

1. Go to Browse Messages (`/`)
2. Apply desired filters:
   - Search query
   - Channel selection
   - Topic, importance, language
   - Date range
   - Media type
   - Engagement thresholds

3. Preview results to verify

### Step 2: Subscribe

1. Click the **"Subscribe Feed"** button (below filters)
2. Choose your format:

**RSS 2.0** (📰)
- Most compatible format
- Works with Feedly, Inoreader, The Old Reader
- Standard XML structure
- Good for general use

**Atom 1.0** (⚛️)
- Modern standard (RFC 4287)
- Better content structure
- Full metadata support
- Preferred for advanced readers

**JSON Feed** (📋)
- JSON format, not XML
- Ideal for developers and APIs
- Easy to parse programmatically
- Growing support

3. Click format to **copy URL** or **open in new tab**

### Step 3: Add to Feed Reader

Copy the generated URL and add to your reader:

**Feedly**
```
1. Click "+" in left sidebar
2. Paste feed URL
3. Choose collection
4. Click "Follow"
```

**Inoreader**
```
1. Click "Add Subscription"
2. Paste feed URL
3. Click "Subscribe"
4. Organize into folders
```

**NewsBlur**
```
1. Click "Add Site"
2. Paste feed URL
3. Click "Add"
```

**Reeder (iOS/Mac)**
```
1. Settings → Add Feed
2. Paste URL
3. Tap/Click Add
```

## Feed Authentication

### Understanding Feed Tokens

Feed tokens provide authenticated access to feeds:

**When Authentication is Required:**
- Feed authentication is on (the default)
- You must create a token to subscribe
- Feeds won't work without a valid token and signature

**When Authentication is Optional:**
- Your administrator has turned feed authentication off
- Feeds work without tokens (public access)
- Tokens still provide tracking and revocation

### Managing Tokens

Access token management at **Settings → Feed Tokens** (`/settings/feed-tokens`).

#### Creating a Token

1. Navigate to Feed Tokens page
2. Enter optional label (e.g., "My Feedly", "Work Reader")
3. Click **"Create Token"**
4. **Save the token immediately** - it's shown only once
5. Copy to secure password manager

#### Token Display

The page shows:
- **Prefix** - Short identifier (e.g., `ft_a1b2c3`)
- **Label** - Your descriptive name
- **Created** - Timestamp
- **Last Used** - Last request timestamp
- **Requests** - Total requests made
- **Status** - Active or Revoked

#### Revoking Tokens

1. Find token in list
2. Click **"Revoke"**
3. Confirm action
4. **All feeds using this token stop working**

To restore access:
1. Create new token
2. Regenerate feed URLs with new token
3. Update feed reader subscriptions

### Signed URLs

When you have a token, feed URLs are cryptographically signed:

```
https://api.example.com/rss/search
  ?format=rss
  &q=tank
  &importance_level=high
  &token=<token id>
  &sig=<signature of the parameters>
```

**Benefits:**
- URL tampering detection
- Parameter validation: changing a filter in the URL invalidates it, so regenerate the URL after editing filters
- Audit trail via the token's request count

## Feed URL Structure

### Base Endpoint

```
GET /rss/search
```

### Common Parameters

Most filters from Browse Messages are supported:

```
format=rss|atom|json          # Feed format
q=search+terms                # Full-text search
channel_username=channelname  # Specific channel
channel_folder=%UA|%RU        # Country filter
importance_level=high|medium|low
topic=combat|equipment|...    # AI topic
language=uk|ru|en|...         # Detected language
media_type=photo|video|...    # Media filter
has_media=true|false          # Any media
min_views=1000                # Engagement threshold
min_forwards=50               # Virality threshold
sentiment=positive|negative|neutral
days=7                        # Last N days
date_from=2024-01-01          # Start date
date_to=2024-12-31            # End date
limit=50                      # Items in feed (1-100, default 50)
```

Items are always newest first (by Telegram post time).

Two shortcut feeds are also available: `/rss/channel/{username}` (one channel; optional `importance_level`, `days`, `limit`) and `/rss/topic/{topic}` (one topic, high importance by default).

### Example URLs

**High-priority combat reports:**
```
/rss/search
  ?format=rss
  &importance_level=high
  &topic=combat
  &days=7
```

**Ukrainian channels with media:**
```
/rss/search
  ?format=atom
  &channel_folder=%UA
  &has_media=true
  &min_views=1000
```

**Specific entity mentions:**
```
/rss/search
  ?format=json
  &q="T-90 tank"
  &media_type=photo
```

## Feed Content

### Feed-Level Metadata

Each feed includes:

**RSS 2.0**
```xml
<channel>
  <title>OSINT Platform - Custom Feed</title>
  <description>Filtered intelligence feed</description>
  <link>https://platform.example.com</link>
  <lastBuildDate>Mon, 09 Dec 2024 12:00:00 +0000</lastBuildDate>
</channel>
```

**Atom 1.0**
```xml
<feed xmlns="http://www.w3.org/2005/Atom">
  <title>OSINT Platform - Custom Feed</title>
  <link href="https://platform.example.com/rss/search?..."/>
  <updated>2024-12-09T12:00:00Z</updated>
  <id>urn:osint-platform:feed:abc123</id>
</feed>
```

**JSON Feed**
```json
{
  "version": "https://jsonfeed.org/version/1.1",
  "title": "OSINT Platform - Custom Feed",
  "home_page_url": "https://platform.example.com",
  "feed_url": "https://platform.example.com/rss/search?...",
  "items": [...]
}
```

### Item-Level Content

Each message becomes a feed item with:

**Core Fields**
- Title: Message snippet (first 100 chars)
- Description/Content: Full message text
- Link: Direct link to message detail page
- Publication date: Telegram post timestamp
- GUID/ID: Unique message identifier

**Extended Metadata**
- Channel name and username
- AI importance level
- Detected topic
- Language code
- Media attachments (enclosures)
- Entity mentions (categories/tags)

**Example RSS Item:**
```xml
<item>
  <title>Ukrainian forces report T-90M destruction near Bakhmut...</title>
  <description>Full message content with context...</description>
  <link>https://platform.example.com/messages/12345</link>
  <pubDate>Mon, 09 Dec 2024 08:30:00 +0000</pubDate>
  <guid isPermaLink="true">https://platform.example.com/messages/12345</guid>
  <category>combat</category>
  <category>equipment</category>
  <category>importance:high</category>
  <enclosure url="https://media.example.com/photo.jpg" type="image/jpeg"/>
</item>
```

## Feed Performance

### Update Frequency

Feeds are generated in real-time on each request:

- **No caching** - Always fresh data
- **Typical response time** - 100-500ms
- **Large result sets** - May take 1-2 seconds
- **Rate limiting** - Respect reader poll intervals

### Recommended Poll Intervals

Configure your feed reader:

**High-Priority Feeds**
- Poll every: 15-30 minutes
- Use for: Combat reports, critical alerts

**Medium-Priority Feeds**
- Poll every: 1-2 hours
- Use for: General monitoring, equipment tracking

**Low-Priority Feeds**
- Poll every: 4-6 hours
- Use for: Background research, archive searches

**Why not faster?**
- Platform resources
- Most Telegram channels post infrequently
- Feed readers may limit poll frequency

### Feed Reader Limits

Most readers limit:
- **Feedly Free**: 100 sources, 3 items shown
- **Inoreader Free**: Unlimited sources, ads
- **NewsBlur Free**: 64 sites
- **Self-hosted**: No limits (Miniflux, FreshRSS)

## Best Practices

### Effective Feed Organization

**Create Specific Feeds**
```
✓ Good: "High-priority Ukrainian combat - Last 24h"
✗ Bad: "All messages" (too broad)

✓ Good: "T-90 tank mentions with photos"
✗ Bad: "Any equipment mention" (too noisy)
```

**Use Folders/Collections**
```
Intelligence/
  ├── Combat Reports/
  │   ├── Ukrainian High-Priority
  │   ├── Russian High-Priority
  │   └── Equipment Losses
  ├── GEOINT/
  │   ├── Location Mentions
  │   └── Infrastructure Damage
  └── OSINT/
      ├── Verified Channels
      └── Analyst Networks
```

### Filter Optimization

**Reduce Noise**
- Set minimum importance level
- Filter by verified channels
- Use topic filters aggressively
- Set engagement thresholds

**Increase Signal**
- Combine filters logically
- Use semantic search for concepts
- Monitor entity mentions
- Track forward chains

### Token Management

**Security**
- Create separate tokens per device
- Label tokens clearly
- Revoke unused tokens quarterly
- Never share token URLs publicly

**Organization**
- "Work Laptop Feedly"
- "Personal iPhone Reeder"
- "Team Shared Reader"
- "API Integration"

## Troubleshooting

### Feed Not Working

**Check authentication:**
```
1. Visit Feed Tokens page
2. Verify token is Active, not Revoked
3. Check "Last Used" timestamp updates
4. Review "Use Count" increments
```

**Verify URL:**
```
1. Open feed URL in browser
2. Should see XML/JSON, not error page
3. Check for "401 Unauthorized" errors
4. Confirm `sig` and `token` are present (if auth required)
```

**Test feed reader:**
```
1. Try adding feed to different reader
2. Check reader's error logs
3. Verify internet connectivity
4. Test with simple feed first
```

### No New Items

**Check filters:**
- Date range might be too narrow
- No new messages match criteria
- Importance threshold too high
- Channel inactive or removed

**Verify search:**
- Open filter URL in browser
- Check Browse Messages with same filters
- Confirm messages exist matching criteria

### Feed Reader Errors

**"Feed could not be found" (404)**
- Feed endpoint changed
- Token revoked
- URL corrupted during copy/paste

**"Unauthorized" (401)**
- Authentication required but no token
- Token revoked or expired
- Token not included in URL

**"Invalid feed format" (400)**
- Malformed filter parameters
- Invalid date format
- URL encoding issues

## Advanced Use Cases

### Monitoring Workflows

**Analyst Dashboard**
```
1. Create high-priority feed (15min poll)
2. Route to dedicated folder
3. Enable mobile notifications
4. Review each morning
```

**Research Archive**
```
1. Create broad semantic search feed
2. Long date range (90+ days)
3. Low poll frequency (daily)
4. Export to Zotero/Notion
```

**Team Collaboration**
```
1. Create shared topic feeds
2. Distribute signed URLs
3. Assign analysts to feeds
4. Aggregate in shared reader
```

### Integration Examples

**Slack/Discord Bot**
```javascript
// Poll feed every 15 minutes
const feedUrl = "https://api.example.com/rss/search?format=json&topic=combat&importance_level=high&token=...";

setInterval(async () => {
  const response = await fetch(feedUrl);
  const feed = await response.json();

  for (const item of feed.items) {
    if (!seenBefore(item.id)) {
      postToSlack(item);
    }
  }
}, 15 * 60 * 1000);
```

**Data Export Pipeline**
```python
import feedparser

# Fetch feed
feed = feedparser.parse("https://api.example.com/rss/search?format=atom&...")

# Extract structured data
for entry in feed.entries:
    message = {
        'id': entry.id,
        'title': entry.title,
        'content': entry.content[0].value,
        'date': entry.published,
        'topics': [tag.term for tag in entry.tags],
    }
    store_in_database(message)
```

---

**Next Steps:**
- [Explore Entities](entities.md) linked in feed items
- [View Social Graph](social-graph.md) for message relationships
- [Configure Notifications](notifications.md) for real-time alerts
