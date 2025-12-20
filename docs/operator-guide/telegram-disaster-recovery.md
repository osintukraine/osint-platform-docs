# Telegram Disaster Recovery Runbook

This runbook covers disaster recovery procedures for Telegram account issues, including account bans, session invalidation, and channel recovery.

## Overview

The OSINT Intelligence Platform uses a multi-layered disaster recovery system:

1. **Shadow Account Sync** - Real-time mirror of channel subscriptions to standby accounts
2. **Automatic Failover** - Health monitoring with automatic switch on ban detection
3. **JSON Backup to MinIO** - Human-readable backups for manual recovery
4. **Recovery Scripts** - CLI tools for status, failover, and restore operations

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    RUSSIA CLUSTER                               │
│  ┌─────────────────────┐    ┌─────────────────────┐            │
│  │ russia-primary      │    │ russia-shadow       │            │
│  │ (ACTIVE)            │◄──►│ (STANDBY)           │            │
│  │ Archive-RU channels │    │ Mirrors all joins   │            │
│  └─────────────────────┘    └─────────────────────┘            │
│           │ If banned, failover ──────►│                       │
└─────────────────────────────────────────────────────────────────┘
```

Each cluster (russia, ukraine) has independent primary and shadow accounts. A ban in one cluster does not affect the other.

## Quick Reference

| Command | Purpose |
|---------|---------|
| `python scripts/dr_status.py` | Show current DR status |
| `python scripts/dr_failover.py --cluster russia --dry-run` | Test failover |
| `python scripts/dr_failover.py --cluster russia --force` | Execute failover |
| `python scripts/dr_restore.py --list` | List available backups |
| `python scripts/dr_restore.py --from-minio --latest --dry-run` | Preview restore |

## Monitoring

### Grafana Dashboard

Access the Disaster Recovery dashboard at:
```
https://your-domain/grafana/d/disaster-recovery
```

Key panels:
- **Account Health**: Shows healthy/unhealthy/banned status for each account
- **Shadow Sync Status**: Channels synced/pending/failed
- **Failover Events**: Recent failover history
- **Backup Status**: Time since last backup, backup size

### Critical Alerts

| Alert | Severity | Action |
|-------|----------|--------|
| `TelegramAccountBanned` | Critical | Failover triggers automatically (if enabled) |
| `NoShadowChannelsSynced` | Critical | Shadow account unusable - fix immediately |
| `ShadowSyncServiceDown` | Critical | Restart shadow-sync service |
| `BackupCriticallyStale` | Critical | Run manual backup immediately |

---

## Scenario 1: Automatic Failover (Shadow Ready)

**When**: Primary account banned, shadow account is synced, `FAILOVER_AUTO_SWITCH=true`

**What happens automatically**:
1. Health monitor detects `UserDeactivatedBanError`
2. Primary marked as `role=banned` in database
3. Shadow promoted to `role=active`
4. Failover recorded in `dr_failover_history`
5. Alert sent via Prometheus/Grafana

**Operator steps**:

```bash
# 1. Verify failover occurred
python scripts/dr_status.py

# 2. Check the new primary is healthy
docker-compose logs -f shadow-sync | grep "health"

# 3. Restart listener to use new credentials
# Note: The listener reads from telegram_accounts table,
# so it should pick up the new active account on restart
docker-compose restart listener

# 4. Verify messages are flowing
docker-compose logs -f listener | head -50

# 5. Monitor for 10 minutes
watch -n 10 'docker-compose exec postgres psql -U osint_user -d osint_platform -c "SELECT COUNT(*) FROM messages WHERE created_at > NOW() - INTERVAL '\''10 minutes'\'';"'
```

**Expected outcome**: Messages resume flowing within 5 minutes.

---

## Scenario 2: Manual Failover

**When**: Primary account banned, `FAILOVER_AUTO_SWITCH=false`, or automatic failover failed

**Steps**:

```bash
# 1. Verify the ban
docker-compose logs listener | grep -i "ban\|deactivated" | tail -20

# 2. Check DR status
python scripts/dr_status.py

# 3. Test what failover would do (dry-run)
python scripts/dr_failover.py --cluster russia --dry-run

# Expected output:
# Current Active: russia-primary
#   Health: banned
# Available Standbys:
#   - russia-shadow (priority=0, health=healthy)
# Next Standby: russia-shadow
# Channels Affected: 42
# ✅ Ready for failover

# 4. Execute failover
python scripts/dr_failover.py --cluster russia --force --reason "Manual failover due to ban"

# 5. Restart listener with new account
docker-compose restart listener

# 6. Verify
python scripts/dr_status.py
```

---

## Scenario 3: Full Manual Restore (No Shadow Available)

**When**: Both primary and shadow accounts are banned or unavailable

**Prerequisites**:
- A new Telegram account with API credentials from [my.telegram.org](https://my.telegram.org/apps)
- The account must be authenticated (has a valid .session file)

**Steps**:

```bash
# 1. Create session for new account
# Run this interactively - will prompt for phone and code
python scripts/telegram_auth.py --account new-russia-primary

# 2. List available backups
python scripts/dr_restore.py --list

# Example output:
# Available backups in osint-backups:
#   2025-12-20 00:00 - channel-state/2025/12/20/channel_state_2025-12-20T00-00-00.json (45.2 KB)
#   2025-12-19 00:00 - channel-state/2025/12/19/channel_state_2025-12-19T00-00-00.json (44.8 KB)

# 3. Preview restore (dry-run)
python scripts/dr_restore.py \
    --from-minio \
    --latest \
    --cluster russia \
    --dry-run

# 4. Execute restore
python scripts/dr_restore.py \
    --from-minio \
    --latest \
    --account new-russia-primary \
    --cluster russia

# This will:
# - Join all public channels via @username
# - Join private channels via invite_link (if available)
# - Rate-limited to 5 joins/minute to avoid Telegram limits

# 5. Register new account in database
python scripts/dr_failover.py \
    --add-account new-russia-primary \
    --cluster russia \
    --role active

# 6. Update environment variables
# Edit .env file:
# RUSSIA_PRIMARY_API_ID=<new_id>
# RUSSIA_PRIMARY_API_HASH=<new_hash>
# RUSSIA_PRIMARY_PHONE=<new_phone>

# 7. Restart services
docker-compose up -d listener
```

**Note**: Private channels without `invite_link` stored cannot be recovered automatically. You'll need to obtain new invite links manually.

---

## Scenario 4: Rollback a Failed Failover

**When**: Failover was triggered but the new account is not working correctly

**Steps**:

```bash
# 1. Get failover ID from history
python scripts/dr_status.py --json | jq '.recent_failovers'

# 2. Rollback
python scripts/dr_failover.py \
    --rollback 123 \
    --reason "False positive - original account is still working"

# 3. Verify rollback
python scripts/dr_status.py

# 4. Restart listener
docker-compose restart listener
```

---

## Setting Up Shadow Accounts

### Initial Setup

1. **Create Shadow Account Credentials**

   Go to [my.telegram.org](https://my.telegram.org/apps) and create API credentials for your shadow phone number.

2. **Configure Environment Variables**

   ```bash
   # .env
   DR_ENABLED=true

   # Russia cluster
   RUSSIA_PRIMARY_API_ID=12345678
   RUSSIA_PRIMARY_API_HASH=abc123...
   RUSSIA_PRIMARY_PHONE=+1234567890

   RUSSIA_SHADOW_API_ID=87654321
   RUSSIA_SHADOW_API_HASH=xyz789...
   RUSSIA_SHADOW_PHONE=+0987654321

   # Shadow sync settings
   SHADOW_SYNC_INTERVAL=300      # 5 minutes
   SHADOW_JOIN_RATE_LIMIT=5      # 5 joins per minute
   HEALTH_CHECK_INTERVAL=60      # 1 minute
   FAILOVER_AUTO_SWITCH=true     # Enable automatic failover
   ```

3. **Authenticate Shadow Account**

   ```bash
   python scripts/telegram_auth.py --account russia-shadow
   # Enter phone number and verification code when prompted
   ```

4. **Register Account in Database**

   ```bash
   python scripts/dr_failover.py \
       --add-account russia-shadow \
       --cluster russia \
       --role standby \
       --priority 0
   ```

5. **Start Shadow Sync Service**

   ```bash
   docker-compose --profile dr up -d shadow-sync
   ```

6. **Verify Sync is Working**

   ```bash
   # Wait 5-10 minutes for first sync
   python scripts/dr_status.py

   # Check logs
   docker-compose logs -f shadow-sync
   ```

### Best Practices for Shadow Accounts

1. **Use Different Phone Numbers**: Shadow accounts should use different phone numbers than primary accounts. If Telegram bans by phone number pattern, this provides isolation.

2. **Different Devices/IPs**: Ideally, authenticate shadow accounts from different networks to avoid pattern detection.

3. **Minimal Activity**: Shadow accounts only join/leave channels. They never read messages or download media, minimizing ban risk.

4. **Regular Health Checks**: The shadow-sync service checks health every 60 seconds. Monitor the `TelegramHealthCheckStale` alert.

5. **Test Failover Periodically**: Run `--dry-run` failover monthly to ensure shadow accounts are healthy.

---

## Backup Management

### Manual Backup

```bash
# Create immediate backup
python scripts/export_channel_state.py --type manual --triggered-by "operator:rick"

# Create backup with local copy
python scripts/export_channel_state.py --output /tmp/channel_backup.json
```

### Backup Schedule

Backups run automatically via cron (if configured). Default: daily at midnight.

```bash
# Example cron entry
0 0 * * * cd /app && python scripts/export_channel_state.py --type scheduled --triggered-by cron
```

### Backup Retention

Old backups are automatically deleted after `BACKUP_RETENTION_DAYS` (default: 90).

### Verify Backups

```bash
# List recent backups
python scripts/dr_restore.py --list

# Download and inspect
python scripts/dr_restore.py --from-minio --latest --dry-run
```

---

## Troubleshooting

### Shadow Sync Not Running

```bash
# Check if service is running
docker-compose --profile dr ps

# Check logs
docker-compose --profile dr logs shadow-sync

# Common issues:
# - DR_ENABLED=false (must be true)
# - Missing shadow account credentials
# - Session file not found (run telegram_auth.py)
```

### Channels Failing to Sync

```bash
# Check failed channels
python scripts/dr_status.py --json | jq '.sync_status.failed_details'

# Common causes:
# - Private channel (needs invite_link)
# - Channel deleted
# - Shadow account banned from specific channel
# - Rate limiting (FloodWait)
```

### Health Checks Failing

```bash
# Check health monitor logs
docker-compose logs shadow-sync | grep -i "health"

# Possible causes:
# - Network issues
# - Telegram API down
# - Session expired (re-authenticate)
```

### Failover Not Triggering

```bash
# Check if auto-switch is enabled
grep FAILOVER_AUTO_SWITCH .env

# Check failover conditions
python scripts/dr_failover.py --cluster russia --dry-run

# Requirements for auto-failover:
# 1. FAILOVER_AUTO_SWITCH=true
# 2. Shadow account exists with role=standby
# 3. Shadow account health_status=healthy
# 4. Primary account detected as banned
```

---

## Database Tables

| Table | Purpose |
|-------|---------|
| `telegram_accounts` | Registry of all accounts (primary + shadow) |
| `shadow_account_state` | Tracks which channels each shadow has joined |
| `dr_backup_history` | Audit trail of backup operations |
| `dr_failover_history` | Audit trail of failover events |

### Useful Queries

```sql
-- Check account status
SELECT name, cluster, role, health_status, channels_synced, last_sync_at
FROM telegram_accounts ORDER BY cluster, role;

-- Check sync failures
SELECT sas.*, c.username, c.name
FROM shadow_account_state sas
JOIN channels c ON c.id = sas.channel_id
WHERE sas.sync_status = 'failed';

-- Recent failovers
SELECT * FROM dr_failover_history ORDER BY initiated_at DESC LIMIT 5;

-- Recent backups
SELECT * FROM dr_backup_history ORDER BY started_at DESC LIMIT 5;
```

---

## Environment Variables Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `DR_ENABLED` | `false` | Enable disaster recovery features |
| `SHADOW_SYNC_INTERVAL` | `300` | Seconds between sync runs |
| `SHADOW_JOIN_RATE_LIMIT` | `5` | Max channel joins per minute |
| `HEALTH_CHECK_INTERVAL` | `60` | Seconds between health checks |
| `FAILOVER_AUTO_SWITCH` | `false` | Auto-promote shadow on ban |
| `BACKUP_ENABLED` | `true` | Enable backup system |
| `BACKUP_BUCKET` | `osint-backups` | MinIO bucket for backups |
| `BACKUP_RETENTION_DAYS` | `90` | Days to keep old backups |

---

## Related Documentation

- [Telegram Setup Guide](telegram-setup.md) - Initial Telegram configuration
- [Backup & Restore](backup-restore.md) - General backup procedures
- [Monitoring Guide](monitoring.md) - Prometheus/Grafana setup
- [Troubleshooting Guide](troubleshooting.md) - General troubleshooting

---

*Last updated: 2025-12-20*
