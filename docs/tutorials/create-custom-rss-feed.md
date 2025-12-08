# Tutorial: Create Custom RSS Feed

Learn how to create custom RSS feeds with advanced filtering for intelligence consumption.

## Learning Objectives

By the end of this tutorial, you will:

- Understand RSS feed capabilities
- Create a custom filtered feed
- Configure advanced filters
- Consume the feed in an RSS reader

## Prerequisites

- Platform installed and running
- User account with feed creation permissions
- RSS reader application (Feedly, Inoreader, etc.)
- Channels being monitored

## Estimated Time

15-20 minutes

## Step 1: Access Feed Creator

**TODO: Content to be generated from codebase analysis**

1. Open frontend at http://localhost:3000
2. Navigate to RSS Feeds section
3. Click "Create New Feed"

## Step 2: Configure Basic Settings

**TODO: Add screenshots and step-by-step instructions:**

### Feed Name

Give your feed a descriptive name:

- Example: "Russian Military Updates"
- Example: "Ukraine Government Channels"

### Feed Description

Describe what the feed contains:

- Example: "Messages from Russian military channels mentioning equipment losses"

### Public vs Private

**TODO: Explain public/private feed options**

## Step 3: Configure Filters

**TODO: Document available filters:**

### Channel Selection

Select specific channels to include:

1. Browse channel list
2. Select channels
3. Use folder-based selection

### Entity Filters

Include only messages mentioning specific entities:

- Filter by entity type (person, organization, location)
- Select specific entities
- Use entity categories

### Tag Filters

Filter by AI-generated tags:

- Threat level (critical, high, medium, low)
- Content categories (military, political, economic)
- Sentiment (positive, negative, neutral)

### Keyword Filters

**TODO: Document keyword filtering:**

- Include keywords
- Exclude keywords
- Boolean operators

### Date Range

**TODO: Document date filtering:**

- Last 24 hours
- Last 7 days
- Custom date range
- No date limit

## Step 4: Configure Feed Options

**TODO: Document feed options:**

### Item Limit

Maximum number of items in feed:

- 50 items (default)
- 100 items
- 250 items

### Sort Order

- Newest first (default)
- Oldest first
- Relevance

### Update Frequency

**TODO: Document feed refresh rates**

## Step 5: Generate Feed

**TODO: Add screenshots and step-by-step instructions:**

1. Review filter configuration
2. Click "Generate Feed"
3. System creates feed and provides URL

## Step 6: Get Feed URL

**TODO: Document feed URL structure:**

```
https://your-domain.com/api/rss/feeds/{feed_id}
```

For authenticated feeds:

```
https://your-domain.com/api/rss/feeds/{feed_id}?token={api_token}
```

## Step 7: Add Feed to RSS Reader

**TODO: Provide examples for popular readers:**

### Feedly

1. Click "Add Content"
2. Paste feed URL
3. Configure update frequency

### Inoreader

1. Click "Add Feed"
2. Paste feed URL
3. Add to folder

### NewsBlur

**TODO: Add NewsBlur instructions**

## Step 8: Verify Feed is Working

**TODO: Add verification steps:**

1. Check feed in RSS reader
2. Verify items appear
3. Test filters by checking content
4. Verify updates are received

## Advanced Filtering Examples

**TODO: Provide concrete examples:**

### Example 1: High-Priority Military Intelligence

- Channels: Archive-Russia
- Tags: military, equipment, losses
- Threat level: high, critical
- Entities: specific military units

### Example 2: Ukrainian Government Announcements

- Channels: Monitor-Ukraine-Government
- Tags: political, announcement
- Exclude: spam, low-priority

### Example 3: Entity-Focused Feed

- All channels
- Entity: "Wagner Group"
- Date: Last 30 days

## Managing Feeds

**TODO: Document feed management:**

### Editing Feeds

1. Navigate to feed list
2. Select feed
3. Edit configuration
4. Save changes

### Deleting Feeds

**TODO: Add deletion instructions**

### Feed Analytics

**TODO: Document feed usage analytics if available**

## Troubleshooting

**TODO: Common issues:**

### Feed Empty

- Check filter criteria
- Verify channels are being monitored
- Check date range

### Feed Not Updating

- Check platform status
- Verify feed URL
- Check RSS reader settings

### Authentication Issues

- Verify API token
- Check token permissions
- Regenerate token if needed

## Next Steps

After creating your feed:

- [Set up Discord alerts](setup-discord-alerts.md) for real-time notifications
- Create multiple feeds for different intelligence areas
- Share feeds with team members

---

!!! tip "Testing Filters"
    Start with broad filters and progressively narrow them based on the results you see.

!!! note "Documentation Status"
    This page is a placeholder. Content will be generated from RSS ingestor service and frontend feed creator.
