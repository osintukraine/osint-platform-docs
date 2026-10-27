# Performance Tuning Guide

Optimize OSINT Intelligence Platform performance for your hardware and workload.

---

## Ollama Optimization

!!! note "GPU gateway"
    These sections apply to the local-Ollama (CPU) setup, `LLM_PROVIDER=ollama`. With an LLM gateway (`LLM_PROVIDER=openai`), classification runs on the GPU host: tune lane keys, aliases and concurrency instead (`backlog_concurrency`, `FACT_CHECK_*`), see [LLM Gateway](external-llm.md). The embedding spam filter and the pre-filter threshold reduce LLM load on either setup, see [Embedding Models](model-provisioning.md).

Ollama (self-hosted LLM) is typically the most impactful component to tune on CPU hardware.

### CPU Allocation

The compose file defines two Ollama instances:
- **ollama** (always on): search-query embeddings (`all-minilm`) and, with `LLM_PROVIDER=ollama`, processor classification
- **ollama-batch** (enrichment profiles only, port `OLLAMA_BATCH_PORT`=11435): background enrichment tasks that still use the Ollama endpoint `OLLAMA_BATCH_URL`

**Recommended CPU allocation** (8-core system):

```yaml
# docker-compose.yml (shipped values)
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

Both have CPU/memory reservations as well. Lower the limits on small hosts; the sum of all limits should not exceed your core count by much.

**Tuning parameters** in `.env`:

```bash
# Realtime Ollama (low latency priority) - compose defaults
OLLAMA_NUM_PARALLEL=1      # Process one request at a time
OLLAMA_MAX_LOADED_MODELS=2 # Keep 2 models in memory
OLLAMA_CPU_THREADS=6       # Match cpus limit
OLLAMA_KEEP_ALIVE=5m       # Keep model loaded

# Small host (laptop)
OLLAMA_CPU_THREADS=2       # Reduce for laptop
OLLAMA_KEEP_ALIVE=2m       # Faster unload
```

`./scripts/stack-manager-py.sh hardware` writes tier-appropriate values for these into `.env` (see [Stack Manager](stack-manager.md)).

### Model Selection

The `.env.example` default is `OLLAMA_MODEL=qwen2.5:1.5b`, a small CPU-friendly model. Larger models classify better but need proportionally more RAM and CPU; benchmark on your own data (`scripts/benchmark_llm_models.py`) rather than trusting generic tables. Which model is used at runtime is also controlled by the `model_configuration` table (Admin UI, Model Configuration). If you have a GPU host, prefer the LLM gateway ([LLM Gateway](external-llm.md)).

**Switch model**:
```bash
# In .env
OLLAMA_MODEL=<smaller model>  # Faster, lower quality; pull it first

# Restart processor
docker compose restart processor-worker
```

### Memory Management

```bash
# Check model memory usage
docker stats osint-ollama

# If memory pressure, reduce loaded models
OLLAMA_MAX_LOADED_MODELS=1

# Or use a smaller model
OLLAMA_MODEL=<smaller model>
```

---

## PostgreSQL Tuning

### Connection Pooling

The code default is 40 connections per process (`POSTGRES_POOL_SIZE`, overflow 40), which is per service replica, so total connections grow with the replica count. PostgreSQL's `max_connections` is 200 in `postgresql.conf`; keep `replicas x (pool + overflow)` under it, or lower the pool.

```bash
# .env (code defaults shown)
POSTGRES_POOL_SIZE=40        # Base connections per process
POSTGRES_MAX_OVERFLOW=40     # Burst connections
POSTGRES_POOL_TIMEOUT=30     # Wait time (seconds)
POSTGRES_POOL_RECYCLE=3600   # Recycle connections hourly
```

**Rule of thumb**: size the pool for the busiest service, not the sum; every service replica opens its own pool.

### Memory Settings

Memory settings are passed on the PostgreSQL command line by `docker-compose.yml` and **override** `infrastructure/postgres/postgresql.conf`. Change them in `.env`:

```bash
POSTGRES_SHARED_BUFFERS=4GB      # ~25% of RAM (compose default 1GB)
POSTGRES_EFFECTIVE_CACHE=12GB    # ~75% of RAM (compose default 4GB)
POSTGRES_WORK_MEM=64MB           # per sort/hash operation (compose default 128MB)
```

Other settings (`maintenance_work_mem`, `max_connections`, parallel workers, logging) come from `postgresql.conf`, which is tuned for a 32 GB host. `hardware` in the stack manager sets `POSTGRES_SHARED_BUFFERS` and `POSTGRES_WORK_MEM` per tier. Recreate PostgreSQL after changes:

```bash
docker compose up -d postgres
```

### Vacuum and Analyze

The platform creates frequent writes. Auto-vacuum is enabled, but manual optimization helps:

```bash
# Analyze all tables (update query planner statistics)
docker compose exec postgres psql -U osint_user -d osint_platform \
  -c "ANALYZE;"

# Vacuum and analyze large tables
docker compose exec postgres psql -U osint_user -d osint_platform \
  -c "VACUUM ANALYZE messages;"
```

**Schedule weekly** (cron):
```bash
0 3 * * 0 docker compose exec -T postgres psql -U osint_user -d osint_platform -c "VACUUM ANALYZE;" >> /var/log/postgres-vacuum.log 2>&1
```

### Index Optimization

Critical indexes are created in `init.sql` and migrations. Monitor slow queries (`pg_stat_statements` is loaded by `postgresql.conf`; on PostgreSQL 13+ the columns are `total_exec_time` and `mean_exec_time`):

```sql
-- Find slow queries (>1 second)
SELECT query, calls, total_exec_time/1000 as total_seconds, mean_exec_time/1000 as mean_seconds
FROM pg_stat_statements
WHERE mean_exec_time > 1000
ORDER BY total_exec_time DESC
LIMIT 10;
```

---

## Redis Stream Management

### Stream Length

The listener writes to two streams, `telegram:messages:realtime` (live) and `telegram:messages:backfill` (historical, consumed when idle). Each is capped at 100,000 entries (`MAX_STREAM_LENGTH` in `services/listener/src/redis_queue.py`; it is not an environment variable). Size Redis memory for a full stream.

**Memory estimate**: 100K queued messages ≈ 500MB-1GB RAM

### Memory Configuration

Redis memory is set in `.env` (compose runs `redis-server --maxmemory ${REDIS_MAXMEMORY:-1gb} --maxmemory-policy allkeys-lru`):

```bash
REDIS_MAXMEMORY=2gb   # raise for high-volume or large backfills
```

```bash
docker compose up -d redis
```

!!! warning
    `allkeys-lru` can evict stream data under memory pressure. If `RedisHighEvictionRate` fires, raise `REDIS_MAXMEMORY` rather than waiting.

### Monitor Queue Health

```bash
# Check stream length
docker compose exec redis redis-cli XLEN telegram:messages:realtime
docker compose exec redis redis-cli XLEN telegram:messages:backfill

# Check consumer group lag
docker compose exec redis redis-cli XINFO GROUPS telegram:messages:realtime

# Check memory usage
docker compose exec redis redis-cli INFO memory | grep used_memory_human
```

---

## MinIO (Media Storage) Optimization

### Content-Addressed Deduplication

The platform stores media by SHA-256 (`media_files.sha256` is unique), so identical files are stored once. This is automatic; the saving varies with the channels (expect roughly 30-40% on typical content).

**Verify deduplication**:
```bash
# Links from messages vs distinct stored files
docker compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT (SELECT COUNT(*) FROM message_media) AS links, (SELECT COUNT(*) FROM media_files) AS files;"
```

### Storage Tiers

For large deployments the platform has built-in tiers: a local SSD media buffer (`MEDIA_BUFFER_PATH`) served directly by Caddy, and Hetzner Storage Boxes behind MinIO for cold files. See [Hetzner Storage](hetzner-storage.md), [Multi-Storage Setup](multi-storage-setup.md) and [Media Lanes](media-lanes.md) (background download so the realtime path stays fast).

### Cleanup Old Media

Do not move or delete files under the MinIO data directory by hand: the database records each object's key and storage box. Use the media repair tooling instead ([Media Repair](media-repair.md)).

---

## Processor Worker Tuning

### Batch Size

```bash
# .env
PROCESSOR_BATCH_SIZE=10      # Messages per XREADGROUP call (default 10)
BACKFILL_BATCH_SIZE=100      # Batch size for the backfill lane (default 100)
```

**Increase for higher throughput** (but higher latency):
```bash
PROCESSOR_BATCH_SIZE=25
```

### Worker Count

```bash
# .env
PROCESSOR_WORKER_REPLICAS=2   # processor-worker container replicas (default 2)
PROCESSOR_CONCURRENCY=1       # concurrent messages per worker (see .env.example)
BACKFILL_WORKER_REPLICAS=2    # processor-worker-backfill replicas (backfill profile)
```

**Total concurrency** = `PROCESSOR_WORKER_REPLICAS * PROCESSOR_CONCURRENCY`

**CPU rule**: keep total workers at or below CPU cores minus 2 when Ollama runs on the same host. With a GPU gateway the LLM load is off-box.

### Feature Toggles

Disable features to reduce load:

```bash
# .env
SPAM_FILTER_ENABLED=true          # Keep enabled (saves resources!)
ENTITY_EXTRACTION_ENABLED=true    # Optional
TRANSLATION_ENABLED=true          # Disable if not needed (also saves DeepL quota)
LLM_ENABLED=true                  # Core feature; LLM_PROVIDER selects ollama, openai (gateway) or none
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
# .env - Development optimizations (or run: ./scripts/stack-manager-py.sh hardware --tier=laptop)
# Lighter Ollama
OLLAMA_CPU_THREADS=2
OLLAMA_NUM_PARALLEL=1
OLLAMA_KEEP_ALIVE=2m

# Smaller pools and fewer workers
POSTGRES_POOL_SIZE=10
POSTGRES_MAX_OVERFLOW=10
PROCESSOR_WORKER_REPLICAS=1
REDIS_MAXMEMORY=256mb

# Reduced features
TRANSLATION_ENABLED=false  # Save API calls

# Profiles: enrichment-lite (CPU only) or none
```

---

## Production Checklist

Before deploying to production:

- [ ] Set `ENVIRONMENT=production` and `DEBUG=false`
- [ ] Allocate 4-6 CPU cores to Ollama if it runs classification locally
- [ ] Choose a model that fits your hardware (`./scripts/bootstrap-hardware.sh show`) when using Ollama
- [ ] Check `POSTGRES_POOL_SIZE` x replicas against `max_connections`
- [ ] Set `REDIS_MAXMEMORY` and `POSTGRES_SHARED_BUFFERS` for your RAM
- [ ] Configure log rotation
- [ ] Set up monitoring alerts
- [ ] Test backup/restore procedure

---

## Benchmark Commands

### LLM Performance

```bash
# Test Ollama response time
time docker compose exec ollama ollama run <your OLLAMA_MODEL> "Classify: troop movement near Kharkiv"

# Expected: <3 seconds for classification
```

### Database Performance

```bash
# Test query performance
docker compose exec postgres psql -U osint_user -d osint_platform \
  -c "\timing on" \
  -c "SELECT COUNT(*) FROM messages WHERE telegram_date > NOW() - INTERVAL '1 hour';"

# Expected: <100ms for count queries
```

### Queue Throughput

```bash
# Watch processing rate
watch -n 5 'docker compose exec redis redis-cli XLEN telegram:messages:realtime'

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
2. Use a smaller model (or the `enrichment-lite` profile) for development
3. Enable spam filter (saves 73% processing)
4. Run `VACUUM ANALYZE` weekly
