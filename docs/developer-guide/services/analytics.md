# Analytics Service

## Overview

The Analytics Service provides automated social graph data collection from Telegram, enabling influence mapping, virality analysis, and cross-channel network detection for OSINT analysis.

**Key Responsibilities:**

- **Engagement Polling** - Hourly collection of views, forwards, and reactions
- **Materialized View Refresh** - Daily refresh of pre-computed social graph analytics
- **Channel Metadata Discovery** - Auto-detection of linked discussion groups and reaction capabilities

!!! info "Comment Scraping"
    Comment scraping functionality has been moved to the **Enrichment Service** (`comment_realtime` and `comment_backfill` tasks). See [Enrichment Service](enrichment.md) for details.

**Service Type:** Background scheduler (APScheduler)
**Language:** Python 3.11+
**Key Libraries:** Telethon, asyncpg, APScheduler
**Docker Profile:** `enrichment`

---

## Architecture

### Component Overview

```
Telegram API (Polling)
    ↓
┌────────────────────────────────────────────┐
│   Analytics Service (APScheduler)          │
│                                            │
│   ┌─────────────────────────────────────┐ │
│   │  Engagement Poller                  │ │
│   │  - Poll views/reactions/forwards    │ │
│   │  - Calculate deltas                 │ │
│   │  - Store timeline snapshots         │ │
│   │  Schedule: Hourly at :00            │ │
│   └─────────────────────────────────────┘ │
│                                            │
│   ┌─────────────────────────────────────┐ │
│   │  View Refresher                     │ │
│   │  - Refresh materialized views       │ │
│   │  - Update social graph data         │ │
│   │  Schedule: Daily at 03:00 UTC       │ │
│   └─────────────────────────────────────┘ │
│                                            │
│   ┌─────────────────────────────────────┐ │
│   │  Channel Metadata Updater           │ │
│   │  - Detect discussion groups         │ │
│   │  - Check reaction capabilities      │ │
│   │  Schedule: At startup               │ │
│   └─────────────────────────────────────┘ │
└────────────────────────────────────────────┘
    ↓
PostgreSQL Database
├── message_engagement_timeline (time-series data)
├── message_reactions (emoji reactions)
└── Materialized views (pre-computed analytics)
```

### Key Files

| File | Purpose | Lines |
|------|---------|-------|
| `src/main.py` | Service entry point, scheduler setup | 324 |
| `src/engagement_poller.py` | Hourly engagement metric polling | 421 |
| `src/view_refresher.py` | Daily materialized view refresh | 228 |
| `src/telethon_client.py` | Shared Telegram client (singleton) | 133 |
| `Dockerfile` | Container image definition | 48 |
| `requirements.txt` | Python dependencies | 27 |

---

## Features

### 1. Engagement Polling

**Schedule:** Hourly at :00 (e.g., 13:00, 14:00, 15:00)

**What it does:**

- Polls Telegram API for last 24 hours of messages
- Fetches: `message.views`, `message.forwards`, `message.reactions`
- Calculates deltas (change since last snapshot)
- Stores time-series data in `message_engagement_timeline`
- Updates `message_reactions` table with emoji reaction details

**Key Metrics:**

| Metric | Formula | Purpose |
|--------|---------|---------|
| `propagation_rate` | `forwards / message_age_hours` | Virality indicator (how fast content spreads) |
| `engagement_rate` | `(reactions + comments) / views * 100` | Audience interaction percentage |
| `views_delta` | `current_views - previous_views` | View growth since last poll |
| `forwards_delta` | `current_forwards - previous_forwards` | Forward growth since last poll |

**Reaction Tracking:**

The service implements a two-tier reaction tracking strategy:

1. **Aggregate Counts** (always available)
    - Total count per emoji from `message.reactions.results`
    - Stored with `user_id = NULL` in `message_reactions`

2. **User-Level Reactions** (when available)
    - Individual user reactions from `message.reactions.recent_reactions`
    - Enables influence mapping (who reacts to what)
    - Stored with specific `user_id` in `message_reactions`

!!! tip "Rate Limiting"
    - Batch size: 100 messages per API call (Telegram's limit)
    - Sleep: 1-2 seconds between batches
    - Flood-wait handling: Automatic retry with exponential backoff (max 3 attempts)

### 2. Materialized View Refresh

**Schedule:** Daily at 03:00 UTC

**What it does:**

- Refreshes pre-computed materialized views for fast API queries
- Uses `REFRESH MATERIALIZED VIEW CONCURRENTLY` (non-blocking)
- Logs refresh duration and row counts for monitoring

**Views Refreshed:**

| View | Purpose | Estimated Rows | Refresh Time |
|------|---------|----------------|--------------|
| `message_social_graph` | Per-message network data (author, forwards, reactions) | 10K-100K | 2-3 minutes |
| `channel_influence_network` | Cross-channel forwarding patterns | 1K-10K | 1-2 minutes |
| `top_influencers` | Most influential users/channels | 100-1K | <1 minute |

**Monitoring:**

All refreshes are logged to `materialized_view_refresh_log` table:

```sql
SELECT
    view_name,
    refreshed_at,
    duration_seconds,
    row_count,
    success
FROM materialized_view_refresh_log
ORDER BY refreshed_at DESC
LIMIT 10;
```

### 3. Channel Metadata Discovery

**Schedule:** At service startup

**What it does:**

- Automatically detects linked discussion groups for each channel
- Checks if channels have reactions enabled
- Updates `channels` table with metadata

**Why This Matters:**

Telegram has a two-layer system:

1. **Main Channel** (broadcast) - One-way communication, no comments
2. **Linked Discussion Group** (optional) - Two-way comments

The analytics service auto-discovers discussion groups via Telegram's `GetFullChannelRequest` API, so you only need to monitor the main channel. Comments are then fetched through the enrichment service's comment tasks.

!!! success "No Manual Configuration Required"
    - Monitor main channel only (e.g., `@russia_news`)
    - Discussion group auto-discovered (e.g., `@russia_news_chat`)
    - No need to add discussion groups to `channels` table
    - Avoids doubling monitored entity count

---

## Configuration

### Environment Variables

All configuration is via environment variables (loaded from `.env`):

**Telegram API:**

```bash
TELEGRAM_API_ID=your_api_id           # From https://my.telegram.org
TELEGRAM_API_HASH=your_api_hash       # From https://my.telegram.org
TELEGRAM_SESSION_PATH=/app/sessions   # Session file location
```

**Database:**

```bash
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=osint_platform
POSTGRES_USER=osint_user
POSTGRES_PASSWORD=your_password
```

**Redis (required by shared config):**

```bash
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=your_redis_password
REDIS_DB=0
```

**MinIO (required by shared config):**

```bash
MINIO_ENDPOINT=minio:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
```

**Logging:**

```bash
LOG_LEVEL=INFO                        # DEBUG, INFO, WARNING, ERROR
TZ=UTC                                # Timezone for scheduler
```

### Scheduler Configuration

Jobs are defined in `src/main.py`:

```python
# Engagement polling: Hourly at :00
scheduler.add_job(
    func=engagement_poller.poll_all_channels,
    trigger=CronTrigger(minute=0),
    id="engagement_polling",
    max_instances=1,
    misfire_grace_time=300,  # 5 minutes
)

# View refresh: Daily at 03:00 UTC
scheduler.add_job(
    func=view_refresher.refresh_all_views,
    trigger=CronTrigger(hour=3, minute=0),
    id="view_refresh",
    max_instances=1,
    misfire_grace_time=3600,  # 1 hour
)
```

**Why Staggered?**

- Engagement polling runs at :00 (top of each hour)
- Comment scraping (in enrichment service) runs at :30 (avoids rate limits)
- View refresh runs at 03:00 UTC (low-traffic time, large batch job)

---

## Running Locally

### Prerequisites

1. **Telegram Authentication**

The analytics service shares Telegram sessions with the listener. You must authenticate once:

```bash
# Authenticate via listener (creates session file)
docker-compose exec listener python -m src.telegram_auth

# Session file is shared with analytics at ./telegram_sessions/analytics.session
```

2. **Environment Configuration**

Create `.env` file in project root:

```bash
TELEGRAM_API_ID=12345678
TELEGRAM_API_HASH=abcdef1234567890abcdef1234567890
POSTGRES_PASSWORD=your_secure_password
```

### Docker Compose (Recommended)

The service is part of the `enrichment` profile:

```bash
# Start analytics service
docker-compose --profile enrichment up -d analytics

# View logs
docker-compose logs -f analytics

# Check scheduler status
docker-compose logs analytics | grep "Next scheduled jobs"

# Stop service
docker-compose stop analytics
```

### Local Development

For development without Docker:

```bash
# Navigate to service directory
cd ~/code/osintukraine/osint-intelligence-platform/services/analytics

# Create virtual environment
python3.11 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
pip install -r ../../shared/python/requirements.txt

# Set environment variables
export POSTGRES_HOST=localhost
export POSTGRES_PORT=5432
export POSTGRES_DB=osint_platform
export POSTGRES_USER=osint_user
export POSTGRES_PASSWORD=your_password
export TELEGRAM_API_ID=12345678
export TELEGRAM_API_HASH=your_hash
export TELEGRAM_SESSION_PATH=../../telegram_sessions
export PYTHONPATH="../../shared/python:$PYTHONPATH"

# Run service
python -m src.main
```

**Expected Output:**

```
2025-12-09 13:00:00 - INFO - Starting Analytics Service v0.1.0
2025-12-09 13:00:01 - INFO - Initializing Analytics Service...
2025-12-09 13:00:02 - INFO - Database pool created successfully
2025-12-09 13:00:03 - INFO - Telethon client initialized
2025-12-09 13:00:05 - INFO - Updating channel metadata (detecting discussion groups)...
2025-12-09 13:00:15 - INFO - Channel metadata updated
2025-12-09 13:00:16 - INFO - Scheduler started with 2 jobs:
2025-12-09 13:00:16 - INFO -   - Engagement polling: Hourly at :00
2025-12-09 13:00:16 - INFO -   - View refresh: Daily at 03:00 UTC
2025-12-09 13:00:17 - INFO - Analytics Service running...
2025-12-09 13:00:17 - INFO - Next scheduled jobs:
2025-12-09 13:00:17 - INFO -   - Engagement Polling: 2025-12-09 14:00:00+00:00
2025-12-09 13:00:17 - INFO -   - Materialized View Refresh: 2025-12-10 03:00:00+00:00
```

---

## Database Schema

### Tables Used

**`message_engagement_timeline`** - Time-series engagement snapshots

```sql
CREATE TABLE message_engagement_timeline (
    id BIGSERIAL PRIMARY KEY,
    message_id BIGINT NOT NULL REFERENCES messages(id),

    -- Snapshot metadata
    snapshot_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    -- Raw metrics
    views_count INTEGER,
    forwards_count INTEGER,
    reactions_count INTEGER,
    comments_count INTEGER,

    -- Deltas (change since last snapshot)
    views_delta INTEGER,
    forwards_delta INTEGER,
    reactions_delta INTEGER,

    -- Derived metrics
    propagation_rate NUMERIC(10, 2),  -- Forwards per hour
    engagement_rate NUMERIC(5, 2)     -- (reactions+comments)/views * 100
);
```

**`message_reactions`** - Emoji reaction tracking

```sql
CREATE TABLE message_reactions (
    id BIGSERIAL PRIMARY KEY,
    message_id BIGINT NOT NULL REFERENCES messages(id),

    -- Reaction details
    emoji VARCHAR(20) NOT NULL,       -- 👍, 👎, ❤️, 🔥, etc.
    count INTEGER DEFAULT 1,          -- Aggregate count or 1 for individual

    -- Optional: Individual user reactions
    user_id BIGINT REFERENCES telegram_users(id),
    reacted_at TIMESTAMP WITH TIME ZONE,

    -- Timestamps
    first_seen TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_updated TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    UNIQUE(message_id, emoji, user_id)
);
```

**`channels`** - Updated with analytics metadata

```sql
ALTER TABLE channels ADD COLUMN IF NOT EXISTS
    linked_discussion_channel_id BIGINT,      -- Telegram ID of discussion group
    has_reactions_enabled BOOLEAN DEFAULT FALSE;  -- Reactions capability
```

### Materialized Views

**`message_social_graph`** - Per-message network data

Pre-computed view combining message data, author info, forward chains, and engagement metrics for fast API queries.

**`channel_influence_network`** - Cross-channel patterns

Tracks which channels forward to which channels, enabling influence mapping and coordination detection.

**`top_influencers`** - Influential users/channels

Ranked by interaction count, reach, and engagement for trending analysis.

---

## Monitoring

### Logs

**Important Log Events:**

```bash
# Successful job completion
INFO - Completed job: engagement_polling in 323.45s

# Flood-wait warnings (should be rare)
WARNING - Flood-wait 30s (attempt 1/3)

# Channel access errors (expected for private channels)
WARNING - Cannot access channel @username: ChannelPrivateError

# View refresh completion
INFO - Refreshed message_social_graph: 12,345 rows in 153.21s
```

**View Logs:**

```bash
# Real-time logs
docker-compose logs -f analytics

# Last 100 lines
docker-compose logs --tail=100 analytics

# Search for errors
docker-compose logs analytics | grep ERROR

# Check scheduler status
docker-compose logs analytics | grep "Next scheduled jobs"
```

### Database Monitoring

**Check Refresh History:**

```sql
SELECT
    view_name,
    refreshed_at,
    duration_seconds,
    row_count,
    success
FROM materialized_view_refresh_log
ORDER BY refreshed_at DESC
LIMIT 10;
```

**Check Latest Engagement Snapshots:**

```sql
SELECT
    m.message_id,
    c.name as channel,
    et.snapshot_at,
    et.views_count,
    et.views_delta,
    et.propagation_rate,
    et.engagement_rate
FROM message_engagement_timeline et
JOIN messages m ON m.id = et.message_id
JOIN channels c ON c.id = m.channel_id
ORDER BY et.snapshot_at DESC
LIMIT 20;
```

**Check Reaction Distribution:**

```sql
SELECT
    emoji,
    COUNT(*) as message_count,
    SUM(count) as total_reactions
FROM message_reactions
WHERE user_id IS NULL  -- Aggregate reactions only
GROUP BY emoji
ORDER BY total_reactions DESC;
```

### Performance Metrics

**Expected Load:**

| Job | Channels | Messages | Runtime |
|-----|----------|----------|---------|
| Engagement Polling | 254 | ~500 per channel | 1-2 hours |
| View Refresh | All data | 10K-100K rows | 5-10 minutes |

**Database Impact:**

- **Writes:** ~1,000-2,000 rows/hour (engagement snapshots)
- **Storage Growth:** ~100MB/month (time-series data)
- **Read Load:** Minimal (only during view refresh)

**Telegram API Usage:**

- **Requests:** ~5,000-10,000/day
- **Rate:** ~3-5 requests/minute (well under 20/sec limit)
- **Flood-waits:** Rare with proper sleep timing

---

## Troubleshooting

### Service Won't Start

**Check Telegram Authentication:**

```bash
docker-compose exec analytics python -c "
from telethon import TelegramClient
from config.settings import settings
import asyncio

async def check():
    client = TelegramClient('test', settings.TELEGRAM_API_ID, settings.TELEGRAM_API_HASH)
    await client.connect()
    print('Authorized:', await client.is_user_authorized())
    await client.disconnect()

asyncio.run(check())
"
```

**Check Database Connectivity:**

```bash
docker-compose exec analytics python -c "
import asyncio
import asyncpg
from config.settings import settings

async def check():
    try:
        conn = await asyncpg.connect(
            host=settings.POSTGRES_HOST,
            port=settings.POSTGRES_PORT,
            database=settings.POSTGRES_DB,
            user=settings.POSTGRES_USER,
            password=settings.POSTGRES_PASSWORD,
        )
        result = await conn.fetchval('SELECT 1')
        print('Database connected:', result == 1)
        await conn.close()
    except Exception as e:
        print('Database error:', e)

asyncio.run(check())
"
```

### Jobs Not Running

**Verify Scheduler:**

```bash
# Check scheduler started
docker-compose logs analytics | grep "Scheduler started"

# Check next scheduled jobs
docker-compose logs analytics | grep "Next scheduled jobs"

# Check for job execution
docker-compose logs analytics | grep "Starting job"
```

**Check Container Time:**

```bash
# Scheduler uses UTC - verify container time
docker-compose exec analytics date -u
```

### Rate Limiting / Flood-Wait

**Check for Flood-Wait Errors:**

```bash
docker-compose logs analytics | grep "FloodWait"
```

**Increase Sleep Between Batches:**

Edit `src/engagement_poller.py` line ~170:

```python
# Increase from 1 to 2 seconds
await asyncio.sleep(2)
```

Rebuild container:

```bash
docker-compose build analytics
docker-compose up -d analytics
```

### Missing Engagement Data

**Check Message Age:**

The service only polls messages from the last 24 hours. Older messages won't get new snapshots.

```sql
-- Verify recent messages exist
SELECT COUNT(*)
FROM messages
WHERE telegram_date >= NOW() - INTERVAL '24 hours'
AND is_spam = false;
```

**Check Channel Activity:**

```sql
-- Verify channels have linked_discussion_channel_id
SELECT
    name,
    username,
    linked_discussion_channel_id,
    has_reactions_enabled
FROM channels
WHERE active = true
ORDER BY last_message_at DESC;
```

---

## Performance Optimization

### Reduce Polling Frequency

For fewer active channels, you can reduce polling frequency:

Edit `src/main.py` scheduler configuration:

```python
# Poll every 2 hours instead of hourly
trigger=CronTrigger(hour='*/2', minute=0)

# Or poll every 6 hours
trigger=CronTrigger(hour='0,6,12,18', minute=0)
```

### Batch Size Tuning

Telegram API limit is 100 messages per request. If you have many small channels:

Edit `src/engagement_poller.py` line ~144:

```python
# Reduce batch size for more frequent sleep breaks
batch_size = 50  # Default: 100
```

### View Refresh Optimization

For large deployments (>100K messages), consider:

1. **Partial Refresh** - Only refresh views for recent data
2. **Separate Schedules** - Refresh views at different times
3. **Incremental Updates** - Use regular views instead of materialized views

---

## Security Considerations

### Credentials

- **Telegram Session Files:** Stored in `./telegram_sessions/` (persistent volume)
- **Database Password:** Via environment variable (never hardcoded)
- **API Keys:** In `.env` file (git-ignored)

!!! warning "Session File Security"
    Session files grant full access to the Telegram account. Protect them:

    ```bash
    # Secure permissions
    chmod 600 telegram_sessions/analytics.session

    # Backup regularly
    cp telegram_sessions/analytics.session telegram_sessions/analytics.session.backup
    ```

### Access Control

- Service uses **read-only** Telegram account (cannot post/delete)
- Database user has INSERT/UPDATE permissions only
- No external API endpoints (internal service only)

### Rate Limit Compliance

The service respects Telegram's rate limits:

- 20 requests/second global limit
- Automatic flood-wait retry with exponential backoff
- Staggered job schedules to spread API load

---

## Related Documentation

- [Enrichment Service](enrichment.md) - Comment scraping tasks
- [Database Schema](../../reference/database-tables.md) - Full schema reference
- [API Documentation](api.md) - Querying analytics data
- [Architecture Overview](../architecture.md) - Overall platform architecture

---

## Future Enhancements

### Planned Features

1. **Prometheus Metrics** - Export job duration, error rates, API call counts
2. **Viral Content Alerts** - Notify via notifier service when `propagation_rate` exceeds threshold
3. **Coordination Detection** - Identify synchronized forwarding patterns across channels
4. **Network Graph Export** - Export data for Gephi/Cytoscape visualization

### Research Ideas

1. **Sentiment Analysis** - Analyze comment sentiment with LLM integration
2. **Influence Scoring** - Rank users by reach, engagement, and network position
3. **Topic Clustering** - Group influencers by topic affinity
4. **Bot Detection** - Identify automated accounts via engagement patterns

---

**Last Updated:** 2025-12-09
**Service Version:** v0.1.0
**Status:** Production-ready
