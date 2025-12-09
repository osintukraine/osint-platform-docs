# Listener Service

> Real-time Telegram channel monitoring service - entry point for all message ingestion

## Overview

The Listener Service is the entry point of the OSINT Intelligence Platform pipeline. It monitors Telegram channels for new messages, enriches them with metadata, and pushes them to Redis for downstream processing.

**Pipeline Position**: `Telegram API → Listener → Redis Streams → Processor → Database`

**Key Technologies**:

- **Telethon**: Telegram MTProto API client
- **Redis Streams**: Message queue with consumer groups
- **asyncio**: High-performance async I/O
- **Prometheus**: Metrics and monitoring

**Core Responsibilities**:

1. Monitor 254+ Telegram channels via single session
2. Extract social graph metadata (forwards, replies, authors)
3. Handle media albums (grouped photos/videos)
4. Push raw messages to Redis queue
5. Discover channels from Telegram folders
6. Historical backfill for new channels

## Architecture

### Component Diagram

```mermaid
graph TB
    subgraph "Telegram API"
        TG[Telegram MTProto]
    end

    subgraph "Listener Service"
        MAIN[main.py<br/>Entry Point]
        TL[TelegramListener<br/>Message Monitor]
        CD[ChannelDiscovery<br/>Folder Sync]
        BS[BackfillService<br/>Historical Fetch]
        RQ[RedisQueue<br/>Stream Client]

        MAIN --> TL
        MAIN --> CD
        MAIN --> BS
        TL --> RQ
        BS --> RQ
    end

    subgraph "External Services"
        REDIS[(Redis Streams)]
        PG[(PostgreSQL)]
        PROM[Prometheus]
    end

    TG -.->|Events| TL
    TG -.->|GetDialogFilters| CD
    TG -.->|iter_messages| BS

    RQ -->|XADD| REDIS
    CD -->|Sync| PG
    TL -->|Metrics| PROM

    style TL fill:#e1f5ff
    style CD fill:#fff3e0
    style BS fill:#f3e5f5
```

### Key Files

| File | Purpose | Lines |
|------|---------|-------|
| `main.py` | Service entry point, initialization, signal handling | 208 |
| `telegram_listener.py` | Core message monitoring, event handlers | 855 |
| `channel_discovery.py` | Folder-based channel discovery and sync | 567 |
| `backfill_service.py` | Historical message fetching | 677 |
| `redis_queue.py` | Redis Streams client for message queue | 262 |
| `social_graph_utils.py` | Extract forwards, replies, author metadata | 306 |
| `media_utils.py` | Media type detection utility | 58 |
| `metrics.py` | Prometheus metrics definitions | 421 |

### Session Management

!!! danger "Critical Rule: NEVER Create Standalone Telegram Clients"

    The Listener Service **owns** the primary Telegram session. All other services (Processor, Enrichment) that need Telegram API access must:

    1. Accept `Optional[TelegramClient]` in constructor
    2. Receive client from `main.py` initialization
    3. Gracefully degrade if client is `None`

    **Why?** Multiple sessions cause rate limits, auth conflicts, and resource waste.

```python
# ✅ CORRECT - Listener owns session
class TelegramListener:
    def __init__(self, redis_queue: RedisQueue, notifier=None):
        self.client: Optional[TelegramClient] = None
        # Client created in start() method

# ✅ CORRECT - Other services receive client
class SocialGraphTask:
    def __init__(self, telegram_client: Optional[TelegramClient]):
        self.client = telegram_client  # Passed from main.py

# ❌ WRONG - Never do this in task classes
class BadTask:
    def __init__(self):
        self.client = TelegramClient(...)  # DON'T CREATE NEW SESSION!
```

### Message Flow

```mermaid
sequenceDiagram
    participant TG as Telegram API
    participant TL as TelegramListener
    participant SG as social_graph_utils
    participant RQ as RedisQueue
    participant REDIS as Redis Streams

    TG->>TL: events.Album (media album)
    TL->>SG: extract_social_metadata(message)
    SG-->>TL: {author_user_id, forward_from, replied_to}
    TL->>SG: fetch_and_upsert_user_from_telegram(client, user_id)
    SG->>SG: Upsert to telegram_users table
    TL->>RQ: push_message(content, metadata, album_ids)
    RQ->>REDIS: XADD telegram:messages

    TG->>TL: events.NewMessage (single message)
    TL->>TL: Check grouped_id
    alt Has grouped_id
        TL->>TL: Buffer for fallback
        Note over TL: Album event should handle this
    else No grouped_id
        TL->>SG: extract_social_metadata(message)
        SG-->>TL: {social metadata}
        TL->>RQ: push_message(content, metadata)
        RQ->>REDIS: XADD telegram:messages
    end

    Note over TL: Periodic stale flush (every 30s)
    TL->>TL: Check buffered groups > 60s old
    TL->>RQ: push_message (fallback album)
    Note over TL: Album event didn't fire (edge case)
```

### Album Handling Strategy

The Listener implements a **dual-handler strategy** for media albums (multiple photos/videos in one post):

1. **Primary Handler**: `events.Album` - Telethon automatically buffers complete albums
2. **Fallback Handler**: `events.NewMessage` + periodic flush - For edge cases where Album event doesn't fire

**Known Edge Cases**:

- Exactly 1 image + 1 video (Telethon issue #4426)
- Cross-datacenter album delivery issues
- Partial album delivery

```python
# PRIMARY: Album event handler
@client.on(events.Album(chats=channel_entities))
async def handle_album(event: events.Album.Event):
    # Telethon provides:
    # - event.text (caption from any message)
    # - event.messages (all media files)
    # - event.grouped_id (album identifier)
    await self._on_album(event)

# SECONDARY: NewMessage handler (buffers grouped messages)
@client.on(events.NewMessage(chats=channel_entities))
async def handle_new_message(event: events.NewMessage.Event):
    if event.message.grouped_id:
        # Buffer for fallback (Album should handle this)
        await self._buffer_grouped_message(event.message)
        return
    await self._on_single_message(event.message)

# FALLBACK: Periodic flush task (every 30s)
async def _periodic_group_flush(self):
    # Flush groups that haven't been claimed by Album event
    # after 60 second timeout
    while not self._shutdown:
        await asyncio.sleep(30)
        for grouped_id, group in self.grouped_messages_cache.items():
            if age_seconds > 60:
                await self._flush_grouped_message(grouped_id)
```

## Configuration

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `TELEGRAM_API_ID` | Telegram API ID from my.telegram.org | - | Yes |
| `TELEGRAM_API_HASH` | Telegram API hash from my.telegram.org | - | Yes |
| `TELEGRAM_PHONE` | Phone number with country code | - | Yes (first auth) |
| `TELEGRAM_SESSION_PATH` | Directory for session files | `/data/sessions` | No |
| `TELEGRAM_SESSION_NAME` | Session file name | `osint_platform` | No |
| `SOURCE_ACCOUNT` | Multi-account identifier | `default` | No |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379/0` | Yes |
| `REDIS_STREAM_NAME` | Redis stream key | `telegram:messages` | No |
| `REDIS_MAX_STREAM_LENGTH` | Max messages in stream | `10000` | No |
| `BACKFILL_ENABLED` | Enable historical message fetching | `false` | No |
| `BACKFILL_START_DATE` | Start date for backfill | `2024-01-01` | No (if enabled) |
| `BACKFILL_MODE` | When to trigger backfill | `manual` | No |
| `BACKFILL_BATCH_SIZE` | Messages per batch | `100` | No |
| `BACKFILL_DELAY_MS` | Delay between batches | `1000` | No |

### Multi-Account Support

The platform supports multiple Telegram accounts for country-specific monitoring:

```bash
# Russia account
TELEGRAM_API_ID_RUSSIA=12345
TELEGRAM_API_HASH_RUSSIA=abc123
TELEGRAM_PHONE_RUSSIA=+1234567890

# Ukraine account
TELEGRAM_API_ID_UKRAINE=67890
TELEGRAM_API_HASH_UKRAINE=def456
TELEGRAM_PHONE_UKRAINE=+0987654321

# Enable multi-account mode
docker-compose --profile multi-account up -d listener-russia listener-ukraine
```

**Benefits**:

- Doubled Telegram API rate limits
- Risk isolation (one account ban doesn't affect the other)
- Organizational clarity (Russia channels use Russia account)

See [Telegram Patterns Documentation](../../architecture/telegram-patterns.md) for full multi-account setup.

## Key Features

### 1. Folder-Based Channel Discovery

!!! success "Revolutionary Innovation"

    Instead of building a custom admin panel, the Listener uses **Telegram's native folder feature** for channel management.

    **Time Saved**: 2-3 weeks development + ongoing maintenance

**User Workflow** (30 seconds):

1. Open Telegram app (mobile or desktop)
2. Find channel you want to monitor
3. Drag channel to appropriate folder
4. Done! Platform detects change within 5 minutes

**Folder Rules** (prefix matching):

| Folder Pattern | Rule | Behavior |
|---------------|------|----------|
| `Archive*` | `archive_all` | Store all non-spam messages |
| `Monitor*` | `selective_archive` | Only store OSINT score ≥70 |
| `Discover*` | `discovery` | Auto-joined channels (14-day probation) |

**Examples** (remember 12-char limit):

- `Archive-RU` → `archive_all` (10 chars ✅)
- `Monitor-UA` → `selective_archive` (10 chars ✅)
- `Discover-RU` → `discovery` (11 chars ✅)
- `Personal` → No match (ignored)

**Technical Implementation**:

```python
# services/listener/src/channel_discovery.py
class ChannelDiscovery:
    FOLDER_RULES = {
        r"^Archive": "archive_all",
        r"^Monitor": "selective_archive",
        r"^Discover": "discovery",
    }

    async def discover_channels(self) -> list[Channel]:
        # Get folders from Telegram API
        result = await self.client(GetDialogFiltersRequest())
        filters: list[DialogFilter] = result.filters

        # Map folders to channels
        for folder in filters:
            rule = self._get_rule_for_folder(folder.title.text)
            for peer in folder.include_peers:
                entity = await self.client.get_entity(peer)
                channel = await self._entity_to_channel(entity, folder.title.text, rule)
                discovered_channels.append(channel)

        return discovered_channels
```

**Background Sync**:

- Runs every 5 minutes
- Detects channel additions/removals
- Updates database automatically
- Triggers backfill for new channels (if enabled)

### 2. Social Graph Metadata Extraction

Every message is enriched with social graph data for OSINT influence mapping:

**Metadata Captured**:

| Field | Source | Purpose |
|-------|--------|---------|
| `author_user_id` | `message.from_id` | Who sent the message |
| `forward_from_channel_id` | `message.fwd_from.from_id` | Original channel (forward chain) |
| `forward_from_message_id` | `message.fwd_from.channel_post` | Original message ID |
| `forward_date` | `message.fwd_from.date` | Original post timestamp |
| `replied_to_message_id` | `message.reply_to.reply_to_msg_id` | Parent message (conversation thread) |
| `has_comments` | `message.replies.comments` | Discussion group enabled? |
| `comments_count` | `message.replies.replies` | Number of comments |
| `linked_chat_id` | `message.replies.channel_id` | Discussion group channel ID |

**User Profile Enrichment**:

When a message has an author, the Listener automatically:

1. Fetches full user profile from Telegram API
2. Upserts to `telegram_users` table
3. Updates `last_seen` and `interaction_count`

```python
# Fire-and-forget pattern (non-blocking)
if social_metadata.get('author_user_id'):
    asyncio.create_task(
        fetch_and_upsert_user_from_telegram(
            self.client,
            social_metadata['author_user_id']
        )
    )
```

### 3. Historical Backfill

Fetch historical messages from channels when first discovered:

**Configuration**:

```bash
BACKFILL_ENABLED=true
BACKFILL_START_DATE=2024-01-01
BACKFILL_MODE=on_discovery  # on_discovery | manual | scheduled
```

**Modes**:

- **on_discovery**: Automatic backfill when channel added to folder
- **manual**: Triggered via API endpoint (`POST /api/admin/channels/{id}/backfill`)
- **scheduled**: Not yet implemented

**Features**:

- Date-based fetching (from configured start date)
- FloodWait error handling (exponential backoff)
- Progress tracking in database
- Resume capability after interruption
- Ephemeral media detection (media that expired)

**Example**:

```python
# Triggered automatically when channel added to "Archive-RU" folder
backfill_service = BackfillService(
    client=telegram_client,
    db=db_session,
    redis_queue=redis_queue,
    media_archiver=media_archiver,
)

stats = await backfill_service.backfill_channel(
    channel=channel,
    from_date=datetime(2024, 1, 1),
)
# → {messages_fetched: 1523, duration_seconds: 345.2, completed: True}
```

### 4. Multi-Account Session Routing

Each message is tagged with `source_account` for multi-account setups:

```python
await redis_queue.push_message(
    message_id=message.id,
    channel_id=channel.telegram_id,
    content=content,
    source_account=settings.SOURCE_ACCOUNT,  # "russia" or "ukraine"
    # ... other fields
)
```

**Downstream Usage**:

- **Processor**: Routes to correct Telegram client for media download
- **Enrichment**: Uses TelegramClientPool to select correct session
- **API**: Displays source account in admin panel

## Redis Integration

### Stream Configuration

| Parameter | Value | Purpose |
|-----------|-------|---------|
| Stream Key | `telegram:messages` | Main message stream |
| Consumer Group | `processor-workers` | Parallel processing |
| Max Length | `10,000` messages | Prevent unbounded growth |
| Encoding | UTF-8 | Text content |

### Message Format

```python
# Fields pushed to Redis stream
{
    "message_id": "12345",                    # Telegram message ID
    "channel_id": "-1001234567890",           # Telegram channel ID
    "content": "Breaking news...",            # Message text
    "media_type": "photo",                    # photo/video/document/etc
    "telegram_date": "2024-01-01T12:00:00Z",  # Original timestamp
    "ingested_at": "2024-01-01T12:01:00Z",    # Listener timestamp
    "trace_id": "uuid-here",                  # Cross-service correlation

    # Album fields (if grouped_id present)
    "grouped_id": "67890",                    # Album identifier
    "media_count": "3",                       # Number of media files
    "album_message_ids": "[12345,12346,12347]", # All message IDs (JSON)

    # Engagement metrics
    "views": "1523",
    "forwards": "42",

    # Multi-account routing
    "source_account": "russia",

    # Social graph metadata
    "author_user_id": "123456789",
    "replied_to_message_id": "12340",
    "forward_from_channel_id": "-1009876543210",
    "forward_from_message_id": "54321",
    "forward_date": "2023-12-31T10:00:00Z",
    "has_comments": "true",
    "comments_count": "15",
    "linked_chat_id": "-1001111111111",
}
```

### Stream Operations

```python
# Push message
stream_id = await redis_queue.push_message(
    message_id=message.id,
    channel_id=channel.telegram_id,
    content=content,
    # ... other fields
)
# → "1234567890-0" (Redis stream ID)

# Get stream info
info = await redis_queue.get_stream_info()
# → {length: 1523, first_entry: {...}, last_entry: {...}, groups: [...]}

# Check pending count
pending = await redis_queue.get_pending_count()
# → 42 (messages waiting for acknowledgment)

# Health check
healthy = await redis_queue.health_check()
# → True/False
```

## Metrics

The Listener exposes Prometheus metrics on **port 8001** (`/metrics` endpoint).

### Key Metrics

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| `telegram_messages_received_total` | Counter | Messages received from Telegram | `channel_id`, `channel_name`, `has_media` |
| `telegram_messages_queued_total` | Counter | Messages pushed to Redis | `channel_id` |
| `telegram_messages_failed_total` | Counter | Failed message processing | `channel_id`, `error_type` |
| `telegram_channels_active_total` | Gauge | Active channels being monitored | - |
| `telegram_channels_discovered` | Gauge | Channels discovered from folders | `folder`, `rule` |
| `telegram_connections_active` | Gauge | Active Telegram connections | - |
| `telegram_flood_waits_total` | Counter | Telegram rate limit errors | `wait_seconds_bucket` |
| `redis_queue_depth` | Gauge | Messages in Redis queue | - |
| `redis_queue_pending` | Gauge | Messages pending acknowledgment | - |
| `backfill_messages_total` | Counter | Backfill messages fetched | `channel_id`, `channel_name`, `status` |
| `backfill_status` | Gauge | Current backfill status | `channel_id`, `channel_name` |

### Grafana Dashboard

Import the pre-configured Listener dashboard:

```bash
# Dashboard JSON available at:
# infrastructure/monitoring/grafana/dashboards/listener-overview.json
```

**Panels**:

- Messages ingested per channel (timeseries)
- Album vs single message ratio
- FloodWait errors by severity
- Redis queue depth
- Active channels by folder
- Backfill progress

### Example Queries

```promql
# Messages received per second
rate(telegram_messages_received_total[5m])

# Top 10 most active channels
topk(10, sum by (channel_name) (increase(telegram_messages_received_total[1h])))

# FloodWait error rate
rate(telegram_flood_waits_total[1h])

# Redis queue backlog
redis_queue_depth + redis_queue_pending
```

## Running Locally

### Docker (Recommended)

```bash
# 1. Create .env file with Telegram credentials
cp .env.example .env
# Edit TELEGRAM_API_ID, TELEGRAM_API_HASH, TELEGRAM_PHONE

# 2. Authenticate Telegram session (first time only)
docker-compose run --rm listener python3 scripts/telegram_auth.py

# 3. Start listener
docker-compose up -d listener

# 4. View logs
docker-compose logs -f listener

# 5. Check metrics
curl http://localhost:8001/metrics
```

### Development (Without Docker)

```bash
# 1. Install dependencies
cd services/listener
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install -r ../../shared/python/requirements.txt

# 2. Set environment variables
export TELEGRAM_API_ID=12345
export TELEGRAM_API_HASH=abc123
export TELEGRAM_PHONE=+1234567890
export REDIS_URL=redis://localhost:6379/0
export POSTGRES_URL=postgresql+asyncpg://user:pass@localhost/osint_platform
export PYTHONPATH="../../shared/python:$PYTHONPATH"

# 3. Authenticate (first time only)
python3 scripts/telegram_auth.py

# 4. Run listener
python -m src.main

# 5. In another terminal, monitor metrics
curl http://localhost:8001/metrics | grep telegram_messages
```

### Development Tips

!!! tip "Hot Reload for Testing"

    For development, use `docker-compose watch` (Docker Compose 2.22+):

    ```yaml
    # docker-compose.yml
    services:
      listener:
        develop:
          watch:
            - path: ./services/listener/src
              action: sync+restart
    ```

**Testing Album Handling**:

```bash
# Send test album to monitored channel
# 1. In Telegram app, select multiple photos
# 2. Send to channel in "Test-Channels" folder
# 3. Check logs for Album event vs fallback flush

docker-compose logs -f listener | grep -E "Album|fallback"
```

## Troubleshooting

### Common Issues

#### 1. Session Errors

**Symptom**: `Telegram not authorized. Please run authentication script first.`

**Solution**:

```bash
# Re-authenticate
docker-compose run --rm listener python3 scripts/telegram_auth.py

# Or manually:
python3 scripts/telegram_auth.py --account default

# For multi-account:
python3 scripts/telegram_auth.py --account russia
python3 scripts/telegram_auth.py --account ukraine
```

#### 2. FloodWait Errors

**Symptom**: `FloodWaitError: must wait 3600 seconds`

**Cause**: Exceeded Telegram API rate limits

**Solution**:

```bash
# Listener automatically sleeps and retries
# Check if you're backfilling too aggressively:

# Reduce backfill speed in .env:
BACKFILL_BATCH_SIZE=50  # default: 100
BACKFILL_DELAY_MS=2000  # default: 1000

# Or disable backfill temporarily:
BACKFILL_ENABLED=false
```

**Monitoring**:

```promql
# Check FloodWait frequency
rate(telegram_flood_waits_total[1h])

# Identify problematic wait times
telegram_flood_waits_total{wait_seconds_bucket="3600+"}
```

#### 3. Album Event Not Firing

**Symptom**: Log shows `FALLBACK flush (Album event didn't fire)`

**Cause**: Known Telethon edge case (1 image + 1 video)

**Impact**: None - fallback mechanism processes correctly

**Verification**:

```bash
# Check fallback vs primary ratio
docker-compose logs listener | grep -c "events.Album"
docker-compose logs listener | grep -c "fallback_flush"

# Should be: Album events >> fallback flushes
# If ratio is reversed, investigate Telethon version
```

#### 4. Redis Connection Failed

**Symptom**: `RedisError: Error connecting to Redis`

**Solution**:

```bash
# Check Redis is running
docker-compose ps redis

# Test connection
docker-compose exec redis redis-cli ping
# → PONG

# Check Redis password
docker-compose exec listener env | grep REDIS_URL

# Restart Redis
docker-compose restart redis listener
```

#### 5. Channel Discovery Not Working

**Symptom**: Channels not appearing in database after adding to folder

**Solution**:

```bash
# Check folder names match patterns
docker-compose logs listener | grep "Processing folder"

# Expected:
# Processing folder 'Archive-RU' with rule 'archive_all'
# Processing folder 'Monitor-UA' with rule 'selective_archive'
# Skipping folder 'Personal' - no matching rule pattern

# Manually trigger discovery
docker-compose exec listener python3 -c "
from src.channel_discovery import ChannelDiscovery
import asyncio

async def test():
    discovery = ChannelDiscovery(...)
    channels = await discovery.discover_channels()
    print(f'Found {len(channels)} channels')

asyncio.run(test())
"
```

#### 6. High Memory Usage

**Symptom**: Container using >2GB RAM

**Cause**: Large album buffer or Redis queue backlog

**Solution**:

```bash
# Check Redis queue depth
curl -s http://localhost:8001/metrics | grep redis_queue_depth

# Clear stale groups (emergency only)
docker-compose exec listener python3 -c "
from src.telegram_listener import TelegramListener
listener = TelegramListener(...)
listener.grouped_messages_cache.clear()
"

# Reduce Redis max length in .env:
REDIS_MAX_STREAM_LENGTH=5000  # default: 10000
```

### Key Log Messages

| Log Message | Severity | Meaning | Action |
|-------------|----------|---------|--------|
| `Connected to Telegram successfully` | INFO | Normal startup | None |
| `Album received via events.Album` | INFO | Album processed normally | None |
| `FALLBACK flush (Album event didn't fire)` | WARNING | Edge case handled | Monitor frequency |
| `FloodWait error - sleeping for X seconds` | WARNING | Rate limited | Reduce request rate |
| `Telegram API schema mismatch` | WARNING | Telethon outdated | `pip install -U telethon` |
| `Failed to push message to queue` | ERROR | Redis connection issue | Check Redis |
| `Session not authorized` | ERROR | Missing auth | Run `telegram_auth.py` |

### Debug Mode

Enable verbose logging for troubleshooting:

```bash
# In .env
LOG_LEVEL=DEBUG

# Or via environment variable
docker-compose run -e LOG_LEVEL=DEBUG listener

# Filter specific components
docker-compose logs listener | grep "telegram_listener"
docker-compose logs listener | grep "channel_discovery"
docker-compose logs listener | grep "backfill_service"
```

### Health Checks

```bash
# Docker health check (automatic)
docker-compose ps listener
# → healthy

# Manual health check
curl http://localhost:8001/health
# → {"status": "healthy", "connections": {"telegram": true, "redis": true}}

# Metrics endpoint
curl http://localhost:8001/metrics | head -20
```

## Related Documentation

- [Processor Service](processor.md) - Downstream message processing
- [Redis Queue Architecture](../../architecture/redis-streams.md) - Message queue design
- [Telegram Patterns](../../architecture/telegram-patterns.md) - Session management, multi-account
- [Channel Management Guide](../../user-guide/channel-management.md) - Folder-based workflow
- [Backfill Guide](../../user-guide/backfill.md) - Historical message fetching
- [Monitoring Setup](../../operations/monitoring.md) - Prometheus + Grafana

---

**Next Steps**:

1. Read [Processor Service Documentation](processor.md) to understand downstream processing
2. Review [Telegram Patterns](../../architecture/telegram-patterns.md) for session management rules
3. Set up [Monitoring](../../operations/monitoring.md) to track listener health
