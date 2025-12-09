# Backup & Restore Guide

**OSINT Intelligence Platform - Comprehensive Backup Strategies and Disaster Recovery**

Complete procedures for protecting platform data, creating recoverable backups, and disaster recovery planning.

---

## Table of Contents

- [Overview](#overview)
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

1. **PostgreSQL Database**: Messages, entities, channels, users, OSINT scores (CRITICAL)
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
config/osint_rules.yml         # OSINT scoring rules
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

### Full System Restore (Disaster Recovery)

**Scenario**: Complete server failure, need to rebuild from scratch.

**Prerequisites**:

- Backup files (PostgreSQL, MinIO, Telegram sessions, configuration)
- Fresh Ubuntu/Debian server
- Docker and Docker Compose installed

**Step-by-step restore:**

```bash
# 1. Clone platform repository
git clone https://github.com/osintukraine/osint-intelligence-platform.git
cd osint-intelligence-platform

# 2. Restore configuration files
tar -xzf /restore-media/config_20251209.tar.gz -C .

# 3. Start infrastructure services only
docker-compose up -d postgres redis minio

# Wait for services to be healthy (30-60 seconds)
docker-compose ps

# 4. Restore PostgreSQL database
gunzip -c /restore-media/osint_platform_20251209.sql.gz | \
  docker-compose exec -T postgres psql -U postgres osint_platform

# Verify database restore
docker-compose exec postgres psql -U postgres -d osint_platform -c "\dt"
# Should list all tables: messages, channels, entities, etc.

# 5. Restore MinIO media files
./bin/mc mirror /restore-media/minio/osint-media minio/osint-media

# Verify media restore
./bin/mc du minio/osint-media
# Should show total media size

# 6. Restore Telegram sessions
tar -xzf /restore-media/sessions_20251209.tar.gz -C telegram_sessions/
chmod 600 telegram_sessions/*.session*

# 7. Start all services
docker-compose up -d

# 8. Verify system health
./scripts/health-check.sh
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

### Automated Backup Script

Create `/scripts/backup-full.sh`:

```bash
#!/bin/bash
# backup-full.sh - Full daily backup of OSINT Platform
# Run via cron: 0 2 * * * /opt/osint-platform/scripts/backup-full.sh

set -e  # Exit on error

# Configuration
BACKUP_DIR="/backups/osint-platform"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Create backup directory
mkdir -p $BACKUP_DIR/postgres $BACKUP_DIR/minio $BACKUP_DIR/config $BACKUP_DIR/sessions

echo "=== OSINT Platform Full Backup - $DATE ==="

# 1. PostgreSQL backup
echo "Backing up PostgreSQL database..."
docker-compose exec -T postgres pg_dump -U postgres -Fc osint_platform \
  > $BACKUP_DIR/postgres/osint_platform_$DATE.dump
echo "PostgreSQL backup complete: $(du -h $BACKUP_DIR/postgres/osint_platform_$DATE.dump | cut -f1)"

# 2. MinIO backup (incremental, only last 24 hours)
echo "Backing up MinIO media (incremental)..."
mc mirror --newer-than 24h production/osint-media $BACKUP_DIR/minio/osint-media
echo "MinIO backup complete"

# 3. Configuration backup
echo "Backing up configuration files..."
tar -czf $BACKUP_DIR/config/config_$DATE.tar.gz \
  .env docker-compose.yml config/ infrastructure/
echo "Configuration backup complete"

# 4. Telegram sessions backup
echo "Backing up Telegram sessions..."
docker-compose exec listener tar -czf /tmp/sessions.tar.gz /app/sessions/
docker cp osint-listener:/tmp/sessions.tar.gz $BACKUP_DIR/sessions/sessions_$DATE.tar.gz
echo "Session backup complete"

# 5. Cleanup old backups
echo "Cleaning up backups older than $RETENTION_DAYS days..."
find $BACKUP_DIR/postgres -name "*.dump" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR/config -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR/sessions -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

# 6. Backup verification
echo "Verifying backup integrity..."
# Check PostgreSQL backup is valid
docker run --rm -v $BACKUP_DIR:/backups postgres:16 pg_restore --list \
  /backups/postgres/osint_platform_$DATE.dump > /dev/null && echo "✓ PostgreSQL backup is valid" || echo "✗ PostgreSQL backup is corrupt!"

# 7. Send notification
if command -v curl &> /dev/null; then
  curl -d "OSINT Platform backup completed successfully - $DATE" http://localhost:8090/osint-backups
fi

echo "=== Backup Complete ==="
echo "Total backup size: $(du -sh $BACKUP_DIR | cut -f1)"
```

**Make executable and schedule:**

```bash
chmod +x scripts/backup-full.sh

# Add to crontab (run daily at 2 AM)
crontab -e
# Add line:
0 2 * * * /opt/osint-platform/scripts/backup-full.sh >> /var/log/osint-backup.log 2>&1
```

### Off-Site Backup Sync

Create `/scripts/backup-offsite.sh`:

```bash
#!/bin/bash
# backup-offsite.sh - Sync backups to off-site storage (cloud/NAS)
# Run via cron: 0 3 * * 0 /opt/osint-platform/scripts/backup-offsite.sh (weekly)

set -e

# Sync to AWS S3 (example)
rclone sync /backups/osint-platform s3:osint-backup-offsite \
  --progress --exclude "minio/**" --log-file=/var/log/offsite-backup.log

# Sync media to Wasabi (cheaper for large files)
rclone sync /backups/osint-platform/minio wasabi:osint-media-backup \
  --progress --log-file=/var/log/offsite-media-backup.log

# Send notification
curl -d "Off-site backup sync completed" http://localhost:8090/osint-backups
```

---

## Testing Backups

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

1. **3-2-1 Rule**: 3 copies, 2 media types, 1 off-site
2. **Automate everything**: Manual backups are forgotten backups
3. **Encrypt sensitive data**: Especially Telegram sessions and .env files
4. **Test monthly**: Untested backups are not real backups
5. **Monitor backup success**: Alert on backup failures
6. **Document procedures**: Recovery is stressful, checklists help
7. **Version control configuration**: Git tracks changes
8. **Separate backup storage**: Don't backup to same disk/server as production

---

## Additional Resources

### Tools

- **pg_dump**: https://www.postgresql.org/docs/current/app-pgdump.html
- **MinIO Client (mc)**: https://min.io/docs/minio/linux/reference/minio-mc.html
- **rclone**: https://rclone.org/docs/
- **gpg**: https://gnupg.org/documentation/

### Project Files

- Backup automation: `/scripts/backup-full.sh`
- Off-site sync: `/scripts/backup-offsite.sh`
- Health check: `/scripts/health-check.sh`

---

**Last Updated**: 2025-12-09
**Version**: 1.0
**Platform Version**: 1.0 (Production-ready)
