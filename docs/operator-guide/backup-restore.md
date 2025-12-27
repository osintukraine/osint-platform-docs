# Backup & Restore Guide

**OSINT Intelligence Platform - Comprehensive Backup Strategies and Disaster Recovery**

Complete procedures for protecting platform data, creating recoverable backups, and disaster recovery planning.

---

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Automated Scripts](#automated-scripts)
- [What Needs Backup](#what-needs-backup)
- [Backup Strategies](#backup-strategies)
- [PostgreSQL Database Backup](#postgresql-database-backup)
- [MinIO Media Backup](#minio-media-backup)
- [Telegram Session Backup](#telegram-session-backup)
- [Configuration Backup](#configuration-backup)
- [Redis Backup](#redis-backup)
- [Restore Procedures](#restore-procedures)
- [Backup Automation](#backup-automation)
- [Testing Backups](#testing-backups)
- [Disaster Recovery Planning](#disaster-recovery-planning)

---

## Overview

**Critical Data Components:**

1. **PostgreSQL Database**: Messages, entities, channels, users, classifications (CRITICAL)
2. **MinIO Object Storage**: Media files (photos, videos, documents) (CRITICAL)
3. **Telegram Sessions**: Authentication credentials for listener accounts (CRITICAL)
4. **Configuration Files**: .env, docker-compose.yml, rules, prompts (IMPORTANT)
5. **Redis Queue**: In-flight messages (OPTIONAL - can be replayed from Telegram)

**Backup Frequency Recommendations:**

| Component | Frequency | Retention | Priority |
|-----------|-----------|-----------|----------|
| PostgreSQL (full) | Daily | 30 days | CRITICAL |
| PostgreSQL (incremental) | Every 6 hours | 7 days | CRITICAL |
| MinIO media | Weekly (full), Daily (incremental) | 90 days | CRITICAL |
| Telegram sessions | After any auth change | Forever | CRITICAL |
| Configuration | On every change (git commit) | Forever | IMPORTANT |
| Redis | Not needed (transient data) | - | OPTIONAL |

**Recovery Point Objective (RPO)**: 6 hours (maximum acceptable data loss)
**Recovery Time Objective (RTO)**: 2 hours (maximum acceptable downtime)

---

## Quick Start

!!! tip "Recommended Approach"
    Use the automated backup scripts for consistent, reliable backups. Manual procedures are documented below for reference and troubleshooting.

### Basic Backup

```bash
# Full backup (database, sessions, config, Ollama models)
./scripts/backup.sh

# Fast backup (skip large Ollama models)
./scripts/backup.sh --no-ollama

# Include media files (slow, large - use Hetzner snapshots instead)
./scripts/backup.sh --include-media

# Trigger Hetzner Storage Box snapshot
./scripts/backup.sh --hetzner-snapshot
```

**Output location**: `./backups/YYYYMMDD_HHMMSS/`

### Basic Restore

```bash
# Full restore from backup
./scripts/restore.sh /path/to/backup/YYYYMMDD_HHMMSS

# Preview what would be restored (dry-run)
./scripts/restore.sh /path/to/backup/YYYYMMDD_HHMMSS --dry-run

# Selective restore (skip database)
./scripts/restore.sh /path/to/backup/YYYYMMDD_HHMMSS --skip-db
```

### Production Readiness Check

```bash
# Comprehensive pre-deployment verification
./scripts/preflight-check.sh

# Quick check (essential checks only)
./scripts/preflight-check.sh --quick
```

**Exit codes**:

- `0` = All checks passed (ready for production)
- `1` = Critical failures (do NOT deploy)
- `2` = Warnings present (review before deploy)

---

## Automated Scripts

The platform includes three operational scripts for production deployment:

### backup.sh - Full Platform Backup

**Location**: `scripts/backup.sh`

Creates a complete backup of all platform state including database, sessions, configuration, and optionally media files.

#### What It Backs Up

| Component | Size | Critical | Notes |
|-----------|------|----------|-------|
| **PostgreSQL** | 3-50GB | ✅ Critical | Both custom and plain SQL formats |
| **Telegram Sessions** | ~1MB | ✅ Critical | Auth credentials (KEEP SECURE) |
| **Configuration** | ~10MB | ✅ Critical | .env, docker-compose.yml, infrastructure/ |
| **Redis** | ~100MB | ⚠️ Optional | Transient queue data |
| **Ollama Models** | 5-10GB | ⚠️ Optional | Can be re-downloaded |
| **Media Files** | 100GB-10TB | ✅ Critical | Use Hetzner snapshots instead |

#### Usage

=== "Standard Backup"
    ```bash
    # Full backup (recommended for scheduled backups)
    ./scripts/backup.sh

    # Output example:
    # ============================================================
    #   OSINT Platform Backup - 20251221_143022
    # ============================================================
    #
    # [INFO] Running pre-flight checks...
    # [OK] Backup directory: ./backups/20251221_143022
    # [OK] Database dump complete: 4.2G
    # [OK] Telegram sessions backup complete: 1.2M
    # [OK] Redis snapshot complete: 87M
    # [OK] Ollama models backup complete
    # [OK] Configuration backup complete
    #
    # Total backup size: 15G
    ```

=== "Fast Backup (No Ollama)"
    ```bash
    # Skip Ollama models (saves 5-10GB, reduces backup time by 80%)
    ./scripts/backup.sh --no-ollama

    # Use for frequent backups (every 6 hours)
    # Ollama models can be re-downloaded with: docker compose exec ollama ollama pull qwen2.5:3b
    ```

=== "With Media Files"
    ```bash
    # Include media files via rsync (SLOW, LARGE)
    ./scripts/backup.sh --include-media

    # ⚠️  Warning: This may take hours and use significant disk space
    # Media size: 100GB-10TB depending on archive age
    #
    # Recommended: Use Hetzner Storage Box snapshots instead
    ```

=== "Hetzner Snapshot"
    ```bash
    # Trigger Hetzner Storage Box snapshot via API
    ./scripts/backup.sh --hetzner-snapshot

    # Requires environment variables:
    # - HETZNER_STORAGE_BOX_ID
    # - HETZNER_API_TOKEN
    #
    # Creates snapshot: osint-backup-YYYYMMDD_HHMMSS
    ```

#### Options

| Flag | Description | Use Case |
|------|-------------|----------|
| `--no-ollama` | Skip Ollama models backup | Frequent backups, save space/time |
| `--include-media` | Include media files via rsync | Full archival backup (not recommended) |
| `--hetzner-snapshot` | Trigger Hetzner snapshot via API | Production media backup |
| `--help` | Show help message | Reference |

#### Output Structure

```bash
backups/20251221_143022/
├── database.dump              # PostgreSQL custom format (for pg_restore)
├── database.sql.gz            # Plain SQL backup (emergency fallback)
├── database.log               # Backup process log
├── telegram_sessions.tar.gz   # ⚠️ SENSITIVE - Auth credentials
├── redis.rdb                  # Redis snapshot (optional)
├── ollama_models.tar.gz       # Ollama models (optional)
├── config/
│   ├── .env                   # ⚠️ SENSITIVE - Contains secrets
│   ├── docker-compose*.yml
│   └── infrastructure/
└── MANIFEST.txt               # Backup metadata and stats
```

!!! danger "Security Warning"
    - **telegram_sessions.tar.gz** contains your Telegram login credentials
    - **.env** contains API keys, passwords, and secrets
    - **Encrypt these files** before storing off-site
    - Never commit to public repositories

#### Backup Manifest

Each backup includes a `MANIFEST.txt` with metadata:

```text
OSINT Platform Backup
=====================
Timestamp: 20251221_143022
Created:   Sat Dec 21 14:30:22 UTC 2025
Host:      production-server
Git Hash:  78376a05
Git Branch: master

Contents:
---------
-rw-r--r-- 1 root root 4.2G Dec 21 14:31 database.dump
-rw-r--r-- 1 root root 1.2M Dec 21 14:32 telegram_sessions.tar.gz
-rw-r--r-- 1 root root  87M Dec 21 14:32 redis.rdb
-rw-r--r-- 1 root root 8.9G Dec 21 14:35 ollama_models.tar.gz
-rw-r--r-- 1 root root  12M Dec 21 14:35 config.tar.gz

Database Stats:
---------------
 messages  | channels | media_files |  db_size
-----------+----------+-------------+----------
  1234567  |      89  |   456789    | 42 GB
```

---

### restore.sh - Full Platform Restore

**Location**: `scripts/restore.sh`

Restores a complete platform backup created by `backup.sh`.

!!! warning "Destructive Operation"
    This will **OVERWRITE** existing data. Always test with `--dry-run` first.

#### Usage

=== "Full Restore"
    ```bash
    # Restore everything from backup
    ./scripts/restore.sh ./backups/20251221_143022

    # Process:
    # 1. Shows backup manifest
    # 2. Asks for confirmation (type 'yes')
    # 3. Stops services
    # 4. Restores configuration
    # 5. Restores database
    # 6. Restores Telegram sessions
    # 7. Restores Ollama models (if present)
    # 8. Starts all services
    # 9. Shows service status
    ```

=== "Dry Run (Preview)"
    ```bash
    # Preview what would be restored WITHOUT making changes
    ./scripts/restore.sh ./backups/20251221_143022 --dry-run

    # Output example:
    # [DRY-RUN] Would restore .env
    # [DRY-RUN] Would restore database from database.dump
    # [DRY-RUN] Would restore Telegram sessions
    # [DRY-RUN] Would start services with: docker compose up -d
    ```

=== "Selective Restore"
    ```bash
    # Restore only sessions and config (skip database)
    ./scripts/restore.sh ./backups/20251221_143022 --skip-db

    # Restore database and config (skip sessions)
    ./scripts/restore.sh ./backups/20251221_143022 --skip-sessions

    # Restore everything except Ollama models
    ./scripts/restore.sh ./backups/20251221_143022 --skip-ollama

    # Combine flags
    ./scripts/restore.sh ./backups/20251221_143022 --skip-db --skip-ollama
    ```

#### Options

| Flag | Description | Use Case |
|------|-------------|----------|
| `--dry-run` | Preview without changes | Verify backup before restore |
| `--skip-db` | Skip database restore | Restore only sessions/config |
| `--skip-sessions` | Skip Telegram sessions | New Telegram account |
| `--skip-ollama` | Skip Ollama models | Re-download models instead |
| `--help` | Show help message | Reference |

#### Restore Process Steps

The script follows a careful sequence to ensure safe restoration:

1. **Display Manifest** - Shows backup contents and metadata
2. **Stop Services** - Stops application services (keeps database running)
3. **Restore Configuration** - Copies .env and config files
4. **Restore Database** - Drops and recreates database, then restores
5. **Restore Sessions** - Extracts Telegram session files to volume
6. **Restore Ollama** - Unpacks model files (if present)
7. **Restore Media** - Optional rsync of media files
8. **Start Services** - Brings all services back online
9. **Verify Health** - Shows service status and next steps

#### Post-Restore Verification

```bash
# After restore completes, verify system health:

# 1. Check service status
docker compose ps

# 2. Check API health
curl http://localhost:8000/health

# 3. Verify database
docker compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(*) FROM messages;"

# 4. Check Telegram authentication
docker compose logs listener | grep "Logged in as"

# 5. Test frontend
curl http://localhost:3000
```

#### Media Restoration

!!! info "Media Files Handled Separately"
    Media files are NOT included in standard backups. Restore media using:

    **Option 1: Hetzner Storage Box Snapshot** (Recommended)
    ```bash
    # Restore from Hetzner snapshot via Hetzner Console
    # https://robot.hetzner.com/storage
    # Then ensure SSHFS mount is configured:
    ./scripts/setup-hetzner-storage.sh
    ```

    **Option 2: rsync from backup** (if --include-media was used)
    ```bash
    # Restore will prompt if media/ directory exists in backup
    rsync -avz /backup/20251221_143022/media/ /data/minio/
    ```

!!! warning "MinIO and Hetzner Storage Box"
    **Critical Understanding**: MinIO writes directly to the Hetzner Storage Box via SSHFS mount. They are **the same files**, not separate copies.

    - Deleting from MinIO = deleting from Hetzner
    - Hetzner snapshots are the **primary media backup method**
    - Don't rely on rsync for routine media backups (too slow, too large)

---

### preflight-check.sh - Production Readiness Verification

**Location**: `scripts/preflight-check.sh`

Comprehensive verification before deploying to production. Checks configuration, services, connectivity, and security settings.

#### Usage

=== "Full Check"
    ```bash
    # Run all checks (recommended before production deployment)
    ./scripts/preflight-check.sh

    # Example output:
    # ╔══════════════════════════════════════════════════════════════╗
    # ║       OSINT Platform - Production Preflight Check            ║
    # ╚══════════════════════════════════════════════════════════════╝
    #
    # ━━━ Environment Configuration ━━━
    #   ✓ .env file exists
    #   ✓ POSTGRES_PASSWORD is set (custom)
    #   ✓ MINIO_ROOT_PASSWORD is set (custom)
    #   ✓ JWT_SECRET_KEY is set (custom)
    #   ✓ Telegram API credentials configured
    #   ✓ DeepL API key configured
    #   ⚠ DOMAIN not set or is localhost - set for production HTTPS
    #
    # ━━━ Docker Environment ━━━
    #   ✓ Docker daemon is running
    #   ✓ Docker Compose available: v2.24.0
    #   ✓ All required images built
    #   ✓ Disk space OK: 450GB available
    #
    # ... (continues with all checks)
    #
    # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    #
    #   Passed:   45
    #   Warnings: 3
    #   Failed:   0
    #
    # ╔══════════════════════════════════════════════════════════════╗
    # ║  WARNINGS FOUND - Review before deploying                    ║
    # ╚══════════════════════════════════════════════════════════════╝
    ```

=== "Quick Check"
    ```bash
    # Essential checks only (faster, good for CI/CD)
    ./scripts/preflight-check.sh --quick

    # Skips:
    # - Ollama model availability
    # - Network connectivity tests
    # - Backup status checks
    ```

#### Exit Codes

| Code | Status | Meaning | Action |
|------|--------|---------|--------|
| `0` | ✅ Pass | All checks passed | Safe to deploy |
| `1` | ❌ Critical | Critical failures found | Fix issues before deploying |
| `2` | ⚠️ Warning | Warnings present | Review warnings, deploy at discretion |

**Use in automation**:

```bash
#!/bin/bash
# Deployment script with preflight check

if ./scripts/preflight-check.sh; then
    echo "Preflight passed, deploying..."
    docker compose up -d
else
    echo "Preflight failed, aborting deployment"
    exit 1
fi
```

#### What It Checks

??? abstract "Environment Configuration (9 checks)"
    - `.env` file exists
    - `POSTGRES_PASSWORD` is custom (not default)
    - `MINIO_ROOT_PASSWORD` is custom (not default)
    - `JWT_SECRET_KEY` is custom (not default)
    - Telegram API credentials configured
    - DeepL API key configured (optional)
    - Domain configuration (for HTTPS)
    - Hetzner mount path accessible
    - Secret strength validation

??? abstract "Docker Environment (4 checks)"
    - Docker daemon running
    - Docker Compose version
    - Required images built
    - Available disk space

??? abstract "Service Health (8+ checks)"
    - PostgreSQL running and healthy
    - Redis running and healthy
    - MinIO running and healthy
    - API running and healthy
    - Listener running
    - Processor workers running
    - Frontend running
    - Ollama running (if not --quick mode)
    - API health endpoint responding

??? abstract "Database State (5 checks)"
    - PostgreSQL connection OK
    - Database schema initialized
    - Table count validation
    - Active channel count
    - Database size reporting

??? abstract "Redis Queue State (3 checks)"
    - Redis connection OK
    - Message queue backlog status
    - Active queue consumers

??? abstract "Storage Configuration (6 checks)"
    - MinIO container running
    - MinIO health endpoint responding
    - Media storage path exists
    - Media storage writable
    - Available storage space
    - SSHFS mount active (if configured)

??? abstract "Security Configuration (6 checks)"
    - NODE_ENV=production
    - Exposed port audit
    - .env file permissions (should be 600)
    - Telegram session files present
    - Caddy/HTTPS configuration
    - Sensitive file security

??? abstract "Telegram Configuration (4 checks)"
    - API credentials configured
    - Phone number configured
    - Session volume exists
    - Session files present

??? abstract "LLM/Ollama (3 checks, skipped in --quick mode)"
    - Ollama container running
    - LLM model available
    - Ollama API responding

??? abstract "Network Connectivity (4 checks, skipped in --quick mode)"
    - Internet access (api.telegram.org)
    - DeepL API reachable (if configured)
    - API port accessible
    - Frontend port accessible

??? abstract "Backup Status (3 checks, skipped in --quick mode)"
    - Backup script available
    - Recent backup exists
    - Backup age verification

#### Common Failures and Fixes

| Check | Failure | Fix |
|-------|---------|-----|
| **POSTGRES_PASSWORD** | Default password detected | Change in `.env`: `openssl rand -base64 32` |
| **Telegram credentials** | Missing API ID/hash | Get from https://my.telegram.org |
| **Disk space** | <10GB available | Clean up old logs, backups, or expand storage |
| **Database schema** | <10 tables found | Schema auto-creates on first run, wait for init |
| **Queue backlog** | >1000 messages pending | Check processor health: `docker compose logs processor-worker` |
| **Hetzner mount** | Path not accessible | Run `./scripts/setup-hetzner-storage.sh` |
| **No active channels** | 0 channels monitored | Configure Telegram folders or add channels manually |

#### Using in CI/CD

```yaml
# GitHub Actions example
name: Deploy to Production

on:
  push:
    branches: [master]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run preflight check
        run: |
          ./scripts/preflight-check.sh --quick

      - name: Deploy if checks pass
        if: success()
        run: |
          docker compose pull
          docker compose up -d --build
```

```bash
# Cron job for daily health monitoring
# Add to crontab: 0 8 * * * /opt/osint-platform/scripts/preflight-check.sh
0 8 * * * /opt/osint-platform/scripts/preflight-check.sh >> /var/log/osint-preflight.log 2>&1 || \
  curl -d "Preflight check failed" http://localhost:8090/osint-alerts
```

---

## What Needs Backup

### Critical Data (Must Backup)

**1. PostgreSQL Database** (3-50GB typical)

```sql
-- Tables requiring backup
messages              -- Core message archive (largest table)
channels              -- Monitored Telegram channels
entities              -- Extracted entities (people, places, units, equipment)
message_entities      -- Entity mentions in messages
osint_rules           -- Scoring and classification rules
llm_prompts           -- AI classification prompts
spam_patterns         -- Spam filter patterns
users                 -- Platform users
media_files           -- Media metadata (references MinIO)
```

**Why critical**: Irreplaceable historical intelligence data. Cannot be re-fetched from Telegram (messages may be deleted, channels may go private).

**2. MinIO Object Storage** (100GB-10TB typical)

```bash
# Bucket: osint-media
media/
├── {sha256[:2]}/{sha256[2:4]}/{sha256}.{ext}  # Content-addressed storage
├── Example: media/a1/b2/a1b2c3d4e5f6...789.jpg
```

**Why critical**: Media files cannot be re-downloaded after Telegram URLs expire (typically 24-48 hours). Content-addressed storage (SHA-256) provides deduplication and integrity verification.

**3. Telegram Session Files** (~1MB)

```bash
telegram_sessions/
├── listener.session
├── listener.session-journal
```

**Why critical**: Loss requires manual re-authentication (2FA, phone verification). May require physical access to phone number. Service downtime during re-auth.

### Important Data (Should Backup)

**4. Configuration Files**

```bash
.env                           # Environment variables and secrets
docker-compose.yml             # Service configuration
config/osint_rules.yml         # Classification rules
infrastructure/postgres/init.sql  # Database schema (in git)
```

**Why important**: Defines platform behavior. Most files in git, but .env contains secrets and deployment-specific settings.

### Optional Data (Nice to Backup)

**5. Redis Queue** (transient, can be replayed)

```bash
# Redis streams
telegram_messages   # In-flight messages being processed
```

**Why optional**: Losing Redis queue means messages in-flight are lost, but listener will re-fetch from Telegram on restart. Acceptable data loss (<1 minute of messages).

**6. Monitoring Data** (Prometheus, Grafana)

```bash
prometheus_data/    # Metrics history (30 days)
grafana_data/       # Dashboard customizations
```

**Why optional**: Historical metrics useful for trend analysis, but not critical for platform operation. Can be rebuilt.

---

## Backup Strategies

### Recommended: 3-2-1 Backup Rule

- **3 copies** of data: 1 production + 2 backups
- **2 different media types**: Local disk + cloud/NAS
- **1 off-site copy**: Geographic redundancy

### Strategy 1: Daily Full + Incremental (Recommended)

**Schedule:**

```bash
# Daily full backups (2 AM)
0 2 * * * /scripts/backup-full.sh

# Incremental backups every 6 hours
0 */6 * * * /scripts/backup-incremental.sh

# Weekly media sync to off-site storage (Sunday 3 AM)
0 3 * * 0 /scripts/backup-media-offsite.sh
```

**Disk Space Required:**

- PostgreSQL: 30 days × 50GB = ~1.5TB (with compression ~300GB)
- MinIO: 90 days × 10TB = 900TB (impractical, use incremental sync)
- Incremental MinIO: ~100GB/week (only new media)

### Strategy 2: Continuous Replication (Advanced)

**PostgreSQL Streaming Replication:**

```yaml
# docker-compose.replication.yml
services:
  postgres-replica:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_PRIMARY_HOST: postgres
      POSTGRES_REPLICATION_MODE: slave
    volumes:
      - postgres_replica_data:/var/lib/postgresql/data
```

**MinIO Replication:**

```bash
# Configure site replication between MinIO instances
mc admin replicate add primary-minio https://backup-minio:9000 \
  --admin-user backup-admin --admin-password <password>
```

**Benefits**: Near-zero RPO (<1 second), automatic failover
**Drawbacks**: Requires second server, doubles infrastructure cost

---

## PostgreSQL Database Backup

### Manual Full Backup

```bash
# Create full database dump
docker-compose exec -T postgres pg_dump -U postgres osint_platform \
  > backups/postgres/osint_platform_$(date +%Y%m%d_%H%M%S).sql

# With compression (recommended, 5-10x smaller)
docker-compose exec -T postgres pg_dump -U postgres osint_platform | \
  gzip > backups/postgres/osint_platform_$(date +%Y%m%d_%H%M%S).sql.gz

# Custom format (allows selective restore, parallel dump)
docker-compose exec -T postgres pg_dump -U postgres -Fc osint_platform \
  > backups/postgres/osint_platform_$(date +%Y%m%d_%H%M%S).dump
```

**Verify backup:**

```bash
# Check file size (should be >100MB for production database)
ls -lh backups/postgres/osint_platform_*.sql.gz

# Check backup integrity
gunzip -c backups/postgres/osint_platform_20251209.sql.gz | head -50
# Should show: PostgreSQL dump header and CREATE TABLE statements
```

### Incremental Backup (WAL Archiving)

**Enable WAL archiving** in `/infrastructure/postgres/postgresql.conf`:

```ini
# Write-Ahead Logging
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /backups/wal/%f && cp %p /backups/wal/%f'
archive_timeout = 3600  # Force WAL switch every hour
```

**Create base backup:**

```bash
# Full base backup (run weekly)
docker-compose exec postgres pg_basebackup -U postgres -D /backups/base \
  -Ft -z -P -X stream

# Incremental WAL files are automatically archived to /backups/wal/
```

**Recovery using WAL:**

```bash
# Restore base backup
tar -xzf /backups/base/base.tar.gz -C /var/lib/postgresql/data

# PostgreSQL will automatically replay WAL files on startup
# Result: Point-in-time recovery to any second between backups
```

### Schema-Only Backup (for version control)

```bash
# Backup schema without data (useful for tracking schema changes)
docker-compose exec -T postgres pg_dump -U postgres -s osint_platform \
  > backups/schema/schema_$(date +%Y%m%d).sql

# Commit to git for version tracking
git add backups/schema/schema_$(date +%Y%m%d).sql
git commit -m "Database schema snapshot $(date +%Y-%m-%d)"
```

### Table-Specific Backup (large tables)

```bash
# Backup only messages table (largest table)
docker-compose exec -T postgres pg_dump -U postgres -t messages osint_platform | \
  gzip > backups/postgres/messages_$(date +%Y%m%d).sql.gz

# Backup all except large tables (faster for testing restore)
docker-compose exec -T postgres pg_dump -U postgres -T messages -T media_files osint_platform | \
  gzip > backups/postgres/metadata_$(date +%Y%m%d).sql.gz
```

---

## MinIO Media Backup

### Strategy: Content-Addressed Storage Advantage

**Key insight**: SHA-256 content addressing means files never change. Once written, immutable.

**Backup strategy**:
- **Full sync weekly** (baseline)
- **Incremental sync daily** (only new files since last sync)

### Using `mc mirror` (Recommended)

```bash
# Install MinIO Client (if not already installed)
docker run --rm -v $(pwd)/bin:/out minio/mc:latest cp /usr/bin/mc /out/mc
chmod +x bin/mc

# Configure MinIO alias
./bin/mc alias set production http://localhost:9000 minioadmin minioadmin

# Full mirror sync (creates exact copy)
./bin/mc mirror production/osint-media /backups/minio/osint-media

# Incremental sync (only new files, --newer-than 24h)
./bin/mc mirror --newer-than 24h production/osint-media /backups/minio/osint-media

# Verify sync integrity
./bin/mc diff production/osint-media /backups/minio/osint-media
# Output: (empty if identical)
```

### Using `rclone` (for cloud backup)

```bash
# Install rclone
curl https://rclone.org/install.sh | sudo bash

# Configure S3-compatible backend
rclone config create s3-backup s3 \
  provider=Minio \
  endpoint=http://localhost:9000 \
  access_key_id=minioadmin \
  secret_access_key=minioadmin

# Sync to cloud storage (AWS S3, Google Cloud Storage, etc.)
rclone sync s3-backup:osint-media /backups/minio/osint-media --progress

# Or to another cloud provider
rclone sync s3-backup:osint-media wasabi:osint-backup/media --progress
```

### Verify Media Backup Integrity

```bash
# Count files
./bin/mc du production/osint-media
./bin/mc du /backups/minio/osint-media
# Should match

# Verify random sample of files (SHA-256 integrity check)
docker-compose exec minio sh -c '
  for file in /data/osint-media/media/a1/b2/*.jpg; do
    sha256sum $file
  done
' | head -10

# Compare with backup
for file in /backups/minio/osint-media/media/a1/b2/*.jpg; do
  sha256sum $file
done | head -10

# SHA-256 hashes should match
```

### Pruning Old Backups

```bash
# Delete backups older than 90 days
find /backups/minio -name "*.jpg" -mtime +90 -delete
find /backups/minio -name "*.mp4" -mtime +90 -delete

# Keep metadata index of deleted backups (for forensics)
find /backups/minio -name "*" -mtime +90 > deleted_files_$(date +%Y%m%d).txt
```

---

## Telegram Session Backup

### Critical: Session Files Must Be Backed Up

**Session files are critical and irreplaceable.** Losing them requires:

1. Manual re-authentication (phone verification, 2FA code)
2. Physical access to phone number (may not be available)
3. Service downtime during re-authentication (30+ minutes)
4. Risk of Telegram flagging account for suspicious activity

### Backup Telegram Sessions

```bash
# Create backup of session files
docker-compose exec listener tar -czf /tmp/sessions-backup.tar.gz /app/sessions/*.session*

# Copy to host
docker cp osint-listener:/tmp/sessions-backup.tar.gz backups/sessions/sessions_$(date +%Y%m%d_%H%M%S).tar.gz

# Verify backup
tar -tzf backups/sessions/sessions_$(date +%Y%m%d_%H%M%S).tar.gz
# Should list: sessions/listener.session, sessions/listener.session-journal
```

### Encrypt Session Backups (Recommended)

**Session files contain authentication secrets. Encrypt before storing off-site.**

```bash
# Encrypt with GPG
gpg --symmetric --cipher-algo AES256 backups/sessions/sessions_$(date +%Y%m%d).tar.gz

# Result: sessions_20251209.tar.gz.gpg (encrypted, safe for cloud storage)

# Decrypt when needed
gpg --decrypt backups/sessions/sessions_20251209.tar.gz.gpg > sessions.tar.gz
```

### Restore Telegram Sessions

```bash
# Stop listener
docker-compose stop listener

# Extract sessions to telegram_sessions/ directory
tar -xzf backups/sessions/sessions_20251209.tar.gz -C telegram_sessions/

# Set correct permissions
chmod 600 telegram_sessions/*.session*

# Restart listener
docker-compose start listener

# Verify authentication
docker-compose logs -f listener | grep "Logged in as"
# Should show: "Logged in as +1234567890"
```

---

## Configuration Backup

### What to Backup

```bash
# Critical configuration files
.env                              # Secrets and environment variables
docker-compose.yml                # Service definitions
config/osint_rules.yml            # OSINT scoring rules

# Important configuration files
infrastructure/postgres/postgresql.conf  # Database tuning
infrastructure/prometheus/prometheus.yml  # Metrics scraping
infrastructure/grafana/provisioning/      # Grafana dashboards
```

### Backup Configuration Files

```bash
# Create configuration backup archive
tar -czf backups/config/config_$(date +%Y%m%d_%H%M%S).tar.gz \
  .env \
  docker-compose.yml \
  config/ \
  infrastructure/postgres/postgresql.conf \
  infrastructure/prometheus/ \
  infrastructure/grafana/provisioning/

# Verify backup
tar -tzf backups/config/config_20251209.tar.gz
```

### Version Control with Git (Recommended)

```bash
# Initialize git repository (if not already done)
git init
git add docker-compose.yml config/ infrastructure/

# Commit configuration changes
git commit -m "Configuration snapshot $(date +%Y-%m-%d)"

# Push to remote repository (encrypted private repo)
git remote add origin git@github.com:org/osint-platform-config.git
git push origin master
```

**Note**: Do NOT commit `.env` to git (contains secrets). Use `.env.example` instead:

```bash
# Create .env template (safe to commit)
cp .env .env.example
# Edit .env.example: Replace actual values with placeholders
# Example: POSTGRES_PASSWORD=<your-postgres-password>

# Commit template, ignore actual .env
echo ".env" >> .gitignore
git add .env.example .gitignore
git commit -m "Add .env template"
```

---

## Redis Backup

### Why Redis Backup is Optional

**Redis contains only transient queue data:**

- Messages being processed (in-flight for <1 minute)
- Rate limiting counters (resets automatically)
- Temporary caches (rebuilt automatically)

**On Redis failure**: Listener re-fetches recent messages from Telegram. Acceptable data loss: <1 minute.

### If You Need Redis Backup (Advanced)

**Enable Redis persistence** in `docker-compose.yml`:

```yaml
services:
  redis:
    command: redis-server --appendonly yes --save 60 1000
    volumes:
      - redis_data:/data
```

**Manual backup:**

```bash
# Trigger RDB snapshot
docker-compose exec redis redis-cli BGSAVE

# Wait for save to complete
docker-compose exec redis redis-cli LASTSAVE

# Copy RDB file
docker cp osint-redis:/data/dump.rdb backups/redis/dump_$(date +%Y%m%d).rdb
```

---

## Restore Procedures

!!! info "Automated Restore"
    Use the automated `restore.sh` script for safe, reliable restoration. See [Automated Scripts](#automated-scripts) section above for full details.

### Full System Restore (Disaster Recovery)

**Scenario**: Complete server failure, need to rebuild from scratch.

**Prerequisites**:

- Backup files (created by `backup.sh` or manual backup)
- Fresh Ubuntu/Debian server
- Docker and Docker Compose installed

#### Using Automated Restore Script (Recommended)

```bash
# 1. Clone platform repository
git clone https://github.com/osintukraine/osint-intelligence-platform.git
cd osint-intelligence-platform

# 2. Copy backup directory to server
scp -r /local/backups/20251221_143022 user@newserver:/opt/osint-platform/backups/

# 3. Run automated restore
./scripts/restore.sh /opt/osint-platform/backups/20251221_143022

# Script will:
# - Display backup manifest
# - Ask for confirmation
# - Stop services
# - Restore configuration (.env, docker-compose.yml)
# - Restore database
# - Restore Telegram sessions
# - Restore Ollama models (if present)
# - Start all services
# - Show verification steps

# 4. Verify system health
./scripts/preflight-check.sh

# 5. Restore media from Hetzner snapshot
# Follow Hetzner Console instructions to restore snapshot
# Then mount: ./scripts/setup-hetzner-storage.sh
```

**Estimated restore time**:

- Database: 10-30 minutes (depends on size)
- Configuration: 1 minute
- Sessions: 1 minute
- Ollama models: 5-10 minutes (or skip and re-download)
- Media: Via Hetzner snapshot restore (separate process)
- **Total: ~20-45 minutes** (excluding media)

#### Manual Step-by-Step Restore (Fallback)

If the automated script fails or you need fine-grained control:

```bash
# 1. Clone platform repository
git clone https://github.com/osintukraine/osint-intelligence-platform.git
cd osint-intelligence-platform

# 2. Restore configuration files
tar -xzf /restore-media/config_20251209.tar.gz -C .

# OR if using backup.sh output structure:
cp /restore-media/20251221_143022/config/.env .
cp -r /restore-media/20251221_143022/config/infrastructure ./

# 3. Start infrastructure services only
docker compose up -d postgres redis minio

# Wait for services to be healthy (30-60 seconds)
docker compose ps

# 4. Restore PostgreSQL database
docker compose exec -T postgres pg_restore \
  -U osint_user \
  -d osint_platform \
  --clean \
  --if-exists \
  --no-owner \
  --no-privileges \
  < /restore-media/20251221_143022/database.dump

# Verify database restore
docker compose exec postgres psql -U osint_user -d osint_platform -c "\dt"
# Should list all tables: messages, channels, entities, etc.

# 5. Restore MinIO media files (from Hetzner snapshot)
# Follow Hetzner Console to restore snapshot, then:
./scripts/setup-hetzner-storage.sh

# 6. Restore Telegram sessions
SESSIONS_VOLUME="osint-intelligence-platform_telegram_sessions"
docker volume create "$SESSIONS_VOLUME"
docker run --rm \
  -v "${SESSIONS_VOLUME}:/data" \
  -v "/restore-media/20251221_143022:/backup:ro" \
  alpine \
  sh -c "tar xzf /backup/telegram_sessions.tar.gz -C /data"

# 7. Start all services
docker compose up -d

# 8. Verify system health
./scripts/preflight-check.sh
```

**Estimated restore time**: 30 minutes (database) + media sync time (depends on bandwidth)

### Selective Restore (Specific Table)

**Scenario**: Accidentally deleted data from one table, need to restore just that table.

```bash
# 1. Extract table from backup
gunzip -c backups/postgres/osint_platform_20251209.sql.gz | \
  grep -A 10000 "CREATE TABLE channels" > channels_restore.sql

# 2. Drop and recreate table
docker-compose exec postgres psql -U postgres -d osint_platform -c "DROP TABLE channels CASCADE;"

# 3. Restore table from backup
docker-compose exec -T postgres psql -U postgres osint_platform < channels_restore.sql

# 4. Rebuild foreign key references
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  ALTER TABLE messages ADD CONSTRAINT fk_messages_channel
    FOREIGN KEY (channel_id) REFERENCES channels(id);
"
```

### Point-in-Time Recovery (Using WAL)

**Scenario**: Restore database to specific timestamp (e.g., before accidental deletion).

```bash
# 1. Stop PostgreSQL
docker-compose stop postgres

# 2. Restore base backup
tar -xzf backups/base/base.tar.gz -C /var/lib/postgresql/data

# 3. Create recovery.conf
cat > /var/lib/postgresql/data/recovery.conf <<EOF
restore_command = 'cp /backups/wal/%f %p'
recovery_target_time = '2025-12-09 14:30:00'  # Restore to this timestamp
recovery_target_action = 'promote'
EOF

# 4. Start PostgreSQL (will replay WAL to target time)
docker-compose start postgres

# Monitor recovery progress
docker-compose logs -f postgres | grep "recovery"
```

---

## Backup Automation

!!! info "Built-in Automation"
    The platform includes `backup.sh` and `restore.sh` scripts ready for automation. See [Automated Scripts](#automated-scripts) section above for full details.

### Scheduled Backups with Cron

The recommended backup schedule uses the automated `backup.sh` script:

=== "Daily Full Backup"
    ```bash
    # Add to crontab (run daily at 2 AM)
    crontab -e

    # Daily backup without Ollama models (faster, 80% less space)
    0 2 * * * /opt/osint-platform/scripts/backup.sh --no-ollama >> /var/log/osint-backup.log 2>&1
    ```

=== "6-Hour Incremental"
    ```bash
    # Add to crontab (incremental backups every 6 hours)
    crontab -e

    # Every 6 hours (2AM, 8AM, 2PM, 8PM)
    0 */6 * * * /opt/osint-platform/scripts/backup.sh --no-ollama >> /var/log/osint-backup.log 2>&1
    ```

=== "Weekly Full with Hetzner"
    ```bash
    # Add to crontab (weekly full backup with Hetzner snapshot)
    crontab -e

    # Sunday at 3 AM - full backup with Hetzner snapshot
    0 3 * * 0 /opt/osint-platform/scripts/backup.sh --hetzner-snapshot >> /var/log/osint-backup-weekly.log 2>&1

    # Daily incremental backups
    0 2 * * 1-6 /opt/osint-platform/scripts/backup.sh --no-ollama >> /var/log/osint-backup.log 2>&1
    ```

### Backup Retention Policy

The automated `backup.sh` script does NOT auto-delete old backups. Implement retention with a cleanup script:

```bash
#!/bin/bash
# cleanup-old-backups.sh - Remove backups older than retention period

BACKUP_DIR="/opt/osint-platform/backups"
RETENTION_DAYS=30  # Keep 30 days of backups

# Find and delete backup directories older than retention period
find "$BACKUP_DIR" -maxdepth 1 -type d -name "2*" -mtime +$RETENTION_DAYS -exec rm -rf {} \;

echo "Cleaned up backups older than $RETENTION_DAYS days"
```

**Schedule cleanup weekly:**

```bash
# Add to crontab (run weekly on Monday at 4 AM)
0 4 * * 1 /opt/osint-platform/scripts/cleanup-old-backups.sh >> /var/log/backup-cleanup.log 2>&1
```

### Off-Site Backup Sync

Create `/scripts/backup-offsite.sh` for cloud/NAS sync:

```bash
#!/bin/bash
# backup-offsite.sh - Sync backups to off-site storage (cloud/NAS)
# Run via cron: 0 3 * * 0 /opt/osint-platform/scripts/backup-offsite.sh (weekly)

set -e

BACKUP_DIR="/opt/osint-platform/backups"
LATEST_BACKUP=$(ls -td "$BACKUP_DIR"/*/ | head -1)

echo "Syncing latest backup to off-site storage: $LATEST_BACKUP"

# Option 1: Sync to AWS S3
rclone sync "$LATEST_BACKUP" s3:osint-backup-offsite/$(basename "$LATEST_BACKUP") \
  --progress --log-file=/var/log/offsite-backup.log

# Option 2: Sync to Wasabi (cheaper for large files)
# rclone sync "$LATEST_BACKUP" wasabi:osint-backup/$(basename "$LATEST_BACKUP") \
#   --progress --log-file=/var/log/offsite-backup.log

# Option 3: Sync to network NAS
# rsync -avz --progress "$LATEST_BACKUP" /mnt/nas/osint-backups/

# Send notification on success
if command -v curl &> /dev/null; then
  curl -d "Off-site backup sync completed: $(basename $LATEST_BACKUP)" http://localhost:8090/osint-backups
fi

echo "Off-site sync complete"
```

**Make executable and schedule:**

```bash
chmod +x scripts/backup-offsite.sh

# Add to crontab (weekly on Sunday at 5 AM, after backup completes)
0 5 * * 0 /opt/osint-platform/scripts/backup-offsite.sh >> /var/log/offsite-backup.log 2>&1
```

### Backup Monitoring and Notifications

Monitor backup success/failure using ntfy notifications:

```bash
#!/bin/bash
# backup-with-notification.sh - Wrapper for backup.sh with notifications

BACKUP_SCRIPT="/opt/osint-platform/scripts/backup.sh"
NTFY_TOPIC="http://localhost:8090/osint-backups"

# Run backup
if $BACKUP_SCRIPT --no-ollama >> /var/log/osint-backup.log 2>&1; then
    # Success
    BACKUP_SIZE=$(du -sh /opt/osint-platform/backups | cut -f1)
    curl -H "Title: Backup Success" \
         -H "Priority: low" \
         -H "Tags: white_check_mark" \
         -d "Daily backup completed successfully. Total size: $BACKUP_SIZE" \
         "$NTFY_TOPIC"
else
    # Failure
    curl -H "Title: Backup FAILED" \
         -H "Priority: urgent" \
         -H "Tags: rotating_light" \
         -d "Daily backup FAILED! Check logs: /var/log/osint-backup.log" \
         "$NTFY_TOPIC"
    exit 1
fi
```

### Automated Restore Testing

Test backup integrity monthly with automated restore to test environment:

```bash
#!/bin/bash
# test-backup-restore.sh - Monthly backup restore test

BACKUP_DIR="/opt/osint-platform/backups"
LATEST_BACKUP=$(ls -td "$BACKUP_DIR"/*/ | head -1)
TEST_ENV="/opt/osint-platform-test"

echo "Testing restore of: $LATEST_BACKUP"

# 1. Clone platform to test directory
cd /opt
if [ ! -d "$TEST_ENV" ]; then
    git clone https://github.com/osintukraine/osint-intelligence-platform.git osint-platform-test
fi

cd "$TEST_ENV"

# 2. Test restore (dry-run)
./scripts/restore.sh "$LATEST_BACKUP" --dry-run

# 3. Verify backup integrity
if [ $? -eq 0 ]; then
    echo "✓ Backup integrity test PASSED"
    curl -H "Title: Backup Test Success" \
         -d "Monthly backup restore test passed: $(basename $LATEST_BACKUP)" \
         http://localhost:8090/osint-backups
else
    echo "✗ Backup integrity test FAILED"
    curl -H "Title: Backup Test FAILED" \
         -H "Priority: urgent" \
         -d "Monthly backup restore test FAILED: $(basename $LATEST_BACKUP)" \
         http://localhost:8090/osint-backups
    exit 1
fi
```

**Schedule monthly:**

```bash
# Add to crontab (first Sunday of each month at 6 AM)
0 6 1-7 * 0 /opt/osint-platform/scripts/test-backup-restore.sh >> /var/log/backup-test.log 2>&1
```

---

## Common Operational Workflows

Practical workflows for common backup/restore scenarios using the automated scripts.

### Scenario 1: First Production Deployment

**Goal**: Deploy platform to production with proper backup configuration.

```bash
# 1. Clone and configure platform
git clone https://github.com/osintukraine/osint-intelligence-platform.git
cd osint-intelligence-platform
cp .env.example .env
# Edit .env with production values

# 2. Run preflight check BEFORE deploying
./scripts/preflight-check.sh

# Expected output: Some warnings about services not running yet (OK)
# Fix any CRITICAL issues (default passwords, missing credentials)

# 3. Deploy services
docker compose up -d

# Wait for services to initialize (2-3 minutes)

# 4. Run preflight check AGAIN (verify deployment)
./scripts/preflight-check.sh

# Expected output: All checks should PASS (green)

# 5. Create first backup immediately
./scripts/backup.sh --no-ollama

# 6. Schedule automated backups
crontab -e
# Add: 0 2 * * * /opt/osint-platform/scripts/backup.sh --no-ollama >> /var/log/osint-backup.log 2>&1

# 7. Configure Hetzner Storage Box (for media)
./scripts/setup-hetzner-storage.sh
./scripts/backup.sh --hetzner-snapshot

# 8. Verify everything is working
./scripts/preflight-check.sh
```

### Scenario 2: Regular Maintenance Backup

**Goal**: Weekly full backup before maintenance/upgrades.

```bash
# Before any maintenance or upgrade work:

# 1. Create full backup (with Ollama models)
./scripts/backup.sh

# Output saved to: ./backups/YYYYMMDD_HHMMSS/

# 2. Trigger Hetzner snapshot for media
./scripts/backup.sh --hetzner-snapshot

# 3. Verify backup integrity
ls -lh ./backups/$(ls -t ./backups | head -1)/

# Should see:
# - database.dump (4-50GB)
# - telegram_sessions.tar.gz (~1MB)
# - ollama_models.tar.gz (5-10GB)
# - config/ directory
# - MANIFEST.txt

# 4. Test restore with dry-run
LATEST_BACKUP=$(ls -td ./backups/*/ | head -1)
./scripts/restore.sh "$LATEST_BACKUP" --dry-run

# Expected output: List of operations that would be performed

# 5. Now safe to proceed with maintenance
git pull origin master
docker compose up -d --build
```

### Scenario 3: Emergency Database Restore

**Goal**: Database corruption detected, need to restore from backup.

```bash
# 1. Identify the issue
docker compose exec postgres psql -U osint_user -d osint_platform
# ERROR: relation "messages" does not exist

# 2. Find latest good backup
ls -lht ./backups/

# 3. Stop application services (keep database running)
docker compose stop listener processor-worker api frontend

# 4. Restore ONLY the database
LATEST_BACKUP=$(ls -td ./backups/*/ | head -1)
./scripts/restore.sh "$LATEST_BACKUP" --skip-sessions --skip-ollama

# This will:
# - Restore database from backup
# - Skip Telegram sessions (already OK)
# - Skip Ollama models (already OK)

# 5. Verify database is restored
docker compose exec postgres psql -U osint_user -d osint_platform -c "SELECT COUNT(*) FROM messages;"

# 6. Restart all services
docker compose up -d

# 7. Monitor for issues
docker compose logs -f --tail=100
```

### Scenario 4: Migrate to New Server

**Goal**: Move entire platform from old server to new server.

```bash
# ON OLD SERVER:

# 1. Create full backup
cd /opt/osint-platform
./scripts/backup.sh --hetzner-snapshot

# 2. Copy backup to new server
LATEST_BACKUP=$(ls -td ./backups/*/ | head -1)
rsync -avz --progress "$LATEST_BACKUP" user@newserver:/tmp/osint-backup/

# ON NEW SERVER:

# 3. Install prerequisites
sudo apt update
sudo apt install docker.io docker-compose git

# 4. Clone platform
git clone https://github.com/osintukraine/osint-intelligence-platform.git /opt/osint-platform
cd /opt/osint-platform

# 5. Restore from backup
./scripts/restore.sh /tmp/osint-backup

# 6. Configure Hetzner Storage Box mount
./scripts/setup-hetzner-storage.sh

# 7. Restore media from Hetzner snapshot
# Via Hetzner Console: https://robot.hetzner.com/storage

# 8. Verify everything works
./scripts/preflight-check.sh

# Expected: All checks PASS

# 9. Test services
curl http://localhost:8000/health
curl http://localhost:3000

# 10. Update DNS to point to new server

# 11. Schedule backups on new server
crontab -e
# Add backup schedule
```

### Scenario 5: Telegram Session Recovery

**Goal**: Lost Telegram sessions, need to restore authentication.

```bash
# 1. Find backup with sessions
ls -lh ./backups/*/telegram_sessions.tar.gz

# 2. Stop listener service
docker compose stop listener

# 3. Restore ONLY Telegram sessions
LATEST_BACKUP=$(ls -td ./backups/*/ | head -1)
./scripts/restore.sh "$LATEST_BACKUP" --skip-db --skip-ollama

# This will:
# - Skip database (already OK)
# - Restore Telegram sessions
# - Skip Ollama models (already OK)

# 4. Restart listener
docker compose start listener

# 5. Verify authentication
docker compose logs listener | grep "Logged in as"

# Expected: "Logged in as +1234567890"
```

### Scenario 6: Test Backup Before Critical Change

**Goal**: Verify backup is restorable before risky operation.

```bash
# Before making risky changes (schema migration, major upgrade):

# 1. Create fresh backup
./scripts/backup.sh

# 2. Test restore on another machine/directory
# Clone to test location
git clone https://github.com/osintukraine/osint-intelligence-platform.git /opt/osint-test
cd /opt/osint-test

# 3. Modify docker-compose.yml to use different ports
# Edit ports: 8000->8001, 3000->3001, 5432->5433, etc.

# 4. Test restore
LATEST_BACKUP=$(ls -td /opt/osint-platform/backups/*/ | head -1)
./scripts/restore.sh "$LATEST_BACKUP"

# 5. Verify test environment works
curl http://localhost:8001/health
docker compose exec postgres psql -U osint_user -d osint_platform -c "SELECT COUNT(*) FROM messages;"

# 6. If test succeeds, safe to proceed with risky change on production

# 7. Cleanup test environment
docker compose down -v
rm -rf /opt/osint-test
```

### Scenario 7: Scheduled Backup Health Check

**Goal**: Monthly verification that backups are working.

```bash
# Run on first Monday of each month:

# 1. Check latest backup exists and is recent
LATEST_BACKUP=$(ls -td ./backups/*/ | head -1)
BACKUP_AGE_DAYS=$(( ($(date +%s) - $(stat -c %Y "$LATEST_BACKUP")) / 86400 ))

if [ $BACKUP_AGE_DAYS -gt 7 ]; then
    echo "❌ WARNING: Latest backup is $BACKUP_AGE_DAYS days old!"
    # Send alert
    curl -H "Priority: urgent" -d "Backup is stale: $BACKUP_AGE_DAYS days old" http://localhost:8090/osint-backups
fi

# 2. Verify backup contents
ls -lh "$LATEST_BACKUP"

# Should see:
# - database.dump (>100MB)
# - telegram_sessions.tar.gz
# - MANIFEST.txt

# 3. Test restore dry-run
./scripts/restore.sh "$LATEST_BACKUP" --dry-run

# 4. Check backup size trends
du -sh ./backups/*/
# Look for unexpected size changes (could indicate issues)

# 5. Verify Hetzner snapshots exist
# Check Hetzner Console: https://robot.hetzner.com/storage
# Should have weekly snapshots for past 4+ weeks
```

---

## Testing Backups

!!! warning "Critical Practice"
    **Untested backups are not real backups.** Test restore procedures monthly to ensure recovery capability.

### Monthly Backup Test Procedure

**Test procedure** (run monthly):

```bash
# 1. Create test environment
docker-compose -f docker-compose.test.yml up -d

# 2. Restore latest backup to test environment
gunzip -c backups/postgres/osint_platform_latest.sql.gz | \
  docker-compose -f docker-compose.test.yml exec -T postgres psql -U postgres test_platform

# 3. Verify data integrity
docker-compose -f docker-compose.test.yml exec postgres psql -U postgres test_platform -c "
  SELECT COUNT(*) FROM messages;
  SELECT COUNT(*) FROM channels;
  SELECT COUNT(*) FROM entities;
"

# 4. Test API functionality
curl http://localhost:8001/api/health
curl http://localhost:8001/api/messages?limit=10

# 5. Cleanup test environment
docker-compose -f docker-compose.test.yml down -v
```

**Document test results:**

```bash
# Create test report
cat > backups/test-reports/test_$(date +%Y%m%d).txt <<EOF
Backup Test Report - $(date)
============================

Backup File: osint_platform_$(date +%Y%m%d).sql.gz
Backup Size: $(du -h backups/postgres/osint_platform_latest.sql.gz | cut -f1)

Database Restore: ✓ Success
Table Row Counts:
  - messages: $(docker-compose -f docker-compose.test.yml exec postgres psql -U postgres test_platform -t -c "SELECT COUNT(*) FROM messages")
  - channels: $(docker-compose -f docker-compose.test.yml exec postgres psql -U postgres test_platform -t -c "SELECT COUNT(*) FROM channels")

API Health Check: ✓ Success

Tested By: $(whoami)
EOF
```

---

## Disaster Recovery Planning

### Disaster Scenarios and Recovery Plans

**Scenario 1: Database Corruption**

- **RPO**: 6 hours (incremental backup frequency)
- **RTO**: 30 minutes (restore from latest backup)
- **Procedure**: Restore from latest PostgreSQL backup + WAL replay

**Scenario 2: Complete Server Failure**

- **RPO**: 24 hours (daily backup frequency)
- **RTO**: 2 hours (rebuild server + restore)
- **Procedure**: Follow "Full System Restore" procedure above

**Scenario 3: Ransomware Attack**

- **RPO**: 24 hours (off-site backup lag)
- **RTO**: 4 hours (clean server + restore from off-site)
- **Procedure**: Rebuild server from scratch, restore from off-site encrypted backups

**Scenario 4: Telegram Account Ban**

- **RPO**: 0 (sessions backed up after auth)
- **RTO**: 2 hours (get new phone number, re-auth, restore sessions)
- **Procedure**: Use backup phone number, re-authenticate, join channels from backup list

### Disaster Recovery Checklist

**Pre-disaster preparation:**

- [ ] Backup automation running (verify via cron logs)
- [ ] Off-site backups enabled (verify via rclone logs)
- [ ] Backup encryption enabled (verify .gpg files exist)
- [ ] Monthly backup tests documented
- [ ] Emergency contact list up to date
- [ ] Backup server credentials stored in password manager
- [ ] Recovery procedures documented and tested

**During disaster:**

- [ ] Assess scope of failure (database, server, network?)
- [ ] Notify team and stakeholders
- [ ] Identify most recent valid backup
- [ ] Spin up replacement infrastructure
- [ ] Begin restore procedure
- [ ] Verify data integrity after restore
- [ ] Resume service and monitor

**Post-disaster:**

- [ ] Document incident timeline
- [ ] Analyze root cause
- [ ] Update disaster recovery procedures
- [ ] Test new backup strategy
- [ ] Train team on lessons learned

---

## Best Practices

### Backup Best Practices

1. **Use Automated Scripts**: Use `backup.sh` for consistency and reliability
2. **3-2-1 Rule**: 3 copies, 2 media types, 1 off-site location
3. **Automate Everything**: Manual backups are forgotten backups - use cron
4. **Test Monthly**: Untested backups are not real backups - verify with `--dry-run`
5. **Monitor Success**: Alert on backup failures via ntfy/Discord
6. **Encrypt Sensitive Data**: Especially Telegram sessions and .env files
7. **Separate Storage**: Don't backup to same disk/server as production
8. **Version Control Config**: Track changes with git (except .env)
9. **Document Procedures**: Keep disaster recovery runbook updated
10. **Hetzner Snapshots for Media**: Use Hetzner Storage Box snapshots, not rsync

### Restore Best Practices

1. **Always Test with --dry-run**: Preview changes before applying
2. **Verify Backup Manifest**: Check MANIFEST.txt before restore
3. **Stop Services First**: Prevent data corruption during restore
4. **Use Selective Restore**: Only restore what's needed (--skip-db, --skip-sessions)
5. **Verify After Restore**: Run preflight-check.sh to confirm health
6. **Keep Old Backup**: Don't delete old backup until new system is verified
7. **Test on Non-Production**: Test restore on test server before production

### Security Best Practices

1. **Encrypt Backups**: Use GPG for sensitive files before off-site storage
2. **Protect .env**: Never commit to git, encrypt in backups
3. **Secure Sessions**: Telegram sessions are as sensitive as passwords
4. **Rotate Credentials**: Change production passwords after test restores
5. **Audit Access**: Track who has access to backups
6. **Off-Site Security**: Use encrypted connections (SSH, HTTPS) for transfers

### Operational Best Practices

1. **Pre-Flight Before Deploy**: Always run `preflight-check.sh` before production deploy
2. **Backup Before Changes**: Create backup before upgrades/migrations
3. **Schedule Regular Checks**: Run preflight checks daily via cron
4. **Monitor Disk Space**: Backups consume significant space, monitor proactively
5. **Clean Old Backups**: Implement retention policy (30 days recommended)
6. **Document Changes**: Keep notes of what changed in each backup
7. **Test Disaster Recovery**: Quarterly full disaster recovery drills

---

## Quick Reference

### Essential Commands

```bash
# Create backup (fast, no Ollama models)
./scripts/backup.sh --no-ollama

# Create backup with Hetzner snapshot
./scripts/backup.sh --hetzner-snapshot

# Restore from backup (with confirmation)
./scripts/restore.sh /path/to/backup/YYYYMMDD_HHMMSS

# Test restore without changes
./scripts/restore.sh /path/to/backup/YYYYMMDD_HHMMSS --dry-run

# Selective restore (database only)
./scripts/restore.sh /path/to/backup/YYYYMMDD_HHMMSS --skip-sessions --skip-ollama

# Production readiness check
./scripts/preflight-check.sh

# Quick health check
./scripts/preflight-check.sh --quick
```

### Script Locations

| Script | Path | Purpose |
|--------|------|---------|
| **backup.sh** | `/scripts/backup.sh` | Full platform backup |
| **restore.sh** | `/scripts/restore.sh` | Full platform restore |
| **preflight-check.sh** | `/scripts/preflight-check.sh` | Production readiness verification |

### Backup Components

| Component | Location in Backup | Critical | Size |
|-----------|-------------------|----------|------|
| PostgreSQL | `database.dump` | ✅ Yes | 3-50GB |
| PostgreSQL (SQL) | `database.sql.gz` | ✅ Yes | 1-20GB |
| Telegram Sessions | `telegram_sessions.tar.gz` | ✅ Yes | ~1MB |
| Configuration | `config/.env` | ✅ Yes | ~10KB |
| Configuration | `config/infrastructure/` | ⚠️ Important | ~1MB |
| Redis | `redis.rdb` | ⚪ Optional | ~100MB |
| Ollama Models | `ollama_models.tar.gz` | ⚪ Optional | 5-10GB |
| Media Files | Not in standard backup | ✅ Yes | 100GB-10TB |
| Manifest | `MANIFEST.txt` | ⚠️ Important | ~1KB |

### Recommended Cron Schedule

```bash
# Daily backup at 2 AM (without Ollama models)
0 2 * * * /opt/osint-platform/scripts/backup.sh --no-ollama >> /var/log/osint-backup.log 2>&1

# Weekly full backup with Hetzner snapshot (Sunday 3 AM)
0 3 * * 0 /opt/osint-platform/scripts/backup.sh --hetzner-snapshot >> /var/log/osint-backup-weekly.log 2>&1

# Daily preflight check (8 AM)
0 8 * * * /opt/osint-platform/scripts/preflight-check.sh >> /var/log/osint-preflight.log 2>&1

# Weekly backup cleanup (Monday 4 AM)
0 4 * * 1 /opt/osint-platform/scripts/cleanup-old-backups.sh >> /var/log/backup-cleanup.log 2>&1

# Monthly backup test (first Sunday 6 AM)
0 6 1-7 * 0 /opt/osint-platform/scripts/test-backup-restore.sh >> /var/log/backup-test.log 2>&1
```

### Exit Codes

| Command | Code | Meaning |
|---------|------|---------|
| `backup.sh` | 0 | Backup successful |
| `backup.sh` | 1 | Backup failed |
| `restore.sh` | 0 | Restore successful |
| `restore.sh` | 1 | Restore failed |
| `preflight-check.sh` | 0 | All checks passed |
| `preflight-check.sh` | 1 | Critical failures (do NOT deploy) |
| `preflight-check.sh` | 2 | Warnings (review before deploy) |

### Common Flags

| Script | Flag | Purpose |
|--------|------|---------|
| `backup.sh` | `--no-ollama` | Skip Ollama models (saves 5-10GB) |
| `backup.sh` | `--include-media` | Include media files (not recommended) |
| `backup.sh` | `--hetzner-snapshot` | Trigger Hetzner snapshot |
| `restore.sh` | `--dry-run` | Preview without changes |
| `restore.sh` | `--skip-db` | Skip database restore |
| `restore.sh` | `--skip-sessions` | Skip Telegram sessions |
| `restore.sh` | `--skip-ollama` | Skip Ollama models |
| `preflight-check.sh` | `--quick` | Essential checks only |

### Troubleshooting

| Problem | Solution |
|---------|----------|
| Backup fails: "Docker not running" | Start Docker: `sudo systemctl start docker` |
| Backup fails: "No space left" | Clean old backups or expand storage |
| Restore fails: "Database already exists" | Normal - script drops and recreates |
| Preflight check fails: "Default password" | Change passwords in `.env` |
| Preflight check fails: "Queue backlog >1000" | Check processor health: `docker compose logs processor-worker` |
| Backup very large (>100GB) | Don't use `--include-media`, use Hetzner snapshots |
| Restore slow (hours) | Normal for large databases, be patient |

---

## Additional Resources

### Official Tools

- **PostgreSQL pg_dump**: https://www.postgresql.org/docs/current/app-pgdump.html
- **MinIO Client (mc)**: https://min.io/docs/minio/linux/reference/minio-mc.html
- **rclone**: https://rclone.org/docs/
- **GPG encryption**: https://gnupg.org/documentation/

### Platform Documentation

- **Installation Guide**: [operator-guide/installation.md](installation.md)
- **Configuration Guide**: [operator-guide/configuration.md](configuration.md)
- **Monitoring Guide**: [operator-guide/monitoring.md](monitoring.md)
- **Troubleshooting Guide**: [operator-guide/troubleshooting.md](troubleshooting.md)
- **Hetzner Storage Setup**: [operator-guide/hetzner-storage.md](hetzner-storage.md)

### Platform Scripts

| Script | Location | Documentation |
|--------|----------|---------------|
| `backup.sh` | `/scripts/backup.sh` | [Automated Scripts](#backupsh-full-platform-backup) |
| `restore.sh` | `/scripts/restore.sh` | [Automated Scripts](#restoresh-full-platform-restore) |
| `preflight-check.sh` | `/scripts/preflight-check.sh` | [Production Preflight Check](#preflight-checksh-production-readiness-verification) |
| `setup-hetzner-storage.sh` | `/scripts/setup-hetzner-storage.sh` | [Hetzner Storage Guide](hetzner-storage.md) |

### Related Guides

- **Telegram Disaster Recovery**: [operator-guide/telegram-disaster-recovery.md](telegram-disaster-recovery.md) - Telegram session recovery procedures
- **Upgrades**: [operator-guide/upgrades.md](upgrades.md) - Platform upgrade procedures with backup integration
- **Scaling**: [operator-guide/scaling.md](scaling.md) - Scaling considerations for backup/restore

---

**Last Updated**: 2025-12-21
**Version**: 2.0
**Platform Version**: 1.0 (Production-ready)
**Scripts Version**: backup.sh v1.0, restore.sh v1.0, preflight-check.sh v1.0
