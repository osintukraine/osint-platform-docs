# Media Sync Service

> Background worker for asynchronous media synchronization from local buffer to remote storage

## Overview

The Media Sync Service handles the asynchronous transfer of media files from the local SSD buffer to remote storage. By default (`SYNC_MODE=filesystem`) it copies files straight onto the Hetzner SSHFS mount; the legacy `minio` mode uploads to MinIO instead. This decouples fast local writes from slow network transfers, allowing the processor to return immediately after saving media locally.

**Pipeline Position**: `Processor / media-worker → Local Buffer → Media Sync → Hetzner SSHFS mount` (or MinIO in legacy mode)

!!! note "Media lane"
    With the `media_lane_enabled` admin setting on, the `media-worker` service (not the processor) downloads media into the buffer in the background (one `media-worker` per Telegram account in multi mode: `media-worker-russia`, `media-worker-ukraine`). See [Media Lanes](../../operator-guide/media-lanes.md).

**Key Technologies**:

- **Redis**: Job queue with BRPOP (blocking pop)
- **Filesystem copy** (`shutil.copy2` on the Hetzner SSHFS mount): default sync mode
- **MinIO**: S3-compatible client, legacy `minio` mode only (`MinioClientPool`)
- **asyncio**: High-performance async I/O
- **SQLAlchemy**: Database updates for sync status

**Core Responsibilities**:

1. Monitor Redis queue for pending sync jobs
2. Copy files from the local buffer to the Hetzner mount (or upload to MinIO in legacy mode)
3. Verify integrity (size comparison)
4. Update `media_files.synced_at` (and clear `local_path`) plus `storage_boxes` usage
5. Delete local files after successful sync
6. Handle failures with linearly increasing retry delays

## Architecture

### Component Diagram

```mermaid
graph TB
    subgraph "Processor Service"
        PROC[Message Processor]
        BUFFER[Local SSD Buffer]
    end

    subgraph "Media Sync Service"
        WORKER[MediaSyncWorker]
        RETRY[Retry Logic]
    end

    subgraph "External Services"
        REDIS[(Redis Queue)]
        PG[(PostgreSQL)]
        HETZNER[Hetzner SSHFS mount]
        MINIO[MinIO - legacy mode]
    end

    PROC -->|1. Write media| BUFFER
    PROC -->|2. Queue sync job| REDIS
    REDIS -->|BRPOP| WORKER
    WORKER -->|Read file| BUFFER
    WORKER -->|copy2 default| HETZNER
    WORKER -.->|legacy minio mode| MINIO
    WORKER -->|Update synced_at| PG
    WORKER -->|Delete after sync| BUFFER
    WORKER -->|On error| RETRY
    RETRY -->|Requeue| REDIS
```

### Key Files

| File | Purpose |
|------|---------|
| `src/worker.py` | Main worker implementation, queue processing |
| `src/poster.py` | Video poster frames (`{sha256}.poster.jpg`): copied to the box right after the video |
| `src/retention.py` | Daily media retention loop (age rotation and storage budget) |
| `src/repair.py` | Operator repair CLI (`requeue-failed`, `buffer-reconcile`) |
| `Dockerfile` | Container build definition |
| `requirements.txt` | Python dependencies |

### Data Flow

```mermaid
sequenceDiagram
    participant PROC as Processor
    participant BUFFER as Local Buffer
    participant REDIS as Redis Queue
    participant WORKER as Media Sync Worker
    participant DEST as Hetzner mount (or MinIO)
    participant DB as PostgreSQL

    PROC->>BUFFER: 1. Write media file
    PROC->>DB: 2. Insert media_files record
    PROC->>REDIS: 3. LPUSH sync job (media:sync:pending)
    Note over PROC: Returns immediately

    WORKER->>REDIS: BRPOP (blocking, 5s timeout)
    REDIS-->>WORKER: Job data (JSON)
    WORKER->>BUFFER: Read file
    WORKER->>DEST: copy to s3_key path (minio mode: fput_object)
    WORKER->>DEST: verify size
    WORKER->>DB: UPDATE synced_at, clear local_path, update storage_boxes
    WORKER->>BUFFER: Delete local file
```

## Configuration

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `REDIS_URL` | Redis connection string | `redis://localhost:6379/0` | Yes |
| `DATABASE_URL` | PostgreSQL connection (asyncpg) | dev URL in code | Yes |
| `SYNC_MODE` | `filesystem` (copy to Hetzner mount) or `minio` (legacy). Compose sets it from `MEDIA_SYNC_MODE` | `filesystem` | No |
| `HETZNER_MEDIA_PATH` | Destination mount (filesystem mode); the worker refuses to start if it is missing | `/mnt/hetzner-media/osint-media` | filesystem mode |
| `WORKER_ID` | Worker identifier in logs | `sync-worker-<pid>` (compose: `media-sync-1`) | No |
| `MEDIA_SYNC_ALLOW_LOCAL_BOX` | Allow the box path to sit on the same filesystem as the buffer. Off by default: the worker compares device ids and refuses to copy into a bare directory left behind by an unmounted box. Enable only when the "box" is deliberately a local disk | `false` | No |
| `MEDIA_RETENTION_REPORT_DIR` | Where retention writes its CSV report | `/var/cache/osint-media-buffer/media-retention` | No |
| `MINIO_ENDPOINT` | MinIO endpoint | `localhost:9000` | minio mode |
| `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY` | MinIO credentials | `minioadmin` | minio mode |
| `MINIO_BUCKET` | Target bucket (compose: `MINIO_BUCKET_NAME`) | `osint-media` | minio mode |
| `MINIO_SECURE` | Use HTTPS | `false` | minio mode |

!!! note "No Prometheus metrics"
    The worker exposes no metrics endpoint. It logs counters (files synced, failed, bytes) on shutdown; monitor via the Redis queue lengths and `media_files.synced_at` queries in [Monitoring](#monitoring).

### Queue Configuration

Built-in constants in `worker.py`:

```python
MEDIA_SYNC_QUEUE = "media:sync:pending"        # Main job queue
MEDIA_SYNC_FAILED_QUEUE = "media:sync:failed"  # Failed jobs queue
MAX_RETRIES = 3                                # Retry attempts
RETRY_DELAY_SECONDS = 5                        # Base delay
```

### Volume Mounts

```yaml
# docker-compose.yml
services:
  media-sync:
    volumes:
      - ${MEDIA_BUFFER_PATH:-./data/media-buffer}:/var/cache/osint-media-buffer
      - /mnt/hetznerstorage/osint-platform:/mnt/hetzner-media   # SSHFS destination
```

!!! danger "Critical: Same Buffer Path as Processor"
    The media-sync service MUST mount the same buffer path as the processor service.

## Poster Frames

For every new video the archiver writes a poster frame next to the buffer file, `{sha256}.poster.jpg` (see [Processor](processor.md)). It has no `media_files` row and no sync job of its own: `src/poster.py` (`carry_poster`) copies it to the storage box right after its video and drops the buffer copy. A poster failure is logged, never raised, because the video is already safe. Retention removes the poster together with its video.

## Media Retention

In `filesystem` mode the worker starts a retention loop (`src/retention.py`) that runs one pass a day at `media_retention_hour_utc` (UTC, default 3). Settings are admin runtime keys, re-read on every pass:

| Key | Default | Meaning |
|-----|---------|---------|
| `media_retention_mode` | `off` | `off`, `dry_run` (report only) or `delete` |
| `media_retention_hour_utc` | `3` | Hour of the daily pass |
| `media_retention_max_age_days` | `365` | Remove files whose newest post is older than this (0 = no age rule) |
| `media_retention_min_age_days` | `180` | Floor: the budget rule never removes younger media |
| `media_storage_budget_gb` | `0` | Over this budget, remove the oldest eligible files down to 98 % (0 = age rule only) |

Protected files are kept (private channels, posts in events or validated clusters, high importance, human-reviewed, bookmarked or commented, posts gone from Telegram, channels no account follows). Unless `media_retention_require_telegram_check` is off, a file is only removed after the Telegram worker of the owning account confirmed the post still carries its media.

Each pass is a row in `media_retention_runs` plus a CSV of the removable files in `MEDIA_RETENTION_REPORT_DIR`. A removal moves the links to `message_media_expired`, marks the post's media `expired` (the frontend then shows the Telegram embed), sets `media_files.deleted_at` and unlinks the file (and its poster) from the box. Run `python -m src.retention --now` for an immediate report-only pass (nothing is deleted; it does nothing while the mode is `off`). Read a `dry_run` report before switching to `delete`.

## Repair CLI

Run inside the `media-sync` container (the only one with the buffer and the box mounted). Without `--apply` both commands only report:

```bash
# Jobs parked in media:sync:failed go back to media:sync:pending
docker compose exec media-sync python -m src.repair requeue-failed [--apply]

# Buffer files that never left the buffer: mark synced if already on the box, otherwise queue a sync job
docker compose exec media-sync python -m src.repair buffer-reconcile [--min-age-hours 24] [--apply]
```

Run `requeue-failed` first and let `media:sync:pending` drain before `buffer-reconcile`.

## Multi-Box Storage Support

Every job must carry a `storage_box_id` (jobs without one are sent straight to the failed queue). In legacy `minio` mode the worker routes uploads per box using `MinioClientPool`:

```python
from storage import MinioClientPool

# Initialize pool (credentials shared across all boxes)
pool = MinioClientPool(
    access_key=MINIO_ACCESS_KEY,
    secret_key=MINIO_SECRET_KEY,
    bucket_name=MINIO_BUCKET
)

# Get client for specific box (lazy-loaded, cached)
async with session() as db:
    client = await pool.get_client(db, storage_box_id)
    client.fput_object(bucket, s3_key, local_path)
```

### How It Works

1. **Job contains `storage_box_id`**: Each sync job specifies which box to upload to
2. **Pool looks up endpoint**: Queries `storage_boxes.minio_endpoint` from database
3. **Client is cached**: Subsequent uploads to same box reuse the client
4. **Endpoint changes detected**: If config changes, client is recreated

### Job Format (Multi-Box)

```json
{
  "sha256": "abc123...",
  "s3_key": "media/ab/c1/abc123.jpg",
  "local_path": "/var/cache/osint-media-buffer/media/ab/c1/abc123.jpg",
  "storage_box_id": "russia-1",   // minio mode: routes to that box's endpoint
  "file_size": 12345,
  "queued_at": "2024-01-15T12:00:00"
}
```

### Usage Tracking

After successful sync, the worker updates box usage:

```sql
UPDATE storage_boxes
SET used_bytes = used_bytes + :file_size,
    reserved_bytes = GREATEST(0, reserved_bytes - :file_size),
    used_gb = (used_bytes + :file_size) / (1024.0 * 1024.0 * 1024.0)
WHERE id = (SELECT storage_box_id FROM media_files WHERE sha256 = :sha256)
```

## Key Concepts

### Write-Through Cache Pattern

The media-sync service implements **write-through cache**:

1. **Write**: Processor writes media to local buffer (fast, synchronous)
2. **Propagate**: Background sync to remote storage (slow, asynchronous)
3. **Serve**: API serves from local buffer if available, otherwise remote

This decouples fast local writes from slow network uploads.

### Content-Addressed Storage

All media uses SHA-256 based paths:

```
media/{hash[:2]}/{hash[2:4]}/{hash}.{ext}
```

Example: `media/ab/c1/abc123def456.jpg`

### Retry Backoff

Failed transfers are re-queued after a delay that grows linearly (`RETRY_DELAY_SECONDS * attempt`):

- Attempt 1: 5 seconds
- Attempt 2: 10 seconds
- Attempt 3: 15 seconds
- After 3 retries (4th failure): move to `media:sync:failed`
- `ValueError` (missing `storage_box_id`, minio-mode size mismatch) is not retried and goes straight to the failed queue

## Job Format

Jobs are JSON payloads in Redis:

```json
{
  "sha256": "abc123def456...",
  "s3_key": "media/ab/c1/abc123def456.jpg",
  "local_path": "/var/cache/osint-media-buffer/media/ab/c1/abc123def456.jpg",
  "storage_box_id": "russia-1",
  "file_size": 123456,
  "queued_at": "2024-01-15T12:00:00",
  "retry_count": 0
}
```

## Monitoring

### Queue Metrics

```bash
# Check pending jobs
docker-compose exec redis redis-cli LLEN media:sync:pending

# Check failed jobs
docker-compose exec redis redis-cli LLEN media:sync:failed

# View failed job details
docker-compose exec redis redis-cli LRANGE media:sync:failed 0 -1
```

### Database Queries

```sql
-- Files pending sync
SELECT COUNT(*) FROM media_files WHERE synced_at IS NULL;

-- Sync completion rate
SELECT
  COUNT(*) FILTER (WHERE synced_at IS NOT NULL) AS synced,
  COUNT(*) FILTER (WHERE synced_at IS NULL) AS pending
FROM media_files;

-- Average sync delay
SELECT
  AVG(EXTRACT(EPOCH FROM (synced_at - created_at))) AS avg_delay_seconds
FROM media_files
WHERE synced_at IS NOT NULL;
```

## Running the Service

### Docker Compose

```bash
# Start media-sync service
docker-compose up -d media-sync

# View logs
docker-compose logs -f media-sync

# Scale to multiple workers (compose sets a fixed WORKER_ID; it only affects log labels)
docker-compose up -d --scale media-sync=3
```

### Local Development

```bash
cd services/media-sync
pip install -r requirements.txt

export REDIS_URL=redis://localhost:6379/0
export DATABASE_URL=postgresql+asyncpg://osint_user:password@localhost:5432/osint_platform
export SYNC_MODE=filesystem
export HETZNER_MEDIA_PATH=/tmp/media-dest   # must exist
# legacy minio mode: SYNC_MODE=minio plus MINIO_ENDPOINT/MINIO_ACCESS_KEY/MINIO_SECRET_KEY

python -m src.worker
```

## Troubleshooting

### High Queue Backlog

**Symptom**: `media:sync:pending` length increasing

**Causes**:
- Slow SSHFS link to Hetzner
- Sync workers down/crashed (check the Hetzner mount exists: the worker exits at start if not)
- MinIO overloaded (minio mode only)

**Solutions**:
```bash
# Scale workers
docker-compose up -d --scale media-sync=5

# Check the mount is alive (filesystem mode)
docker-compose exec media-sync ls /mnt/hetzner-media/osint-media | head
```

### Files Not Syncing

**Debug Steps**:
```bash
# Check worker is running
docker-compose ps media-sync

# Check queue has jobs
docker-compose exec redis redis-cli LLEN media:sync:pending

# Filesystem mode: confirm destination is writable
docker-compose exec media-sync sh -c 'touch /mnt/hetzner-media/osint-media/.probe && echo ok'
```

In `minio` mode, test connectivity with the `minio` Python client using your `MINIO_*` credentials.

### Permanent Failures

**Inspect Failed Jobs**:
```bash
docker-compose exec redis redis-cli LRANGE media:sync:failed 0 0
```

**Common errors**:
- `File missing from both local buffer and Hetzner` (or `MinIO` in legacy mode): logged and counted as failed, not re-queued
- `Size mismatch after copy` (filesystem) / `Size mismatch: expected X, got Y` (minio)
- `Job missing storage_box_id`
- `Connection refused` (MinIO down, minio mode)

**Requeue failed jobs**:
```bash
docker-compose exec redis redis-cli LMOVE media:sync:failed media:sync:pending RIGHT LEFT
```

## Performance

| Operation | Throughput | Latency (p95) |
|-----------|-----------|---------------|
| Queue pop (Redis) | N/A | <1ms |
| Local file read | 100 MB/s | <100ms |
| Remote copy/upload | depends on link | 1-10s |
| Database update | N/A | <50ms |

**Bottleneck**: Network upload to Hetzner (limited by uplink bandwidth).

**Scaling**: Add more workers - they process jobs independently from the same queue.

## Related Documentation

- [Media Storage Architecture](../media-storage.md) - Complete storage design (includes multi-box)
- [Multi-Storage Setup](../../operator-guide/multi-storage-setup.md) - Adding multiple storage boxes
- [Processor Service](processor.md) - Upstream service that queues sync jobs
- [API Service](api.md) - Media delivery and routing
- [Hetzner Storage](../../operator-guide/hetzner-storage.md) - Single-box storage configuration
