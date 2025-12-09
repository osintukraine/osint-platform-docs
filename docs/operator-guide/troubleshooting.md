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
- [LLM and AI Issues](#llm-and-ai-issues)
- [Search Issues](#search-issues)
- [Network and Connectivity Issues](#network-and-connectivity-issues)
- [Getting Help](#getting-help)

---

## Overview

**Key Resources:**

- **Logs**: Dozzle (http://localhost:8888) for real-time container logs
- **Metrics**: Grafana (http://localhost:3001) for performance dashboards
- **Alerts**: Prometheus (http://localhost:9090/alerts) for active alerts
- **Health checks**: `./scripts/health-check.sh` for automated status

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
# Run automated health check
./scripts/health-check.sh

# Expected output:
# === OSINT Platform Health Check ===
#
# Listener: healthy
# Processor: healthy
# API: healthy
# Enrichment: healthy
#
# PostgreSQL: healthy
# Redis: PONG
# MinIO: healthy
# Ollama: healthy
#
# Prometheus: healthy
# Grafana: healthy
#
# === Health Check Complete ===
```

### Quick Service Status

```bash
# Check all services
docker-compose ps

# Expected: All services "Up" with "(healthy)" where applicable
# Unhealthy or "Exit 1" indicates a problem

# Check specific service
docker-compose ps listener
docker-compose ps processor-worker
docker-compose ps postgres
```

### View Recent Logs

```bash
# Last 50 lines of all services
docker-compose logs --tail=50

# Last 100 lines of specific service
docker-compose logs --tail=100 processor-worker

# Follow logs in real-time
docker-compose logs -f processor-worker

# Search logs for errors
docker-compose logs processor-worker | grep -i error
docker-compose logs processor-worker | grep -i exception
```

### Check Resource Usage

```bash
# Container resource usage (CPU, memory)
docker stats

# Disk space
df -h
docker system df

# Database size
docker-compose exec postgres psql -U postgres -d osint_platform -c "
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
docker-compose ps
# Shows: osint-listener    Exit 1
```

**Diagnosis:**

```bash
# Check exit logs
docker-compose logs --tail=50 listener

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
docker-compose ps postgres
# Should show: Up (healthy)

# If not healthy, check postgres logs
docker-compose logs postgres

# Wait for PostgreSQL to be ready
docker-compose up -d postgres
sleep 30  # Wait 30 seconds
docker-compose up -d listener
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
docker-compose config

# Should output formatted YAML with no errors
# If errors, check line numbers in error message
```

### Issue: Service Crashes After Starting

**Symptoms:**

```bash
docker-compose ps
# Shows service "Restarting" continuously
```

**Diagnosis:**

```bash
# Watch logs during restart
docker-compose logs -f processor-worker

# Look for repeating error patterns
# Common: Out of memory, database connection pool exhausted
```

**Solutions:**

**A. Out of memory (OOM killed):**

```bash
# Check for OOM in logs
docker-compose logs processor-worker | grep -i "killed"

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
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT count(*) FROM pg_stat_activity;
"

# If near max_connections (100), check for connection leaks
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT application_name, state, count(*)
  FROM pg_stat_activity
  GROUP BY application_name, state;
"

# Fix: Restart service to release connections
docker-compose restart processor-worker
```

### Issue: Healthcheck Failing

**Symptoms:**

```bash
docker-compose ps postgres
# Shows: Up (unhealthy)
```

**Diagnosis:**

```bash
# Check healthcheck command manually
docker-compose exec postgres pg_isready -U postgres -d osint_platform

# Should output: osint_platform:5432 - accepting connections
# If not, database is not ready
```

**Solutions:**

```bash
# Wait longer (database may be recovering)
sleep 60
docker-compose ps postgres

# If still unhealthy, restart service
docker-compose restart postgres

# Check logs for initialization errors
docker-compose logs postgres | grep -i error
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
docker-compose ps postgres
# Should show: Up (healthy)

# Check network connectivity
docker-compose exec processor ping postgres
# Should respond (Ctrl+C to stop)

# Check PostgreSQL is listening
docker-compose exec postgres netstat -tlnp | grep 5432
# Should show: LISTEN on port 5432
```

**Solutions:**

**A. PostgreSQL not started:**

```bash
docker-compose up -d postgres
sleep 30  # Wait for PostgreSQL to be ready
docker-compose restart processor-worker
```

**B. Network misconfiguration:**

```bash
# Verify services are on same network
docker network ls
docker network inspect osint-intelligence-platform_backend

# Ensure postgres and processor-worker are both listed

# If not, recreate network
docker-compose down
docker-compose up -d
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
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT count(*) FROM pg_stat_activity;
"

# Check max_connections setting
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SHOW max_connections;
"

# Identify connection leakers
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT application_name, state, count(*)
  FROM pg_stat_activity
  GROUP BY application_name, state
  ORDER BY count(*) DESC;
"
```

**Solutions:**

**A. Increase max_connections (temporary fix):**

```bash
# Edit infrastructure/postgres/postgresql.conf
max_connections = 200  # Increase from 100

# Restart PostgreSQL
docker-compose restart postgres
```

**B. Fix connection leaks (permanent fix):**

```python
# Check code for unclosed database sessions
# services/processor/src/worker.py

# BAD: Session not closed
session = SessionLocal()
messages = session.query(Message).all()
# If exception occurs here, session never closed!

# GOOD: Use context manager
from shared.database import get_db_session

with get_db_session() as session:
    messages = session.query(Message).all()
# Session automatically closed, even if exception occurs
```

**C. Kill idle connections:**

```bash
# Kill connections idle for >10 minutes
docker-compose exec postgres psql -U postgres -d osint_platform -c "
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
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT calls, mean_exec_time, query
  FROM pg_stat_statements
  ORDER BY mean_exec_time DESC
  LIMIT 10;
"

# Check for missing indexes
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT schemaname, tablename, indexname
  FROM pg_indexes
  WHERE tablename = 'messages';
"

# Check table bloat (needs VACUUM)
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
  FROM pg_tables
  WHERE schemaname = 'public'
  ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
  LIMIT 10;
"
```

**Solutions:**

**A. Add missing indexes:**

```sql
# Common missing indexes (from production experience)

-- Index for date range queries (most common)
CREATE INDEX CONCURRENTLY idx_messages_date_id ON messages(date DESC, id DESC);

-- Index for channel-specific queries
CREATE INDEX CONCURRENTLY idx_messages_channel_date ON messages(channel_id, date DESC);

-- Index for OSINT score filtering
CREATE INDEX CONCURRENTLY idx_messages_osint_score ON messages(osint_score DESC) WHERE osint_score > 50;

-- Index for spam filtering
CREATE INDEX CONCURRENTLY idx_messages_spam ON messages(spam, date DESC);
```

**B. Run VACUUM to reduce bloat:**

```bash
# Analyze and vacuum (can run online, safe for production)
docker-compose exec postgres psql -U postgres -d osint_platform -c "VACUUM ANALYZE messages;"

# For heavily bloated tables, use VACUUM FULL (requires table lock, downtime!)
# Only run during maintenance window
docker-compose exec postgres psql -U postgres -d osint_platform -c "VACUUM FULL messages;"
```

**C. Tune PostgreSQL configuration:**

```bash
# Edit infrastructure/postgres/postgresql.conf

# Increase shared_buffers (if cache hit ratio <90%)
shared_buffers = 512MB  # Increase from 256MB

# Increase work_mem (for complex queries)
work_mem = 16MB  # Increase from 4MB

# Increase effective_cache_size (total RAM available)
effective_cache_size = 2GB

# Restart PostgreSQL
docker-compose restart postgres
```

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
docker-compose logs listener | grep -i floodwait

# Count occurrences
docker-compose logs listener | grep -c FloodWaitError

# If frequent (>10/hour), Telegram API is rate limiting
```

**Solutions:**

**A. Automatic retry (already implemented):**

The listener automatically waits for the required time and retries. No action needed. Wait for the specified seconds.

**B. Reduce request rate:**

```bash
# Edit services/listener/src/config.py

# Increase delay between channel polls
POLL_INTERVAL_SECONDS = 60  # Increase from 30

# Reduce concurrent channel fetches
MAX_CONCURRENT_CHANNELS = 5  # Reduce from 10

# Restart listener
docker-compose restart listener
```

**C. Multi-account strategy (if persistent):**

```yaml
# Use multiple Telegram accounts to distribute load
# docker-compose.yml
services:
  listener-account-1:
    environment:
      TELEGRAM_SESSION_NAME: account1
      MONITORED_CHANNELS: "channel1,channel2,channel3"

  listener-account-2:
    environment:
      TELEGRAM_SESSION_NAME: account2
      MONITORED_CHANNELS: "channel4,channel5,channel6"
```

**Reference**: See `/osint-intelligence-platform/docs/PITFALLS_FROM_PRODUCTION.md` HIGH #1 for detailed multi-account strategy.

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
ls -la telegram_sessions/listener.session

# If missing or 0 bytes, session needs re-authentication
```

**Solutions:**

**A. Re-authenticate (interactive):**

```bash
# Stop listener
docker-compose stop listener

# Run interactive auth
docker-compose run --rm listener python -m services.listener.src.auth

# Follow prompts:
# - Enter phone number (with country code: +1234567890)
# - Enter verification code sent to Telegram app
# - Enter 2FA password (if enabled)

# Verify session created
ls -la telegram_sessions/listener.session
# Should be >1KB

# Start listener
docker-compose up -d listener

# Verify authentication
docker-compose logs -f listener | grep "Logged in as"
```

**B. Restore from backup:**

```bash
# If you have backup session files (see backup-restore.md)
tar -xzf backups/sessions/sessions_latest.tar.gz -C telegram_sessions/
chmod 600 telegram_sessions/*.session*

docker-compose restart listener
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

**A. Join channel via invite link:**

```python
# If channel went private, get invite link from admin
# services/listener/src/manual_join.py

from telethon.sync import TelegramClient

client = TelegramClient('listener', api_id, api_hash)
client.start()

# Join via invite link
client(JoinChannelRequest('https://t.me/+INVITE_CODE'))
```

**B. Remove inaccessible channel from monitoring:**

```bash
# Edit config to skip this channel
# config/channels.yml (or database)

# Mark channel as inactive
docker-compose exec postgres psql -U postgres -d osint_platform -c "
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
docker-compose logs listener | grep "New message"
# Should show recent messages

# 2. Check Redis queue has messages
docker-compose exec redis redis-cli XLEN telegram_messages
# Should show count > 0

# 3. Check processor is consuming from queue
docker-compose logs processor-worker | grep "Processing message"
# Should show recent processing

# 4. Check database for recent messages
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT COUNT(*) FROM messages WHERE date > NOW() - INTERVAL '1 hour';
"
# Should match messages received by listener
```

**Solutions:**

**A. Listener not forwarding to queue:**

```bash
# Check listener logs for errors
docker-compose logs listener | grep -i error

# Common issues:
# - Redis connection failed
# - Message size exceeds limit
# - Spam filter blocking all messages

# Verify Redis connectivity
docker-compose exec listener ping redis
# Should respond

# Check spam filter rate
docker-compose logs listener | grep "spam detected"
# If >95%, spam filter may be misconfigured
```

**B. Processor not consuming from queue:**

```bash
# Check processor is running
docker-compose ps processor-worker
# Should show: Up

# Check processor logs for errors
docker-compose logs processor-worker | grep -i error

# Common errors:
# - Database connection failed
# - Ollama LLM timeout
# - MinIO media upload failed

# Restart processor
docker-compose restart processor-worker
```

**C. Queue backlog:**

```bash
# Check queue depth
docker-compose exec redis redis-cli XLEN telegram_messages

# If >10,000, processor can't keep up
# Scale up processor workers
docker-compose up -d --scale processor-worker=4

# Monitor queue drain
watch 'docker-compose exec redis redis-cli XLEN telegram_messages'
```

### Issue: High Message Skip Rate (>90%)

**Symptoms:**

- Grafana shows >90% messages skipped
- Very few messages in database despite high volume

**Diagnosis:**

```bash
# Check OSINT scoring metrics
docker-compose exec processor-worker curl -s http://localhost:8002/metrics | grep osint_messages_skipped

# Check OSINT score distribution
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT
    CASE
      WHEN osint_score < 10 THEN '0-10'
      WHEN osint_score < 30 THEN '10-30'
      WHEN osint_score < 50 THEN '30-50'
      WHEN osint_score < 70 THEN '50-70'
      ELSE '70-100'
    END AS score_range,
    COUNT(*) as count
  FROM messages
  WHERE date > NOW() - INTERVAL '24 hours'
  GROUP BY score_range
  ORDER BY score_range;
"
```

**Solutions:**

**A. OSINT scoring threshold too high:**

```bash
# Check current threshold in config/osint_rules.yml or .env
grep OSINT_SCORE_THRESHOLD .env

# If >70, lowering threshold will archive more messages
# Edit .env:
OSINT_SCORE_THRESHOLD=50  # Lower from 70

# Restart processor
docker-compose restart processor-worker
```

**B. Rule patterns too strict:**

```bash
# Check rule coverage (% messages matched by rules)
# Grafana → RuleEngine Performance → Rule Coverage Over Time

# If <30%, rules may be too specific
# Review and broaden rule patterns in config/osint_rules.yml

# Example: Broaden location matching
rules:
  - name: bakhmut_mentions
    pattern: "Bakhmut|Бахмут|Артемовск"  # Add alternate spellings
    score_adjustment: +30
```

**C. LLM scoring too conservative:**

```bash
# Check LLM prompt version
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT id, version, active FROM llm_prompts ORDER BY created_at DESC LIMIT 5;
"

# If using older prompt version, may be more conservative
# See docs/architecture/LLM_PROMPTS.md for prompt evolution

# Activate newer prompt version (if available)
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  UPDATE llm_prompts SET active = false WHERE version = 'v2';
  UPDATE llm_prompts SET active = true WHERE version = 'v6';
"

# Restart processor to reload prompts
docker-compose restart processor-worker
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
docker-compose exec processor-worker top

# Check for CPU-intensive tasks:
# - LLM inference (Ollama)
# - Entity extraction (spaCy)
# - Embedding generation

# Check Grafana → cAdvisor Dashboard → Container CPU Usage
# Identify which container(s) are CPU-bound
```

**Solutions:**

**A. LLM inference bottleneck:**

```bash
# Check LLM response time
docker-compose logs processor-worker | grep "LLM inference took"

# If >15s per message, LLM is bottleneck
# Options:
# 1. Use faster model
# 2. Reduce LLM calls (increase rule coverage)
# 3. Scale horizontally (dedicated Ollama instance)

# Option 1: Switch to faster model
docker-compose exec ollama ollama pull qwen2.5:1.5b  # Faster, less accurate
# Update .env:
OLLAMA_MODEL=qwen2.5:1.5b

# Option 2: Increase rule coverage (skip LLM for obvious cases)
# See config/osint_rules.yml

# Option 3: Dedicated Ollama (see docker-compose.yml ollama-batch service)
```

**B. Too many concurrent workers:**

```bash
# Check how many processor workers are running
docker-compose ps | grep processor-worker

# If >4 on single CPU, workers compete for CPU
# Scale down:
docker-compose up -d --scale processor-worker=2

# Monitor CPU usage
docker stats
```

**C. Database query optimization:**

```bash
# Check for slow queries consuming CPU
# See "Slow Query Performance" section above

# Check cache hit ratio
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT sum(blks_hit)::float / (sum(blks_hit) + sum(blks_read)) AS cache_hit_ratio
  FROM pg_stat_database;
"

# If <0.9, increase shared_buffers
# Edit infrastructure/postgres/postgresql.conf:
shared_buffers = 512MB

docker-compose restart postgres
```

### Issue: High Memory Usage / OOM Kills

**Symptoms:**

```bash
docker stats
# Shows: Container using >90% of memory limit

docker-compose logs processor-worker | grep -i killed
# Shows: "Killed" (OOM killer)
```

**Diagnosis:**

```bash
# Check container memory limits
docker-compose config | grep -A 5 "processor-worker" | grep memory

# Check actual memory usage
docker stats --no-stream processor-worker

# Identify memory-intensive processes
docker-compose exec processor-worker ps aux --sort=-%mem | head -10
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
docker-compose up -d processor-worker
```

**B. Fix memory leaks:**

```python
# Common memory leak: Not closing sessions
# services/processor/src/worker.py

# BAD: Session never closed
session = SessionLocal()
for message in messages:
    session.query(Message).filter_by(id=message.id).first()
# Session accumulates objects, memory grows

# GOOD: Use context manager
with SessionLocal() as session:
    for message in messages:
        msg = session.query(Message).filter_by(id=message.id).first()
        session.expunge(msg)  # Remove from session to free memory
```

**C. Reduce batch sizes:**

```python
# If processing large batches, split into smaller chunks
# services/enrichment/src/tasks/ai_tagging.py

# BAD: Load all pending messages (may be 100,000+)
messages = session.query(Message).filter(Message.ai_tags.is_(None)).all()

# GOOD: Process in batches
BATCH_SIZE = 100
offset = 0
while True:
    messages = session.query(Message).filter(Message.ai_tags.is_(None)) \
        .limit(BATCH_SIZE).offset(offset).all()
    if not messages:
        break
    process_batch(messages)
    offset += BATCH_SIZE
```

---

## Storage Issues

### Issue: Disk Space Full

**Symptoms:**

```bash
df -h
# Shows: /dev/sda1  95%  (or 100%)

docker-compose logs postgres
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
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT pg_size_pretty(pg_database_size('osint_platform'));
"

# Check MinIO bucket size
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

**C. Archive old database data:**

```sql
-- Move old messages to archive table
CREATE TABLE messages_archive AS
SELECT * FROM messages WHERE date < '2024-01-01';

DELETE FROM messages WHERE date < '2024-01-01';

VACUUM FULL messages;
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
docker-compose ps minio
# Should show: Up (healthy)

# Check MinIO health
curl http://localhost:9000/minio/health/live
# Should return HTTP 200

# Test S3 API
./bin/mc alias set test http://localhost:9000 minioadmin minioadmin
./bin/mc ls test/
# Should list buckets
```

**Solutions:**

**A. MinIO not started:**

```bash
docker-compose up -d minio
sleep 10  # Wait for MinIO to be ready
docker-compose restart processor-worker
```

**B. Bucket not created:**

```bash
# Check if bucket exists
./bin/mc ls production/
# Should show: osint-media

# If missing, create bucket
./bin/mc mb production/osint-media
./bin/mc anonymous set download production/osint-media
```

**C. MinIO credentials incorrect:**

```bash
# Verify .env has correct credentials
grep MINIO .env

# Should match docker-compose.yml minio service:
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin

# Update processor .env if different
docker-compose restart processor-worker
```

---

## LLM and AI Issues

### Issue: Ollama Service Not Responding

**Symptoms:**

```bash
# Processor logs show:
# Error: LLM request timeout after 30 seconds
# Error connecting to Ollama: Connection refused
```

**Diagnosis:**

```bash
# Check Ollama is running
docker-compose ps ollama
# Should show: Up

# Check Ollama API
curl http://localhost:11434/api/tags
# Should return: {"models": [...]}

# Check Ollama logs
docker-compose logs ollama | tail -50
```

**Solutions:**

**A. Ollama not started:**

```bash
docker-compose up -d ollama
sleep 30  # Wait for Ollama to load models
docker-compose restart processor-worker
```

**B. Model not loaded:**

```bash
# List available models
docker-compose exec ollama ollama list

# If empty or missing qwen2.5:3b, pull model
docker-compose exec ollama ollama pull qwen2.5:3b

# Verify model loaded
docker-compose exec ollama ollama list
# Should show: qwen2.5:3b
```

**C. Ollama overloaded:**

```bash
# Check Ollama resource usage
docker stats ollama

# If CPU >90% or memory >90%, Ollama is bottleneck
# Options:
# 1. Use faster model (smaller parameters)
# 2. Scale horizontally (dedicated Ollama instance)
# 3. Increase timeout in processor config

# Option 1: Use faster model
docker-compose exec ollama ollama pull qwen2.5:1.5b
# Update .env:
OLLAMA_MODEL=qwen2.5:1.5b

# Option 3: Increase timeout
# Edit services/processor/src/llm_classifier.py:
LLM_TIMEOUT_SECONDS = 60  # Increase from 30

docker-compose restart processor-worker
```

### Issue: Low LLM Success Rate (<90%)

**Symptoms:**

- Grafana → AI/ML Processing → LLM Success Rate shows <90%
- Many messages have NULL osint_score

**Diagnosis:**

```bash
# Check LLM error rate
docker-compose logs processor-worker | grep "LLM error" | tail -20

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
# Check prompt version active:
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT id, version, active FROM llm_prompts WHERE active = true;
"

# Verify prompt includes JSON schema instructions
# See docs/architecture/LLM_PROMPTS.md for prompt design

# If using old prompt (v2-v4), activate v6:
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  UPDATE llm_prompts SET active = false;
  UPDATE llm_prompts SET active = true WHERE version = 'v6';
"

docker-compose restart processor-worker
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
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT COUNT(*) FROM messages WHERE search_vector @@ websearch_to_tsquery('english', 'Bakhmut');
"

# If 0, search_vector may not be populated
# Check if search_vector column exists
docker-compose exec postgres psql -U postgres -d osint_platform -c "
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
docker-compose exec postgres psql -U postgres -d osint_platform -c "
  SELECT * FROM pg_extension WHERE extname = 'vector';
"

# If empty, pgvector not installed
```

**Solutions:**

**A. Install pgvector extension:**

```sql
-- Connect as superuser
docker-compose exec postgres psql -U postgres -d osint_platform

-- Create extension
CREATE EXTENSION vector;

-- Verify installation
SELECT * FROM pg_extension WHERE extname = 'vector';
```

**B. Regenerate embeddings:**

```bash
# Embeddings may not be generated yet
# Check enrichment service logs
docker-compose logs enrichment | grep "embedding"

# Manually trigger embedding generation for recent messages
docker-compose exec enrichment python -c "
from tasks.embedding import EmbeddingTask
task = EmbeddingTask()
task.run()
"
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
docker-compose ps

# Check if ports are mapped correctly
docker-compose ps | grep "0.0.0.0"
# Should show port mappings like: 0.0.0.0:3001->3000/tcp

# Check if something else is using the port
sudo lsof -i :3001
sudo lsof -i :8000

# Test connectivity from inside container
docker-compose exec api curl -s http://localhost:8000/health
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

docker-compose up -d grafana
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
docker-compose down
docker network prune
docker-compose up -d
```

---

## Getting Help

### Before Asking for Help

**Gather diagnostic information:**

```bash
# 1. Platform version
git log -1 --oneline

# 2. Service status
docker-compose ps > status.txt

# 3. Recent logs (all services)
docker-compose logs --tail=100 > logs.txt

# 4. System resources
docker stats --no-stream > resources.txt
df -h > disk.txt

# 5. Configuration (sanitized)
cp .env .env.sanitized
# EDIT .env.sanitized: Remove actual passwords/secrets
# Replace with placeholders: POSTGRES_PASSWORD=<redacted>

# 6. Error messages
docker-compose logs | grep -i error > errors.txt
```

### Where to Get Help

1. **Check this troubleshooting guide first**
2. **Review logs** via Dozzle (http://localhost:8888)
3. **Check metrics** via Grafana (http://localhost:3001)
4. **Search GitHub Issues**: https://github.com/osintukraine/osint-intelligence-platform/issues
5. **Create new issue** with diagnostic information above

### Creating a Good Bug Report

```markdown
**Environment:**
- OS: Ubuntu 22.04 / macOS 13 / Windows 11
- Docker version: (output of `docker --version`)
- Docker Compose version: (output of `docker-compose --version`)
- Platform version: (output of `git log -1 --oneline`)

**Issue Description:**
Clear description of the problem.

**Steps to Reproduce:**
1. Start services: `docker-compose up -d`
2. Wait 2 minutes
3. Check logs: `docker-compose logs processor-worker`
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

- Health check script: `/scripts/health-check.sh`
- Stack manager: `/scripts/stack-manager.sh`
- Monitoring dashboards: `/infrastructure/grafana/dashboards/`

---

**Last Updated**: 2025-12-09
**Version**: 1.0
**Platform Version**: 1.0 (Production-ready)

**Contributing**: Found a solution not listed here? Please contribute by creating a Pull Request!
