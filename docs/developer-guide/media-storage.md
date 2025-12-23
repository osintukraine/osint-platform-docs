# Media Storage Architecture

This guide covers the technical architecture of the media storage system, including the hybrid local buffer + Hetzner storage design.

## Overview

The platform uses a **hybrid storage architecture** designed for:

1. **Fast browser response** - Local SSD buffer for hot files
2. **Cost-effective scaling** - Hetzner Storage Boxes (~€3.80/TB/month)
3. **High availability** - Redis-cached routing (99%+ hit rate)
4. **Zero data loss** - Local buffer first, async sync to Hetzner

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         WRITE PATH                                       │
└─────────────────────────────────────────────────────────────────────────┘

Telegram Message with Media
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          Processor Service                                │
│                                                                          │
│  1. Download to .tmp/                                                    │
│     /var/cache/osint-media-buffer/.tmp/{uuid}.tmp                       │
│                                                                          │
│  2. Compute SHA-256 hash                                                 │
│                                                                          │
│  3. Check deduplication (media_files table)                             │
│     └── EXISTS? → Return existing record                                │
│                                                                          │
│  4. Atomic move to local buffer:                                        │
│     .tmp/{uuid}.tmp → osint-media/media/ab/cd/abcd1234.jpg             │
│                                                                          │
│  5. Insert into media_files:                                            │
│     - sha256, s3_key, storage_box_id                                    │
│     - local_path (buffer path)                                          │
│     - synced_at = NULL (not yet synced)                                 │
│                                                                          │
│  6. Queue sync job to Redis: media:sync:pending                         │
│     { sha256, s3_key, local_path, storage_box_id, file_size }          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼ (async, background worker)
┌──────────────────────────────────────────────────────────────────────────┐
│                       Media-Sync Worker                                   │
│                                                                          │
│  1. BRPOP from media:sync:pending queue                                 │
│                                                                          │
│  2. Upload to MinIO (backed by Hetzner SSHFS)                           │
│     minio.fput_object(bucket, s3_key, local_path)                       │
│                                                                          │
│  3. Verify upload (stat_object, check size)                             │
│                                                                          │
│  4. Update database:                                                     │
│     UPDATE media_files SET synced_at = NOW(), local_path = NULL         │
│     WHERE sha256 = :sha256                                              │
│                                                                          │
│  5. Delete local file (space reclaimed)                                 │
│                                                                          │
│  6. Populate Redis cache:                                                │
│     SET media:route:{sha256} {storage_box_id} EX 86400                  │
└──────────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         READ PATH                                        │
└─────────────────────────────────────────────────────────────────────────┘

Browser Request: GET /media/ab/cd/abcd1234.jpg
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                           Caddy                                          │
│                                                                          │
│  Layer 1: Local Buffer Check (SSD)                                      │
│  └── Path: /var/cache/osint-media-buffer/osint-media/media/ab/cd/...   │
│  └── EXISTS? → Serve directly (< 1ms)                                   │
│      Header: X-Media-Source: local-buffer                               │
│                                                                          │
│  Layer 2: API Redirect (cache miss)                                     │
│  └── Rewrite: /api/media/internal/media-redirect/ab/cd/abcd1234.jpg    │
│  └── Proxy to API service                                               │
└──────────────────────────────────────────────────────────────────────────┘
         │ (cache miss)
         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                           API Service                                    │
│                                                                          │
│  Layer 3: Redis Cache Lookup                                            │
│  └── Key: media:route:{sha256_prefix}                                   │
│  └── HIT? → Return 302 redirect to storage box URL                      │
│      Header: X-Cache-Status: hit                                        │
│                                                                          │
│  Layer 4: Database Lookup (cache miss)                                  │
│  └── SELECT storage_box_id FROM media_files WHERE sha256 = :hash       │
│  └── Populate Redis cache (24h TTL)                                     │
│  └── Return 302 redirect                                                │
│      Header: X-Cache-Status: miss                                       │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
    HTTP 302 Redirect → Storage Box URL
```

## Key Components

### 1. Local Buffer

**Location**: `/var/cache/osint-media-buffer/`

**Structure**:
```
/var/cache/osint-media-buffer/
├── osint-media/
│   └── media/
│       └── {hash[:2]}/
│           └── {hash[2:4]}/
│               └── {hash}.{ext}
└── .tmp/                        # Atomic download staging
```

**Purpose**:
- Fast SSD writes during message processing
- Instant browser access for newly uploaded media
- Staging area before async sync to Hetzner

**Code** (`services/processor/src/media_archiver.py`):
```python
# Constants
MEDIA_BUFFER_PATH = os.environ.get("MEDIA_BUFFER_PATH", "/var/cache/osint-media-buffer")
LOCAL_BUFFER_ROOT = Path(MEDIA_BUFFER_PATH) / "osint-media"
LOCAL_BUFFER_TMP = Path(MEDIA_BUFFER_PATH) / ".tmp"

def _get_local_buffer_path(self, sha256: str, extension: str) -> Path:
    """Generate local buffer path using content-addressed structure."""
    return LOCAL_BUFFER_ROOT / "media" / sha256[:2] / sha256[2:4] / f"{sha256}{extension}"
```

### 2. Redis Sync Queue

**Queue Name**: `media:sync:pending`

**Job Format**:
```json
{
  "sha256": "abcd1234567890...",
  "s3_key": "media/ab/cd/abcd1234567890.jpg",
  "local_path": "/var/cache/osint-media-buffer/osint-media/media/ab/cd/abcd1234567890.jpg",
  "storage_box_id": "default",
  "file_size": 12345,
  "queued_at": "2024-01-15T12:00:00",
  "retry_count": 0
}
```

**Failed Queue**: `media:sync:failed` (after 3 retries)

**Code** (`services/processor/src/media_archiver.py`):
```python
async def _queue_sync_job(self, sha256: str, s3_key: str, local_path: str, file_size: int):
    """Queue a media file for background sync to Hetzner storage."""
    job = {
        "sha256": sha256,
        "s3_key": s3_key,
        "local_path": local_path,
        "storage_box_id": self.storage_box_id,
        "file_size": file_size,
        "queued_at": datetime.utcnow().isoformat(),
    }
    await self.redis.lpush(MEDIA_SYNC_QUEUE, json.dumps(job))
```

### 3. Media-Sync Worker

**Location**: `services/media-sync/src/worker.py`

**Flow**:
1. `BRPOP media:sync:pending` (blocking wait)
2. Upload file to MinIO via `fput_object()`
3. Verify upload with `stat_object()`
4. Update `media_files.synced_at` timestamp
5. Delete local file to reclaim space
6. On failure: retry up to 3 times, then move to failed queue

**Key Methods**:
```python
class MediaSyncWorker:
    async def process_job(self, job_data: str):
        """Process a single sync job."""
        job = json.loads(job_data)
        sha256 = job["sha256"]
        s3_key = job["s3_key"]
        local_path = Path(job["local_path"])

        # Upload to MinIO
        self.minio.fput_object(
            bucket_name=MINIO_BUCKET,
            object_name=s3_key,
            file_path=str(local_path),
            content_type=self._guess_mime_type(local_path)
        )

        # Verify and update
        stat = self.minio.stat_object(MINIO_BUCKET, s3_key)
        await self._mark_synced(sha256)
        local_path.unlink(missing_ok=True)

    async def _mark_synced(self, sha256: str):
        """Update database to mark file as synced."""
        async with self.async_session() as session:
            await session.execute(
                text("""
                    UPDATE media_files
                    SET synced_at = :now, local_path = NULL
                    WHERE sha256 = :sha256
                """),
                {"sha256": sha256, "now": datetime.utcnow()}
            )
            await session.commit()
```

### 4. Redis Route Cache

**Purpose**: Eliminate database lookups for media routing

**Key Format**: `media:route:{sha256_prefix}`

**Value**: Storage box ID (e.g., "default", "russia-1")

**TTL**: 86,400 seconds (24 hours)

**Code** (`services/api/src/routers/media.py`):
```python
# Check Redis cache first
cached_box = await redis.get(f"media:route:{clean_hash}")
if cached_box:
    return RedirectResponse(
        url=f"/storage/{cached_box}/{path}",
        status_code=307,
        headers={
            "X-Cache-Status": "hit",
            "X-Media-Source": "storage-box"
        }
    )

# Cache miss - query database
result = await db.execute(
    text("SELECT storage_box_id FROM media_files WHERE sha256 = :hash"),
    {"hash": clean_hash}
)
box_id = result.scalar_one_or_none()

# Populate cache
if box_id:
    await redis.setex(f"media:route:{clean_hash}", CACHE_TTL, box_id)
```

### 5. Caddy Media Routing

**Location**: `infrastructure/caddy/Caddyfile.ory`

**Strategy**:
1. **Fast path**: Try local buffer with `file` matcher
2. **Fallback**: Rewrite to API redirect endpoint

```caddyfile
handle /media/* {
    # Set root to local media buffer
    root * /var/cache/osint-media-buffer/osint-media

    # Matcher: file exists in local buffer (hot path)
    @local_file file {path}

    # Serve from local buffer if file exists
    handle @local_file {
        header Cache-Control "public, max-age=31536000, immutable"
        header X-Media-Source "local-buffer"
        file_server
    }

    # Fallback: API redirect for files not in local buffer
    handle {
        rewrite * /api/media/internal/media-redirect{path}
        reverse_proxy api:8000 {
            header_up X-Internal-Request "caddy-media-fallback"
        }
    }
}
```

## Database Schema

### storage_boxes Table

```sql
CREATE TABLE IF NOT EXISTS storage_boxes (
    id VARCHAR(50) PRIMARY KEY,                  -- "default", "russia-1", "ukraine-1"
    hetzner_host VARCHAR(255) NOT NULL,          -- "uXXXXXX.your-storagebox.de"
    hetzner_user VARCHAR(50) NOT NULL,           -- "uXXXXXX"
    hetzner_port INTEGER DEFAULT 23,
    mount_path VARCHAR(255) NOT NULL,            -- "/mnt/hetzner-storage"
    capacity_gb INTEGER NOT NULL,
    used_gb INTEGER DEFAULT 0,
    account_region VARCHAR(20) NOT NULL,         -- "russia" or "ukraine"
    is_active BOOLEAN DEFAULT true,
    is_full BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITHOUT TIME ZONE DEFAULT NOW(),
    last_health_check TIMESTAMP WITHOUT TIME ZONE
);
```

### media_files Table (Extended Columns)

```sql
-- Added columns for storage routing
ALTER TABLE media_files ADD COLUMN IF NOT EXISTS
    storage_box_id VARCHAR(50) REFERENCES storage_boxes(id);
ALTER TABLE media_files ADD COLUMN IF NOT EXISTS
    synced_at TIMESTAMP WITHOUT TIME ZONE;       -- NULL = pending sync
ALTER TABLE media_files ADD COLUMN IF NOT EXISTS
    local_path TEXT;                             -- Path in local buffer (if not synced)
ALTER TABLE media_files ADD COLUMN IF NOT EXISTS
    last_accessed_at TIMESTAMP WITHOUT TIME ZONE;
ALTER TABLE media_files ADD COLUMN IF NOT EXISTS
    access_count INTEGER DEFAULT 0;

-- Indexes for performance
CREATE INDEX idx_media_files_pending_sync ON media_files(synced_at) WHERE synced_at IS NULL;
CREATE INDEX idx_media_files_routing ON media_files(sha256, storage_box_id);
```

## API Endpoints

### Internal Media Redirect

**Endpoint**: `GET /api/media/internal/media-redirect/{file_hash:path}`

**Purpose**: Route media requests to appropriate storage (local buffer or Hetzner)

**Flow**:
1. Extract hash from path (`ab/cd/abcd1234.jpg` → `abcd1234`)
2. Check Redis cache: `media:route:{hash}`
3. Cache miss: Query `media_files.storage_box_id`
4. Cache result (24h TTL)
5. Return 302 redirect

**Response Headers**:
```http
HTTP/1.1 302 Found
Location: /storage/default/media/ab/cd/abcd1234.jpg
X-Media-Source: storage-box
X-Cache-Status: hit|miss|populated
```

### Cache Invalidation

**Endpoint**: `POST /api/media/internal/media-invalidate/{file_hash}`

**Purpose**: Invalidate Redis cache when file is moved or deleted

### Cache Statistics

**Endpoint**: `GET /api/media/internal/media-stats`

**Response**:
```json
{
  "cache_size": 15234,
  "estimated_hit_rate": 0.994,
  "cache_ttl_seconds": 86400
}
```

## Performance Characteristics

### Latency Breakdown

| Layer | Cache Hit | Cache Miss | Notes |
|-------|-----------|------------|-------|
| **Local Buffer** | <1ms | N/A | SSD read |
| **Redis Cache** | ~1ms | N/A | In-memory lookup |
| **Database Query** | N/A | 5-10ms | PostgreSQL SELECT |
| **SSHFS Read** | N/A | 20-50ms | Network + disk |
| **Total (Hit)** | 1-2ms | N/A | 99%+ of requests |
| **Total (Miss)** | N/A | 30-70ms | First access only |

### Expected Cache Hit Rates

| Scenario | Hit Rate | Notes |
|----------|----------|-------|
| **Local Buffer** | 80-95% | Recent uploads, hot files |
| **Redis Cache** | 99%+ | After warm-up period |
| **Combined** | 99.5%+ | Almost no DB queries |

## Adding a New Storage Box

1. **Create database entry**:
```sql
INSERT INTO storage_boxes (id, hetzner_host, hetzner_user, hetzner_port, mount_path, capacity_gb, account_region)
VALUES ('russia-2', 'uXXXXXX.your-storagebox.de', 'uXXXXXX', 23, '/mnt/storage/russia-2', 20000, 'russia');
```

2. **Update Caddy configuration** to handle the new storage path

3. **Configure SSHFS mount** via systemd

4. **Update storage box selection logic** (if using automatic routing)

## Extending the Architecture

### Custom Storage Box Selection

Override in `media_archiver.py`:
```python
def _select_storage_box(self, account_region: str) -> str:
    """Select storage box based on account region and capacity."""
    # Query storage_boxes table for available boxes
    # Implement round-robin or least-full strategy
    pass
```

### Cache Warming

Pre-populate cache for popular files:
```python
async def warm_cache(db, redis, limit=10000):
    """Warm Redis cache with popular files."""
    popular = await db.execute(
        text("""
            SELECT sha256, storage_box_id
            FROM media_files
            WHERE last_accessed_at > NOW() - INTERVAL '7 days'
            ORDER BY access_count DESC
            LIMIT :limit
        """),
        {"limit": limit}
    )
    for sha256, box_id in popular:
        await redis.setex(f"media:route:{sha256}", 86400, box_id)
```

## Related Documentation

- [Operator Guide: Hetzner Storage](../operator-guide/hetzner-storage.md) - Setup and maintenance
- [Reference: API Endpoints](../reference/api-endpoints.md) - API endpoint reference (includes media endpoints)
- [Architecture Overview](./architecture.md) - System architecture
