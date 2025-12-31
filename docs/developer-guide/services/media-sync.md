# Media Sync Service

> Background worker for asynchronous media synchronization from local buffer to remote storage

## Overview

The Media Sync Service handles the asynchronous transfer of media files from the local SSD buffer to Hetzner Storage Boxes via MinIO. This decouples fast local writes from slow network uploads, allowing the processor to return immediately after saving media locally.

**Pipeline Position**: `Processor → Local Buffer → Media Sync → MinIO → Hetzner Storage`

**Key Technologies**:

- **Redis**: Job queue with BRPOP (blocking pop)
- **MinIO**: S3-compatible object storage client
- **asyncio**: High-performance async I/O
- **SQLAlchemy**: Database updates for sync status

**Core Responsibilities**:

1. Monitor Redis queue for pending sync jobs
2. Upload files from local buffer to MinIO
3. Verify upload integrity (size comparison)
4. Update database with sync timestamp
5. Delete local files after successful sync
6. Handle failures with exponential backoff retry

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
        MINIO[MinIO]
        HETZNER[Hetzner Storage Box]
    end

    PROC -->|1. Write media| BUFFER
    PROC -->|2. Queue sync job| REDIS
    REDIS -->|BRPOP| WORKER
    WORKER -->|Read file| BUFFER
    WORKER -->|Upload| MINIO
    MINIO -->|SSHFS| HETZNER
    WORKER -->|Update synced_at| PG
    WORKER -->|Delete after sync| BUFFER
    WORKER -->|On error| RETRY
    RETRY -->|Requeue| REDIS
```

### Key Files

| File | Purpose |
|------|---------|
| `src/worker.py` | Main worker implementation, queue processing |
| `Dockerfile` | Container build definition |
| `requirements.txt` | Python dependencies |

### Data Flow

```mermaid
sequenceDiagram
    participant PROC as Processor
    participant BUFFER as Local Buffer
    participant REDIS as Redis Queue
    participant WORKER as Media Sync Worker
    participant MINIO as MinIO
    participant DB as PostgreSQL

    PROC->>BUFFER: 1. Write media file
    PROC->>DB: 2. Insert media_files record
    PROC->>REDIS: 3. LPUSH sync job
    Note over PROC: Returns immediately

    WORKER->>REDIS: BRPOP (blocking, 5s timeout)
    REDIS-->>WORKER: Job data (JSON)
    WORKER->>BUFFER: Read file
    WORKER->>MINIO: fput_object(s3_key)
    WORKER->>MINIO: stat_object (verify size)
    WORKER->>DB: UPDATE synced_at = NOW()
    WORKER->>BUFFER: Delete local file
```

## Configuration

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `REDIS_URL` | Redis connection string | `redis://localhost:6379/0` | Yes |
| `DATABASE_URL` | PostgreSQL connection | - | Yes |
| `MINIO_ENDPOINT` | MinIO endpoint | `localhost:9000` | Yes |
| `MINIO_ACCESS_KEY` | MinIO access key | `minioadmin` | Yes |
| `MINIO_SECRET_KEY` | MinIO secret key | `minioadmin` | Yes |
| `MINIO_BUCKET` | Target bucket name | `osint-media` | No |
| `MINIO_SECURE` | Use HTTPS | `false` | No |

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
      - ${MEDIA_BUFFER_PATH:-./data/media-buffer}:/var/cache/osint-media-buffer:rshared
```

!!! danger "Critical: Same Buffer Path as Processor"
    The media-sync service MUST mount the same buffer path as the processor service.

## Multi-Box Storage Support

The media-sync worker supports multiple storage boxes using `MinioClientPool`:

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
  "storage_box_id": "russia-1",   // Routes to minio-russia-1
  "file_size": 12345,
  "queued_at": "2024-01-15T12:00:00"
}
```

### Usage Tracking

After successful sync, the worker updates box usage:

```sql
UPDATE storage_boxes
SET used_bytes = used_bytes + :file_size,
    reserved_bytes = GREATEST(0, reserved_bytes - :file_size)
WHERE id = :storage_box_id
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

### Exponential Backoff

Failed uploads retry with increasing delays:

- Attempt 1: 5 seconds
- Attempt 2: 10 seconds
- Attempt 3: 15 seconds
- After 3 failures: Move to failed queue

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

# Scale to multiple workers
docker-compose up -d --scale media-sync=3
```

### Local Development

```bash
cd services/media-sync
pip install -r requirements.txt

export REDIS_URL=redis://localhost:6379/0
export DATABASE_URL=postgresql+asyncpg://osint_user:password@localhost:5432/osint_platform
export MINIO_ENDPOINT=localhost:9000
export MINIO_ACCESS_KEY=minioadmin
export MINIO_SECRET_KEY=minioadmin

python -m src.worker
```

## Troubleshooting

### High Queue Backlog

**Symptom**: `media:sync:pending` length increasing

**Causes**:
- Slow network to Hetzner
- Sync workers down/crashed
- MinIO overloaded

**Solutions**:
```bash
# Scale workers
docker-compose up -d --scale media-sync=5

# Check MinIO health
docker-compose logs minio | tail -50
```

### Files Not Syncing

**Debug Steps**:
```bash
# Check worker is running
docker-compose ps media-sync

# Check queue has jobs
docker-compose exec redis redis-cli LLEN media:sync:pending

# Test MinIO connectivity
docker-compose exec media-sync python3 -c "
from minio import Minio
client = Minio('minio:9000', 'minioadmin', 'minioadmin', secure=False)
print(client.bucket_exists('osint-media'))
"
```

### Permanent Failures

**Inspect Failed Jobs**:
```bash
docker-compose exec redis redis-cli LRANGE media:sync:failed 0 0
```

**Common errors**:
- `File missing from both local buffer and MinIO`
- `Size mismatch: expected X, got Y`
- `Connection refused` (MinIO down)

**Requeue failed jobs**:
```bash
docker-compose exec redis redis-cli RPOPLPUSH media:sync:failed media:sync:pending
```

## Performance

| Operation | Throughput | Latency (p95) |
|-----------|-----------|---------------|
| Queue pop (Redis) | N/A | <1ms |
| Local file read | 100 MB/s | <100ms |
| MinIO upload | 10-50 files/sec | 1-10s |
| Database update | N/A | <50ms |

**Bottleneck**: Network upload to Hetzner (limited by uplink bandwidth).

**Scaling**: Add more workers - they process jobs independently from the same queue.

## Related Documentation

- [Media Storage Architecture](../media-storage.md) - Complete storage design (includes multi-box)
- [Multi-Storage Setup](../../operator-guide/multi-storage-setup.md) - Adding multiple storage boxes
- [Processor Service](processor.md) - Upstream service that queues sync jobs
- [API Service](api.md) - Media delivery and routing
- [Hetzner Storage](../../operator-guide/hetzner-storage.md) - Single-box storage configuration
