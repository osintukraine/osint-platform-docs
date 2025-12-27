# Performance Tuning Guide

Optimize OSINT Intelligence Platform performance for your hardware and workload.

---

## Ollama Optimization

Ollama (self-hosted LLM) is typically the most impactful component to tune.

### CPU Allocation

The platform uses two Ollama instances:
- **ollama** (realtime): Processor classification, API queries
- **ollama-batch** (background): Enrichment tasks

**Recommended CPU allocation** (8-core system):

```yaml
# docker-compose.yml
ollama:
  deploy:
    resources:
      limits:
        cpus: '6.0'   # Realtime gets priority
        memory: 8G

ollama-batch:
  deploy:
    resources:
      limits:
        cpus: '4.0'   # Background can share
        memory: 6G
```

**Tuning parameters** in `.env`:

```bash
# Realtime Ollama (low latency priority)
OLLAMA_NUM_PARALLEL=1      # Process one request at a time
OLLAMA_MAX_LOADED_MODELS=2 # Keep 2 models in memory
OLLAMA_CPU_THREADS=6       # Match cpus limit
OLLAMA_KEEP_ALIVE=5m       # Keep model loaded

# Development mode (laptop)
DEVELOPMENT_MODE=true
OLLAMA_CPU_THREADS=2       # Reduce for laptop
OLLAMA_KEEP_ALIVE=2m       # Faster unload
```

### Model Selection

| Model | RAM | Speed | Quality | Use Case |
|-------|-----|-------|---------|----------|
| `gemma2:2b` | 1.5GB | Fast (30-35 tok/s) | 75% | Development, testing |
| `qwen2.5:3b` | 2.4GB | Medium (18-25 tok/s) | 87% | Production (RU/UK best) |
| `llama3.2:3b` | 2.5GB | Medium (20-25 tok/s) | 85% | Production (fallback) |
| `phi3.5:3.8b` | 3GB | Slow (12-15 tok/s) | 90% | Quality-critical |

**Switch model**:
```bash
# In .env
OLLAMA_MODEL=gemma2:2b  # Faster for development

# Restart processor
docker-compose restart processor-worker
```

### Memory Management

```bash
# Check model memory usage
docker stats osint-ollama

# If memory pressure, reduce loaded models
OLLAMA_MAX_LOADED_MODELS=1

# Or use smaller model
OLLAMA_MODEL=gemma2:2b
```

---

## PostgreSQL Tuning

### Connection Pooling

Default pool is 20 connections. Adjust based on worker count.

```bash
# .env
POSTGRES_POOL_SIZE=20        # Base connections
POSTGRES_MAX_OVERFLOW=10     # Burst connections
POSTGRES_POOL_TIMEOUT=30     # Wait time (seconds)
POSTGRES_POOL_RECYCLE=3600   # Recycle connections hourly
```

**Rule of thumb**: `POSTGRES_POOL_SIZE = (processor_workers * 2) + (enrichment_workers * 2) + 10`

### Memory Settings

Edit `infrastructure/postgres/postgresql.conf`:

```conf
# For 16GB RAM system
shared_buffers = 4GB          # 25% of RAM
effective_cache_size = 12GB   # 75% of RAM
maintenance_work_mem = 1GB    # For vacuum/index
work_mem = 64MB               # Per operation

# For 32GB RAM system
shared_buffers = 8GB
effective_cache_size = 24GB
maintenance_work_mem = 2GB
work_mem = 128MB
```

Restart PostgreSQL after changes:
```bash
docker-compose restart postgres
```

### Vacuum and Analyze

The platform creates frequent writes. Auto-vacuum is enabled, but manual optimization helps:

```bash
# Analyze all tables (update query planner statistics)
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "ANALYZE;"

# Vacuum and analyze large tables
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "VACUUM ANALYZE messages;"
```

**Schedule weekly** (cron):
```bash
0 3 * * 0 docker-compose exec -T postgres psql -U osint_user -d osint_platform -c "VACUUM ANALYZE;" >> /var/log/postgres-vacuum.log 2>&1
```

### Index Optimization

Critical indexes are created in `init.sql`. Monitor slow queries:

```sql
-- Find slow queries (>1 second)
SELECT query, calls, total_time/1000 as total_seconds, mean_time/1000 as mean_seconds
FROM pg_stat_statements
WHERE mean_time > 1000
ORDER BY total_time DESC
LIMIT 10;
```

---

## Redis Stream Management

### Stream Length

Redis Streams can grow indefinitely. Limit to prevent memory issues:

```bash
# .env
REDIS_MAX_STREAM_LENGTH=100000  # Max messages in queue
```

**Memory estimate**: 100K messages ≈ 500MB-1GB RAM

### Memory Configuration

```bash
# docker-compose.yml redis command
command: redis-server --appendonly yes --maxmemory 512mb --maxmemory-policy allkeys-lru
```

**Increase for high-volume**:
```yaml
command: redis-server --appendonly yes --maxmemory 2gb --maxmemory-policy allkeys-lru
```

### Monitor Queue Health

```bash
# Check stream length
docker-compose exec redis redis-cli XLEN telegram_messages

# Check consumer group lag
docker-compose exec redis redis-cli XINFO GROUPS telegram_messages

# Check memory usage
docker-compose exec redis redis-cli INFO memory | grep used_memory_human
```

---

## MinIO (Media Storage) Optimization

### Content-Addressed Deduplication

The platform uses SHA-256 hashing for deduplication. This is automatic and saves 50-70% storage for media-heavy channels.

**Verify deduplication**:
```bash
# Check unique vs total media
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(DISTINCT media_hash), COUNT(*) FROM message_media;"
```

### Storage Tiers

For large deployments, consider tiered storage:

```yaml
# docker-compose.yml
minio:
  volumes:
    - /fast-ssd/minio:/data        # Hot storage (recent)
    # Archive script moves old media to:
    # /slow-hdd/minio-archive      # Cold storage (>30 days)
```

### Cleanup Old Media

Telegram URLs expire, but archived media persists. Archive script (optional):

```bash
#!/bin/bash
# Move media older than 90 days to archive storage
find /data/minio/telegram-archive -mtime +90 -type f \
  -exec mv {} /archive/minio/ \;
```

---

## Processor Worker Tuning

### Batch Size

```bash
# .env
PROCESSOR_BATCH_SIZE=10  # Messages per batch
WORKER_BATCH_SIZE=50     # Worker batch size
```

**Increase for higher throughput** (but higher latency):
```bash
PROCESSOR_BATCH_SIZE=25
WORKER_BATCH_SIZE=100
```

### Worker Count

```bash
# .env
WORKER_COUNT=4           # Internal workers per container
PROCESSOR_REPLICAS=2     # Container replicas
```

**Total workers** = `WORKER_COUNT * PROCESSOR_REPLICAS`

**CPU rule**: Total workers ≤ CPU cores - 2 (leave room for Ollama)

### Feature Toggles

Disable features to reduce load:

```bash
# .env
SPAM_FILTER_ENABLED=true          # Keep enabled (saves resources!)
ENTITY_EXTRACTION_ENABLED=true    # Optional
TRANSLATION_ENABLED_IN_WORKER=true # Disable if not needed
LLM_SCORING_ENABLED=true          # Core feature
```

---

## Network Optimization

### Docker Network

Use bridge network (default) for single-host. For multi-host:

```yaml
# docker-compose.yml
networks:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### External Access

For production, use reverse proxy:

```yaml
# Caddy (included in auth profile)
caddy:
  ports:
    - "80:80"
    - "443:443"
```

---

## Development Mode Settings

For laptops and testing:

```bash
# .env - Development optimizations
DEVELOPMENT_MODE=true

# Lighter Ollama
OLLAMA_MODEL=gemma2:2b
OLLAMA_CPU_THREADS=2
OLLAMA_NUM_PARALLEL=1
OLLAMA_KEEP_ALIVE=2m

# Smaller pools
POSTGRES_POOL_SIZE=5
WORKER_COUNT=2
PROCESSOR_REPLICAS=1

# Reduced features
TRANSLATION_ENABLED=false  # Save API calls
```

---

## Production Checklist

Before deploying to production:

- [ ] Set `DEVELOPMENT_MODE=false`
- [ ] Allocate 4-6 CPU cores to Ollama
- [ ] Set `OLLAMA_MODEL=qwen2.5:3b` (production model)
- [ ] Configure `POSTGRES_POOL_SIZE` for worker count
- [ ] Set Redis memory limit (`--maxmemory 1gb`)
- [ ] Configure log rotation
- [ ] Set up monitoring alerts
- [ ] Test backup/restore procedure

---

## Benchmark Commands

### LLM Performance

```bash
# Test Ollama response time
time docker-compose exec ollama ollama run qwen2.5:3b "Classify: Russian troops near Kharkiv"

# Expected: <3 seconds for classification
```

### Database Performance

```bash
# Test query performance
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "\timing on" \
  -c "SELECT COUNT(*) FROM messages WHERE telegram_date > NOW() - INTERVAL '1 hour';"

# Expected: <100ms for count queries
```

### Queue Throughput

```bash
# Watch processing rate
watch -n 5 'docker-compose exec redis redis-cli XLEN telegram_messages'

# Should decrease or stay stable, not grow continuously
```

---

## Related Documentation

- [Scaling Guide](scaling.md) - When and how to scale
- [Monitoring](monitoring.md) - Performance dashboards
- [Docker Services Reference](../reference/docker-services.md) - Resource limits
- [Environment Variables](../reference/environment-variables.md) - All tuning parameters

---

**Quick Wins**:
1. Increase Ollama CPU to 6 cores (biggest impact)
2. Use `gemma2:2b` for development
3. Enable spam filter (saves 73% processing)
4. Run `VACUUM ANALYZE` weekly
