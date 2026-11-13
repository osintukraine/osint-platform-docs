# Notifier Service

Real-time event aggregation and notification routing service for the OSINT Intelligence Platform.

## Overview

The **Notifier Service** (also called **Aggregator Service**) is a central intelligence hub that receives notification events from all platform services via Redis Pub/Sub, applies intelligent batching and routing rules, formats rich notifications, and delivers them to the self-hosted ntfy server for push notification delivery.

### Purpose

- **Event Aggregation**: Subscribe to all platform events via single Redis channel
- **Smart Batching**: Group routine events into 5-minute summaries, send urgent events immediately
- **Topic Routing**: Route events to appropriate ntfy topics based on event type
- **Rich Formatting**: Create detailed notifications with media attachments and action buttons
- **Prometheus Integration**: Convert AlertManager webhooks to notifications
- **Decoupling**: Keep services lightweight by centralizing notification logic

### Key Features

- Event-driven architecture (Redis Pub/Sub)
- Priority-based batching (urgent/high immediate, default/low/min batched)
- Flat topic structure with underscore naming (`osint_telegram_listener`)
- Rich notification formatting with markdown support
- Media attachment support (MinIO URL integration)
- Prometheus AlertManager webhook endpoints
- Admin-controlled event filtering (`filters.py`, via `runtime_config("notifier")`)
- Comprehensive Prometheus metrics
- FastAPI health checks

## Architecture

### Service Flow

```
Platform Services (Listener, Processor, API, etc.)
    ↓ NotificationClient.emit()
Redis Pub/Sub (notifications:events channel)
    ↓ subscribe
Notifier Service (Aggregator)
    ├─ filters.decide() → Drop / keep / push at once (admin settings)
    ├─ EventRouter → Determine ntfy topic
    ├─ EventBatcher → Buffer or send immediately
    ├─ NotificationFormatter → Create rich notification
    └─ NtfyPublisher → POST to ntfy server
    ↓
ntfy Server (self-hosted)
    ↓ push notifications
Mobile/Desktop/Web Clients
```

### Components

| Component | File | Responsibility |
|-----------|------|----------------|
| **AggregatorService** | `main.py` | Main service, Redis subscription, event loop |
| **EventRouter** | `router.py` | Route events to ntfy topics, determine priority |
| **EventBatcher** | `batcher.py` | Batch low-priority events (5-minute window) |
| **NotificationFormatter** | `formatter.py` | Format events into rich ntfy notifications |
| **NtfyPublisher** | `publisher.py` | HTTP client for ntfy server |
| **Settings** | `config.py` | Env-based defaults (pydantic-settings) |
| **decide()** | `filters.py` | Drop or escalate events using the admin-page notification settings |

### Event Flow Details

1. **Service emits event**:
   ```python
   await notifier.emit("channel.discovered", {...}, priority="default")
   ```

2. **Redis publishes**: Event sent to `notifications:events` channel

3. **Aggregator receives**: Subscribed to channel, receives JSON event

4. **Filter decides** (`filters.decide`): the event is dropped (counted as `reason="disabled_by_setting"`), kept, or - for archived posts at or above `notification_min_importance` - escalated to immediate `high` priority

5. **Router determines topic**:
   - `channel.discovered` → `osint_telegram_discovery`
   - Priority override based on event type

6. **Batcher decides**:
   - Urgent/high (priority 4-5) → Send immediately
   - Default/low/min (priority 1-3) → Buffer for 5 minutes

7. **Formatter creates notification**:
   - Specialized formatters per event type
   - Media attachments, action buttons, emoji, markdown

8. **Publisher sends to ntfy**:
   - HTTP POST to ntfy server
   - Topic, title, message, priority, tags, attachments

9. **ntfy delivers**: Push notifications to subscribed clients

## Key Files

```
services/notifier/
├── src/
│   ├── main.py              # Main service, FastAPI app, event loop
│   ├── config.py            # Env-based settings defaults
│   ├── filters.py           # Admin-setting event filter (drop / keep / push now)
│   ├── router.py            # Event → topic routing logic
│   ├── batcher.py           # Event batching (5-minute window)
│   ├── formatter.py         # Rich notification formatting
│   └── publisher.py         # ntfy HTTP client
├── tests/
├── Dockerfile               # Python 3.11-slim container
└── requirements.txt         # FastAPI, Redis, httpx, prometheus-client
```

## Notification Types

### Event Routing Table

The router maps event types to ntfy topics:

| Event Type | Topic | Priority | Batched? |
|------------|-------|----------|----------|
| `message.archived` / `message.received` | `osint_data_archived` | default (high if importance filter matches) | Yes (unless escalated) |
| `media.downloaded` / `media.deduplicated` | `osint_data_media` | default | Yes |
| `media.archival_failed` | `osint_data_errors` | high | No |
| `spam.detected` | `osint_processing_spam` | default | Yes |
| `message.quarantined` | `osint_data_quarantine` | default | Yes |
| `entity.extracted` | `osint_processing_entities` | default | Yes |
| `llm.request` | `osint_processing_llm` | default | Yes |
| `llm.error` | `osint_processing_llm` | high | No |
| `channel.discovered` | `osint_telegram_discovery` | default | Yes |
| `channel.removed` | `osint_telegram_discovery` | default | Yes |
| `session.disconnected` | `osint_telegram_listener` | urgent | No |
| `flood_wait.triggered` | `osint_telegram_listener` | high | No |
| `backfill.started` | `osint_telegram_backfill` | default | Yes |
| `backfill.failed` | `osint_telegram_backfill` | high | No |
| `container.unhealthy` | `osint_system_containers` | urgent | No |
| `api.error_5xx` | `osint_api_errors` | high | No |
| `db.error` / `storage.error` / `redis.error` | `osint_data_errors` | urgent | No |
| `alert.*` | `osint_prometheus_alerts` | by severity (critical urgent, warning high, else default) | varies |

Unrouted event types go to `osint_system_unknown`.

### Priority Levels

| Priority | ntfy Int | When Used | Batching |
|----------|----------|-----------|----------|
| `urgent` | 5 | Platform down/broken (db.error, container.unhealthy) | Immediate |
| `high` | 4 | Significant issues (api.error_5xx, backfill.failed) | Immediate |
| `default` | 3 | Routine operations (message.archived, channel.discovered) | Batched (5min) |
| `low` | 2 | Defined in the map but no event type is assigned it | Batched (5min) |
| `min` | 1 | Defined in the map but no event type is assigned it | Batched (5min) |

!!! note "Priority Override"
    The router determines priority based on event type, overriding any priority set by the emitting service. Event types not in `event_priority_map` are `default` (batched). Batches are always published at ntfy priority 3.

## ntfy.sh Integration

### Topic Naming Convention

ntfy requires **flat topics** (single-level identifiers):

```
✅ CORRECT: osint_telegram_listener
❌ WRONG:   osint/telegram/listener
```

All topics follow the pattern: `osint_<category>_<subcategory>`

### Topic Categories

```
Data Events:
  osint_data_archived         - Message archival notifications
  osint_data_media            - Media download & deduplication
  osint_data_errors           - Database, Redis, storage errors, media archival failures
  osint_data_quarantine       - Quarantined messages


Processing Events:
  osint_processing_entities   - Extracted entities
  osint_processing_spam       - Spam detected
  osint_processing_llm        - LLM requests and errors

Telegram Events:
  osint_telegram_discovery    - Channel discovered/removed
  osint_telegram_listener     - Session connect/disconnect, flood waits
  osint_telegram_backfill     - Historical backfill operations

API Events:
  osint_api_requests          - API request tracking
  osint_api_errors            - API errors (4xx, 5xx)
  osint_api_rss               - RSS feed generation

System Events:
  osint_system_containers     - Container health events
  osint_system_resources      - CPU, memory, disk alerts
  osint_system_unknown        - Uncategorized events

Prometheus Alerts:
  osint_prometheus_alerts     - AlertManager alerts
```

### Publishing to ntfy

The `NtfyPublisher` sends HTTP POST requests to the ntfy server:

```python
# services/notifier/src/publisher.py
async def publish(
    topic: str,
    title: str,
    message: str,
    priority: int = 3,
    tags: Optional[list[str]] = None,
    attach_url: Optional[str] = None,
    actions: Optional[list[dict]] = None,
) -> bool:
    payload = {
        "topic": topic,
        "title": title,
        "message": message,
        "priority": priority,
        "tags": tags or [],
    }

    if attach_url:
        payload["attach"] = attach_url

    if actions:
        payload["actions"] = actions

    async with httpx.AsyncClient(timeout=10) as client:
        response = await client.post(ntfy_url, json=payload)
        response.raise_for_status()
```

The JSON publish goes to the ntfy base URL (`NTFY_URL`), with the topic in the body. Failures are logged and return `False` (counted as dropped); there is no retry.

### Rich Notification Features

**Markdown Support**:
```markdown
**Channel:** @UkraineNews
**Importance:** HIGH
**Media:** ✓ Attached

_Message ID: 123456_
```

**Media Attachments**:
```json
{
  "attach": "http://localhost:9000/media/abc123.jpg"
}
```

**Action Buttons**:
```json
{
  "actions": [{
    "action": "view",
    "label": "Open Channel",
    "url": "https://t.me/UkraineNews"
  }]
}
```

## Notification Rules

### Batching Logic

The `EventBatcher` groups low-priority events into 5-minute summaries:

```python
# services/notifier/src/batcher.py
def should_batch(self, priority: str) -> bool:
    # Batch low-priority events, send high-priority immediately
    return priority in ("default", "low", "min")
```

**Batching Behavior**:

- Events added to topic-specific batches
- Timer starts on first event (5 minutes)
- Immediate flush when batch reaches max size (50 events)
- Automatic flush on timer expiry
- All batches flushed on service shutdown

**Batch Notification Format**:

```
📊 23 osint_data_archived events

**Summary of 23 events:**

• message.archived: 20
• media.downloaded: 3

_Batched at 14:35:00 UTC_
```

### Event Filtering

`filters.decide(event, settings)` runs before routing. Settings are read per event through `runtime_config("notifier")` (precedence: admin page override, then `.env`, then default; cached about 30 s). The keys are declared in `shared/python/config/runtime_keys.py`:

| Runtime key | Env var | Default | Effect |
|-------------|---------|---------|--------|
| `enabled` | `NOTIFICATIONS_ENABLED` | true | Master switch; off drops every event |
| `notify_message_archived` | `NOTIFY_MESSAGE_ARCHIVED` | true | Routine archived-post stream |
| `notify_high_importance` | `NOTIFY_OSINT_HIGH` | true | Push posts at or above the minimum importance at once, as `high` |
| `notification_min_importance` | `NOTIFICATION_MIN_IMPORTANCE` | `high` | `low`, `medium` or `high` |
| `notify_spam_detected` | `NOTIFY_SPAM_DETECTED` | true | `spam.detected` |
| `notify_channel_discovery` | `NOTIFY_CHANNEL_DISCOVERY` | true | `channel.discovered` / `channel.removed` |
| `notify_llm_activity` | `NOTIFY_LLM_ACTIVITY` | false | `llm.*` events (no service emits them at present) |
| `aggregator_batch_interval` | `AGGREGATOR_BATCH_INTERVAL` | 300 | Read once at start; restart to apply |
| `aggregator_max_batch_size` | `AGGREGATOR_MAX_BATCH_SIZE` | 50 | Read once at start; restart to apply |

!!! warning "Unused settings"
    `NOTIFY_OSINT_CRITICAL`, `NOTIFY_SYSTEM_HEALTH`, `NOTIFY_API_ERRORS`, `AGGREGATOR_ENABLED`, `NTFY_TOPIC_PREFIX`, `NTFY_RATE_LIMIT_EVENTS` and `METRICS_PORT` exist in `config.py` and compose but no code reads them.

### Prometheus AlertManager Integration

The notifier service exposes webhook endpoints for AlertManager:

```python
# services/notifier/src/main.py
@app.post("/alertmanager/critical")
async def alertmanager_critical(request: Request):
    payload = await request.json()
    await _process_alertmanager_webhook(payload, severity="critical", priority="urgent")
    return {"status": "ok", "alerts_processed": len(payload.get("alerts", []))}

@app.post("/alertmanager/warning")
async def alertmanager_warning(request: Request):
    # ...
```

`/alertmanager/info` and `/alertmanager/default` also exist (priority `default`). The AlertManager receivers in `infrastructure/alertmanager/alertmanager.yml` use all four.

**AlertManager Configuration** (`infrastructure/alertmanager/alertmanager.yml`):

```yaml
receivers:
  - name: 'ntfy-critical'
    webhook_configs:
      - url: 'http://notifier:8000/alertmanager/critical'

  - name: 'ntfy-warning'
    webhook_configs:
      - url: 'http://notifier:8000/alertmanager/warning'
  # info and default receivers point to /alertmanager/info and /alertmanager/default
```

**Alert Processing**:

1. AlertManager sends webhook with alerts array
2. Notifier extracts alertname, severity, annotations
3. Creates notification event with Prometheus metadata
4. Publishes to Redis `notifications:events` channel
5. Router sends to `osint_prometheus_alerts` topic
6. Priority based on severity (critical → urgent, warning → high)

## Configuration

### Environment Variables

```bash
# Environment
ENVIRONMENT=development
DEBUG=false
LOG_LEVEL=INFO

# Redis connection (subscribe to notification events)
REDIS_URL=redis://redis:6379/0
REDIS_CHANNEL=notifications:events

# ntfy server connection (publish notifications)
NTFY_URL=http://ntfy:80
NTFY_TOPIC_PREFIX=osint-platform
NTFY_RATE_LIMIT_EVENTS=100  # per minute per topic

# Batching configuration
AGGREGATOR_ENABLED=true
AGGREGATOR_BATCH_INTERVAL=300  # seconds (5 minutes)
AGGREGATOR_MAX_BATCH_SIZE=50  # notifications per batch

# Feature flags (see Event Filtering; admin-page overrides win)
NOTIFICATIONS_ENABLED=
NOTIFICATION_MIN_IMPORTANCE=
NOTIFY_MESSAGE_ARCHIVED=true
NOTIFY_SPAM_DETECTED=true
NOTIFY_OSINT_HIGH=true
NOTIFY_OSINT_CRITICAL=true       # unused
NOTIFY_LLM_ACTIVITY=true
NOTIFY_CHANNEL_DISCOVERY=true
NOTIFY_SYSTEM_HEALTH=true        # unused
NOTIFY_API_ERRORS=true           # unused

# Metrics (unused: metrics are served by the app on port 8000)
METRICS_PORT=9094
```

The service also needs the shared platform settings (`POSTGRES_*`, `REDIS_*`, `MINIO_*`, `JWT_SECRET_KEY`) because `runtime_config` reads the admin page values from the `platform_config` table. Compose refuses to start without `POSTGRES_PASSWORD`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY` and `JWT_SECRET_KEY`.

### Docker Compose Configuration

```yaml
notifier:
  build:
    context: .
    dockerfile: services/notifier/Dockerfile
  container_name: osint-notifier
  profiles: ["monitoring", "notifications"]
  restart: unless-stopped

  depends_on:
    redis:
      condition: service_healthy
    ntfy:
      condition: service_healthy
    postgres:
      condition: service_healthy

  environment:
    REDIS_URL: redis://redis:6379/0
    REDIS_CHANNEL: notifications:events
    NTFY_URL: http://ntfy:80
    LOG_LEVEL: INFO

  ports:
    - "9094:9094"  # host mapping only: the app listens on 8000, so nothing answers on 9094

  networks:
    - backend

  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 10s
```

### Service Dependencies

- **Redis**: Event pub/sub
- **PostgreSQL**: Admin-page notification settings (`runtime_config`)
- **ntfy**: Push notification server
- **Prometheus** (optional): Metrics collection
- **AlertManager** (optional): Alert routing

## Running Locally

### Start with Docker Compose

```bash
# Start notifier service (monitoring or notifications profile)
docker-compose --profile notifications up -d notifier

# View logs
docker-compose logs -f notifier

# Check health
curl http://localhost:8000/health

# Check Prometheus metrics (served on the app port; note the trailing slash)
docker-compose exec notifier python -c "import urllib.request as u; print(u.urlopen('http://localhost:8000/metrics/').read().decode()[:500])"
```

!!! note "Port 8000 is not published"
    The `curl http://localhost:8000/...` examples on this page work from inside the container (`docker-compose exec notifier ...`, using Python `urllib` if `curl` is missing) or after you add a `8000:8000` port mapping for local debugging. Prometheus scrapes `notifier:8000` at `/metrics/` (see `infrastructure/prometheus/prometheus.yml`).

### Standalone Development

```bash
cd services/notifier

# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Install shared dependencies
pip install -r ../../shared/python/requirements.txt

# Set environment variables
export REDIS_URL=redis://localhost:6379/0
export NTFY_URL=http://localhost:80
export LOG_LEVEL=DEBUG

# Run service
python -m src.main
```

Service runs on port 8000 by default (FastAPI).

### Testing Notifications

**1. Send test event to Redis**:

```bash
docker-compose exec redis redis-cli PUBLISH notifications:events '{
  "service": "test",
  "type": "channel.discovered",
  "data": {
    "channel": "Test Channel",
    "username": "testchannel",
    "folder": "Archive-UA"
  },
  "priority": "default",
  "tags": ["test"],
  "timestamp": "2025-12-09T10:00:00Z"
}'
```

**2. Watch notifier logs**:

```bash
docker-compose logs -f notifier
```

**3. Verify notification in ntfy**:

Open browser: `http://localhost:8090/osint_telegram_discovery` (ntfy is bound to 127.0.0.1; `NTFY_PORT` defaults to 8090)

**4. Send test alert to AlertManager webhook**:

```bash
curl -X POST http://localhost:8000/alertmanager/critical \
  -H "Content-Type: application/json" \
  -d '{
    "version": "4",
    "status": "firing",
    "alerts": [{
      "status": "firing",
      "labels": {
        "alertname": "TestAlert",
        "severity": "critical",
        "component": "test"
      },
      "annotations": {
        "summary": "Test alert from curl",
        "description": "This is a test alert"
      },
      "startsAt": "2025-12-09T10:00:00Z"
    }]
  }'
```

## Troubleshooting

### No Notifications Received

**Check ntfy server**:

```bash
curl http://localhost:8090/v1/health
# Expected: {"healthy":true}
```

**Check notifier service health**:

```bash
curl http://localhost:8000/health
# Expected: {"status":"healthy","redis_connected":true,...}
```

**Check Redis subscription**:

```bash
docker-compose exec redis redis-cli PUBSUB NUMSUB notifications:events
# Expected: notifications:events 1
```

**Check notifier logs**:

```bash
docker-compose logs notifier | grep -E "ERROR|WARNING"

# Look for:
# - Redis connection errors
# - ntfy publish failures
# - Event processing errors
```

### Events Not Batching

**Verify priority settings**:

- `urgent` / `high` (priority 4-5) = immediate (not batched)
- `default` / `low` / `min` (priority 1-3) = batched

**Check batch configuration**:

```bash
docker-compose exec notifier env | grep AGGREGATOR

# Expected (admin-page overrides win over these):
# AGGREGATOR_BATCH_INTERVAL=300
# AGGREGATOR_MAX_BATCH_SIZE=50
```

**Watch for batch flush messages**:

```bash
docker-compose logs -f notifier | grep "Flushing batch"

# Expected (after 5 minutes):
# Flushing batch for osint_data_archived: 23 events (reason: timeout)
```

### High Latency

**Check Prometheus metrics**:

```bash
docker-compose exec notifier python -c "import urllib.request as u; print(u.urlopen('http://localhost:8000/metrics/').read().decode())" | grep aggregator_

# Key metrics:
# aggregator_events_received_total
# aggregator_events_published_total
# aggregator_events_dropped_total
# aggregator_batch_size
```

**Check ntfy server performance**:

```bash
docker stats osint-ntfy

# Look for high CPU or memory usage
```

**Reduce batch window**:

```bash
# In .env
AGGREGATOR_BATCH_INTERVAL=60  # Down from 300

# Restart service
docker-compose restart notifier
```

### AlertManager Alerts Not Received

**Verify AlertManager connection**:

```bash
docker-compose ps alertmanager

# Check AlertManager status
curl http://localhost:9093/api/v1/status
```

**Check webhook configuration**:

```bash
cat infrastructure/alertmanager/alertmanager.yml

# Verify webhook URLs point to:
# http://notifier:8000/alertmanager/critical
# http://notifier:8000/alertmanager/warning
```

**Test webhook manually**:

```bash
curl -X POST http://localhost:8000/alertmanager/critical \
  -H "Content-Type: application/json" \
  -d '{...}'  # See "Testing Notifications" section
```

### Events Dropped

**Check dropped events metric**:

```bash
docker-compose exec notifier python -c "import urllib.request as u; print(u.urlopen('http://localhost:8000/metrics/').read().decode())" | grep aggregator_events_dropped_total

# Shows reasons:
# - disabled_by_setting (filtered by admin settings; expected)
# - processing_error
# - publish_failed / publish_error
# - batch_publish_failed / batch_publish_error
```

**Check notifier logs for errors**:

```bash
docker-compose logs notifier | grep -E "Failed|dropped"
```

**Common causes**:

- Invalid JSON in event data
- ntfy server unreachable
- Redis connection lost
- Event formatting error

## Prometheus Metrics

The notifier service exposes metrics from the FastAPI app at `http://notifier:8000/metrics/` (inside the compose network; the `9094` host mapping does not reach it):

### Event Metrics

```prometheus
# Events received from Redis
aggregator_events_received_total{service="listener",event_type="channel.discovered"} 42

# Events published to ntfy
aggregator_events_published_total{topic="osint_telegram_discovery",priority="default"} 38

# Events dropped (errors or filtered by settings)
aggregator_events_dropped_total{service="listener",event_type="channel.discovered",reason="publish_failed"} 4
```

### Batch Metrics

```prometheus
# Batch sizes (histogram)
aggregator_batch_size_bucket{priority="batch",le="10"} 5
aggregator_batch_size_bucket{priority="batch",le="25"} 12
aggregator_batch_size_bucket{priority="batch",le="50"} 15

# Active Redis subscriptions
aggregator_active_subscriptions 1
```

### Health Endpoint

```bash
curl http://localhost:8000/health
```

**Response**:

```json
{
  "status": "healthy",   // "stopped" when the loop is not running
  "redis_connected": true,
  "batcher_stats": {
    "total_batched": 156,
    "total_immediate": 23,
    "pending_batches": 2,
    "pending_events": 8,
    "active_timers": 2
  }
}
```

## Related Documentation

- **[Notifications User Guide](../../user-guide/notifications.md)** - Complete ntfy configuration and usage
- **[Monitoring Guide](../../operator-guide/monitoring.md)** - Prometheus and AlertManager setup
- **[Architecture Overview](../architecture.md)** - Platform architecture
- **NotificationClient API**: `shared/python/notifications/client.py` - How services emit events

## Usage Examples

### Emitting Events from Services

Services use the lightweight `NotificationClient` to emit events:

```python
# services/listener/src/main.py
from notifications import NotificationClient

notifier = NotificationClient("listener", os.getenv("REDIS_URL"))

# Emit channel discovery event
await notifier.emit(
    "channel.discovered",
    {
        "channel": "Ukraine News",
        "username": "UkraineNews",
        "folder": "Archive-UA",
        "rule": "archive_all",
        "members_count": 50000,
        "verified": True
    },
    priority="default",
    tags=["discovery", "telegram"]
)

# Emit session disconnected event (urgent)
await notifier.emit(
    "session.disconnected",
    {
        "reason": "connection_lost",
        "reconnect_attempts": 3
    },
    priority="urgent",
    tags=["telegram", "session"]
)
```

### Custom Notification Formatting

Add new event types by extending the formatter:

```python
# services/notifier/src/formatter.py
def _format_custom_event(
    self, service: str, data: dict, priority: str
) -> dict[str, Any]:
    """Format custom event type."""
    title = f"🎯 Custom Event"

    message_parts = [
        f"**Service:** {service}",
        f"**Data:** {data.get('custom_field')}"
    ]

    return {
        "title": title,
        "message": "\n".join(message_parts),
        "tags": ["custom", service],
        "attach_url": None,
        "actions": None,
    }
```

Update routing table:

```python
# services/notifier/src/router.py
self.routing_table = {
    # ...
    "custom.event": "osint_custom_topic",
}
```

---

**Last Updated**: 2026-10-27
