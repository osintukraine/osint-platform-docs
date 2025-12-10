# Migration Service

## Overview

The Migration Service handles the migration of historical data from legacy tg-archive SQLite databases to the new OSINT Intelligence Platform. This service is designed to transfer 3 years of archived Telegram data while applying production-quality spam filtering and deduplication.

**Status**: Under Development

**Purpose**: Migrate historical tg-archive data to PostgreSQL and MinIO with enrichment and filtering

**Location**: `/services/migration/`

### Migration Scope

| Metric | Value |
|--------|-------|
| **Databases** | 3 SQLite databases |
| **Messages** | 671,418 messages (before filtering) |
| **Media Files** | 531,880 files |
| **Date Range** | 2022-04-07 to 2025-04-19 (3 years) |
| **Expected Output** | ~200K-400K messages after spam filtering |

### Database Inventory

Based on tg-archive-video archives:

| Database | Messages | Media | Date Range |
|----------|----------|-------|------------|
| ruvideos2022-2024 | 467,866 | 353,678 | 2022-04-07 → 2024-12-24 |
| uavideos | 126,212 | 117,311 | 2022-12-08 → 2025-04-19 |
| ruvideos | 77,340 | 60,891 | 2024-12-23 → 2025-04-19 |

## Architecture

### Key Components

```
services/migration/
├── src/
│   ├── indexers/           # Database scanning and inventory
│   │   ├── sqlite_indexer.py
│   │   └── sqlite_schema.py
│   ├── storage/            # Storage backend abstraction
│   │   ├── storage_backend.py
│   │   ├── minio_backend.py
│   │   └── hetzner_backend.py
│   ├── processors/         # Message enrichment pipeline
│   ├── media/              # Media handling
│   └── utils/              # Progress tracking, validation
├── tests/                  # Unit tests and integration tests
└── migrations/             # Progress tracking database
```

### Schema Mapping

The migration maps legacy tg-archive schema to the new platform schema:

#### Legacy Schema (SQLite)

```sql
-- tg-archive database structure
messages (
    id INTEGER,              -- Telegram message ID
    type TEXT,              -- 'message' or 'service'
    date TEXT,              -- ISO timestamp
    content TEXT,           -- Message text
    user_id INTEGER,        -- Channel Telegram ID
    media_id INTEGER        -- Foreign key to media table
)

users (
    id INTEGER,             -- Telegram channel ID
    username TEXT,
    first_name TEXT,
    last_name TEXT
)

media (
    id INTEGER,
    type TEXT,              -- MIME type
    url TEXT,               -- Relative path: '2022-12-12/video.mp4'
    title TEXT              -- Original filename
)
```

#### New Platform Schema (PostgreSQL)

```sql
-- OSINT Platform schema
channels (
    id SERIAL PRIMARY KEY,
    telegram_id BIGINT UNIQUE,  -- Maps from users.id
    username TEXT,
    name TEXT,
    folder TEXT,
    rule TEXT
)

messages (
    id SERIAL PRIMARY KEY,
    channel_id INTEGER REFERENCES channels(id),
    telegram_message_id BIGINT,    -- Maps from messages.id
    content TEXT,
    created_at TIMESTAMP,
    osint_score INTEGER,
    is_spam BOOLEAN
)

media_files (
    id SERIAL PRIMARY KEY,
    sha256_hash TEXT UNIQUE,       -- Computed during migration
    file_path TEXT,                -- S3 path
    size_bytes BIGINT,
    mime_type TEXT
)

message_media (
    message_id INTEGER REFERENCES messages(id),
    media_file_id INTEGER REFERENCES media_files(id),
    media_order INTEGER
)
```

#### Key Mapping Logic

| Legacy | New Platform | Transformation |
|--------|-------------|----------------|
| `users.id` | `channels.telegram_id` | Direct mapping |
| `messages.user_id` | `channels.telegram_id` | Lookup channel |
| `messages.id` | `messages.telegram_message_id` | Not primary key |
| `media.url` | `media_files.sha256_hash` | Compute hash from file |

!!! warning "Important Schema Differences"
    - Legacy `messages.id` is the Telegram message ID, not a database primary key
    - New platform uses auto-increment primary keys with `telegram_message_id` as separate field
    - Media deduplication via SHA-256 hash instead of path-based storage

## Migration Process

The migration follows a phased approach with validation checkpoints.

### Phase 1: Indexing (Week 1-2)

**Goal**: Build comprehensive inventory of legacy data

**Process**:

1. Scan all SQLite databases in tg-archive sites directory
2. Extract message counts, date ranges, channel mappings
3. Identify media references and file paths
4. Generate inventory report

**Implementation**: `src/indexers/sqlite_indexer.py`

```python
from migration.src.indexers.sqlite_indexer import SQLiteIndexer
from pathlib import Path

indexer = SQLiteIndexer()
inventories = await indexer.scan_all_databases(
    Path("/path/to/tg-archive-video/sites")
)

# Output statistics
total_messages = sum(inv.total_messages for inv in inventories)
total_media = sum(inv.total_media for inv in inventories)
```

**Deliverables**:

- Message count per database
- Date range coverage
- Channel/user mapping
- Media reference inventory

### Phase 2: Database Migration (Week 3-6)

**Goal**: Migrate messages to PostgreSQL with enrichment

**Process per message**:

1. Extract message from SQLite
2. **Spam filter** using production rules
3. **OSINT scoring** (optional, can be skipped for speed)
4. **Entity extraction** (hashtags, mentions, coordinates)
5. Insert into PostgreSQL
6. Track progress in `progress.db`

**Expected Filtering**:

```
671,418 messages (raw)
    ↓ Spam filter (30-60% reduction)
270,000-470,000 messages
    ↓ Low OSINT score filter (<50)
200,000-400,000 messages (migrated)
```

**Spam Filter Applied**:

The migration reuses the production spam filter from the processor service:

- Content patterns (forwarding artifacts, formatting noise)
- Low-content messages
- Service messages (ignored completely)
- Duplicate detection

!!! tip "Performance Tuning"
    OSINT scoring can be disabled during migration for faster processing. Enrichment can be run afterward as a batch job using the enrichment service.

### Phase 3: Media Migration (Week 7-10)

**Goal**: Copy media files to S3 with deduplication

**Process per media file**:

1. Compute SHA-256 hash of file
2. Check if hash already exists in S3 (deduplication)
3. If not exists:
    - Read from Hetzner storage box mount, OR
    - Download from Telegram API (if file missing)
4. Upload to MinIO with content-addressed path
5. Update `message_media` junction table

**Deduplication Strategy**:

```
531,880 media files (raw)
    ↓ SHA-256 deduplication (30-40% reduction)
320,000-372,000 unique files
    ↓ Orphaned media (spam filtered messages)
200,000-300,000 files (migrated)
```

**Content-Addressed Storage**:

Media files are stored using SHA-256 hash paths:

```
media/{hash[:2]}/{hash[2:4]}/{hash}.{ext}
```

Example: `media/a1/b2/a1b2c3d4...xyz.mp4`

### Phase 4: Validation (Week 11-12)

**Goal**: Verify data integrity and completeness

**Validation Checks**:

- [ ] Message counts match expectations (<5% discrepancy)
- [ ] Media accessibility (>95% success rate)
- [ ] Random sample verification vs Telegram API (>90% match)
- [ ] Database constraints verified
- [ ] Foreign key relationships intact
- [ ] No orphaned records

## Storage Backend

The migration service uses a pluggable storage backend system to support different deployment scenarios.

### Storage Backend Abstraction

Abstract interface defined in `src/storage/storage_backend.py`:

```python
class StorageBackend(ABC):
    """Abstract storage backend for object storage"""

    @abstractmethod
    async def put_object(bucket: str, key: str, data: BinaryIO) -> bool

    @abstractmethod
    async def get_object(bucket: str, key: str) -> Optional[bytes]

    @abstractmethod
    async def stat_object(bucket: str, key: str) -> Optional[dict]

    @abstractmethod
    async def remove_object(bucket: str, key: str) -> bool

    @abstractmethod
    async def list_objects(bucket: str, prefix: str = "") -> list[str]
```

### MinIO Backend (Default)

Standard S3-compatible storage using MinIO client.

**Implementation**: `src/storage/minio_backend.py`

```python
from migration.src.storage.minio_backend import MinIOBackend

storage = MinIOBackend(
    endpoint="localhost:9000",
    access_key="minioadmin",
    secret_key="minioadmin",
    secure=False
)

# Upload media file
with open("video.mp4", "rb") as f:
    await storage.put_object(
        bucket="media",
        key=f"migrated/{sha256_hash}.mp4",
        data=f,
        length=file_size,
        content_type="video/mp4"
    )
```

**Use Cases**:

- Standard deployments
- No external dependencies
- Works everywhere

### Hetzner S3 Bridge (Cost-Optimized)

Bridges S3 API to Hetzner Storage Box via filesystem mount (SSHFS/WebDAV/CIFS).

**Implementation**: `src/storage/hetzner_backend.py`

```python
from migration.src.storage.hetzner_backend import HetznerS3Bridge

# Prerequisites: Storage box must be mounted
# sshfs u123456@u123456.your-storagebox.de:/ /mnt/hetzner-storage

storage = HetznerS3Bridge(
    mount_point="/mnt/hetzner-storage",
    protocol="sshfs"
)

# Same API as MinIO backend
await storage.put_object(bucket, key, data, length)
```

**Use Cases**:

- Large media libraries
- Cost-sensitive deployments
- Hetzner Storage Box customers

**Cost Comparison**:

| Storage | Capacity | Cost/Month | Use Case |
|---------|----------|------------|----------|
| MinIO (local) | Limited by disk | Included | Development, small datasets |
| MinIO (S3) | Unlimited | Variable | Cloud deployments |
| Hetzner Bridge | 20TB | €5.49 | Large archives, cost optimization |

!!! note "Storage Box Mounting"
    The Hetzner backend requires the storage box to be mounted on the host filesystem before use. Multiple mount protocols are supported:

    - **SSHFS**: Most compatible, easier setup
    - **WebDAV**: Built-in support
    - **CIFS**: Windows compatibility

## Running the Migration

### Prerequisites

1. **Platform services running**:
    ```bash
    cd /path/to/osint-intelligence-platform
    docker-compose up -d postgres redis minio
    ```

2. **Access to legacy databases**:
    ```bash
    # Verify SQLite databases are accessible
    ls -lh /path/to/tg-archive-video/sites/*/data.sqlite
    ```

3. **Storage box mounted** (if using Hetzner backend):
    ```bash
    # Mount Hetzner storage box
    sshfs u123456@u123456.your-storagebox.de:/ /mnt/hetzner-storage

    # Verify mount
    ls -lh /mnt/hetzner-storage
    ```

### Phase 1: Run Indexing

Scan all legacy databases and build inventory:

```bash
python -m services.migration.src.indexers.sqlite_indexer \
    --sites-dir /path/to/tg-archive-video/sites \
    --output-json /tmp/migration_inventory.json
```

**Expected Output**:

```
============================================================
PHASE 1: DATABASE INDEXING
============================================================

Scanning ruvideos2022-2024...
  Database: ruvideos2022-2024
  Messages: 467,866
  Media: 353,678
  Date range: 2022-04-07 → 2024-12-24
  Channels: 45

Scanning uavideos...
  Database: uavideos
  Messages: 126,212
  Media: 117,311
  Date range: 2022-12-08 → 2025-04-19
  Channels: 28

Scanning ruvideos...
  Database: ruvideos
  Messages: 77,340
  Media: 60,891
  Date range: 2024-12-23 → 2025-04-19
  Channels: 15

============================================================
INDEXING SUMMARY
============================================================
Databases scanned: 3
Total messages: 671,418
Total media: 531,880
Earliest date: 2022-04-07
Latest date: 2025-04-19
```

### Phase 2: Run Database Migration

!!! warning "Work in Progress"
    Database migration implementation is under development. The following is the planned interface.

```bash
# Planned command
python -m services.migration.src.controllers.migration_controller \
    --phase database \
    --config services/migration/config/migration_config.yaml \
    --inventory /tmp/migration_inventory.json
```

### Phase 3: Run Media Migration

!!! warning "Work in Progress"
    Media migration implementation is under development. The following is the planned interface.

```bash
# Planned command
python -m services.migration.src.controllers.migration_controller \
    --phase media \
    --config services/migration/config/migration_config.yaml \
    --storage-backend hetzner_bridge
```

## Configuration

Migration behavior is configured via YAML file.

**Location**: `services/migration/config/migration_config.yaml`

```yaml
# Source databases
source:
  sqlite_dir: "/path/to/tg-archive-video/sites"
  exclude_patterns:
    - "*/backup/*"
    - "*/test/*"

# Storage backend selection
storage:
  backend: "hetzner_bridge"  # or "minio"

  # Hetzner bridge configuration
  hetzner:
    mount_point: "/mnt/hetzner-storage"
    protocol: "sshfs"

  # MinIO configuration (alternative)
  minio:
    endpoint: "localhost:9000"
    access_key: "${MINIO_ROOT_USER}"
    secret_key: "${MINIO_ROOT_PASSWORD}"
    secure: false

# Processing options
processing:
  batch_size: 1000              # Messages per batch
  enable_spam_filter: true      # Apply production spam rules
  enable_osint_scoring: false   # Disable for faster migration
  min_osint_score: 50          # More inclusive than production (70)
  parallel_workers: 4           # Concurrent processing workers

# Database connection
database:
  postgres:
    host: "localhost"
    port: 5432
    database: "osint_platform"
    user: "osint_user"
    password: "${POSTGRES_PASSWORD}"

# Progress tracking
progress:
  database: "services/migration/migrations/progress.db"
  checkpoint_frequency: 100     # Save progress every N messages
  log_level: "INFO"
```

### Environment Variables

The following environment variables can be used:

| Variable | Description | Default |
|----------|-------------|---------|
| `POSTGRES_PASSWORD` | PostgreSQL password | (required) |
| `MINIO_ROOT_USER` | MinIO access key | `minioadmin` |
| `MINIO_ROOT_PASSWORD` | MinIO secret key | `minioadmin` |
| `MIGRATION_LOG_LEVEL` | Logging verbosity | `INFO` |

## Progress Tracking

Migration progress is tracked in a SQLite database to support resumption after interruption.

**Location**: `services/migration/migrations/progress.db`

### Progress Schema

```sql
-- Overall phase progress
CREATE TABLE migration_progress (
    id INTEGER PRIMARY KEY,
    phase TEXT NOT NULL,        -- 'indexing', 'database', 'media'
    database_name TEXT,
    status TEXT NOT NULL,       -- 'pending', 'processing', 'completed', 'failed'
    total_items INTEGER,
    processed_items INTEGER DEFAULT 0,
    started_at TIMESTAMP,
    completed_at TIMESTAMP
);

-- Individual message tracking
CREATE TABLE message_migration (
    id INTEGER PRIMARY KEY,
    source_db TEXT NOT NULL,
    source_message_id INTEGER NOT NULL,
    target_message_id INTEGER,  -- New PostgreSQL ID
    status TEXT NOT NULL,       -- 'pending', 'spam', 'migrated', 'failed'
    spam_reason TEXT,
    osint_score INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Media file tracking
CREATE TABLE media_migration (
    id INTEGER PRIMARY KEY,
    source_path TEXT NOT NULL,
    sha256_hash TEXT,
    target_s3_key TEXT,
    status TEXT NOT NULL,       -- 'pending', 'exists', 'copied', 'failed'
    size_bytes INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Viewing Progress

Check migration progress:

```bash
sqlite3 services/migration/migrations/progress.db "
  SELECT
    phase,
    database_name,
    status,
    processed_items || '/' || total_items as progress,
    ROUND(processed_items * 100.0 / total_items, 1) || '%' as pct
  FROM migration_progress
  ORDER BY started_at;
"
```

**Example Output**:

```
phase     database_name         status      progress      pct
--------  --------------------  ----------  ------------  ------
indexing  ruvideos2022-2024     completed   467866/467866 100.0%
indexing  uavideos              completed   126212/126212 100.0%
indexing  ruvideos              completed   77340/77340   100.0%
database  ruvideos2022-2024     processing  234567/467866 50.1%
database  uavideos              pending     0/126212      0.0%
```

### Resuming Migration

If migration is interrupted, it can be resumed from the last checkpoint:

```bash
# Migration automatically detects incomplete phases
python -m services.migration.src.controllers.migration_controller \
    --resume
```

## Testing

The migration service includes comprehensive tests.

### Running Tests

```bash
# All tests
pytest services/migration/tests/

# SQLite indexer tests
pytest services/migration/tests/test_sqlite_indexer.py -v

# Storage backend tests
pytest services/migration/tests/test_storage_backends.py -v
```

### Test Storage Setup

For testing storage backends, use the provided setup scripts:

```bash
# Setup test storage (with sudo)
./services/migration/tests/setup-test-storage.sh

# Setup test storage (without sudo, using user namespaces)
./services/migration/tests/setup-test-storage-no-sudo.sh
```

### Integration Tests

The migration service includes integration tests that validate the complete flow:

```bash
# Run integration tests with Docker Compose
cd services/migration/tests
docker-compose -f docker-compose.storage-test.yml up -d
pytest test_integration.py -v
docker-compose -f docker-compose.storage-test.yml down
```

## Troubleshooting

### Issue: Mount Point Not Accessible

**Symptom**: `ValueError: Mount point /mnt/hetzner-storage does not exist`

**Solution**:

```bash
# Check if storage box is mounted
mount | grep hetzner

# Remount if needed
fusermount -u /mnt/hetzner-storage
sshfs u123456@u123456.your-storagebox.de:/ /mnt/hetzner-storage -o allow_other

# Verify mount is writable
touch /mnt/hetzner-storage/test.txt && rm /mnt/hetzner-storage/test.txt
```

### Issue: Database Locked

**Symptom**: `sqlite3.OperationalError: database is locked`

**Solution**:

```bash
# Check for active connections
lsof /path/to/tg-archive-video/sites/*/data.sqlite

# Close tg-archive if running
pkill -f tg-archive

# Ensure no other processes are accessing the database
```

### Issue: Out of Disk Space

**Symptom**: Migration fails with disk space errors

**Solution**:

```bash
# Check available space
df -h

# Clean up temporary files
rm -rf /tmp/migration_*.json
rm -rf /tmp/migration_cache/*

# Consider using Hetzner bridge to avoid local storage
```

### Issue: MinIO Connection Failed

**Symptom**: `Connection refused` or `S3 error` during media upload

**Solution**:

```bash
# Verify MinIO is running
docker-compose ps minio

# Check MinIO logs
docker-compose logs minio

# Test connection
curl http://localhost:9000/minio/health/live

# Restart MinIO if needed
docker-compose restart minio
```

### Issue: Slow Migration Performance

**Symptom**: Migration taking longer than expected

**Solutions**:

1. **Disable OSINT scoring** during migration:
    ```yaml
    processing:
      enable_osint_scoring: false
    ```

2. **Increase parallel workers**:
    ```yaml
    processing:
      parallel_workers: 8  # Increase from 4
    ```

3. **Use larger batch sizes**:
    ```yaml
    processing:
      batch_size: 5000  # Increase from 1000
    ```

4. **Skip media migration** and run separately:
    ```bash
    # Migrate database only first
    python -m services.migration.src.controllers.migration_controller \
        --phase database

    # Then migrate media in background
    nohup python -m services.migration.src.controllers.migration_controller \
        --phase media &
    ```

## Rollback Strategy

Migration is designed to be non-destructive.

### Rollback Principles

- **Original data never modified**: Legacy SQLite databases remain intact
- **Original media never deleted**: Storage box files remain untouched
- **Checkpointed progress**: Can restart from any phase
- **Separate progress database**: Migration state is isolated

### Performing Rollback

If migration needs to be rolled back:

```bash
# 1. Stop migration
pkill -f migration

# 2. Drop migrated data from PostgreSQL (if needed)
docker-compose exec postgres psql -U osint_user -d osint_platform -c "
  TRUNCATE messages, media_files, message_media, channels CASCADE;
"

# 3. Reset progress tracker
rm services/migration/migrations/progress.db

# 4. Restart from Phase 1
python -m services.migration.src.indexers.sqlite_indexer \
    --sites-dir /path/to/tg-archive-video/sites
```

!!! warning "MinIO Data"
    If media files were uploaded to MinIO, they will need to be manually deleted:

    ```bash
    # List migrated media
    mc ls local/media/migrated/

    # Remove migrated media
    mc rm --recursive --force local/media/migrated/
    ```

## Expected Outcomes

### Data Volume Reduction

The migration applies production-quality filtering to reduce data volume:

| Metric | Before | After | Reduction |
|--------|--------|-------|-----------|
| **Messages** | 671,418 | 200K-400K | 30-70% |
| **Media Files** | 531,880 | 200K-300K | 40-60% |
| **Storage** | ~60TB | ~20-30TB | 50-67% |

**Cost Impact**: €299/month → ~€230/month (23% reduction)

### Timeline Estimate

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| Phase 1: Indexing | Week 1-2 | Complete inventory |
| Phase 2: Database | Week 3-6 | Messages in PostgreSQL |
| Phase 3: Media | Week 7-10 | Media in MinIO/S3 |
| Phase 4: Validation | Week 11-12 | Verified migration |
| **Total** | **2-3 months** | **Production ready** |

### Quality Metrics

Target quality thresholds:

- **Message accuracy**: <5% discrepancy from source
- **Media accessibility**: >95% success rate
- **API verification**: >90% match with Telegram
- **Database integrity**: 100% constraint validation
- **Performance**: <2s average query time

## Related Documentation

### Architecture & Design

- [Database Schema](../database-schema.md) - New platform schema details
- [Architecture Overview](../architecture.md) - Platform architecture
- [Shared Libraries](../shared-libraries.md) - Common Python modules

### Other Services

- [Processor Service](processor.md) - Message processing and spam filtering
- [Enrichment Service](enrichment.md) - Background enrichment tasks
- [API Service](api.md) - REST API endpoints

### Operations

- [Deployment Guide](../../tutorials/deploy-to-production.md) - Docker Compose deployment
- [Monitoring Guide](../../operator-guide/monitoring.md) - Prometheus & Grafana setup

### Development

- [Contributing Guide](../contributing.md) - Development workflow
- [Adding Features](../adding-features.md) - Feature development guide

---

**Last Updated**: 2025-12-09

**Status**: Under Development - Phase 1 (Indexing) Complete
