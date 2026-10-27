# Scaling Guide

Guide to scaling the OSINT Intelligence Platform as monitoring volume increases.

---

## When to Scale

### Signs You Need More Processor Workers

| Indicator | Threshold | Action |
|-----------|-----------|--------|
| Redis queue depth (`osint_queue_messages_pending`) | >1000 messages | Add processor workers |
| Processing latency | >5 seconds | Add processor workers |
| CPU consistently | >80% on workers | Add processor workers or upgrade CPU |
| LLM response time | >3 seconds | Check Ollama resources or the LLM gateway |

**Monitor these in Grafana**: `OSINT Platform Overview` dashboard

### Signs You Need More Enrichment Workers

| Indicator | Threshold | Action |
|-----------|-----------|--------|
| Embedding backlog | >10,000 messages | Scale enrichment-fast-pool |
| Tagging backlog | >5,000 messages | Scale enrichment-ai-tagging |
| Translation queue | >1,000 messages | Scale enrichment-fast-pool |

**Monitor these in Grafana**: `Enrichment Performance` dashboard

### Signs of Infrastructure Bottlenecks

| Indicator | Threshold | Action |
|-----------|-----------|--------|
| PostgreSQL connections | >80% pool used | Increase `POSTGRES_POOL_SIZE` |
| Redis memory | >80% of limit | Increase Redis memory or reduce stream length |
| Ollama response time | >5 seconds | Increase CPU/memory allocation |
| Disk space | <20% free | Add storage or archive old media |

---

## How to Scale

### Scale Processor Workers

The processor handles real-time message classification. Default is 2 replicas (`PROCESSOR_WORKER_REPLICAS`). The service publishes no host port, so it scales freely.

```bash
# Scale to 4 workers (immediate)
docker compose up -d --scale processor-worker=4

# Check workers are running
docker compose ps | grep processor

# Verify load distribution
docker compose logs --tail=20 processor-worker
```

**Permanent scaling** - edit `.env`:
```bash
PROCESSOR_WORKER_REPLICAS=4
```

Other lanes scale the same way: `MEDIA_WORKER_REPLICAS` (media-worker, default 1) and `BACKFILL_WORKER_REPLICAS` (processor-worker-backfill, `backfill` profile, default 2). Each replica opens its own database pool, so check `POSTGRES_POOL_SIZE` against `max_connections` ([Performance Tuning](performance-tuning.md#connection-pooling)).

Then restart:
```bash
docker compose up -d
```

### Scale Enrichment Workers

Enrichment workers are selected by the `enrichment-lite`, `enrichment-standard` and `enrichment-full` profiles (plus `llm-gateway` for the gateway-based ones). Each service handles specific tasks.

```bash
# Start enrichment profile (if not running)
docker compose --profile enrichment-lite up -d

# Replicas of the enrichment services come from one variable
ENRICHMENT_WORKER_REPLICAS=2
docker compose up -d
```

!!! warning "Fixed host ports"
    Enrichment services publish a fixed host port for metrics (for example `9199:9199` for fast-pool). More than one replica of such a service fails with a port conflict unless you remove or change that `ports:` mapping first (Prometheus reaches them over the Docker network and does not need it). Check `docker compose ps` after scaling.

**Worker specialization** (cannot mix tasks):

| Worker | Tasks | When to Scale |
|--------|-------|---------------|
| `enrichment-router` | Routes messages to task queues | Routing lag |
| `enrichment-fast-pool` | Embeddings, translation, entity matching | Large translation backlog |
| `enrichment-ai-tagging` | LLM topic tagging | Tagging backlog >5K |
| `enrichment-telegram` | Engagement polling, comments (in multi-account mode one `enrichment-telegram-<account>` per account) | Comment backlog |
| `enrichment-rss-validation` | Article validation | RSS correlation backlog |
| `enrichment-event-detection`, `enrichment-geolocation-llm`, `enrichment-cluster-*` | Events, LLM geolocation, clusters | Matching backlog |

### Scale Ollama (LLM)

!!! note "GPU gateway"
    With `LLM_PROVIDER=openai` the LLM runs on the gateway's GPU host, not in this stack. Its capacity is set by the per-lane parallel caps on the gateway; match `backlog_concurrency` to the batch lane cap. See [LLM Gateway](external-llm.md). The local `ollama` then only serves search embeddings, and `ollama-batch` is not used.

On a CPU-only Ollama setup it is the most common bottleneck. Two instances exist:
- `ollama` - Realtime inference (processor, API)
- `ollama-batch` - Background inference (enrichment)

**CPU allocation** (docker-compose.yml, shipped values shown):
```yaml
ollama:
  deploy:
    resources:
      limits:
        cpus: '6.0'
        memory: 8G
```

**Increase batch Ollama** (for enrichment):
```yaml
ollama-batch:
  deploy:
    resources:
      limits:
        cpus: '4.0'
        memory: 6G
```

After editing:
```bash
docker compose up -d ollama ollama-batch
```

**Switch to a faster model** (trades quality for speed):
```bash
# In .env
OLLAMA_MODEL=<smaller model>  # pull it first; see Performance Tuning
```

---

## Resource Planning

### By Channel Count

| Channels | Processor Workers | RAM | CPU Cores | Storage/Year |
|----------|-------------------|-----|-----------|--------------|
| 10-50 | 2 | 16GB | 4 | 200GB |
| 50-100 | 2-4 | 16GB | 8 | 500GB |
| 100-250 | 4 | 32GB | 8-12 | 1TB |
| 250-500 | 4-6 | 32-64GB | 12-16 | 2TB |
| 500+ | 6-8 | 64GB+ | 16+ | 4TB+ |

### By Message Volume

| Messages/Hour | Processor Workers | Ollama CPU | Notes |
|---------------|-------------------|------------|-------|
| <500 | 2 | 2 cores | Default config |
| 500-2000 | 2-4 | 4 cores | Monitor queue depth |
| 2000-5000 | 4 | 6 cores | Consider faster model |
| 5000+ | 4-8 | 8+ cores | May need second server |

### Component Memory Requirements

| Component | Minimum | Recommended | Notes |
|-----------|---------|-------------|-------|
| PostgreSQL | 2GB | 4GB | Shared buffers = 25% of RAM |
| Redis | 512MB | 1GB | Stream length determines usage |
| Ollama (realtime) | 4GB | 8GB | Model size dependent |
| Ollama (batch) | 4GB | 6GB | Can share model cache |
| Processor (per worker) | 512MB | 1GB | Includes sentence-transformers |
| Enrichment (per worker) | 1GB | 2GB | Varies by task |

---

## Bottleneck Identification

### Using Grafana Dashboards

**OSINT Platform Overview**:
- `Message Processing Rate` - Should be steady, not dropping
- `Queue Depth` - Should stay under 100 normally
- `LLM Latency` - Should be under 3 seconds

**Enrichment Workers**:
- `Backlog by Task Type` - Identify which queue is growing
- `Worker Throughput` - Messages processed per minute
- `Error Rate` - Should be <1%

### Using Prometheus Queries

```promql
# Queue depth (messages waiting, published by the processor)
osint_queue_messages_pending

# Processing rate (messages/second)
rate(osint_messages_processed_total[5m])

# LLM latency (p95)
histogram_quantile(0.95, rate(osint_llm_response_duration_seconds_bucket[5m]))

# Worker CPU usage
sum(rate(container_cpu_usage_seconds_total{name=~"osint-processor.*"}[5m])) by (name)
```

### Quick Bottleneck Check

```bash
# Check Redis queue depth
docker compose exec redis redis-cli XLEN telegram:messages:realtime
# Normal: <100, Warning: >500, Critical: >1000
docker compose exec redis redis-cli XLEN telegram:messages:backfill   # drains only when idle

# Check processor worker CPU
docker stats --no-stream | grep processor
# Warning if >80% sustained

# Check Ollama load
docker stats --no-stream | grep ollama
# Warning if >90% CPU sustained

# Check PostgreSQL connections
docker compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT count(*) FROM pg_stat_activity;"
# Warning if approaching max_connections (200); each replica holds up to
# POSTGRES_POOL_SIZE + POSTGRES_MAX_OVERFLOW
```

---

## Scaling Strategies

### Horizontal Scaling (Add Workers)

Best for:
- Message processing bottlenecks
- Enrichment backlogs
- CPU-bound tasks

```bash
# Processor: scale horizontally
docker compose up -d --scale processor-worker=4

# Enrichment: raise replicas (mind the fixed host ports above)
ENRICHMENT_WORKER_REPLICAS=2 docker compose up -d
```

### Vertical Scaling (More Resources)

Best for:
- LLM inference (Ollama)
- Database performance
- Memory pressure

Edit `docker-compose.yml` resource limits, then:
```bash
docker compose up -d
```

### Mixed Strategy (Recommended)

1. **Start with vertical** for Ollama (CPU/memory)
2. **Add horizontal** for processor workers
3. **Scale enrichment** workers as needed
4. **Upgrade database** last (usually not the bottleneck)

---

## Multi-Server Deployment

For very large deployments (500+ channels, 10K+ messages/hour):

### Architecture

```
Server 1 (Ingestion):
  - listener
  - processor-worker (4x)
  - ollama (realtime)
  - redis

Server 2 (Data):
  - postgres
  - minio

Server 3 (Enrichment):
  - enrichment workers
  - ollama-batch

Server 4 (Frontend):
  - api
  - frontend
  - monitoring stack
```

### Network Configuration

Use Docker Swarm or external Redis/PostgreSQL:

```bash
# .env on ingestion server
POSTGRES_HOST=server2.internal
REDIS_HOST=server1.internal
OLLAMA_BASE_URL=http://localhost:11434

# .env on enrichment server
POSTGRES_HOST=server2.internal
REDIS_HOST=server1.internal
OLLAMA_BATCH_URL=http://localhost:11435  # Local batch instance
```

!!! warning
    The stock compose file assumes one Docker network and uses service names (`postgres`, `redis`, `minio`) in many places beyond these variables (`DATABASE_URL`, `REDIS_URL`, MinIO endpoints). A multi-server split needs those overridden too, and PostgreSQL/Redis/MinIO must not be exposed beyond a private network. This layout is a design sketch, not a tested deployment.

---

## Monitoring After Scaling

After any scaling change:

1. **Watch queue depth** for 30 minutes
   ```bash
   watch -n 10 'docker compose exec redis redis-cli XLEN telegram:messages:realtime'
   ```

2. **Check worker logs** for errors
   ```bash
   docker compose logs --tail=50 -f processor-worker
   ```

3. **Verify Grafana dashboards** show improvement

4. **Check resource usage** stabilizes
   ```bash
   docker stats
   ```

---

## Common Scaling Mistakes

### Over-Scaling
- Adding workers without addressing bottleneck (LLM, database)
- Solution: Identify actual bottleneck first

### Under-Provisioning the LLM
- LLM is often the bottleneck, not workers
- Solution: Allocate 4-6 CPU cores to realtime Ollama, or move to a GPU gateway

### Ignoring Database Connections
- More workers = more connections
- Solution: Keep replicas x (`POSTGRES_POOL_SIZE` + `POSTGRES_MAX_OVERFLOW`) under `max_connections`

### Not Monitoring After Changes
- Changes may shift bottleneck elsewhere
- Solution: Always monitor for 30+ minutes after scaling

---

## Related Documentation

- [Performance Tuning](performance-tuning.md) - Optimize individual services
- [Monitoring](monitoring.md) - Set up alerting for scale triggers
- [Troubleshooting](troubleshooting.md) - Debug scaling issues
- [Docker Services Reference](../reference/docker-services.md) - Service details

---

**Quick Reference**:
```bash
# Scale processor workers
docker compose up -d --scale processor-worker=4

# Check queue depth
docker compose exec redis redis-cli XLEN telegram:messages:realtime

# Monitor resources
docker stats
```
