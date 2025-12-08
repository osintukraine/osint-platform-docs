# Tutorial: Setup Discord Alerts

Configure real-time Discord notifications for critical intelligence.

## Learning Objectives

By the end of this tutorial, you will:

- Create a Discord webhook
- Configure notification rules in the platform
- Test alert delivery
- Set up advanced alert routing

## Prerequisites

- Platform installed and running
- Discord server with admin permissions
- Channels being monitored
- Basic understanding of Discord webhooks

## Estimated Time

20-30 minutes

## Step 1: Create Discord Webhook

**TODO: Content to be generated from codebase analysis**

1. Open Discord server
2. Right-click on target channel
3. Select "Edit Channel"
4. Go to "Integrations"
5. Click "Create Webhook"
6. Name the webhook (e.g., "OSINT Alerts")
7. Copy webhook URL

### Webhook URL Format

```
https://discord.com/api/webhooks/{webhook_id}/{webhook_token}
```

## Step 2: Configure Notification Service

**TODO: Add screenshots and step-by-step instructions:**

1. Navigate to Admin → Notifications
2. Click "Add Discord Webhook"
3. Paste webhook URL
4. Name the notification channel
5. Test connection

## Step 3: Create Notification Rule

**TODO: Document notification rule creation:**

### Rule Configuration

1. Click "Create Rule"
2. Configure trigger conditions
3. Select notification destination
4. Configure message template

### Trigger Conditions

**TODO: Document available triggers:**

- Channel-based triggers
- Entity mention triggers
- Tag-based triggers
- Keyword triggers
- Threat level triggers
- Sentiment triggers

## Step 4: Configure Message Template

**TODO: Document template syntax:**

### Basic Template

```
**New {tag} Alert**

Channel: {channel_name}
Time: {timestamp}

{message_text}

[View Message]({message_url})
```

### Template Variables

**TODO: List available variables:**

- `{channel_name}` - Channel username
- `{message_text}` - Message content
- `{timestamp}` - Message timestamp
- `{message_url}` - Link to message
- `{entity_names}` - Mentioned entities
- `{tags}` - AI tags
- `{threat_level}` - Threat assessment

### Advanced Templates

**TODO: Provide advanced template examples with embeds**

## Step 5: Configure Alert Routing

**TODO: Document routing configuration:**

### Priority-Based Routing

- Critical alerts → `#critical-intel` channel
- High priority → `#high-priority` channel
- Medium priority → `#monitoring` channel
- Low priority → `#low-priority` channel

### Topic-Based Routing

- Military intelligence → `#military` channel
- Political developments → `#political` channel
- Economic news → `#economic` channel

## Step 6: Set Up Rate Limiting

**TODO: Document rate limiting configuration:**

### Why Rate Limit?

- Prevent notification spam
- Avoid Discord rate limits
- Ensure important alerts stand out

### Rate Limit Options

- Maximum alerts per minute
- Maximum alerts per hour
- Deduplication window
- Grouping similar alerts

## Step 7: Test Notifications

**TODO: Add testing steps:**

### Manual Test

1. Click "Send Test Alert"
2. Verify alert appears in Discord
3. Check message formatting
4. Verify links work

### Live Test

1. Wait for matching message
2. Verify alert is triggered
3. Check alert content
4. Verify routing is correct

## Step 8: Monitor Alert Performance

**TODO: Document monitoring:**

1. Check alert delivery rate
2. Monitor failed deliveries
3. Review alert frequency
4. Adjust rules as needed

## Advanced Configuration

**TODO: Document advanced features:**

### Alert Aggregation

Group similar alerts to reduce noise:

- Time window: 5 minutes
- Group by: channel, entity, or tag
- Summary format

### Alert Enrichment

Add context to alerts:

- Entity background information
- Historical mentions
- Related messages
- Social graph connections

### Conditional Formatting

**TODO: Document conditional formatting based on severity:**

- Critical: Red embed
- High: Orange embed
- Medium: Yellow embed
- Low: Blue embed

## Example Notification Rules

**TODO: Provide concrete examples:**

### Example 1: High-Priority Entity Mentions

**Trigger:**
- Entity: "Wagner Group"
- Channels: All Archive-Russia
- Threat level: High or Critical

**Destination:** `#critical-intel`

**Template:**
```
🚨 **CRITICAL: Wagner Group Mention**

Channel: {channel_name}
Entities: {entity_names}

{message_text}

[View Details]({message_url})
```

### Example 2: Equipment Loss Reports

**Trigger:**
- Tags: military, equipment, losses
- Channels: Archive-Russia
- Keywords: "destroyed", "lost", "damaged"

**Destination:** `#military`

### Example 3: Government Announcements

**Trigger:**
- Tags: political, announcement
- Channels: Monitor-Ukraine-Government

**Destination:** `#political`

## Managing Notifications

**TODO: Document notification management:**

### Editing Rules

1. Navigate to notification rules
2. Select rule to edit
3. Modify configuration
4. Save changes

### Disabling Rules

**TODO: Add disable/enable instructions**

### Deleting Rules

**TODO: Add deletion instructions**

## Troubleshooting

**TODO: Common issues:**

### Alerts Not Appearing

- Verify webhook URL is correct
- Check Discord channel permissions
- Review notification rule conditions
- Check platform logs

### Too Many Alerts

- Increase rate limiting
- Narrow trigger conditions
- Use alert aggregation
- Adjust threat level threshold

### Missing Information in Alerts

- Check template variables
- Verify data is being enriched
- Review message processing logs

### Discord Rate Limiting

- Reduce alert frequency
- Use alert grouping
- Distribute across multiple webhooks

## Next Steps

After setting up Discord alerts:

- Create additional rules for different intelligence types
- Set up Telegram notifications as backup
- Configure alert escalation policies
- Monitor and refine rules based on feedback

---

!!! warning "Rate Limits"
    Discord enforces webhook rate limits. Use alert grouping and rate limiting to avoid hitting these limits.

!!! tip "Testing First"
    Always test new rules with a dedicated test channel before deploying to production channels.

!!! note "Documentation Status"
    This page is a placeholder. Content will be generated from notification service code and configuration.
