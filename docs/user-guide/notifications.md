# Notifications & Alerts

Configure real-time notifications for critical intelligence via ntfy.sh mobile and desktop push notifications.

## Overview

The platform uses **ntfy.sh**, a simple HTTP-based notification service, to deliver real-time alerts about high-priority intelligence without requiring account creation or complex setup.

### Key Features

- **No Account Required** - Just subscribe to a topic
- **Multi-Platform** - iOS, Android, Web, Desktop
- **Free & Open Source** - Self-hostable notification service
- **Topic-Based** - Subscribe to specific intelligence categories
- **Priority Levels** - Control notification urgency
- **Offline Queue** - Receive missed alerts when back online

### What ntfy.sh Is

ntfy (pronounced "notify") is a lightweight pub/sub notification service:

1. **Publisher** - OSINT platform sends notifications to ntfy topics
2. **Subscriber** - You subscribe to topics via app or web
3. **Delivery** - ntfy pushes notifications to your devices

No authentication, no tracking, no login - just topics.

## Getting Started

### Install ntfy App

**iOS:**
```
1. Open App Store
2. Search "ntfy"
3. Install "ntfy - PUT/POST to your phone"
4. Open app
```

**Android:**
```
1. Open Google Play Store
2. Search "ntfy"
3. Install "ntfy - PUT/POST to your phone"
4. Open app
```

**Desktop (Web):**
```
Visit: https://ntfy.sh
No installation needed
```

**Desktop (Native):**
- macOS: Download from https://ntfy.sh
- Windows: Download from https://ntfy.sh
- Linux: `sudo snap install ntfy`

### Subscribe to Topics

The platform publishes to these ntfy topics:

#### High-Priority Intelligence

**Topic:** `osint-ukraine-high`
**URL:** `https://ntfy.sh/osint-ukraine-high`

**Contains:**
- Combat reports (importance: high)
- Equipment losses
- Significant military movements
- Critical infrastructure strikes

**Recommended for:**
- Active analysts
- Situation room monitoring
- Urgent intelligence needs

**Average volume:** 10-30 notifications/day

#### Equipment Mentions

**Topic:** `osint-ukraine-equipment`
**URL:** `https://ntfy.sh/osint-ukraine-equipment`

**Contains:**
- T-90 tank mentions
- HIMARS strikes
- Aircraft losses
- Artillery systems

**Recommended for:**
- Military equipment analysts
- OSINT researchers
- Battle damage assessment

**Average volume:** 20-50 notifications/day

#### Sanctioned Entities

**Topic:** `osint-ukraine-sanctions`
**URL:** `https://ntfy.sh/osint-ukraine-sanctions`

**Contains:**
- Mentions of OpenSanctions entities
- Politically exposed persons (PEPs)
- Sanctioned individuals
- Restricted organizations

**Recommended for:**
- Sanctions compliance
- Financial intelligence
- Political analysis

**Average volume:** 5-15 notifications/day

#### Combat Reports

**Topic:** `osint-ukraine-combat`
**URL:** `https://ntfy.sh/osint-ukraine-combat`

**Contains:**
- Frontline updates
- Casualty reports
- Territorial changes
- Tactical operations

**Recommended for:**
- Military analysts
- Conflict monitoring
- Tactical intelligence

**Average volume:** 30-70 notifications/day

#### All Messages (High Volume)

**Topic:** `osint-ukraine-all`
**URL:** `https://ntfy.sh/osint-ukraine-all`

**Contains:**
- Every message above spam threshold
- All importance levels
- All topics

**Recommended for:**
- Comprehensive monitoring
- Testing/development
- Archival purposes

**Average volume:** 100-300+ notifications/day
**Warning:** Very high volume, may drain battery

### Subscribing in ntfy App

**iOS/Android:**
```
1. Open ntfy app
2. Tap "+" or "Add subscription"
3. Enter topic name (e.g., "osint-ukraine-high")
4. Tap "Subscribe"
5. Enable notifications when prompted
```

**Web:**
```
1. Visit https://ntfy.sh
2. Enter topic in search box
3. Click "Subscribe"
4. Allow browser notifications
```

**Direct Link Method:**
```
Just visit:
https://ntfy.sh/osint-ukraine-high

Click "Subscribe" button
```

## Notification Format

### Message Structure

Each notification contains:

**Title:**
```
[Importance] Channel Name
Example: [HIGH] Ukraine Weapons Tracker
```

**Body:**
```
Message content (truncated to 200 chars)
Example: "Ukrainian forces report destruction of T-90M tank near Bakhmut. Video confirmation..."
```

**Tags:**
- Topic emoji (⚔️ combat, 🛡️ equipment, etc.)
- Importance indicator (🔴 high, 🟡 medium)

**Actions (Clickable):**
- **View Message** - Opens message detail page
- **View Channel** - Opens Telegram channel

**Priority Levels:**
- **High** (5) - Importance: high messages
- **Default** (3) - Importance: medium messages
- **Low** (1) - Importance: low messages, background updates

### Priority Behavior

**High Priority (5):**
- Sound + vibration (even in Do Not Disturb on some devices)
- Notification stays visible
- Red badge indicator

**Default Priority (3):**
- Standard notification sound
- Normal visibility
- Standard behavior

**Low Priority (1):**
- Silent notification
- May be grouped/minimized
- No sound/vibration

## Managing Notifications

### Per-Topic Settings

**iOS:**
```
1. Long-press notification
2. Tap "Settings"
3. Adjust:
   - Sound
   - Alert style
   - Show previews
   - Badge app icon
```

**Android:**
```
1. Long-press notification
2. Tap gear icon
3. Adjust:
   - Sound
   - Vibration
   - Importance level
   - Show badge
```

**Web:**
```
Browser notification settings
Control per-site permissions
```

### Muting Topics

**Temporary (iOS/Android):**
```
1. Open ntfy app
2. Find subscription
3. Tap to open
4. Tap bell icon
5. Select "Mute for X hours"
```

**Permanent:**
```
1. Swipe left on subscription (iOS)
2. Tap "Delete"

OR

1. Long-press subscription (Android)
2. Tap "Delete"
```

### Adjusting Volume

**High-volume topics** (e.g., `osint-ukraine-all`):

**Option 1: Reduce notification importance**
```
iOS: Notification > Settings > Alerts > Banners
Android: Long-press > Settings > Importance > Low
```

**Option 2: Disable sound**
```
Keep notifications but silent
Review at your convenience
```

**Option 3: Unsubscribe and use RSS**
```
Subscribe to RSS feed instead
Check manually at intervals
See [RSS Feeds guide](rss-feeds.md)
```

## Advanced Configuration

### Custom Filters (Server-Side)

**Note:** Currently not user-configurable. Contact admin to request custom notification rules.

**Potential filters:**
- Specific channels only
- Minimum engagement threshold
- Specific entity mentions
- Geographic keywords
- Time-based rules

### Self-Hosted ntfy Server

For enhanced privacy or custom rules:

**Steps:**
```
1. Deploy ntfy server (Docker/binary)
2. Configure OSINT platform to use your ntfy URL
3. Subscribe to topics on your server
4. Full control over retention, rate limits
```

**Benefits:**
- Private notifications
- Custom retention policies
- No third-party dependencies
- Advanced authentication

**See:** [Operations Guide](../operations/notifications.md) for admin setup

### Integration with Other Tools

**Zapier/IFTTT:**
```
1. Subscribe to ntfy topic
2. Create webhook automation
3. Forward to Slack, Discord, email, etc.
```

**Custom Scripts:**
```bash
# Subscribe to topic programmatically
curl -s https://ntfy.sh/osint-ukraine-high/raw | while read msg; do
  echo "Received: $msg"
  # Custom processing
done
```

**Discord Bot:**
```javascript
// Forward ntfy notifications to Discord
const fetch = require('node-fetch');

fetch('https://ntfy.sh/osint-ukraine-high/raw')
  .then(res => res.text())
  .then(msg => {
    // Post to Discord webhook
  });
```

## Notification Topics Reference

| Topic | Content | Priority | Volume/Day |
|-------|---------|----------|------------|
| `osint-ukraine-high` | High-priority intelligence | High (5) | 10-30 |
| `osint-ukraine-equipment` | Equipment mentions | Default (3) | 20-50 |
| `osint-ukraine-sanctions` | Sanctioned entities | Default (3) | 5-15 |
| `osint-ukraine-combat` | Combat reports | High (5) | 30-70 |
| `osint-ukraine-all` | All messages | Varies | 100-300+ |

**Topic Naming Convention:**
```
osint-{region}-{category}

Examples:
- osint-ukraine-high
- osint-syria-combat
- osint-global-sanctions
```

## Best Practices

### Effective Notification Management

**Start Selective:**
```
1. Subscribe to osint-ukraine-high only
2. Evaluate volume over 24 hours
3. Add more topics as needed
4. Remove if overwhelmed
```

**Use Multiple Devices:**
```
Work phone: osint-ukraine-high (critical)
Personal phone: osint-ukraine-equipment (interest)
Desktop: osint-ukraine-all (comprehensive)
```

**Time-Based Subscriptions:**
```
Working hours: All topics
Evening: High-priority only
Night: Muted/disabled
Weekends: Email digest instead (via RSS)
```

### Reducing Notification Fatigue

**Symptoms:**
- Ignoring notifications
- Feeling overwhelmed
- Missing important alerts

**Solutions:**
1. **Reduce topics** - Unsubscribe from high-volume
2. **Increase thresholds** - Request higher importance filtering
3. **Use RSS instead** - Pull-based vs. push
4. **Schedule review times** - Batch notifications
5. **Mute during focus time** - DND mode

### Privacy Considerations

**What ntfy.sh knows:**
- Topics you subscribe to
- Your IP address
- Notification delivery success

**What ntfy.sh DOESN'T know:**
- Your identity (no account)
- Your device details (end-to-end)
- Message content (encrypted in transit)

**For maximum privacy:**
- Use VPN when subscribing
- Self-host ntfy server
- Use desktop app (not web)
- Rotate topics periodically

## Troubleshooting

### Not Receiving Notifications

**Check app permissions:**
```
iOS: Settings > Notifications > ntfy > Allow Notifications
Android: Settings > Apps > ntfy > Notifications > Enabled
```

**Verify subscription:**
```
1. Open ntfy app
2. Check subscriptions list
3. Verify topic name is exact
4. Re-subscribe if needed
```

**Test with manual send:**
```bash
# Send test notification
curl -d "Test message" https://ntfy.sh/osint-ukraine-high
```

**Platform issues:**
- Check if other apps send notifications
- Restart device
- Reinstall ntfy app
- Try web version

### Delayed Notifications

**Reasons:**
- Mobile device in low-power mode
- Network connectivity issues
- ntfy.sh server overload (rare)
- App backgrounded by OS

**Solutions:**
- Keep app open (web)
- Disable battery optimization for ntfy (Android)
- Check network connection
- Wait a few minutes (queue processing)

### Duplicate Notifications

**Causes:**
- Subscribed to same topic on multiple devices (intended)
- Subscribed twice in same app (bug)
- Overlapping topics (e.g., both `high` and `all`)

**Fix:**
```
1. Review subscriptions
2. Delete duplicates
3. Unsubscribe from redundant topics
```

### Notification Sound Not Playing

**iOS:**
```
1. Check ringer switch (physical)
2. Settings > Sounds & Haptics > Volume
3. Settings > Notifications > ntfy > Sounds
```

**Android:**
```
1. Settings > Sound > Notification volume
2. Long-press notification > Settings > Sound
3. Check Do Not Disturb exceptions
```

## Alternative Notification Methods

If ntfy doesn't meet your needs:

### Email Notifications (via RSS-to-Email)

**Services:**
- Blogtrottr (free)
- FeedMyInbox (free)
- Feedburner (Google, free)

**Steps:**
```
1. Create RSS feed with filters (see RSS Feeds guide)
2. Sign up for RSS-to-email service
3. Add feed URL
4. Receive digest emails
```

### Slack/Discord Integration

**Webhook Setup:**
```
1. Create incoming webhook in Slack/Discord
2. Set up intermediary service (Zapier, custom bot)
3. Subscribe to ntfy topic programmatically
4. Forward to webhook
```

**Pre-built Solutions:**
- ntfy-to-slack (GitHub)
- ntfy-discord-bridge (GitHub)

### Telegram Bot

**Direct Messages:**
```
1. Request Telegram bot integration (admin)
2. Add bot to your Telegram
3. Receive DMs for notifications
```

**Channel Forwarding:**
```
Already implemented: High-priority messages
automatically forwarded to specific Telegram channels
(admin-configured)
```

## Future Enhancements

**Planned features** (check roadmap for status):

- User-configurable notification rules
- Geo-fencing (notifications for specific regions)
- Entity-specific alerts (follow specific people/equipment)
- Severity escalation (repeated mentions = higher priority)
- Quiet hours (auto-mute during specified times)
- Notification analytics (what you clicked, engagement)

---

**Next Steps:**
- [Subscribe to RSS Feeds](rss-feeds.md) for pull-based monitoring
- [Search Messages](searching.md) to find content for custom alerts
- [Explore Entities](entities.md) to track specific mentions
