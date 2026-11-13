# Notifications & Alerts

Receive push notifications from the platform on your phone or desktop through its built-in ntfy server.

## Overview

The platform includes a **notifier** service that sends events to a self-hosted [ntfy](https://ntfy.sh) server. ntfy is a simple publish/subscribe service: the platform publishes to *topics*, and you subscribe to the topics you care about in the ntfy app or web page.

!!! info "Self-hosted, not ntfy.sh"
    Notifications go to **your platform's own ntfy server**, not the public ntfy.sh service. Ask your administrator for its address (for example `https://your-domain/ntfy` or `http://<host>:8090`), and whether the `monitoring` or `notifications` profile is running, because ntfy is only deployed with one of those.

### What You Get

- **Platform events**: new data, media, spam, channel discovery, Telegram connection changes, backfill progress
- **Health and error alerts**: unhealthy containers, database or storage errors, Prometheus alerts
- **High-importance messages**: messages classified as high importance are sent immediately

!!! note "Notifications are operational, not an intelligence feed"
    There are no per-theme topics such as combat reports or equipment mentions. To follow intelligence on a theme (a topic, channel, entity or keyword), create a filtered [RSS feed](rss-feeds.md) instead.

## Subscribing

### Install the ntfy App

- **iOS / Android**: install "ntfy" from the App Store or Google Play
- **Desktop**: open your platform's ntfy web address in a browser, or install a desktop client from [ntfy.sh](https://ntfy.sh)

### Add Your Server and Topics

In the app, set the **default server** to your platform's ntfy address (Settings), then tap **+** and enter a topic name. If your administrator enabled ntfy authentication, they will give you credentials.

You can also subscribe by visiting `<ntfy address>/<topic>` in a browser.

## Topics

Topic names are flat identifiers. The main ones:

| Topic | Content | Delivery |
|-------|---------|----------|
| `osint_data_archived` | Messages archived or received (includes the high-importance alerts) | Batched, high-importance sent immediately |
| `osint_data_media` | Media downloaded or deduplicated | Batched |
| `osint_data_quarantine` | Messages sent to quarantine | Batched |
| `osint_data_errors` | Database, Redis, storage and media archival failures | Immediate |
| `osint_processing_spam` | Spam detections | Batched |
| `osint_processing_entities` | Entity extraction | Batched |
| `osint_processing_llm` | LLM requests and errors | Batched, errors immediate |
| `osint_telegram_discovery` | Channels discovered or removed | Batched |
| `osint_telegram_listener` | Telegram session connected or disconnected, flood waits | Disconnects immediate |
| `osint_telegram_backfill` | Backfill started, progress, completed, failed | Batched, failures immediate |
| `osint_system_containers` | Container started, stopped or unhealthy | Unhealthy immediate |
| `osint_system_resources` | High CPU, memory or disk | Batched |
| `osint_api_errors` | API 4xx and 5xx errors | 5xx immediate |
| `osint_api_rss` | RSS feeds generated or subscribed | Batched |
| `osint_prometheus_alerts` | Alerts from the monitoring stack | By severity |

Start with `osint_telegram_listener`, `osint_data_errors` and `osint_prometheus_alerts` if you only want to know when something breaks. Subscribe to `osint_data_archived` only if you want to see content arrive; it is high volume.

!!! tip "Which messages count as high importance?"
    By default only messages classified as **high** importance are pushed immediately. Your administrator can change the minimum level and switch individual notification types on or off in the admin settings.

## Notification Format

Each notification has a title, a short body and tags. The ntfy priority tells your device how loudly to alert:

| Priority | Used for | Delivery |
|----------|----------|----------|
| Urgent (5) | Unhealthy containers, Telegram disconnected, database/storage/Redis errors, critical Prometheus alerts | Sent immediately |
| High (4) | API 5xx errors, failed media archival or backfill, flood waits, LLM errors, high-importance messages, warning alerts | Sent immediately |
| Default (3) | Routine events | Combined into a summary every 5 minutes (up to 50 per batch) |

Routine events are batched so a busy channel does not flood your phone.

## Managing Notifications

Use the ntfy app's own controls:

- **Mute** a subscription temporarily from its bell icon
- **Delete** a subscription to stop receiving it
- Change sound, vibration and importance per topic in your device's notification settings for ntfy

If a topic is too noisy, unsubscribe and use an [RSS feed](rss-feeds.md) instead, which you can read on your own schedule.

## Integrations

Anything that can read an ntfy topic can forward it:

```bash
# Stream a topic from your platform's ntfy server
curl -s https://<ntfy address>/osint_telegram_listener/raw
```

To post alerts into Discord or Slack, the platform has no built-in integration; the [Discord alerts tutorial](../tutorials/setup-discord-alerts.md) shows one approach using an RSS feed and your own n8n instance.

## Troubleshooting

### Not Receiving Notifications

- Confirm the app's default server is your platform's ntfy address, not ntfy.sh
- Check the topic name is exact (underscores, not hyphens)
- Check the app has notification permission on your device
- Ask your administrator to confirm the `notifier` and `ntfy` services are running and notifications are enabled in admin settings

### Delayed Notifications

- Routine events are batched into 5-minute summaries by design
- Mobile devices in low-power mode may delay delivery; disable battery optimization for the ntfy app (Android)

### Too Many Notifications

- Unsubscribe from `osint_data_archived`, `osint_data_media` and other batched topics
- Ask your administrator to raise the minimum importance or disable specific notification types

---

**Next Steps:**
- [Subscribe to RSS Feeds](rss-feeds.md) for intelligence monitoring
- [Search Messages](searching.md) to build the filters for a feed
- [Explore Entities](entities.md) to track specific mentions
