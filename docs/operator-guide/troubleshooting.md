# Troubleshooting Guide

**OSINT Intelligence Platform - Common Issues and Solutions**

Comprehensive troubleshooting guide based on 3+ years of production operations. Solutions are categorized by issue type with copy-pasteable diagnostic commands.

---

## Table of Contents

- [Overview](#overview)
- [Quick Diagnostics](#quick-diagnostics)
- [Service Startup Issues](#service-startup-issues)
- [Database Issues](#database-issues)
- [Telegram Issues](#telegram-issues)
- [Message Processing Issues](#message-processing-issues)
- [Performance Issues](#performance-issues)
- [Storage Issues](#storage-issues)
- [LLM and AI Issues](#llm-and-ai-issues) (including [LLM gateway problems](#llm-gateway-problems))
- [Search Issues](#search-issues)
- [Geolocation & Map Issues](#geolocation-map-issues)
- [Network and Connectivity Issues](#network-and-connectivity-issues)
- [Getting Help](#getting-help)

---

## Overview

**Key Resources:**

- **Logs**: `docker compose logs -f SERVICE`, or Grafana Explore with Loki (`monitoring` profile)
- **Metrics**: Grafana (`GRAFANA_PORT`, 3001 with `.env.example`) for performance dashboards
- **Alerts**: Prometheus (http://localhost:9090/alerts) for active alerts
- **Health checks**: `./scripts/platform-health-check.py` (smoke test; `--full` for functional checks) and `./scripts/preflight-check.sh`
- **Pipeline view**: `./scripts/pipeline-status.py`

**Troubleshooting Philosophy:**

1. **Check logs first**: 90% of issues are visible in logs
2. **Verify metrics**: Performance issues show in Grafana dashboards
3. **Check dependencies**: Services often fail due to dependency issues (database, Redis, network)
4. **Isolate the issue**: Restart one service at a time to identify the problem
5. **Document findings**: Update this guide with new solutions

---

## Quick Diagnostics

### Health Check Script

```bash
# Quick smoke test of the components enabled in .env
./scripts/platform-health-check.py

# Full functional verification (add --ci for JSON output)
./scripts/platform-health-check.py --full

# Configuration and readiness checks (secrets, ports, storage, backups)
./scripts/preflight-check.sh
```

### Quick Service Status

```bash
# Check all services
docker compose ps

# Expected: All services "Up" with "(healthy)" where applicable
# Unhealthy or "Exit 1" indicates a problem

# Check specific service
docker compose ps listener
docker compose ps processor-worker
docker compose ps postgres
```

### View Recent Logs

```bash
# Last 50 lines of all services
docker compose logs --tail=50

# Last 100 lines of specific service
docker compose logs --tail=100 processor-worker

# Follow logs in real-time
docker compose logs -f processor-worker

# Search logs for errors
docker compose logs processor-worker | grep -i error
docker compose logs processor-worker | grep -i exception
```

### Check Resource Usage

```bash
# Container resource usage (CPU, memory)
docker stats

# Disk space
df -h
docker system df

# Database size
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT pg_size_pretty(pg_database_size('osint_platform'));
"

# MinIO storage usage
./bin/mc du production/osint-media
```

---

## Service Startup Issues

### Issue: Container Exits Immediately

**Symptoms:**

```bash
docker compose ps
# Shows: osint-listener    Exit 1
```

**Diagnosis:**

```bash
# Check exit logs
docker compose logs --tail=50 listener

# Common errors:
# - "Configuration error: ..."
# - "Database connection refused"
# - "Permission denied: /app/sessions"
```

**Solutions:**

**A. Missing environment variables:**

```bash
# Check .env file exists
ls -la .env

# Verify required variables
grep -E "POSTGRES_|REDIS_|TELEGRAM_" .env

# Compare with .env.example
diff .env .env.example
```

**Fix**: Copy missing variables from `.env.example` to `.env`

**B. Database not ready:**

```bash
# Check PostgreSQL is running
docker compose ps postgres
# Should show: Up (healthy)

# If not healthy, check postgres logs
docker compose logs postgres

# Wait for PostgreSQL to be ready
docker compose up -d postgres
sleep 30  # Wait 30 seconds
docker compose up -d listener
```

**C. Permission errors:**

```bash
# Check telegram_sessions directory permissions
ls -la telegram_sessions/

# Should be writable by container user
# Fix permissions:
chmod 755 telegram_sessions/
chmod 644 telegram_sessions/*.session*
```

**D. Configuration syntax errors:**

```bash
# Validate docker-compose.yml
docker compose config

# Should output formatted YAML with no errors
# If errors, check line numbers in error message
```

### Issue: Service Crashes After Starting

**Symptoms:**

```bash
docker compose ps
# Shows service "Restarting" continuously
```

**Diagnosis:**

```bash
# Watch logs during restart
docker compose logs -f processor-worker

# Look for repeating error patterns
# Common: Out of memory, database connection pool exhausted
```

**Solutions:**

**A. Out of memory (OOM killed):**

```bash
# Check for OOM in logs
docker compose logs processor-worker | grep -i "killed"

# Check memory usage
docker stats processor-worker

# Fix: Increase container memory limit in docker-compose.yml
services:
  processor-worker:
    deploy:
      resources:
        limits:
          memory: 4G  # Increase from 2G
```

**B. Database connection pool exhausted:**

```bash
# Check active connections
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT count(*) FROM pg_stat_activity;
"

# If near max_connections (200 by default), check for connection leaks
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT application_name, state, count(*)
  FROM pg_stat_activity
  GROUP BY application_name, state;
"

# Fix: Restart service to release connections
docker compose restart processor-worker
```

### Issue: Healthcheck Failing

**Symptoms:**

```bash
docker compose ps postgres
# Shows: Up (unhealthy)
```

**Diagnosis:**

```bash
# Check healthcheck command manually
docker compose exec postgres pg_isready -U osint_user -d osint_platform

# Should output: osint_platform:5432 - accepting connections
# If not, database is not ready
```

**Solutions:**

```bash
# Wait longer (database may be recovering)
sleep 60
docker compose ps postgres

# If still unhealthy, restart service
docker compose restart postgres

# Check logs for initialization errors
docker compose logs postgres | grep -i error
```

---

## Database Issues

### Issue: "Connection Refused" Errors

**Symptoms:**

```bash
# Processor logs show:
# Error: could not connect to server: Connection refused
#   Is the server running on host "postgres" (xxx.xxx.xxx.xxx) and accepting TCP/IP connections on port 5432?
```

**Diagnosis:**

```bash
# Check PostgreSQL is running
docker compose ps postgres
# Should show: Up (healthy)

# Check network connectivity
docker compose exec processor ping postgres
# Should respond (Ctrl+C to stop)

# Check PostgreSQL is listening
docker compose exec postgres netstat -tlnp | grep 5432
# Should show: LISTEN on port 5432
```

**Solutions:**

**A. PostgreSQL not started:**

```bash
docker compose up -d postgres
sleep 30  # Wait for PostgreSQL to be ready
docker compose restart processor-worker
```

**B. Network misconfiguration:**

```bash
# Verify services are on same network
docker network ls
docker network inspect osint-intelligence-platform_backend

# Ensure postgres and processor-worker are both listed

# If not, recreate network
docker compose down
docker compose up -d
```

**C. Firewall blocking connection:**

```bash
# Check if host firewall is blocking Docker networks
sudo iptables -L | grep -i docker

# Temporarily disable firewall (for testing only!)
sudo ufw disable

# If this fixes it, add Docker network to firewall allow list
```

### Issue: "Too Many Connections" Error

**Symptoms:**

```bash
# Logs show:
# FATAL: remaining connection slots are reserved for non-replication superuser connections
# FATAL: sorry, too many clients already
```

**Diagnosis:**

```bash
# Count active connections
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT count(*) FROM pg_stat_activity;
"

# Check max_connections setting
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SHOW max_connections;
"

# Identify connection leakers
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT application_name, state, count(*)
  FROM pg_stat_activity
  GROUP BY application_name, state
  ORDER BY count(*) DESC;
"
```

**Solutions:**

**A. Increase max_connections (temporary fix):**

```bash
# Edit infrastructure/postgres/postgresql.conf (default is already 200)
max_connections = 300

# Restart PostgreSQL
docker compose restart postgres

# Better first: every service replica holds up to POSTGRES_POOL_SIZE +
# POSTGRES_MAX_OVERFLOW connections (40 + 40 by default). Lower the pool or the
# replica count (see Performance Tuning)
```

**B. Fix connection leaks (permanent fix):**

Services use SQLAlchemy sessions from `shared/python/models/base.py`. A leak shows up as many `idle in transaction` rows for one `application_name`; fix the code path to close its session (use the session context manager) and restart that service.

**C. Kill idle connections:**

```bash
# Kill connections idle for >10 minutes
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT pg_terminate_backend(pid)
  FROM pg_stat_activity
  WHERE state = 'idle'
    AND state_change < now() - interval '10 minutes'
    AND pid <> pg_backend_pid();
"
```

### Issue: Slow Query Performance

**Symptoms:**

- API responses slow (>2 seconds)
- High CPU usage on PostgreSQL container
- Message processing rate drops

**Diagnosis:**

```bash
# Find slow queries (>100ms)
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT calls, mean_exec_time, query
  FROM pg_stat_statements
  ORDER BY mean_exec_time DESC
  LIMIT 10;
"

# Check for missing indexes
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT schemaname, tablename, indexname
  FROM pg_indexes
  WHERE tablename = 'messages';
"

# Check table bloat (needs VACUUM)
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
  FROM pg_tables
  WHERE schemaname = 'public'
  ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
  LIMIT 10;
"
```

**Solutions:**

**A. Check the indexes:**

The `messages` indexes ship in `init.sql` and the numbered migrations (for example `idx_messages_telegram_date`, `idx_messages_channel_date`, `idx_messages_importance_level`, `idx_messages_is_spam`). Run `./scripts/migrate.sh --status` first: a missing index usually means a pending migration. If you still need a new one, add it through a migration (see [Database Migrations](../developer-guide/database-migrations.md)); the date column is `telegram_date` and the spam flag is `is_spam`.

```sql
-- Example shape of an index created CONCURRENTLY
CREATE INDEX CONCURRENTLY idx_messages_example ON messages(channel_id, telegram_date DESC);
```

**B. Run VACUUM to reduce bloat:**

```bash
# Analyze and vacuum (can run online, safe for production)
docker compose exec postgres psql -U osint_user -d osint_platform -c "VACUUM ANALYZE messages;"

# For heavily bloated tables, use VACUUM FULL (requires table lock, downtime!)
# Only run during maintenance window
docker compose exec postgres psql -U osint_user -d osint_platform -c "VACUUM FULL messages;"
```

**C. Tune PostgreSQL configuration:**

```bash
# Memory settings come from .env (compose passes them on the postgres command line)
POSTGRES_SHARED_BUFFERS=2GB     # raise if cache hit ratio <90%
POSTGRES_WORK_MEM=128MB         # raise for complex queries that spill to disk
POSTGRES_EFFECTIVE_CACHE=8GB

docker compose up -d postgres
```

See [Performance Tuning](performance-tuning.md#postgresql-tuning).

---

## Telegram Issues

### Issue: FloodWaitError

**Symptoms:**

```bash
# Listener logs show:
# telethon.errors.rpcerrorlist.FloodWaitError: A wait of 300 seconds is required (caused by GetHistoryRequest)
```

**Diagnosis:**

```bash
# Check listener logs for flood wait errors
docker compose logs listener | grep -i floodwait

# Count occurrences
docker compose logs listener | grep -c FloodWaitError

# If frequent (>10/hour), Telegram API is rate limiting
```

**Solutions:**

**A. Automatic retry (already implemented):**

The listener automatically waits for the required time and retries. No action needed. Wait for the specified seconds.

**B. Reduce request rate:**

The listener is event driven (live updates), so FloodWait mostly comes from backfill, folder sync and joins. Lower the backfill pressure instead of polling intervals:

```bash
# .env
BACKFILL_BATCH_SIZE=50          # default 100
FOLDER_SYNC_INTERVAL=300        # seconds, minimum 30

docker compose up -d listener
```

**C. Multi-account strategy (if persistent):**

Run one listener per Telegram account with the `multi-account` profile (`listener-russia`, `listener-ukraine`; each handles the channels whose `SOURCE_ACCOUNT` matches). Authenticate each account with `python3 scripts/telegram_auth.py --account NAME`. Details in [Telegram Setup](telegram-setup.md).

```bash
./scripts/switch-telegram-mode.sh multi    # TELEGRAM_MODE=multi; checks prerequisites, then starts the per-account services
```

### Issue: Session Expired / Auth Required

**Symptoms:**

```bash
# Listener logs show:
# telethon.errors.rpcerrorlist.UnauthorizedError: The authorization key has expired
# telethon.errors.rpcerrorlist.SessionPasswordNeededError
```

**Diagnosis:**

```bash
# Check if session file exists
ls -la telegram_sessions/*.session

# If missing or 0 bytes, session needs re-authentication
```

**Solutions:**

**A. Re-authenticate (interactive):**

```bash
# Stop listener
docker compose stop listener

# Run interactive auth (on the host, from the repository root)
python3 scripts/telegram_auth.py            # add --account NAME for named accounts

# Follow prompts:
# - Enter phone number (with country code: +1234567890)
# - Enter verification code sent to Telegram app
# - Enter 2FA password (if enabled)

# Verify session created
ls -la telegram_sessions/listener.session
# Should be >1KB

# Start listener
docker compose up -d listener

# Verify authentication
docker compose logs -f listener
```

**B. Restore from backup:**

```bash
# If you have backup session files (see backup-restore.md)
tar -xzf backups/sessions/sessions_latest.tar.gz   # contains telegram_sessions/
chmod 600 telegram_sessions/*.session*

docker compose restart listener
```

### Issue: Channel Access Denied

**Symptoms:**

```bash
# Listener logs show:
# telethon.errors.rpcerrorlist.ChannelPrivateError: The channel specified is private and you lack permission to access it
```

**Diagnosis:**

```bash
# Channel may have:
# 1. Gone private (requires invite link)
# 2. Banned the account
# 3. Requires phone number verification

# Check channel status manually via Telegram app
```

**Solutions:**

**A. Rejoin the channel**

Channels are managed through Telegram folders. Join the channel again in the Telegram app with the monitoring account (invite link if it went private); the next folder sync (`FOLDER_SYNC_INTERVAL`) picks it up. Sessions are per account, so use the account that owns the folder.

**B. Remove inaccessible channel from monitoring:**

Remove it from the Telegram folder, or mark it inactive:

```bash
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  UPDATE channels SET active = false WHERE telegram_id = 1234567890;
"
```

---

## Message Processing Issues

### Issue: Messages Not Appearing in Database

**Symptoms:**

- Telegram messages are sent to channels
- But not appearing in database/API

**Diagnosis:**

```bash
# 1. Check listener is receiving messages
docker compose logs listener --since 10m | tail -20
# Should show recent activity (queued messages, no errors)

# 2. Check Redis queue has messages
docker compose exec redis redis-cli XLEN telegram:messages:realtime
# Should show count > 0

# 3. Check processor is consuming from queue
docker compose logs processor-worker | grep "Processing message"
# Should show recent processing

# 4. Check database for recent messages
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT COUNT(*) FROM messages WHERE created_at > NOW() - INTERVAL '1 hour';
"
# Should match messages received by listener
```

**Solutions:**

**A. Listener not forwarding to queue:**

```bash
# Check listener logs for errors
docker compose logs listener | grep -i error

# Common issues:
# - Redis connection failed
# - Message size exceeds limit
# - Spam filter blocking all messages

# Verify Redis connectivity
docker compose exec redis redis-cli ping
# Should respond: PONG

# Spam is decided in the processor, not the listener: check the spam rate there
# (Grafana: spam rate; osint:spam_rate:5m). If >95%, the spam filter may be misconfigured
```

**B. Processor not consuming from queue:**

```bash
# Check processor is running
docker compose ps processor-worker
# Should show: Up

# Check processor logs for errors
docker compose logs processor-worker | grep -i error

# Common errors:
# - Database connection failed
# - LLM timeout (gateway or Ollama)
# - MinIO media upload failed

# Restart processor
docker compose restart processor-worker
```

**C. Queue backlog:**

```bash
# Check queue depth
docker compose exec redis redis-cli XLEN telegram:messages:realtime

# If >10,000, processor can't keep up
# Scale up processor workers
docker compose up -d --scale processor-worker=4   # persist: PROCESSOR_WORKER_REPLICAS=4 in .env

# Monitor queue drain
watch 'docker compose exec redis redis-cli XLEN telegram:messages:realtime'
```

### Issue: High Message Skip Rate (>90%)

**Symptoms:**

- Grafana shows >90% messages skipped
- Very few messages in database despite high volume

**Diagnosis:**

```bash
# Check classification metrics
docker compose exec processor-worker curl -s http://localhost:8002/metrics | grep osint_messages_skipped

# Check importance level distribution
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT
    importance_level,
    COUNT(*) as count
  FROM messages
  WHERE created_at > NOW() - INTERVAL '24 hours'
  GROUP BY importance_level
  ORDER BY
    CASE importance_level
      WHEN 'high' THEN 1
      WHEN 'medium' THEN 2
      WHEN 'low' THEN 3
      ELSE 4
    END;
"
```

**Solutions:**

**A. Minimum importance too high:**

The archive/skip decision uses `monitoring_min_importance` (`low`, `medium` or `high`, default `medium`), a runtime setting in **Admin, Config, Platform Settings** (env fallback `MONITORING_MIN_IMPORTANCE`). Lowering it archives more messages. The change applies within about 30 seconds, no restart.

**B. Spam filter too aggressive:**

Messages are skipped as spam by the embedding filter and the LLM. Check the spam settings and recent spam rate:

```bash
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT is_spam, spam_type, COUNT(*) FROM messages
  WHERE created_at > NOW() - INTERVAL '24 hours' GROUP BY 1, 2 ORDER BY 3 DESC;
"
```

Relevant runtime settings: `embedding_spam_threshold`, `llm_prefilter_spam_similarity`, `realtime_use_embedding`. Review false positives in the admin spam queue; see [Backlog Re-classification](backlog-reclassification.md) to re-check spam with the LLM.

**C. LLM scoring too conservative:**

```bash
# Check LLM prompt versions (the active one has is_active = true)
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT id, version, is_active FROM llm_prompts
   WHERE task = 'message_classification' AND prompt_type = 'system' ORDER BY version DESC LIMIT 5;
"

# An older prompt may be more conservative. To switch versions (activate / roll back),
# see "Prompt versions" in LLM Gateway and Providers (operator-guide/external-llm.md).
# Prompt history: docs/architecture/LLM_PROMPTS.md in the platform repo.

# Prompt changes are read from the database; restart the processor if a change does not apply
docker compose restart processor-worker
```

---

## Performance Issues

### Issue: High CPU Usage

**Symptoms:**

```bash
docker stats
# Shows: processor-worker using >90% CPU continuously
```

**Diagnosis:**

```bash
# Identify which process is consuming CPU
docker compose exec processor-worker top

# Check for CPU-intensive tasks:
# - LLM inference (when LLM_PROVIDER=ollama)
# - Entity extraction and matching
# - Embedding generation (sentence-transformers, CPU)

# Check Grafana → cAdvisor Dashboard → Container CPU Usage
# Identify which container(s) are CPU-bound
```

**Solutions:**

**A. LLM inference bottleneck:**

```bash
# Check LLM response time (processor metrics)
docker compose exec processor-worker curl -s localhost:8002/metrics | grep osint_llm_response_duration

# If >15s per message, LLM is bottleneck
# Options:
# 1. Move to the LLM gateway (see LLM Gateway and Providers) - best option
# 2. Skip the LLM for obvious spam (admin setting llm_prefilter_spam_similarity)
# 3. Use a faster model (Ollama only)

# Option 3: Switch to a faster model
docker compose exec ollama ollama pull <smaller model>  # Faster, less accurate
# Update .env:
OLLAMA_MODEL=<smaller model>
```

**B. Too many concurrent workers:**

```bash
# Check how many processor workers are running
docker compose ps | grep processor-worker

# If >4 on single CPU, workers compete for CPU
# Scale down:
docker compose up -d --scale processor-worker=2

# Monitor CPU usage
docker stats
```

**C. Database query optimization:**

```bash
# Check for slow queries consuming CPU
# See "Slow Query Performance" section above

# Check cache hit ratio
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT sum(blks_hit)::float / (sum(blks_hit) + sum(blks_read)) AS cache_hit_ratio
  FROM pg_stat_database;
"

# If <0.9, increase shared_buffers (set in .env, passed to postgres by compose)
POSTGRES_SHARED_BUFFERS=2GB

docker compose up -d postgres
```

### Issue: High Memory Usage / OOM Kills

**Symptoms:**

```bash
docker stats
# Shows: Container using >90% of memory limit

docker compose logs processor-worker | grep -i killed
# Shows: "Killed" (OOM killer)
```

**Diagnosis:**

```bash
# Check container memory limits
docker compose config | grep -A 5 "processor-worker" | grep memory

# Check actual memory usage
docker stats --no-stream processor-worker

# Identify memory-intensive processes
docker compose exec processor-worker ps aux --sort=-%mem | head -10
```

**Solutions:**

**A. Increase container memory limit:**

```yaml
# docker-compose.yml
services:
  processor-worker:
    deploy:
      resources:
        limits:
          memory: 4G  # Increase from 2G
        reservations:
          memory: 2G

# Restart service
docker compose up -d processor-worker
```

**B. Reduce concurrency and batch sizes:**

Memory grows with concurrent work. Lower `PROCESSOR_CONCURRENCY`, `PROCESSOR_BATCH_SIZE`, the replica counts (`PROCESSOR_WORKER_REPLICAS`, `ENRICHMENT_WORKER_REPLICAS`) or the enrichment window settings (`ai_tagging_window_days`, `enrichment_llm_concurrency` in the admin settings), then recreate the service. If one service grows without bound, capture `docker stats` over time and open an issue with the service log.

---

## Storage Issues

### Issue: Disk Space Full

**Symptoms:**

```bash
df -h
# Shows: /dev/sda1  95%  (or 100%)

docker compose logs postgres
# Shows: "FATAL: could not write to file ... No space left on device"
```

**Diagnosis:**

```bash
# Check overall disk usage
df -h

# Check Docker disk usage
docker system df

# Identify largest directories
du -sh /* | sort -h | tail -10

# Check database size
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT pg_size_pretty(pg_database_size('osint_platform'));
"

# Check MinIO bucket size (mc alias as in Backup & Restore)
./bin/mc du production/osint-media
```

**Solutions:**

**A. Clean up Docker resources:**

```bash
# Remove unused containers, images, networks
docker system prune -a

# Remove unused volumes (CAREFUL: may delete data!)
docker volume prune

# Remove old Docker images
docker images | grep "<none>" | awk '{print $3}' | xargs docker rmi
```

**B. Clean up old logs:**

```bash
# Check log file sizes
find /var/lib/docker/containers -name "*-json.log" -exec ls -lh {} \;

# Truncate large log files
truncate -s 0 /var/lib/docker/containers/*/*-json.log

# Configure log rotation in docker-compose.yml:
services:
  processor-worker:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

**C. Database growth:**

```sql
-- Move old messages to archive table
-- The platform is an archive: avoid deleting messages. If you must, export first
-- (pg_dump -t messages, see Backup & Restore) and expect many dependent rows
-- (entities, media links, embeddings) to be affected by cascades.
SELECT pg_size_pretty(pg_total_relation_size('messages'));
```

**D. Prune old MinIO media (with caution):**

```bash
# Delete media older than 365 days (CAREFUL!)
./bin/mc rm --recursive --older-than 365d production/osint-media/media/

# Verify only old files deleted
./bin/mc ls production/osint-media/media/ | head -20
```

### Issue: MinIO Connection Errors

**Symptoms:**

```bash
# Processor logs show:
# Error uploading media to MinIO: Connection refused
# Error: S3 error: Unable to connect to endpoint
```

**Diagnosis:**

```bash
# Check MinIO is running
docker compose ps minio
# Should show: Up (healthy)

# Check MinIO health
curl http://localhost:9000/minio/health/live
# Should return HTTP 200

# Test S3 API
./bin/mc alias set test http://localhost:9000 "$MINIO_ROOT_USER" "$MINIO_ROOT_PASSWORD"
./bin/mc ls test/
# Should list buckets
```

**Solutions:**

**A. MinIO not started:**

```bash
docker compose up -d minio
sleep 10  # Wait for MinIO to be ready
docker compose restart processor-worker
```

**B. Bucket not created:**

```bash
# Check if bucket exists
./bin/mc ls production/
# Should show the bucket from MINIO_BUCKET_NAME (compose default: osint-media)

# If missing, create bucket (minio-init normally does this at startup)
./bin/mc mb production/osint-media
./bin/mc anonymous set download production/osint-media
```

**C. MinIO credentials incorrect:**

```bash
# Verify .env has correct credentials
grep MINIO .env

# The processor, media-worker and API read the same MINIO_* values as the minio service.
# After changing credentials in .env, recreate everything that uses them:
docker compose up -d --force-recreate minio minio-init processor-worker media-worker api
```

---

## LLM and AI Issues

### LLM gateway problems

Applies when `LLM_PROVIDER=openai`. Setup and background: [LLM Gateway and Providers](external-llm.md).

**Gateway unreachable (timeouts, connection errors, name not resolved).**

```bash
docker compose logs --since 15m processor-worker | grep -iE 'gateway|connect|resolve|timeout'
```

- Check `LLM_GATEWAY_URL` in `.env` (do not paste it into tickets or chats).
- If the gateway's hostname resolves on the host but not in containers (for example a private-network name), set `LLM_GATEWAY_HOST` and `LLM_GATEWAY_IP`. Compose pins them with `extra_hosts` for the processor, backlog worker, fact-check and RSS validation. Recreate the services afterwards: `docker compose up -d --force-recreate processor-worker backlog-worker enrichment-fact-check enrichment-rss-validation`.
- While the gateway is down, messages are still archived by the embedding filter; topic and importance stay `pending` until the [backlog worker](backlog-reclassification.md) runs.

**A lane is paused (401 or 403 in the logs).**

A 401 means the lane's key is blocked; a 403 means the key is not allowed to use that model alias. The platform stops sending requests on that lane for 5 minutes, then probes once.

```bash
docker compose logs --since 30m processor-worker backlog-worker enrichment-fact-check enrichment-rss-validation | grep -iE 'paused|401|403'
```

Check the lane's key and alias in `.env` (`LLM_GATEWAY_API_KEY` and `LLM_GATEWAY_MODEL` for realtime, `LLM_GATEWAY_AUDIT_API_KEY` and `FACT_CHECK_MODEL` for audit, `LLM_GATEWAY_BATCH_API_KEY`, `LLM_GATEWAY_BATCH_MODEL` and `RSS_VALIDATION_MODEL` for batch), fix the key or alias on the gateway, then recreate the service that reads it.

**429 responses.** The lane is at its parallel or rate cap. Calls are retried with backoff. If it persists, lower `backlog_concurrency` (admin settings) or the fact-check pacing (`FACT_CHECK_TIME_BUDGET_SECONDS`, `FACT_CHECK_CYCLE_INTERVAL_SECONDS`).

**Fact-check returns empty or truncated answers.** The audit alias is a reasoning model; keep `FACT_CHECK_NUM_PREDICT` at 4096 or higher.

**Messages stuck or processing stalls.** The processor cancels any single message that runs longer than `PROCESSOR_MESSAGE_TIMEOUT_S` (default 600 seconds) and logs where it was waiting, then continues with the next message. Look for the timeout line:

```bash
docker compose logs --since 1h processor-worker | grep -i 'timed out after'
```

Repeated timeouts at the same place usually mean a slow Telegram media download (enable the [media lane](media-lanes.md)) or a hung LLM call (check the gateway as above).

### Issue: Ollama Service Not Responding

Applies when `LLM_PROVIDER=ollama`, and to the local `ollama` service that serves search-query embeddings. For a gateway, see "LLM gateway problems" above.

**Symptoms:**

```bash
# Processor logs show:
# Error: LLM request timeout after 30 seconds
# Error connecting to Ollama: Connection refused
```

**Diagnosis:**

```bash
# Check Ollama is running
docker compose ps ollama
# Should show: Up

# Check Ollama API
curl http://localhost:11434/api/tags
# Should return: {"models": [...]}

# Check Ollama logs
docker compose logs ollama | tail -50
```

**Solutions:**

**A. Ollama not started:**

```bash
docker compose up -d ollama
sleep 30  # Wait for Ollama to load models
docker compose restart processor-worker
```

**B. Model not loaded:**

```bash
# List available models
docker compose exec ollama ollama list

# If empty or missing your OLLAMA_MODEL, pull it
docker compose exec ollama ollama pull <your OLLAMA_MODEL>

# Verify model loaded
docker compose exec ollama ollama list
# Should show: your OLLAMA_MODEL
```

**C. Ollama overloaded:**

```bash
# Check Ollama resource usage
docker stats ollama

# If CPU >90% or memory >90%, Ollama is bottleneck
# Options:
# 1. Use faster model (smaller parameters)
# 2. Scale horizontally (dedicated Ollama instance)
# 3. Increase the timeout (LLM_TIMEOUT in .env, see .env.example)

# Option 1: Use faster model
docker compose exec ollama ollama pull <smaller model>
# Update .env:
OLLAMA_MODEL=<smaller model>

# Option 3: set LLM_TIMEOUT=60 in .env, then
docker compose up -d processor-worker
```

### Issue: Low LLM Success Rate (<90%)

**Symptoms:**

- Grafana → AI/ML Processing → LLM Success Rate shows <90%
- Many messages have NULL importance_level

**Diagnosis:**

```bash
# Check LLM error rate
docker compose logs processor-worker | grep -i "llm" | grep -i "error\|fail" | tail -20

# Common errors:
# - "Model not found"
# - "Context length exceeded"
# - "Timeout"
# - "Invalid JSON response"
```

**Solutions:**

**A. Context length exceeded:**

```bash
# Error: "Context length exceeded (max: 2048 tokens)"

# Truncate message content before sending to LLM
# Edit services/processor/src/llm_classifier.py:

def truncate_for_llm(text: str, max_chars: int = 2000) -> str:
    if len(text) <= max_chars:
        return text
    return text[:max_chars] + "..."
```

**B. Invalid JSON response:**

```bash
# Error: "Failed to parse LLM response as JSON"

# LLM may not be following prompt format
# Check which prompt version is active:
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT id, version, is_active FROM llm_prompts
   WHERE task = 'message_classification' AND is_active = true;
"

# Prompt v12 expects a large model with structured output; with a small Ollama model use an
# older version. Switching: see "Prompt versions" in LLM Gateway and Providers.
# Prompt design: docs/architecture/LLM_PROMPTS.md in the platform repo.

docker compose restart processor-worker
```

---

## Search Issues

### Issue: Search Returns No Results

**Symptoms:**

- API search endpoint returns empty results
- Frontend search shows "No messages found"

**Diagnosis:**

```bash
# Test search directly via PostgreSQL
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT COUNT(*) FROM messages WHERE search_vector @@ websearch_to_tsquery('english', 'Bakhmut');
"

# If 0, search_vector may not be populated
# Check if search_vector column exists
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT column_name FROM information_schema.columns
  WHERE table_name = 'messages' AND column_name = 'search_vector';
"
```

**Solutions:**

**A. search_vector column missing:**

```sql
-- Add search_vector column (from init.sql)
ALTER TABLE messages ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    to_tsvector('english', coalesce(content, ''))
  ) STORED;

CREATE INDEX idx_messages_search ON messages USING gin(search_vector);
```

**B. Search index not built:**

```sql
-- Rebuild search vectors
UPDATE messages SET content = content;  -- Triggers search_vector regeneration

-- Verify search works
SELECT COUNT(*) FROM messages WHERE search_vector @@ websearch_to_tsquery('english', 'war');
```

**C. Wrong search language:**

```sql
# If searching non-English content, use appropriate language
# Russian:
SELECT * FROM messages WHERE to_tsvector('russian', content) @@ websearch_to_tsquery('russian', 'война');

# Ukrainian:
SELECT * FROM messages WHERE to_tsvector('ukrainian', content) @@ websearch_to_tsquery('ukrainian', 'війна');
```

### Issue: Vector Search (Semantic Search) Not Working

**Symptoms:**

- API `/api/search/semantic` returns errors
- Frontend semantic search shows "Error: pgvector extension not found"

**Diagnosis:**

```bash
# Check if pgvector extension is installed
docker compose exec postgres psql -U osint_user -d osint_platform -c "
  SELECT * FROM pg_extension WHERE extname = 'vector';
"

# If empty, pgvector not installed
```

**Solutions:**

**A. Install pgvector extension:**

```sql
-- Connect as superuser
docker compose exec postgres psql -U osint_user -d osint_platform

-- Create extension
CREATE EXTENSION vector;

-- Verify installation
SELECT * FROM pg_extension WHERE extname = 'vector';
```

**B. Regenerate embeddings:**

```bash
# Embeddings may not be generated yet
# Check enrichment service logs
docker compose logs enrichment | grep "embedding"

# Manually trigger embedding generation for recent messages
docker compose exec enrichment python -c "
from tasks.embedding import EmbeddingTask
task = EmbeddingTask()
task.run()
"
```

---

## Geolocation & Map Issues

Issues related to the Event Detection V3 geolocation pipeline and map interface.

### Issue: High Unresolved Location Rate

**Symptoms:**
- Many messages with `extraction_method='unresolved'`
- Map showing fewer markers than expected

**Diagnosis:**

```sql
-- Check geolocation success rate by stage
SELECT extraction_method, COUNT(*) as total,
       ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) as percentage
FROM message_locations
WHERE extracted_at >= NOW() - INTERVAL '24 hours'
GROUP BY extraction_method
ORDER BY total DESC;

-- Expected: gazetteer >50%, unresolved <10%
```

**Solutions:**

1. **Add missing locations to gazetteer:**
```sql
-- Find most common unresolved locations
SELECT location_name, COUNT(*) as occurrences
FROM message_locations
WHERE extraction_method = 'unresolved'
GROUP BY location_name
ORDER BY occurrences DESC
LIMIT 20;

-- Add to gazetteer (get coords from GeoNames)
INSERT INTO location_gazetteer (name_primary, name_local, latitude, longitude, country_code)
VALUES ('MissingCity', 'Місто', 48.123, 37.456, 'UA');
```

2. **Check Nominatim availability:**
```bash
# Test Nominatim API
curl "https://nominatim.openstreetmap.org/search?q=Bakhmut&format=json"

# Check enrichment logs for rate limit errors
docker compose logs enrichment | grep -i nominatim
```

3. **Verify Cyrillic encoding:**
```bash
# Check for encoding issues in logs
docker compose logs enrichment | grep -i "encoding\|unicode"
```

---

### Issue: Cluster Detection Not Working

**Symptoms:**
- No new clusters appearing on map
- Messages not being grouped into events

**Diagnosis:**

```sql
-- Check cluster detection rate
SELECT tier, COUNT(*) as count
FROM telegram_event_clusters
WHERE detected_at >= NOW() - INTERVAL '24 hours'
  AND archived_at IS NULL
GROUP BY tier;

-- Check if messages have embeddings (required for clustering)
SELECT COUNT(*) as total,
       COUNT(content_embedding) as with_embedding
FROM messages
WHERE telegram_date >= NOW() - INTERVAL '24 hours';
```

**Solutions:**

1. **Verify embedding generation:**
```bash
# Check embedding task is running
docker compose logs enrichment | grep -i embedding

# Check embedding queue
docker compose exec redis redis-cli XLEN enrich:fast
```

2. **Adjust detection sensitivity:**
```bash
# Lower threshold for more clusters (in .env)
CLUSTER_VELOCITY_THRESHOLD=1.5  # default 2.0
CLUSTER_SIMILARITY_THRESHOLD=0.75  # default 0.80
MIN_MESSAGES_FOR_CLUSTER=2  # default 3

# Restart enrichment
docker compose restart enrichment
```

---

### Issue: Too Many False Positive Clusters

**Symptoms:**
- Clusters containing unrelated messages
- Low similarity scores in cluster_messages

**Diagnosis:**

```sql
-- Find clusters with low similarity
SELECT c.id, c.tier, AVG(cm.similarity) as avg_similarity
FROM telegram_event_clusters c
JOIN cluster_messages cm ON c.id = cm.cluster_id
WHERE c.detected_at >= NOW() - INTERVAL '7 days'
GROUP BY c.id, c.tier
HAVING AVG(cm.similarity) < 0.85
ORDER BY avg_similarity;
```

**Solutions:**

1. **Increase similarity threshold:**
```bash
# In .env
CLUSTER_SIMILARITY_THRESHOLD=0.85  # stricter matching
docker compose restart enrichment
```

2. **Archive false positives:**
```sql
-- Archive specific cluster
UPDATE telegram_event_clusters
SET archived_at = NOW(), archive_reason = 'false_positive'
WHERE id = 123;
```

---

### Issue: Map API Slow Responses

**Symptoms:**
- Map taking >2s to load markers
- Timeout errors in browser console

**Diagnosis:**

```bash
# Check API response time
time curl "http://localhost:8000/api/map/messages?south=48&west=35&north=50&east=40"

# Check Redis cache hit rate
docker compose exec redis redis-cli INFO stats | grep keyspace
```

**Solutions:**

1. **Verify spatial indexes:**
```sql
-- Check index exists
SELECT indexname FROM pg_indexes
WHERE tablename = 'message_locations'
  AND indexname LIKE '%bbox%';

-- Recreate if missing
CREATE INDEX IF NOT EXISTS idx_message_locations_bbox
ON message_locations (latitude, longitude)
WHERE latitude IS NOT NULL;

ANALYZE message_locations;
```

2. **Enable server-side clustering:**
```
# In frontend, always use cluster=true at zoom < 12
/api/map/messages?...&cluster=true&zoom=10
```

---

### Issue: WebSocket Not Receiving Updates

**Symptoms:**
- Map not showing new locations in real-time
- Connection indicator shows red/yellow

**Diagnosis:**

```bash
# Check Redis pub/sub
docker compose exec redis redis-cli PUBSUB CHANNELS

# Should show: map:new_location

# Check if geolocation task is publishing
docker compose logs enrichment | grep "map:new_location"
```

**Solutions:**

1. **Verify Redis pub/sub:**
```bash
# Subscribe to channel (should see messages)
docker compose exec redis redis-cli SUBSCRIBE map:new_location
```

2. **Check proxy WebSocket support:**
```nginx
# nginx.conf - ensure WebSocket upgrade headers
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_read_timeout 90s;
```

3. **Test WebSocket directly:**
```javascript
// In browser console
const ws = new WebSocket('ws://localhost:8000/api/map/ws/map/live?south=48&west=35&north=50&east=40');
ws.onmessage = (e) => console.log(JSON.parse(e.data));
```

---

## Network and Connectivity Issues

### Issue: Cannot Access Web Interfaces

**Symptoms:**

- http://localhost:3001 (Grafana) not loading
- http://localhost:8000 (API) not responding

**Diagnosis:**

```bash
# Check if services are running
docker compose ps

# Check if ports are mapped correctly
docker compose ps | grep "0.0.0.0"
# Should show port mappings like: 0.0.0.0:3001->3000/tcp

# Check if something else is using the port
sudo lsof -i :3001
sudo lsof -i :8000

# Test connectivity from inside container
docker compose exec api curl -s http://localhost:8000/health
```

**Solutions:**

**A. Port conflict:**

```bash
# If another service is using port, change mapping
# Edit docker-compose.yml:
services:
  grafana:
    ports:
      - "3002:3000"  # Change from 3001 to 3002

docker compose up -d grafana
```

**B. Firewall blocking access:**

```bash
# Check firewall rules
sudo ufw status

# Allow Docker ports
sudo ufw allow 8000/tcp
sudo ufw allow 3001/tcp
sudo ufw allow 9090/tcp

# Or disable firewall (for testing only!)
sudo ufw disable
```

**C. Docker network misconfiguration:**

```bash
# Recreate Docker networks
docker compose down
docker network prune
docker compose up -d
```

---

## Getting Help

### Before Asking for Help

**Gather diagnostic information:**

```bash
# 1. Platform version
git log -1 --oneline

# 2. Service status
docker compose ps > status.txt

# 3. Recent logs (all services)
docker compose logs --tail=100 > logs.txt

# 4. System resources
docker stats --no-stream > resources.txt
df -h > disk.txt

# 5. Configuration (sanitized)
cp .env .env.sanitized
# EDIT .env.sanitized: Remove actual passwords/secrets
# Replace with placeholders: POSTGRES_PASSWORD=<redacted>

# 6. Error messages
docker compose logs | grep -i error > errors.txt
```

### Where to Get Help

1. **Check this troubleshooting guide first**
2. **Review logs** with `docker compose logs` or Grafana Explore (Loki)
3. **Check metrics** via Grafana (http://localhost:3001)
4. **Search GitHub Issues**: https://github.com/osintukraine/osint-intelligence-platform/issues
5. **Create new issue** with diagnostic information above

### Creating a Good Bug Report

```markdown
**Environment:**
- OS: Ubuntu 22.04 / macOS 13 / Windows 11
- Docker version: (output of `docker --version`)
- Docker Compose version: (output of `docker compose --version`)
- Platform version: (output of `git log -1 --oneline`)

**Issue Description:**
Clear description of the problem.

**Steps to Reproduce:**
1. Start services: `docker compose up -d`
2. Wait 2 minutes
3. Check logs: `docker compose logs processor-worker`
4. See error: "..."

**Expected Behavior:**
What you expected to happen.

**Actual Behavior:**
What actually happened.

**Logs:**
```
(Paste relevant log excerpts, sanitize any secrets)
```

**Configuration:**
(Attach .env.sanitized, relevant docker-compose.yml sections)

**Additional Context:**
Any other relevant information.
```

---

## Additional Resources

### Documentation

- [PITFALLS_FROM_PRODUCTION.md](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docs/PITFALLS_FROM_PRODUCTION.md) - Lessons from 3+ years of production
- [ARCHITECTURE.md](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docs/ARCHITECTURE.md) - Technical deep dive
- [Monitoring Guide](monitoring.md) - Performance monitoring and debugging
- [Backup & Restore Guide](backup-restore.md) - Data recovery procedures

### Project Files

- Health check script: `/scripts/platform-health-check.py`
- Stack manager: `/scripts/stack-manager-py.sh` ([Stack Manager](stack-manager.md))
- Monitoring dashboards: `/infrastructure/grafana/dashboards/`

---

**Last Updated**: 2025-12-09
**Version**: 1.0
**Platform Version**: 1.0 (Production-ready)

**Contributing**: Found a solution not listed here? Please contribute by creating a Pull Request!
