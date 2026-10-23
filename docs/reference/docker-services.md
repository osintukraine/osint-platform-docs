# Docker Services Reference

Complete reference for all Docker services in the OSINT Intelligence Platform.

**Source**: [`docker-compose.yml`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docker-compose.yml) in the platform repo

**Last Updated**: 2026-10-23

---

## Overview

The platform's containers are organized into functional categories:

| Category | Purpose |
|----------|---------|
| **Core Infrastructure** | Database, cache, storage |
| **LLM Layer** | Local Ollama, LLM gateway workers, backlog re-classification |
| **Application Services** | Data ingestion, processing, enrichment, API, frontend |
| **Monitoring Stack** | Metrics, logs, alerts, dashboards |
| **Infrastructure Exporters** | PostgreSQL and Redis metrics |
| **Notification System** | Push notifications |
| **Authentication** (optional) | Identity management and access control |

---

## Core Infrastructure

### postgres

| Field | Value |
|-------|-------|
| **Image** | `pgvector/pgvector:pg16` |
| **Container Name** | `osint-postgres` |
| **Purpose** | Primary database with vector extension for semantic search |
| **Ports** | `5432:5432` |
| **Profile** | (none - essential) |
| **Dependencies** | None |
| **Volumes** | `postgres_data:/var/lib/postgresql/data`<br>`./infrastructure/postgres/postgresql.conf:/etc/postgresql/postgresql.conf:ro`<br>`./infrastructure/postgres/init.sql:/docker-entrypoint-initdb.d/001-init.sql:ro` |
| **Environment** | `POSTGRES_DB`: osint_platform<br>`POSTGRES_USER`: postgres<br>`POSTGRES_PASSWORD`: postgres<br>`TZ`: UTC |
| **Healthcheck** | `pg_isready` every 10s |

### redis

| Field | Value |
|-------|-------|
| **Image** | `redis:7-alpine` |
| **Container Name** | `osint-redis` |
| **Purpose** | Message queue (Redis Streams) and caching layer |
| **Ports** | `6379:6379` |
| **Profile** | (none - essential) |
| **Dependencies** | None |
| **Volumes** | `redis_data:/data` |
| **Command** | `redis-server --appendonly yes --maxmemory 512mb --maxmemory-policy allkeys-lru` |
| **Healthcheck** | `redis-cli ping` every 10s |

### minio

| Field | Value |
|-------|-------|
| **Image** | `minio/minio:latest` |
| **Container Name** | `osint-minio` |
| **Purpose** | S3-compatible object storage for media archival (content-addressed SHA-256) |
| **Ports** | `9000:9000` (API)<br>`9001:9001` (Console) |
| **Profile** | (none - essential) |
| **Dependencies** | None |
| **Volumes** | `minio_data:/data` |
| **Environment** | `MINIO_ROOT_USER`: minioadmin<br>`MINIO_ROOT_PASSWORD`: minioadmin<br>`MINIO_PROMETHEUS_AUTH_TYPE`: public |
| **Healthcheck** | `curl -f http://localhost:9000/minio/health/live` every 30s |

### minio-init

| Field | Value |
|-------|-------|
| **Image** | `minio/mc:latest` |
| **Container Name** | `osint-minio-init` |
| **Purpose** | One-time bucket initialization for MinIO |
| **Dependencies** | `minio` (healthy) |
| **Lifecycle** | Exits after creating bucket |

---

## LLM Layer

Production uses an external **LLM gateway** (`LLM_PROVIDER=openai`), which is not a compose service: see [LLM Gateway](../operator-guide/external-llm.md). The services below are the local Ollama option plus the gateway workers.

### ollama

| Field | Value |
|-------|-------|
| **Image** | `ollama/ollama:latest` |
| **Container Name** | `osint-ollama` |
| **Purpose** | Local LLM (when `LLM_PROVIDER=ollama`) and search-query embeddings (`all-minilm`), which are needed even with the gateway |
| **Ports** | `11434` (`OLLAMA_PORT`) |
| **Profile** | (none) |
| **Volumes** | `./data/ollama:/root/.ollama` |

### ollama-init

| Field | Value |
|-------|-------|
| **Container Name** | `osint-ollama-init` |
| **Purpose** | Pulls the models configured for `ollama` |
| **Profile** | (none) |

### ollama-batch and ollama-batch-init

| Field | Value |
|-------|-------|
| **Container Names** | `osint-ollama-batch`, `osint-ollama-batch-init` |
| **Purpose** | CPU-era second Ollama for batch enrichment (AI tagging, RSS validation) |
| **Ports** | `11435` (`OLLAMA_BATCH_PORT`) |
| **Profiles** | `enrichment-standard`, `enrichment-full`, `enrichment` |
| **Notes** | Not used in production: with the gateway, batch work runs through the `llm-gateway` services |

### backlog-worker

| Field | Value |
|-------|-------|
| **Build** | `services/processor/Dockerfile` (command `python -m src.backlog_worker`) |
| **Purpose** | Re-classifies `pending` and spam-flagged messages on the gateway batch lane. See [Backlog Re-classification](../operator-guide/backlog-reclassification.md) |
| **Profile** | `llm-gateway` |
| **Key settings** | `LLM_GATEWAY_BATCH_API_KEY`, `LLM_GATEWAY_BATCH_MODEL`, `BACKLOG_RECLASSIFY_ENABLED`, `BACKLOG_CONCURRENCY` |
| **Healthcheck** | `curl http://localhost:8004/metrics` |

---

## Application Services

### listener

| Field | Value |
|-------|-------|
| **Build** | `services/listener/Dockerfile` |
| **Container Name** | `osint-listener` |
| **Purpose** | Monitors Telegram channels and pushes messages to Redis Streams |
| **Ports** | `8001:8001` (Prometheus metrics) |
| **Profile** | (none - essential) |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Volumes** | `./services/listener:/app/services/listener`<br>`./shared/python:/app/shared/python`<br>`./telegram_sessions:/app/sessions` |
| **Environment** | `TELEGRAM_API_ID`, `TELEGRAM_API_HASH`, `TELEGRAM_PHONE`<br>`BACKFILL_ENABLED`: false<br>`TRANSLATION_ENABLED`: false (moved to processor) |
| **Notes** | Owns Telegram session - never create standalone clients |

### listener-russia

| Field | Value |
|-------|-------|
| **Build** | `services/listener/Dockerfile` |
| **Container Name** | `osint-listener-russia` |
| **Purpose** | Multi-account setup - monitors Russia channels |
| **Ports** | `8011:8001` (Prometheus metrics) |
| **Profile** | `multi-account` |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Environment** | `TELEGRAM_API_ID_RUSSIA`, `TELEGRAM_API_HASH_RUSSIA`<br>`SOURCE_ACCOUNT`: russia |

### listener-ukraine

| Field | Value |
|-------|-------|
| **Build** | `services/listener/Dockerfile` |
| **Container Name** | `osint-listener-ukraine` |
| **Purpose** | Multi-account setup - monitors Ukraine channels |
| **Ports** | `8012:8001` (Prometheus metrics) |
| **Profile** | `multi-account` |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Environment** | `TELEGRAM_API_ID_UKRAINE`, `TELEGRAM_API_HASH_UKRAINE`<br>`SOURCE_ACCOUNT`: ukraine |

### processor-worker

| Field | Value |
|-------|-------|
| **Build** | `services/processor/Dockerfile` |
| **Purpose** | Real-time message processing: embedding spam filter, entity extraction, LLM classification (realtime lane), translation, persistence. Media is downloaded inline unless the [media lane](../operator-guide/media-lanes.md) is on |
| **Profile** | (none - essential) |
| **Replicas** | 2 (`PROCESSOR_WORKER_REPLICAS`) |
| **Volumes** | `./data/models:/models:ro` (see [Embedding Models](../operator-guide/model-provisioning.md)), media buffer |
| **Environment** | `LLM_PROVIDER`, `LLM_GATEWAY_*`, `EMBEDDING_SPAM_MODEL`, `HF_HUB_OFFLINE=1` |
| **Healthcheck** | `curl http://localhost:8002/metrics` |

### media-worker

| Field | Value |
|-------|-------|
| **Build** | `services/processor/Dockerfile` (command `python -m src.media_worker`) |
| **Purpose** | Downloads media for messages with `media_status` `pending`; idles unless the `media_lane_enabled` admin setting is on. See [Media Lanes](../operator-guide/media-lanes.md) |
| **Profile** | (none - core) |
| **Replicas** | 1 (`MEDIA_WORKER_REPLICAS`) |
| **Healthcheck** | `curl http://localhost:8003/metrics` |

### media-sync

| Field | Value |
|-------|-------|
| **Build** | `services/media-sync/Dockerfile` |
| **Purpose** | Moves archived media from the local buffer to long-term storage (`MEDIA_SYNC_MODE`, default `filesystem`). See [Media Sync](../developer-guide/services/media-sync.md) |
| **Profile** | (none - core) |

### api

| Field | Value |
|-------|-------|
| **Build** | `services/api/Dockerfile` |
| **Container Name** | `osint-api` |
| **Purpose** | REST API with FastAPI - serves data, semantic search, admin console |
| **Ports** | `8000:8000` (API)<br>`8003:8003` (Prometheus metrics) |
| **Profile** | (none - essential) |
| **Dependencies** | `postgres` (healthy), `redis` (healthy), `ollama` (healthy) |
| **Volumes** | `./services/api:/app/services/api`<br>`./shared/python:/app/shared/python` |
| **Environment** | `API_WORKERS`: 4<br>`AUTH_PROVIDER`: none<br>`AUTH_REQUIRED`: false |
| **Healthcheck** | `curl http://localhost:8000/health` every 30s |

### frontend

| Field | Value |
|-------|-------|
| **Build** | `services/frontend-nextjs/Dockerfile` |
| **Container Name** | `osint-frontend` |
| **Purpose** | Next.js frontend with Bun runtime |
| **Ports** | `3000:3000` |
| **Profile** | (none - essential) |
| **Dependencies** | `api` (healthy), `minio` (healthy) |
| **Environment** | `API_URL`: http://api:8000 (server-side)<br>`NEXT_PUBLIC_API_URL`: http://localhost:8000 (client-side)<br>`NODE_ENV`: production |
| **Healthcheck** | Process check every 30s |

### rss-ingestor

| Field | Value |
|-------|-------|
| **Build** | `services/rss-ingestor/Dockerfile` |
| **Container Name** | `osint-rss-ingestor` |
| **Purpose** | Fetches and enriches RSS feeds for event correlation |
| **Profile** | (none - essential) |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Volumes** | `./services/rss-ingestor:/app/services/rss-ingestor`<br>`./shared/python:/app/shared/python`<br>`./data/huggingface:/app/.cache/huggingface` |

### analytics

| Field | Value |
|-------|-------|
| **Build** | `services/analytics/Dockerfile` |
| **Container Name** | `osint-analytics` |
| **Purpose** | Social graph data collection - engagement polling, comment scraping, view tracking |
| **Profiles** | `enrichment-lite`, `enrichment-standard`, `enrichment-full` |
| **Dependencies** | `postgres` (healthy), `listener` (healthy) |
| **Volumes** | `./telegram_sessions:/app/sessions`<br>`./shared/python:/shared/python:ro`<br>`./data/logs/analytics:/var/log/analytics` |
| **Environment** | `TELEGRAM_SESSION_PATH`: /app/sessions (uses analytics.session) |

### nocodb

| Field | Value |
|-------|-------|
| **Image** | `nocodb/nocodb:latest` |
| **Container Name** | `osint-nocodb` |
| **Purpose** | Airtable-like database UI for database management |
| **Ports** | `8080:8080` |
| **Profile** | `tools` |
| **Dependencies** | `postgres` (healthy) |
| **Volumes** | `nocodb_data:/usr/app/data` |
| **Environment** | `NC_ADMIN_EMAIL`: admin@osint.local<br>`NC_DISABLE_TELE`: true |
| **Healthcheck** | `curl http://localhost:8080/api/v1/health` every 30s |

---

## Enrichment Workers

Enrichment workers are selected by the `enrichment-lite`, `enrichment-standard` and `enrichment-full` profiles (see Profiles below) and process messages in batches with time budgets. The LLM-based workers other than fact-check and RSS validation (AI tagging, event detection, geolocation, cluster validation) still default to the Ollama endpoint `OLLAMA_BATCH_URL` and small CPU-era models; production runs the `enrichment-lite` set plus the `llm-gateway` workers.

### enrichment-ai-tagging

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-ai-tagging` |
| **Purpose** | LLM-based tag generation (themes, tactics, sentiment) |
| **Ports** | `9196:9196` (Prometheus metrics) |
| **Dependencies** | `postgres` (healthy), `ollama-batch` (healthy) |
| **Environment** | `AI_TAGGING_MODEL`: qwen2.5:1.5b<br>`BATCH_SIZE`: 4<br>`TIME_BUDGET_SECONDS`: 120 |

### enrichment-event-detection

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-event-detection` |
| **Purpose** | Creates events from RSS and matches Telegram messages |
| **Ports** | `9099:9099` (Prometheus metrics) |
| **Dependencies** | `postgres` (healthy), `ollama-batch` (healthy) |
| **Environment** | `EVENT_DETECTION_MODEL`: qwen2.5:3b<br>`BATCH_SIZE`: 10<br>`TIME_BUDGET_SECONDS`: 300 |

### enrichment-router

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-router` |
| **Purpose** | Routes messages to Redis queues (Phase 3 architecture) |
| **Ports** | `9198:9198` (Prometheus metrics) |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Environment** | `ROUTER_POLL_INTERVAL`: 30<br>`ROUTER_BATCH_SIZE`: 100 |

### enrichment-rss-validation

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-rss-validation` |
| **Purpose** | LLM validation of RSS correlations (confirms, contradicts, context, alternative) on the batch lane |
| **Ports** | `9197` (Prometheus metrics) |
| **Profiles** | `enrichment-standard`, `enrichment-full`, `llm-gateway` |
| **Environment** | `RSS_VALIDATION_MODEL`, `LLM_GATEWAY_BATCH_API_KEY`, `RSS_VALIDATION_TIME_BUDGET_SECONDS`, `RSS_VALIDATION_BATCH_SIZE`, `RSS_VALIDATION_CYCLE_INTERVAL_SECONDS` |
| **Notes** | Only recent posts inside the correlation window are queued; the similarity floor is the admin setting `validation_min_similarity` |

### enrichment-fact-check

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-fact-check` |
| **Purpose** | Audits a sample of recent LLM-classified messages on the audit lane and auto-reclassifies on high confidence. See [Fact-Check](../developer-guide/services/fact-check.md) |
| **Ports** | `9203` (Prometheus metrics) |
| **Profiles** | `enrichment-standard`, `enrichment-full`, `llm-gateway` |
| **Environment** | `FACT_CHECK_MODEL`, `LLM_GATEWAY_AUDIT_API_KEY`, `FACT_CHECK_NUM_PREDICT`, `FACT_CHECK_LOOKBACK_DAYS`, `FACT_CHECK_AUTO_ACTION_THRESHOLD`, `FACT_CHECK_TIME_BUDGET_SECONDS`, `FACT_CHECK_CYCLE_INTERVAL_SECONDS` |

### enrichment-fast-pool

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-fast-pool` |
| **Purpose** | CPU-bound tasks: embedding generation, translation, entity matching |
| **Ports** | `9199:9199` (Prometheus metrics) |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Environment** | `EMBEDDING_MODEL`: all-MiniLM-L6-v2<br>`BATCH_SIZE`: 50<br>`TIME_BUDGET_SECONDS`: 60<br>`DEEPL_API_KEY` |

### enrichment-telegram

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-telegram` |
| **Purpose** | Telegram API tasks: engagement polling, social graph, comment fetching |
| **Ports** | `9200:9200` (Prometheus metrics) |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Volumes** | `./telegram_sessions:/app/sessions` |
| **Environment** | `TELEGRAM_SESSION_PATH`: /app/sessions/enrichment.session<br>`BATCH_SIZE`: 20<br>`RATE_LIMIT_PER_SECOND`: 20 |

### enrichment-decision

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-decision` |
| **Purpose** | Decision verification and reprocessing tasks |
| **Ports** | `9201:9201` (Prometheus metrics) |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Environment** | `BATCH_SIZE`: 50<br>`TIME_BUDGET_SECONDS`: 60 |

### enrichment-maintenance

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-maintenance` |
| **Purpose** | Hourly maintenance: channel cleanup, quarantine processing, discovery evaluation |
| **Ports** | `9202:9202` (Prometheus metrics) |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Environment** | `BATCH_SIZE`: 100<br>`TIME_BUDGET_SECONDS`: 120<br>`CYCLE_INTERVAL_SECONDS`: 300 |

### enrichment-geolocation-llm

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-geolocation-llm` |
| **Purpose** | LLM-based location extraction from message content |
| **Ports** | `9099:9099` (Prometheus metrics) |
| **Profile** | `enrichment-standard`, `enrichment-full` |
| **Dependencies** | `postgres` (healthy), `ollama-batch` (healthy) |
| **Environment** | `GEOLOCATION_MODEL`: qwen2.5:3b<br>`BATCH_SIZE`: 5<br>`TIME_BUDGET_SECONDS`: 120 |

### enrichment-cluster-detection

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-cluster-detection` |
| **Purpose** | Event Detection V3: velocity-based cluster detection + auxiliary tasks (archiver, tier updater) |
| **Ports** | `9211:9211` (Prometheus metrics) |
| **Profile** | `enrichment-standard`, `enrichment-full` |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Environment** | `CLUSTER_VELOCITY_THRESHOLD`: 2.0<br>`CLUSTER_TIME_WINDOW_HOURS`: 2<br>`CLUSTER_SIMILARITY_THRESHOLD`: 0.80<br>`BATCH_SIZE`: 50<br>`TIME_BUDGET_SECONDS`: 120 |
| **Tasks** | `cluster_detection`, `cluster_archiver`, `cluster_tier_updater` |

### enrichment-cluster-validation

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-cluster-validation` |
| **Purpose** | LLM-based cluster validation using claim analysis (factual/rumor/propaganda) |
| **Ports** | `9212:9212` (Prometheus metrics) |
| **Profile** | `enrichment-standard`, `enrichment-full` |
| **Dependencies** | `postgres` (healthy), `ollama-batch` (healthy) |
| **Environment** | `VALIDATION_MODEL`: qwen2.5:3b<br>`CLUSTER_RUMOR_TTL_HOURS`: 24<br>`BATCH_SIZE`: 5<br>`TIME_BUDGET_SECONDS`: 120<br>`LLM_TIMEOUT`: 180 |
| **Tasks** | `cluster_validation` |

---

## OpenSanctions Entity Matching

All OpenSanctions services use the `opensanctions` profile.

### yente-index

| Field | Value |
|-------|-------|
| **Image** | `docker.elastic.co/elasticsearch/elasticsearch:8.15.0` |
| **Container Name** | `osint-yente-index` |
| **Purpose** | ElasticSearch for OpenSanctions entity index (8-10GB) |
| **Profile** | `opensanctions` |
| **Volumes** | `yente_elasticsearch_data:/usr/share/elasticsearch/data` |
| **Environment** | `discovery.type`: single-node<br>`ES_JAVA_OPTS`: -Xms2g -Xmx2g |
| **Healthcheck** | Cluster health check every 30s |

### yente

| Field | Value |
|-------|-------|
| **Image** | `ghcr.io/opensanctions/yente:latest` |
| **Container Name** | `osint-yente` |
| **Purpose** | Self-hosted OpenSanctions API for entity matching (no rate limits) |
| **Profile** | `opensanctions` |
| **Dependencies** | `yente-index` (healthy) |
| **Volumes** | `./infrastructure/yente/datasets.yml:/app/datasets.yml:ro` |
| **Environment** | `YENTE_INDEX_URL`: http://yente-index:9200<br>`YENTE_MAX_BATCH`: 100<br>`YENTE_MATCH_FUZZY`: true |
| **Healthcheck** | `curl http://localhost:8000/healthz` every 30s (5min start period) |

### opensanctions

| Field | Value |
|-------|-------|
| **Build** | `services/opensanctions/Dockerfile` |
| **Container Name** | `osint-opensanctions` |
| **Purpose** | Entity intelligence service - enriches entities with sanctions data |
| **Profile** | `opensanctions` |
| **Dependencies** | `postgres` (healthy), `yente` (healthy) |
| **Volumes** | `./services/opensanctions:/app/services/opensanctions`<br>`./shared/python:/app/shared/python`<br>`./data/huggingface:/app/.cache/huggingface` |
| **Environment** | `OPENSANCTIONS_BACKEND`: yente (or api)<br>`OPENSANCTIONS_MATCH_THRESHOLD`: 0.85<br>`OPENSANCTIONS_BATCH_SIZE`: 10 |

### entity-ingestion

| Field | Value |
|-------|-------|
| **Build** | `services/entity-ingestion/Dockerfile` |
| **Container Name** | `osint-entity-ingestion` |
| **Purpose** | CSV to PostgreSQL entity ingestion (ArmyGuide, Root.NK, ODIN) |
| **Profile** | `opensanctions` |
| **Dependencies** | `postgres` (healthy) |
| **Volumes** | `./data/entities:/data/entities`<br>`./data/huggingface:/app/.cache/huggingface` |
| **Environment** | `ENTITY_CSV_DIR`: /data/entities/csv/<br>`ENTITY_SCAN_INTERVAL`: 300<br>`ENTITY_BATCH_SIZE`: 100 |

---

## Monitoring Stack

All monitoring services use the `monitoring` profile.

### prometheus

| Field | Value |
|-------|-------|
| **Image** | `prom/prometheus:v2.48.0` |
| **Container Name** | `osint-prometheus` |
| **Purpose** | Metrics collection and alerting |
| **Ports** | `9090:9090` |
| **Profile** | `monitoring` |
| **Volumes** | `./infrastructure/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro`<br>`./infrastructure/prometheus/rules:/etc/prometheus/rules:ro`<br>`prometheus_data:/prometheus` |
| **Command** | 30-day retention, lifecycle API enabled |
| **Healthcheck** | `wget http://localhost:9090/-/healthy` every 30s |

### grafana

| Field | Value |
|-------|-------|
| **Image** | `grafana/grafana:11.4.0` |
| **Container Name** | `osint-grafana` |
| **Purpose** | Visualization and dashboards |
| **Ports** | `3001:3000` (avoiding conflict with frontend) |
| **Profile** | `monitoring` |
| **Dependencies** | `prometheus` (healthy) |
| **Volumes** | `grafana_data:/var/lib/grafana`<br>`./infrastructure/grafana/provisioning:/etc/grafana/provisioning:ro`<br>`./infrastructure/grafana/dashboards:/var/lib/grafana/dashboards:ro` |
| **Environment** | `GF_SECURITY_ADMIN_USER`: admin<br>`GF_SECURITY_ADMIN_PASSWORD`: admin<br>`GF_DATABASE_TYPE`: sqlite3 |
| **Healthcheck** | `wget http://localhost:3000/api/health` every 30s |

### alertmanager

| Field | Value |
|-------|-------|
| **Image** | `prom/alertmanager:v0.26.0` |
| **Container Name** | `osint-alertmanager` |
| **Purpose** | Alert routing to ntfy via notifier service |
| **Ports** | `9093:9093` |
| **Profile** | `monitoring` |
| **Dependencies** | `prometheus` (healthy), `notifier` (healthy) |
| **Volumes** | `./infrastructure/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro`<br>`alertmanager_data:/alertmanager` |
| **Healthcheck** | `wget http://localhost:9093/-/healthy` every 30s |

### notifier

| Field | Value |
|-------|-------|
| **Build** | `services/notifier/Dockerfile` |
| **Container Name** | `osint-notifier` |
| **Purpose** | Routes Redis events to ntfy topics with batching |
| **Ports** | `9094:9094` (Prometheus metrics) |
| **Profile** | `monitoring` |
| **Dependencies** | `redis` (healthy), `ntfy` (healthy) |
| **Volumes** | `./services/notifier:/app/services/notifier`<br>`./shared/python:/app/shared/python` |
| **Environment** | `NTFY_URL`: http://ntfy:80<br>`AGGREGATOR_ENABLED`: true<br>`AGGREGATOR_BATCH_INTERVAL`: 300 |
| **Healthcheck** | `curl http://localhost:8000/health` every 30s |

### ntfy

| Field | Value |
|-------|-------|
| **Image** | `binwiederhier/ntfy:v2.8.0` |
| **Container Name** | `osint-ntfy` |
| **Purpose** | Self-hosted notification delivery (web/mobile push notifications) |
| **Ports** | `8090:80` (Web UI)<br>`9096:9095` (Prometheus metrics) |
| **Profile** | `monitoring` |
| **Volumes** | `ntfy_cache:/var/cache/ntfy`<br>`ntfy_data:/var/lib/ntfy` |
| **Environment** | `NTFY_ENABLE_METRICS`: true<br>`NTFY_CACHE_DURATION`: 168h<br>`NTFY_VISITOR_REQUEST_LIMIT_EXEMPT_HOSTS`: Docker networks |
| **Healthcheck** | `wget http://localhost:80/v1/health` every 30s |

### cadvisor

| Field | Value |
|-------|-------|
| **Image** | `gcr.io/cadvisor/cadvisor:v0.51.0` |
| **Container Name** | `osint-cadvisor` |
| **Purpose** | Container resource metrics (CPU, memory, network) |
| **Ports** | `8081:8080` |
| **Profile** | `monitoring` |
| **Volumes** | `/:/rootfs:ro`<br>`/var/run:/var/run:ro`<br>`/sys:/sys:ro`<br>`/var/lib/docker/:/var/lib/docker:ro` |
| **Privileged** | Yes (required for container inspection) |
| **Command** | 10s housekeeping, docker-only, minimal metrics |

### loki

| Field | Value |
|-------|-------|
| **Image** | `grafana/loki:3.3.2` |
| **Container Name** | `osint-loki` |
| **Purpose** | Log storage and querying (used from Grafana) |
| **Ports** | `3100` (`LOKI_PORT`) |
| **Profile** | `monitoring` |

### promtail

| Field | Value |
|-------|-------|
| **Image** | `grafana/promtail:3.3.2` |
| **Container Name** | `osint-promtail` |
| **Purpose** | Ships container logs to Loki |
| **Profile** | `monitoring` |

### node-exporter

| Field | Value |
|-------|-------|
| **Image** | `prom/node-exporter:v1.7.0` |
| **Container Name** | `osint-node-exporter` |
| **Purpose** | Host system metrics (CPU, memory, disk, network) |
| **Ports** | `9100:9100` |
| **Profile** | `monitoring` |
| **Volumes** | `/proc:/host/proc:ro`<br>`/sys:/host/sys:ro`<br>`/:/rootfs:ro` |
| **Command** | Excludes Docker virtual interfaces |
| **Healthcheck** | `wget http://localhost:9100/` every 30s |

---

## Infrastructure Exporters

### postgres-exporter

| Field | Value |
|-------|-------|
| **Image** | `prometheuscommunity/postgres-exporter:v0.15.0` |
| **Container Name** | `osint-postgres-exporter` |
| **Purpose** | PostgreSQL metrics for Prometheus |
| **Ports** | `9187:9187` |
| **Profile** | `monitoring` |
| **Dependencies** | `postgres` (healthy) |
| **Volumes** | `./infrastructure/postgres-exporter/queries.yaml:/etc/postgres_exporter/queries.yaml:ro` |
| **Environment** | `DATA_SOURCE_NAME`: PostgreSQL connection string |
| **Healthcheck** | `wget http://localhost:9187/metrics` every 30s |

### redis-exporter

| Field | Value |
|-------|-------|
| **Image** | `oliver006/redis_exporter:v1.55.0` |
| **Container Name** | `osint-redis-exporter` |
| **Purpose** | Redis queue metrics for Prometheus |
| **Ports** | `9121:9121` |
| **Profile** | `monitoring` |
| **Dependencies** | `redis` |
| **Environment** | `REDIS_ADDR`: redis:6379 |
| **Healthcheck** | Disabled (distroless image) |

---

## Authentication & Access Control

All authentication services use the `auth` profile.

### kratos-migrate

| Field | Value |
|-------|-------|
| **Image** | `oryd/kratos:v1.1.0` |
| **Container Name** | `osint-kratos-migrate` |
| **Purpose** | Database migrations for Kratos (identity management) |
| **Profile** | `auth` |
| **Dependencies** | `postgres` (healthy) |
| **Lifecycle** | Runs once on startup |

### kratos

| Field | Value |
|-------|-------|
| **Image** | `oryd/kratos:v1.1.0` |
| **Container Name** | `osint-kratos` |
| **Purpose** | Identity management (registration, login, OAuth) |
| **Ports** | `4433:4433` (Public API)<br>`4434:4434` (Admin API) |
| **Profile** | `auth` |
| **Dependencies** | `postgres` (healthy), `kratos-migrate` (completed) |
| **Volumes** | `./infrastructure/kratos:/etc/config/kratos:ro` |
| **Environment** | OAuth providers (Google, GitHub)<br>SMTP configuration |
| **Healthcheck** | `wget http://localhost:4433/health/ready` every 30s |

### oathkeeper

| Field | Value |
|-------|-------|
| **Image** | `oryd/oathkeeper:v0.40.6` |
| **Container Name** | `osint-oathkeeper` |
| **Purpose** | Access proxy and request authorization |
| **Ports** | `4455:4455` (Proxy)<br>`4456:4456` (API) |
| **Profile** | `auth` |
| **Dependencies** | `kratos` (healthy) |
| **Volumes** | `./infrastructure/oathkeeper:/etc/config/oathkeeper:ro` |
| **Healthcheck** | `wget http://localhost:4456/health/ready` every 30s |

### caddy

| Field | Value |
|-------|-------|
| **Image** | `caddy:2.7-alpine` |
| **Container Name** | `osint-caddy` |
| **Purpose** | Reverse proxy with automatic HTTPS |
| **Ports** | `80:80` (HTTP)<br>`443:443` (HTTPS)<br>`2019:2019` (Admin API) |
| **Profile** | `auth` |
| **Dependencies** | `frontend`, `api` |
| **Volumes** | `./infrastructure/caddy/Caddyfile.local:/etc/caddy/Caddyfile:ro` |

---

## Documentation (tools profile)

### mkdocs

| Field | Value |
|-------|-------|
| **Image** | `squidfunk/mkdocs-material:latest` |
| **Container Name** | `osint-mkdocs` |
| **Purpose** | Documentation site with Material theme |
| **Ports** | `8001:8000` |
| **Profile** | `tools` |
| **Volumes** | `.:/docs:ro` (entire project) |
| **Command** | `serve --dev-addr=0.0.0.0:8000` |
| **Healthcheck** | `wget http://127.0.0.1:8000` every 60s |

---

## Network & Volumes

### Network

| Name | Driver |
|------|--------|
| `backend` | bridge |

### Volumes

| Volume | Purpose | Type |
|--------|---------|------|
| `postgres_data` | PostgreSQL database | Docker volume |
| `redis_data` | Redis persistence | Docker volume |
| `minio_data` | S3 object storage | Docker volume |
| `telegram_sessions` | Telegram session files | Bind mount (`./telegram_sessions`) |
| `prometheus_data` | Prometheus metrics | Docker volume |
| `grafana_data` | Grafana dashboards | Docker volume |
| `ntfy_cache` | ntfy message cache | Docker volume |
| `ntfy_data` | ntfy persistence | Docker volume |
| `nocodb_data` | NocoDB metadata | Docker volume |
| `alertmanager_data` | AlertManager state | Docker volume |
| `yente_elasticsearch_data` | OpenSanctions index (8-10GB) | Docker volume |

---

## Profiles

Profiles select which service groups start. Set them persistently with `COMPOSE_PROFILES` in `.env` (production uses `enrichment-lite,monitoring,auth,llm-gateway`). Services without a profile (postgres, redis, minio, ollama, listener, processor-worker, media-worker, media-sync, api, frontend, rss-ingestor) always start.

| Profile | Adds |
|---------|------|
| `enrichment-lite` | router, fast-pool, telegram, decision, maintenance, cluster-detection, analytics |
| `enrichment-standard` | Lite plus AI tagging, event detection, fact-check, geolocation-llm, cluster-validation, RSS validation, ollama-batch, opensanctions, entity-ingestion |
| `enrichment-full` | Same service set as standard |
| `enrichment` | Legacy name, same set as standard |
| `llm-gateway` | backlog-worker, enrichment-fact-check, enrichment-rss-validation. See [LLM Gateway](../operator-guide/external-llm.md) |
| `monitoring` | prometheus, grafana, alertmanager, loki, promtail, exporters, cadvisor, notifier, ntfy |
| `notifications` | notifier and ntfy without the rest of monitoring |
| `auth` | kratos, kratos-migrate, kratos-init, oathkeeper, crowdsec, caddy |
| `cloudflare` | crowdsec-cloudflare-bouncer |
| `opensanctions` | yente-index, yente, opensanctions, entity-ingestion |
| `multi-account` | listener-russia, listener-ukraine, shadow-sync |
| `dr` | shadow-sync (disaster recovery) |
| `backfill` | processor-worker-backfill |
| `tools` | nocodb, mkdocs |

**Start with profiles:**
```bash
docker compose --profile monitoring --profile llm-gateway up -d
```

**Check what is running:**
```bash
docker compose ps
```

---

## Port Reference

Host ports from `docker-compose.yml` (most are overridable in `.env`):

| Service | Port | Notes |
|---------|------|-------|
| Frontend | 3000 | |
| API | 8000 (metrics 8003) | Docs at `/docs` |
| PostgreSQL | 5432 | |
| Redis | 6379 | |
| MinIO / console | 9000 / 9001 | |
| Ollama / ollama-batch | 11434 / 11435 | |
| Prometheus | 9090 | |
| Grafana | 3000 (`GRAFANA_PORT`) | Same default as the frontend: change one if they share a host |
| Alertmanager | 9093 | |
| Loki | 3100 | |
| ntfy | 127.0.0.1:8090 | Bound to localhost |
| NocoDB | 8080 | `tools` profile |
| MkDocs | 8001 (`MKDOCS_PORT`) | `tools` profile |
| Listener metrics | 8001 | |
| Enrichment workers | 9099, 9196 to 9203, 9210 to 9212 | One metrics port per worker |

---

## Dependencies

- `listener` publishes to Redis; `processor-worker` consumes it and writes to PostgreSQL and MinIO; `api` and `frontend` read from them.
- `processor-worker`, `backlog-worker`, `enrichment-fact-check` and `enrichment-rss-validation` call the LLM provider (gateway or Ollama).
- `media-worker` and `backlog-worker` use `messages` rows in PostgreSQL as their queue.
- Monitoring: `prometheus` feeds `grafana` and `alertmanager` (then `notifier`, `ntfy`); `promtail` feeds `loki`, also shown in `grafana`.

---

## Environment Variables

[`.env.example`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/.env.example) is the canonical reference. [Environment Variables](environment-variables.md) gives an overview of the groups.

---

## Healthcheck Summary

Most services use a 30 s interval, 10 s timeout and 3 retries. Methods: HTTP (`curl`, `wget`) against a metrics or health endpoint, `pg_isready`, `redis-cli ping`, or a Python process check in slim images.

---

## Related Documentation

- [Architecture Overview](../developer-guide/architecture.md)
- [Installation Guide](../operator-guide/installation.md)
- [Monitoring Setup](../operator-guide/monitoring.md)
- [LLM Gateway](../operator-guide/external-llm.md)
