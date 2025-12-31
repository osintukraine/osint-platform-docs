# Media Repair Tool

**OSINT Intelligence Platform - Unified Media Repair Utilities**

The media repair tool is a unified CLI utility for diagnosing and fixing media archival issues. It's baked into the processor container and provides commands for repairing albums, individual messages, verifying file integrity, and managing sync queues.

---

## Table of Contents

- [Overview](#overview)
- [Quick Reference](#quick-reference)
- [Commands](#commands)
  - [albums](#albums-command)
  - [message](#message-command)
  - [verify](#verify-command)
  - [sync-queue](#sync-queue-command)
- [Common Scenarios](#common-scenarios)
- [Troubleshooting](#troubleshooting)

---

## Overview

**Location**: `services/processor/tools/media_repair.py`

**Access**: Run from the processor-worker container:

```bash
docker-compose exec processor-worker python tools/media_repair.py <command>
```

**Why this tool exists**:

1. **Album detection failures**: Telethon's `events.Album` occasionally fails to fire (~5% of albums), causing incomplete media archival
2. **Sync queue issues**: Files downloaded to local buffer but not synced to MinIO
3. **Ghost entries**: Database records pointing to non-existent files
4. **Caption loss**: Album messages missing their text content

---

## Quick Reference

```bash
# Show help
docker-compose exec processor-worker python tools/media_repair.py --help

# Repair all incomplete albums (dry-run first)
docker-compose exec processor-worker python tools/media_repair.py albums --dry-run
docker-compose exec processor-worker python tools/media_repair.py albums

# Repair a specific message
docker-compose exec processor-worker python tools/media_repair.py message 8170603

# Verify files exist and fix ghost entries
docker-compose exec processor-worker python tools/media_repair.py verify --fix

# Queue pending files for MinIO sync
docker-compose exec processor-worker python tools/media_repair.py sync-queue
```

---

## Commands

### albums Command

Repairs Telegram albums (grouped media) that have missing media files or captions.

```bash
# Preview what would be repaired
docker-compose exec processor-worker python tools/media_repair.py albums --dry-run

# Repair all albums
docker-compose exec processor-worker python tools/media_repair.py albums

# Repair with limit
docker-compose exec processor-worker python tools/media_repair.py albums --limit 100
```

**What it does**:

1. Finds all messages with `grouped_id` (albums)
2. For each album, fetches complete media from Telegram
3. Archives any missing media files
4. Links media to the primary message
5. Restores missing captions from Telegram

**Options**:

| Option | Description |
|--------|-------------|
| `--limit N` | Process only N albums |
| `--dry-run` | Report issues without making changes |
| `--temp-session` | Use separate Telegram session (avoids lock) |

---

### message Command

Repairs media for a specific message by ID.

```bash
# Repair message with database ID 8170603
docker-compose exec processor-worker python tools/media_repair.py message 8170603

# Dry run
docker-compose exec processor-worker python tools/media_repair.py message 8170603 --dry-run
```

**What it does**:

1. Looks up the message in the database
2. Fetches media from Telegram (including all album items if grouped)
3. Downloads to local buffer
4. Queues for sync to MinIO

**Options**:

| Option | Description |
|--------|-------------|
| `--dry-run` | Report only, no downloads |
| `--temp-session` | Use separate Telegram session |

---

### verify Command

Verifies that files referenced in the database actually exist on disk.

```bash
# Check for ghost entries
docker-compose exec processor-worker python tools/media_repair.py verify

# Fix ghost entries (clear local_path for missing files)
docker-compose exec processor-worker python tools/media_repair.py verify --fix

# Limit check to N files
docker-compose exec processor-worker python tools/media_repair.py verify --limit 1000
```

**What it does**:

1. Queries all `media_files` with `local_path` set but `synced_at` NULL
2. Checks if each file exists on disk
3. Reports missing files
4. With `--fix`, clears `local_path` for missing files (marks them for re-download)

**Options**:

| Option | Description |
|--------|-------------|
| `--fix` | Clear local_path for missing files |
| `--limit N` | Check only N files |

---

### sync-queue Command

Queues files that are in local buffer but not yet synced to MinIO.

```bash
# Queue all pending files
docker-compose exec processor-worker python tools/media_repair.py sync-queue

# Preview what would be queued
docker-compose exec processor-worker python tools/media_repair.py sync-queue --dry-run

# Queue with limit
docker-compose exec processor-worker python tools/media_repair.py sync-queue --limit 500
```

**What it does**:

1. Finds all `media_files` where `synced_at IS NULL` and `local_path IS NOT NULL`
2. Verifies each file exists on disk
3. Pushes sync jobs to Redis queue `media:sync:pending`
4. The `media-sync` service processes the queue

**Options**:

| Option | Description |
|--------|-------------|
| `--dry-run` | Report only, don't queue |
| `--limit N` | Queue only N files |

---

## Common Scenarios

### Scenario 1: Album with Missing Photos

**Symptom**: Album shows 1 photo in frontend but Telegram has 5

```bash
# Find the message ID from the frontend URL or database
# Repair the specific message
docker-compose exec processor-worker python tools/media_repair.py message <message_id>

# Check sync queue is processing
docker-compose exec redis redis-cli LLEN media:sync:pending
```

### Scenario 2: Bulk Repair After Outage

**Symptom**: Many messages archived during an issue period have missing media

```bash
# First, verify file integrity
docker-compose exec processor-worker python tools/media_repair.py verify --fix

# Then repair all albums
docker-compose exec processor-worker python tools/media_repair.py albums

# Finally, ensure pending files are queued
docker-compose exec processor-worker python tools/media_repair.py sync-queue
```

### Scenario 3: Files Stuck in Local Buffer

**Symptom**: Files downloaded but not appearing in MinIO

```bash
# Check pending sync count
docker-compose exec redis redis-cli LLEN media:sync:pending

# If queue is empty but files pending, re-queue them
docker-compose exec processor-worker python tools/media_repair.py sync-queue

# Verify media-sync service is running
docker-compose logs media-sync --tail 50
```

### Scenario 4: Telegram Session Lock

**Symptom**: Error "database is locked" when running repair

```bash
# Use temp session flag to avoid conflict with listener
docker-compose exec processor-worker python tools/media_repair.py albums --temp-session

# Or stop listener temporarily
docker-compose stop listener
docker-compose exec processor-worker python tools/media_repair.py albums
docker-compose start listener
```

---

## Troubleshooting

### Error: "Telegram not authorized"

The Telegram session file doesn't exist or is expired.

```bash
# Check session files exist
docker-compose exec processor-worker ls -la /data/telegram_sessions/

# If missing, copy from listener or re-authenticate
docker-compose exec listener ls -la /data/telegram_sessions/
```

### Error: "Connection refused" to Redis

Redis isn't running or network issue.

```bash
# Check Redis is healthy
docker-compose ps redis
docker-compose exec redis redis-cli PING
```

### Repair runs but files not appearing

Check the media-sync service:

```bash
# Check sync service logs
docker-compose logs media-sync --tail 100

# Check queue depth
docker-compose exec redis redis-cli LLEN media:sync:pending

# Check MinIO connectivity
docker-compose exec media-sync mc ls minio/osint-media/ | head
```

### Very slow repair

Telegram rate limiting. The tool handles flood-wait automatically, but you can:

```bash
# Use smaller batches
docker-compose exec processor-worker python tools/media_repair.py albums --limit 50

# Check for flood-wait in logs
docker-compose logs processor-worker --tail 100 | grep -i flood
```

---

## Database Queries for Diagnostics

```sql
-- Albums with no media files
SELECT m.id, m.grouped_id, c.name
FROM messages m
JOIN channels c ON m.channel_id = c.id
WHERE m.grouped_id IS NOT NULL
AND NOT EXISTS (SELECT 1 FROM message_media mm WHERE mm.message_id = m.id);

-- Files pending sync
SELECT COUNT(*) FROM media_files WHERE synced_at IS NULL AND local_path IS NOT NULL;

-- Albums by media count
SELECT
    CASE
        WHEN cnt = 1 THEN '1 file'
        WHEN cnt = 2 THEN '2 files'
        ELSE '3+ files'
    END as media_count,
    COUNT(*) as albums
FROM (
    SELECT m.id, COUNT(mm.media_id) as cnt
    FROM messages m
    LEFT JOIN message_media mm ON m.id = mm.message_id
    WHERE m.grouped_id IS NOT NULL
    GROUP BY m.id
) sub
GROUP BY 1;
```

---

## See Also

- [Troubleshooting Guide](troubleshooting.md) - General platform troubleshooting
- [Backup & Restore](backup-restore.md) - Media backup procedures
- [Hetzner Storage](hetzner-storage.md) - Storage box configuration
