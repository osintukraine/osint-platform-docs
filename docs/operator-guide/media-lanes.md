# Media Lanes and Telegram Embeds

Downloading Telegram media can take seconds to many minutes. In the original design the processor downloaded media *before* saving a message, so one slow video delayed its own text and everything queued behind it. **Media lanes** split the two:

- **Text lane** (processor): saves the message immediately with `media_status = 'pending'`.
- **Media lane** (`media-worker`): downloads the media in the background and updates `media_status`.

Design record: [media lanes plan](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docs/plans/2026-10-21-media-lanes-and-telegram-embeds.md).

## Enabling

1. Apply migration 036 (`./scripts/migrate.sh`). It adds `media_status` and related columns to `messages`.
2. Make sure the `media-worker` service is running (it is part of the core stack, no profile needed). Scale it with `MEDIA_WORKER_REPLICAS` (default 1). `MEDIA_DOWNLOAD_TIMEOUT_S` (default 1800) is the per-file timeout.
3. Turn on the admin setting `media_lane_enabled` (Admin, Config, Platform Settings). The default is **off**, which keeps the old behaviour (download inline, then save).

The media-worker and the processor share one Telegram account, so keep media concurrency modest.

## Media status

| `messages.media_status` | Meaning |
|---|---|
| `NULL` | No media, or archived before the media lane existed |
| `pending` | Waiting for the media-worker |
| `downloading` | Claimed by a media-worker |
| `archived` | Downloaded and stored |
| `failed` | Download failed (retried with backoff) |
| `unavailable` | Post or media is gone at the source |
| `expired` | File removed by media retention |

The `messages` rows are the queue; there is no separate Redis stream, so no job is lost between "message saved" and "job queued". Archived files still flow through [media-sync](../developer-guide/services/media-sync.md) to long-term storage.

## Telegram embeds

When a message has media that is not archived (pending, failed, unavailable, or lost with old storage) and its channel is public, the feed shows the original post as an embedded Telegram post instead of "No media", with a link to open it on Telegram. Embeds load content from `t.me` in the viewer's browser. The reverse proxy must allow framing it: the Caddyfiles in `infrastructure/caddy/` already include `https://t.me` in `frame-src`.

## Checking the queue

```bash
docker compose logs -f media-worker

docker compose exec postgres psql -U osint_user -d osint_platform -c \
  "SELECT media_status, count(*) FROM messages GROUP BY 1 ORDER BY 2 DESC;"
```

A growing `pending` count with a healthy worker usually means Telegram is rate-limiting the account (FloodWait); see [Troubleshooting](troubleshooting.md).
