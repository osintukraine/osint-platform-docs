# Docker Services Reference

Complete reference for all Docker services in the OSINT Intelligence Platform.

**Source**: [`docker-compose.yml`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docker-compose.yml) in the platform repo

**Last Updated**: 2026-10-27

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
| **Image** | `osint-platform-postgres:latest` (built from `infrastructure/docker/postgres-pgvector-postgis.Dockerfile`: PostgreSQL 16 with pgvector and PostGIS) |
| **Container Name** | `osint-postgres` |
| **Purpose** | Primary database with vector extension for semantic search |
| **Ports** | `${POSTGRES_PORT:-5432}:5432` |
| **Profile** | (none - essential) |
| **Dependencies** | None |
| **Command** | `postgres -c config_file=/etc/postgresql/postgresql.conf` plus `shared_buffers`, `work_mem`, `effective_cache_size` from `POSTGRES_SHARED_BUFFERS` (1GB), `POSTGRES_WORK_MEM` (128MB), `POSTGRES_EFFECTIVE_CACHE` (4GB) |
| **Volumes** | `postgres_data:/var/lib/postgresql/data`<br>`./infrastructure/postgres/postgresql.conf:/etc/postgresql/postgresql.conf:ro`<br>`./infrastructure/postgres/init.sql:/docker-entrypoint-initdb.d/001-init.sql:ro` |
| **Environment** | `POSTGRES_DB`: osint_platform<br>`POSTGRES_USER`: postgres (compose default; `.env.example` sets osint_user)<br>`POSTGRES_PASSWORD`: required, no default<br>`TZ`: UTC |
| **Healthcheck** | `pg_isready` every 10s |

### redis

| Field | Value |
|-------|-------|
| **Image** | `redis:7.4.11-alpine` |
| **Container Name** | `osint-redis` |
| **Purpose** | Message queue (Redis Streams) and caching layer |
| **Ports** | `${REDIS_PORT:-6379}:6379` |
| **Profile** | (none - essential) |
| **Dependencies** | None |
| **Volumes** | `redis_data:/data` |
| **Command** | `redis-server --maxmemory ${REDIS_MAXMEMORY:-1gb} --maxmemory-policy allkeys-lru --appendonly yes` |
| **Healthcheck** | `redis-cli ping` every 10s |

### minio

| Field | Value |
|-------|-------|
| **Image** | `pgsty/silo:RELEASE.2026-09-16T00-00-00Z` (S3-compatible MinIO fork) |
| **Container Name** | `osint-minio` |
| **Purpose** | S3-compatible object storage for media archival (content-addressed SHA-256) |
| **Ports** | `${MINIO_PORT:-9000}:9000` (API)<br>`${MINIO_CONSOLE_PORT:-9001}:9001` (Console) |
| **Profile** | (none - essential) |
| **Dependencies** | None |
| **Volumes** | `${HETZNER_MOUNT_PATH:-./data/minio}:/data` (bind mount; the Hetzner path when set) |
| **Environment** | `MINIO_ROOT_USER`: minioadmin (default)<br>`MINIO_ROOT_PASSWORD`: required, no default<br>`MINIO_PROMETHEUS_AUTH_TYPE`: public |
| **Healthcheck** | `curl -f http://localhost:9000/minio/health/live` every 30s |

### minio-init

| Field | Value |
|-------|-------|
| **Image** | `pgsty/mc:RELEASE.2026-09-16T00-00-00Z` |
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
| **Build** | `infrastructure/ollama/Dockerfile.init` |
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
| **Key settings** | `LLM_GATEWAY_API_KEY`, `LLM_GATEWAY_MODEL`, `BACKLOG_RECLASSIFY_ENABLED`, `BACKLOG_CONCURRENCY`, `BACKLOG_SPAM_FIRST` |
| **Healthcheck** | `curl http://localhost:8004/metrics` |

### backlog-enrich and backlog-reprocess

| Field | Value |
|-------|-------|
| **Build** | `services/processor/Dockerfile` (commands `python -m src.backlog_jobs_worker enrich` and `... reprocess`) |
| **Purpose** | Gateway batch-lane job workers that run one job over archived messages, newest first, paced by live admin settings and braked when live classification latency rises (night concurrency 22:00 to 06:00 UTC). `enrich` fills in missing enrichment; `reprocess` re-classifies by group (`BACKLOG_REPROCESS_GROUP`) and samples results for the fact-check council (`COUNCIL_SAMPLE_RATE`) |
| **Profile** | `llm-gateway` |
| **Key settings** | `BACKLOG_ENRICH_FROM`, `BACKLOG_ENRICH_TO`, `BACKLOG_ENRICH_ENABLED`, `BACKLOG_ENRICH_CONCURRENCY`, `BACKLOG_REPROCESS_ENABLED`, `BACKLOG_REPROCESS_GROUP`, `BACKLOG_REPROCESS_CONCURRENCY`, `BACKLOG_REPROCESS_NIGHT_CONCURRENCY` |
| **Healthcheck** | `curl http://localhost:8005/metrics` |

---

## Application Services

### listener

| Field | Value |
|-------|-------|
| **Build** | `services/listener/Dockerfile` |
| **Container Name** | `osint-listener` |
| **Purpose** | Monitors Telegram channels and pushes messages to Redis Streams |
| **Ports** | `${LISTENER_METRICS_PORT:-8001}:8001` (Prometheus metrics)<br>`8002:8002` (health endpoint) |
| **Profile** | (none - essential) |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Healthcheck** | `curl http://localhost:8002/health` every 60s (600s start period) |
| **Volumes** | `./services/listener:/app/services/listener`<br>`./shared/python:/app/shared/python`<br>`./telegram_sessions:/app/sessions` |
| **Environment** | `TELEGRAM_API_ID`, `TELEGRAM_API_HASH`, `TELEGRAM_PHONE`<br>`BACKFILL_ENABLED`: false (default)<br>`FOLDER_SYNC_INTERVAL`: 300<br>`TRANSLATION_ENABLED`: false (translation runs in the processor) |
| **Replicas** | `${SINGLE_ACCOUNT_REPLICAS:-1}`: 0 when `TELEGRAM_MODE=multi` (see [Telegram mode](#telegram-mode-single-or-multi)) |
| **Notes** | Owns Telegram session - never create standalone clients. The default account's listener (single mode) |

### listener-russia

| Field | Value |
|-------|-------|
| **Build** | `services/listener/Dockerfile` |
| **Container Name** | `osint-listener-russia` |
| **Purpose** | Multi-account mode - the Russia account; handles only channels whose `source_account` is `russia`. Opens sessions `listener-russia` and `discovery-russia` |
| **Ports** | `8011:8001` (Prometheus metrics) |
| **Profile** | `multi-account` |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Environment** | `TELEGRAM_API_ID_RUSSIA`, `TELEGRAM_API_HASH_RUSSIA`, `TELEGRAM_PHONE_RUSSIA`<br>`SOURCE_ACCOUNT`: russia<br>`TELEGRAM_SESSION_NAME`: listener-russia |

### listener-ukraine

| Field | Value |
|-------|-------|
| **Build** | `services/listener/Dockerfile` |
| **Container Name** | `osint-listener-ukraine` |
| **Purpose** | Multi-account mode - the Ukraine account; handles only channels whose `source_account` is `ukraine`. Opens sessions `listener-ukraine` and `discovery-ukraine` |
| **Ports** | `8012:8001` (Prometheus metrics) |
| **Profile** | `multi-account` |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Environment** | `TELEGRAM_API_ID_UKRAINE`, `TELEGRAM_API_HASH_UKRAINE`, `TELEGRAM_PHONE_UKRAINE`<br>`SOURCE_ACCOUNT`: ukraine<br>`TELEGRAM_SESSION_NAME`: listener-ukraine |

### shadow-sync

| Field | Value |
|-------|-------|
| **Build** | `services/shadow-sync/Dockerfile` |
| **Container Name** | `osint-shadow-sync` |
| **Purpose** | Disaster recovery: keeps shadow Telegram accounts joined to the same channels as the primary accounts and can fail over (`FAILOVER_AUTO_SWITCH`) |
| **Ports** | `8013:8001` (Prometheus metrics) |
| **Profile** | `dr` (not part of multi-account mode) |
| **Dependencies** | `postgres` (healthy) |
| **Environment** | `DR_ENABLED`, `SHADOW_SYNC_INTERVAL`, `SHADOW_JOIN_RATE_LIMIT`, `HEALTH_CHECK_INTERVAL`, `{RUSSIA,UKRAINE}_{PRIMARY,SHADOW}_{API_ID,API_HASH,PHONE}` |

### processor-worker

| Field | Value |
|-------|-------|
| **Build** | `services/processor/Dockerfile` |
| **Purpose** | Real-time message processing: embedding spam filter, entity extraction, LLM classification (realtime lane), translation, persistence. Media is downloaded inline unless the [media lane](../operator-guide/media-lanes.md) is on |
| **Profile** | (none - essential) |
| **Replicas** | 2 (`PROCESSOR_WORKER_REPLICAS`) |
| **Dependencies** | `postgres`, `redis`, `ollama` (all healthy) |
| **Volumes** | `./data/models:/models:ro` (see [Embedding Models](../operator-guide/model-provisioning.md)), `./config:/config:ro`, `./data/huggingface`, media buffer (`MEDIA_BUFFER_PATH`) |
| **Environment** | `LLM_PROVIDER`, `LLM_GATEWAY_*`, `EMBEDDING_SPAM_MODEL`, `HF_HUB_OFFLINE=1` |
| **Healthcheck** | `curl http://localhost:8002/metrics` |

### media-worker

| Field | Value |
|-------|-------|
| **Build** | `services/processor/Dockerfile` (command `python -m src.media_worker`) |
| **Purpose** | Downloads media for messages with `media_status` `pending`; idles unless the `media_lane_enabled` admin setting is on. See [Media Lanes](../operator-guide/media-lanes.md) |
| **Profile** | (none - core) |
| **Replicas** | `${SINGLE_ACCOUNT_REPLICAS:-${MEDIA_WORKER_REPLICAS:-1}}`: 0 in multi mode |
| **Scope** | The default account: every channel no named account follows (`SOURCE_ACCOUNT` unset) |
| **Healthcheck** | `curl http://localhost:8003/metrics` |

### media-worker-russia and media-worker-ukraine

| Field | Value |
|-------|-------|
| **Build** | `services/processor/Dockerfile` (command `python -m src.media_worker`) |
| **Purpose** | Media lane of one account: claims only that account's channels, with session slots `media-worker-<account>-N` seeded from `<account>.session` |
| **Profile** | `multi-account` (1 replica each) |
| **Environment** | `TELEGRAM_API_ID_{RUSSIA,UKRAINE}`, `TELEGRAM_API_HASH_{RUSSIA,UKRAINE}`, `SOURCE_ACCOUNT` |

### media-sync

| Field | Value |
|-------|-------|
| **Build** | `services/media-sync/Dockerfile` |
| **Purpose** | Moves archived media from the local buffer to long-term storage (`MEDIA_SYNC_MODE`, default `filesystem`). See [Media Sync](../developer-guide/services/media-sync.md) |
| **Profile** | (none - core) |

### processor-worker-backfill

| Field | Value |
|-------|-------|
| **Build** | `services/processor/Dockerfile` |
| **Purpose** | Extra processor replicas that drain the Redis backfill queue (historical messages) without competing with the realtime workers |
| **Profile** | `backfill` |
| **Replicas** | 2 (`BACKFILL_WORKER_REPLICAS`) |
| **Environment** | Same as `processor-worker`, plus `BACKFILL_ONLY_MODE`, `BACKFILL_CONCURRENCY`, `BACKFILL_NIGHT_CONCURRENCY` |

### api

| Field | Value |
|-------|-------|
| **Build** | `services/api/Dockerfile` |
| **Container Name** | `osint-api` |
| **Purpose** | REST API with FastAPI - serves data, semantic search, admin console |
| **Ports** | `${API_PORT:-8000}:8000` (API)<br>`${API_METRICS_PORT:-8003}:8003` (Prometheus metrics) |
| **Profile** | (none - essential) |
| **Dependencies** | `postgres` (healthy), `redis` (healthy), `ollama` (healthy) |
| **Volumes** | `./services/api:/app/services/api`<br>`./shared/python:/app/shared/python` |
| **Environment** | `AUTH_PROVIDER`: none (default)<br>`AUTH_REQUIRED`: false (default)<br>`API_CORS_ORIGINS`, `FEED_AUTH_REQUIRED`, `OPENSANCTIONS_*` |
| **Healthcheck** | `curl http://localhost:8000/health` every 30s |

### frontend

| Field | Value |
|-------|-------|
| **Build** | `services/frontend-nextjs/Dockerfile` (`NEXT_PUBLIC_*` values are build arguments) |
| **Container Name** | `osint-frontend` |
| **Purpose** | Next.js frontend with Bun runtime |
| **Ports** | `3000:3000` |
| **Profile** | (none - essential) |
| **Dependencies** | `api` (healthy), `minio` (healthy) |
| **Environment** | `API_URL`: http://api:8000 (server-side)<br>`MINIO_URL`: http://minio:9000/osint-media<br>`NODE_ENV`: production<br>Client-side URLs (`NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_MINIO_URL`, `NEXT_PUBLIC_BASE_URL`) are baked in at build time |
| **Healthcheck** | Node fetch of `http://127.0.0.1:3000/` every 30s |

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
| **Profiles** | `enrichment-lite`, `enrichment-standard`, `enrichment-full`, `enrichment` |
| **Dependencies** | `postgres` (healthy), `listener` (healthy) |
| **Volumes** | `./telegram_sessions:/app/sessions`<br>`./shared/python:/shared/python:ro`<br>`./data/logs/analytics:/var/log/analytics` |
| **Environment** | `TELEGRAM_SESSION_PATH`: /app/sessions (uses analytics.session) |
| **Replicas** | `${SINGLE_ACCOUNT_REPLICAS:-1}`: 0 in multi mode |

### analytics-russia and analytics-ukraine

| Field | Value |
|-------|-------|
| **Build** | `services/analytics/Dockerfile` |
| **Container Names** | `osint-analytics-russia`, `osint-analytics-ukraine` |
| **Purpose** | Analytics of one account: session `analytics-<account>`, engagement and metadata of that account's channels only |
| **Profile** | `multi-account` |
| **Dependencies** | `postgres` (healthy), `listener-<account>` (healthy) |
| **Environment** | `TELEGRAM_API_ID_{RUSSIA,UKRAINE}`, `TELEGRAM_API_HASH_{RUSSIA,UKRAINE}`, `SOURCE_ACCOUNT`<br>`VIEW_REFRESH`: false on `analytics-ukraine` (`analytics-russia` refreshes the materialized views) |

### nocodb

| Field | Value |
|-------|-------|
| **Image** | `nocodb/nocodb:latest` |
| **Container Name** | `osint-nocodb` |
| **Purpose** | Airtable-like database UI for database management |
| **Ports** | `${NOCODB_PORT:-8080}:8080` |
| **Profile** | `tools` |
| **Dependencies** | `postgres` (healthy) |
| **Volumes** | `nocodb_data:/usr/app/data` |
| **Environment** | `NC_ADMIN_EMAIL`: `NOCODB_ADMIN_EMAIL` (default admin@osint.local)<br>`NC_DISABLE_TELE`: true |
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
| **Profiles** | `enrichment-standard`, `enrichment-full`, `llm-gateway` |
| **Dependencies** | `postgres` (healthy); `ollama-batch` only if running |
| **Environment** | `AI_TAGGING_MODEL`: qwen2.5:1.5b (Ollama option)<br>`LLM_GATEWAY_MODEL`: `LLM_GATEWAY_BATCH_MODEL` (default osint-batch)<br>`BATCH_SIZE`: `AI_TAGGING_BATCH_SIZE` (12)<br>`TIME_BUDGET_SECONDS`: 120 |

### enrichment-event-detection

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-event-detection` |
| **Purpose** | Creates events from RSS and matches Telegram messages |
| **Ports** | `9099:9099` (Prometheus metrics) |
| **Profiles** | `enrichment-standard`, `enrichment-full`, `llm-gateway` |
| **Dependencies** | `postgres` (healthy); `ollama-batch` only if running |
| **Environment** | `EVENT_DETECTION_MODEL`: qwen2.5:3b (Ollama option)<br>`LLM_GATEWAY_MODEL`: `LLM_GATEWAY_BATCH_MODEL`<br>`BATCH_SIZE`: 10<br>`TIME_BUDGET_SECONDS`: 300 |

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
| **Environment** | `RSS_VALIDATION_MODEL` (compose default granite3-dense:2b, set the gateway alias in `.env`), `LLM_GATEWAY_BATCH_API_KEY`, `RSS_VALIDATION_TIME_BUDGET_SECONDS` (60), `RSS_VALIDATION_BATCH_SIZE` (6), `RSS_VALIDATION_CYCLE_INTERVAL_SECONDS` (30) |
| **Notes** | Only recent posts inside the correlation window are queued; the similarity floor is the admin setting `validation_min_similarity` |

### enrichment-fact-check

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-fact-check` |
| **Purpose** | Audits a sample of recent LLM-classified messages on the audit lane and auto-reclassifies on high confidence. See [Fact-Check](../developer-guide/services/fact-check.md) |
| **Ports** | `9203` (Prometheus metrics) |
| **Profiles** | `enrichment-standard`, `enrichment-full`, `llm-gateway` |
| **Environment** | `FACT_CHECK_MODEL` (compose default qwen2.5:3b, set the gateway alias in `.env`), `LLM_GATEWAY_AUDIT_API_KEY`, `FACT_CHECK_NUM_PREDICT` (500), `FACT_CHECK_LOOKBACK_DAYS` (7), `FACT_CHECK_SAMPLE_SIZE` (50), `FACT_CHECK_AUTO_ACTION_THRESHOLD` (0.85), `FACT_CHECK_HUMAN_REVIEW_THRESHOLD` (0.60), `FACT_CHECK_TIME_BUDGET_SECONDS` (600), `FACT_CHECK_CYCLE_INTERVAL_SECONDS` (300) |

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
| **Replicas** | `${SINGLE_ACCOUNT_REPLICAS:-${ENRICHMENT_WORKER_REPLICAS:-1}}`: 0 in multi mode |
| **Environment** | `TELEGRAM_SESSION_PATH`: /app/sessions/enrichment.session<br>`BATCH_SIZE`: 20<br>`RATE_LIMIT_PER_SECOND`: 20<br>`CHANNEL_COUNTRY_MODEL`, `SOCIAL_GRAPH_AUTO_JOIN`, `CHANNEL_JOINS_ENABLED` |

### enrichment-telegram-russia and enrichment-telegram-ukraine

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` (same worker as `enrichment-telegram`) |
| **Container Names** | `osint-enrichment-telegram-russia`, `osint-enrichment-telegram-ukraine` |
| **Purpose** | Telegram enrichment of one account: reads its own Redis stream `enrich:telegram:<account>`, session `enrichment-<account>.session`, database scans limited to that account's channels |
| **Profile** | `multi-account` (1 replica each) |
| **Ports** | none on the host; Prometheus scrapes over the backend network |
| **Environment** | `TELEGRAM_API_ID_{RUSSIA,UKRAINE}`, `TELEGRAM_API_HASH_{RUSSIA,UKRAINE}`, `TELEGRAM_SESSION_PATH`: /app/sessions/enrichment-<account>.session, `SOURCE_ACCOUNT` |

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
| **Ports** | `9210:9210` (Prometheus metrics) |
| **Profiles** | `enrichment-standard`, `enrichment-full`, `llm-gateway` |
| **Dependencies** | `postgres` (healthy); `ollama-batch` only if running |
| **Environment** | `GEOLOCATION_MODEL`: qwen2.5:1.5b (Ollama option)<br>`LLM_GATEWAY_MODEL`: `LLM_GATEWAY_BATCH_MODEL`<br>`BATCH_SIZE`: `GEOLOCATION_LLM_BATCH_SIZE` (12)<br>`TIME_BUDGET_SECONDS`: 600 |

### enrichment-cluster-detection

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-cluster-detection` |
| **Purpose** | Event Detection V3: velocity-based cluster detection + auxiliary tasks (archiver, tier updater) |
| **Ports** | `9211:9211` (Prometheus metrics) |
| **Profiles** | `enrichment-lite`, `enrichment-standard`, `enrichment-full` |
| **Dependencies** | `postgres` (healthy), `redis` (healthy) |
| **Environment** | `CLUSTER_VELOCITY_THRESHOLD`: 2.0<br>`CLUSTER_TIME_WINDOW_HOURS`: 720<br>`CLUSTER_SIMILARITY_THRESHOLD`: 0.80<br>`MIN_MESSAGES_FOR_CLUSTER`: 3<br>`BATCH_SIZE`: 50<br>`TIME_BUDGET_SECONDS`: 120 |
| **Tasks** | `cluster_detection`, `cluster_archiver`, `cluster_tier_updater` |

### enrichment-cluster-validation

| Field | Value |
|-------|-------|
| **Build** | `services/enrichment/Dockerfile` |
| **Container Name** | `osint-enrichment-cluster-validation` |
| **Purpose** | LLM-based cluster validation using claim analysis (factual/rumor/propaganda) |
| **Ports** | `9212:9212` (Prometheus metrics) |
| **Profiles** | `enrichment-standard`, `enrichment-full`, `llm-gateway` |
| **Dependencies** | `postgres` (healthy); `ollama-batch` only if running |
| **Environment** | `VALIDATION_MODEL`: qwen2.5:3b (Ollama option)<br>`LLM_GATEWAY_MODEL`: `LLM_GATEWAY_BATCH_MODEL`<br>`CLUSTER_RUMOR_TTL_HOURS`: 24<br>`BATCH_SIZE`: `CLUSTER_VALIDATION_BATCH_SIZE` (10)<br>`TIME_BUDGET_SECONDS`: 120<br>`LLM_TIMEOUT`: 180 |
| **Tasks** | `cluster_validation` |

---

## OpenSanctions Entity Matching

`yente-index` and `yente` use the `opensanctions` profile only; `opensanctions` and `entity-ingestion` also start with the `enrichment-standard`, `enrichment-full` and `enrichment` profiles.

### yente-index

| Field | Value |
|-------|-------|
| **Image** | `docker.elastic.co/elasticsearch/elasticsearch:9.4.2` |
| **Container Name** | `osint-yente-index` |
| **Purpose** | ElasticSearch for OpenSanctions entity index (8-10GB) |
| **Profile** | `opensanctions` |
| **Volumes** | `yente_elasticsearch_data:/usr/share/elasticsearch/data` |
| **Environment** | `discovery.type`: single-node<br>`ES_JAVA_OPTS`: -Xms2g -Xmx2g |
| **Healthcheck** | Cluster health check every 30s |

### yente

| Field | Value |
|-------|-------|
| **Image** | `ghcr.io/opensanctions/yente:5.5.0` |
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
| **Dependencies** | `postgres` (healthy) |
| **Volumes** | `./services/opensanctions:/app/services/opensanctions`<br>`./shared/python:/app/shared/python`<br>`./data/huggingface:/app/.cache/huggingface` |
| **Environment** | `OPENSANCTIONS_BACKEND`: yente (or api)<br>`OPENSANCTIONS_MATCH_THRESHOLD`: 0.85<br>`OPENSANCTIONS_BATCH_SIZE`: 5<br>`OPENSANCTIONS_POLL_INTERVAL`: 300 |

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
| **Ports** | `${PROMETHEUS_PORT:-9090}:9090` |
| **Profile** | `monitoring` |
| **Volumes** | `./infrastructure/prometheus:/etc/prometheus/config:ro`<br>`prometheus_data:/prometheus` |
| **Command** | 30-day retention, lifecycle API enabled |
| **Healthcheck** | `wget http://localhost:9090/-/healthy` every 30s |

### grafana

| Field | Value |
|-------|-------|
| **Image** | `grafana/grafana:11.4.0` |
| **Container Name** | `osint-grafana` |
| **Purpose** | Visualization and dashboards |
| **Ports** | `${GRAFANA_PORT:-3000}:3000` (compose default 3000, which collides with the frontend; `.env.example` sets `GRAFANA_PORT=3001`) |
| **Profile** | `monitoring` |
| **Dependencies** | `prometheus` (healthy) |
| **Volumes** | `grafana_data:/var/lib/grafana`<br>`./infrastructure/grafana/provisioning:/etc/grafana/provisioning:ro`<br>`./infrastructure/grafana/dashboards:/var/lib/grafana/dashboards:ro` |
| **Environment** | `GF_SECURITY_ADMIN_USER`: `GRAFANA_ADMIN_USER` (admin)<br>`GF_SECURITY_ADMIN_PASSWORD`: `GRAFANA_ADMIN_PASSWORD`, required<br>`GF_SERVER_ROOT_URL`: `GRAFANA_ROOT_URL`, served from sub-path `/grafana/`<br>`GF_DATABASE_TYPE`: sqlite3 |
| **Healthcheck** | `wget http://localhost:3000/api/health` every 30s |

### alertmanager

| Field | Value |
|-------|-------|
| **Image** | `prom/alertmanager:v0.26.0` |
| **Container Name** | `osint-alertmanager` |
| **Purpose** | Alert routing to ntfy via notifier service |
| **Ports** | `${ALERTMANAGER_PORT:-9093}:9093` |
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
| **Profiles** | `monitoring`, `notifications` |
| **Dependencies** | `redis`, `ntfy`, `postgres` (all healthy) |
| **Volumes** | `./services/notifier:/app/services/notifier`<br>`./shared/python:/app/shared/python` |
| **Environment** | `NTFY_URL`: http://ntfy:80<br>`AGGREGATOR_ENABLED`: true<br>`AGGREGATOR_BATCH_INTERVAL`: 300 |
| **Healthcheck** | `curl http://localhost:8000/health` every 30s |

### ntfy

| Field | Value |
|-------|-------|
| **Image** | `binwiederhier/ntfy:v2.8.0` |
| **Container Name** | `osint-ntfy` |
| **Purpose** | Self-hosted notification delivery (web/mobile push notifications) |
| **Ports** | `127.0.0.1:${NTFY_PORT:-8090}:80` (Web UI)<br>`127.0.0.1:9096:9095` (Prometheus metrics) |
| **Profiles** | `monitoring`, `notifications` |
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

### kratos-init

| Field | Value |
|-------|-------|
| **Image** | `python:3.11-slim` (runs `bootstrap.py`) |
| **Container Name** | `osint-kratos-init` |
| **Purpose** | One-time bootstrap of Kratos identities and roles |
| **Profile** | `auth` |
| **Dependencies** | `kratos` (healthy) |

### crowdsec

| Field | Value |
|-------|-------|
| **Image** | `crowdsecurity/crowdsec:v1.7.4` |
| **Container Name** | `osint-crowdsec` |
| **Purpose** | Intrusion detection engine fed by Caddy logs; Caddy's bouncer enforces its decisions. See [Security Guide](../security-guide/index.md) |
| **Profile** | `auth` |
| **Healthcheck** | `cscli lapi status` every 30s |
| **Notes** | `crowdsec-cloudflare-bouncer` (profile `cloudflare`) pushes decisions to Cloudflare |

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
| **Image** | `asyafiqe/caddy-crowdsec-bouncer:2.7.6` (Caddy with the CrowdSec bouncer) |
| **Container Name** | `osint-caddy` |
| **Purpose** | Reverse proxy with automatic HTTPS |
| **Ports** | `80:80` (HTTP)<br>`443:443` (HTTPS)<br>`2019:2019` (Admin API) |
| **Profile** | `auth` |
| **Dependencies** | `frontend`, `api`, `oathkeeper`, `kratos` |
| **Volumes** | `./infrastructure/caddy/Caddyfile.production:/etc/caddy/Caddyfile:ro`, `storage-routes.snippet`, `caddy_data`, `caddy_config`, `caddy_logs`, media buffer (read-only) |

---

## Documentation (tools profile)

### mkdocs

| Field | Value |
|-------|-------|
| **Build** | `${DOCS_REPO_PATH:-../osint-platform-docs}/Dockerfile` (this documentation repo) |
| **Container Name** | `osint-mkdocs` |
| **Purpose** | Documentation site with Material theme |
| **Ports** | `8001:8000` |
| **Profile** | `tools` |
| **Volumes** | `${DOCS_REPO_PATH}/docs` and `mkdocs.yml`, read-only |
| **Command** | `mkdocs serve --dev-addr=0.0.0.0:8000 --no-livereload` |
| **Healthcheck** | `wget http://127.0.0.1:8000` every 60s |

---

## Other Compose Files

`docker-compose.yml` is the platform. The other files are overlays or alternatives in the repo root:

| File | Use |
|------|-----|
| `docker-compose.production.yml` | Production overlay (Caddy with CrowdSec, Oathkeeper routing; the multi-account listeners live in `docker-compose.yml`, selected by `TELEGRAM_MODE`). Merge with `COMPOSE_FILE=docker-compose.yml:docker-compose.production.yml` in `.env` |
| `docker-compose.security.yml` | CrowdSec stack (CrowdSec, Caddy bouncer, Cloudflare bouncer, dashboard); layer on top of the production file |
| `docker-compose.development.yml` | Laptop override that trims resources: `-f docker-compose.yml -f docker-compose.development.yml` |
| `docker-compose.minimal.yml` | Standalone core stack (postgres, redis, minio, ollama, enricher, listener, processor, api, frontend) |
| `docker-compose.monitoring.yml` | Older standalone monitoring overlay; the `monitoring` profile in the main file is the maintained path |
| `docker-compose.nocodb.yml` | Older NocoDB overlay; use the `tools` profile instead |

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
| `minio_data` | Declared but unused by default: `minio` mounts `./data/minio` (or `HETZNER_MOUNT_PATH`) | Bind mount |
| `telegram_sessions` | Telegram session files | Bind mount (`./telegram_sessions`) |
| `prometheus_data` | Prometheus metrics | Docker volume |
| `grafana_data` | Grafana dashboards | Docker volume |
| `ntfy_cache` | ntfy message cache | Docker volume |
| `ntfy_data` | ntfy persistence | Docker volume |
| `nocodb_data` | NocoDB metadata | Docker volume |
| `alertmanager_data` | AlertManager state | Docker volume |
| `yente_elasticsearch_data` | OpenSanctions index (8-10GB) | Docker volume |
| `loki_data` | Loki logs | Docker volume |
| `caddy_data`, `caddy_config`, `caddy_logs` | Caddy certificates, config, logs | Docker volume |
| `crowdsec_data`, `crowdsec_config` | CrowdSec state | Docker volume |

---

## Profiles

Profiles select which service groups start. Set them persistently with `COMPOSE_PROFILES` in `.env` (production uses `enrichment-lite,monitoring,auth,llm-gateway`). Services without a profile (postgres, redis, minio, ollama, listener (0 replicas in multi mode), processor-worker, media-worker, media-sync, api, frontend, rss-ingestor) always start.

| Profile | Adds |
|---------|------|
| `enrichment-lite` | router, fast-pool, telegram, decision, maintenance, cluster-detection, analytics |
| `enrichment-standard` | Lite plus AI tagging, event detection, fact-check, geolocation-llm, cluster-validation, RSS validation, ollama-batch, opensanctions, entity-ingestion |
| `enrichment-full` | Same service set as standard |
| `enrichment` | Legacy name, same set as standard |
| `llm-gateway` | backlog-worker, backlog-enrich, backlog-reprocess, plus the gateway-capable enrichment workers (ai-tagging, event-detection, fact-check, geolocation-llm, cluster-validation, rss-validation). See [LLM Gateway](../operator-guide/external-llm.md) |
| `monitoring` | prometheus, grafana, alertmanager, loki, promtail, exporters, cadvisor, notifier, ntfy |
| `notifications` | notifier and ntfy without the rest of monitoring |
| `auth` | kratos, kratos-migrate, kratos-init, oathkeeper, crowdsec, caddy |
| `cloudflare` | crowdsec-cloudflare-bouncer |
| `opensanctions` | yente-index, yente, opensanctions, entity-ingestion |
| `multi-account` | Per account (russia, ukraine): listener, media-worker, enrichment-telegram, analytics. See [Telegram mode](#telegram-mode-single-or-multi) |
| `dr` | shadow-sync (disaster recovery) |
| `backfill` | processor-worker-backfill |
| `tools` | nocodb, mkdocs |

### Telegram mode (single or multi)

`TELEGRAM_MODE` in `.env` selects which Telegram services run (unset = `single`):

| Mode | Runs | `.env` lines |
|------|------|--------------|
| `single` | The default account: `listener`, `media-worker`, `enrichment-telegram`, `analytics` | `SINGLE_ACCOUNT_REPLICAS` unset, no `multi-account` in `COMPOSE_PROFILES` |
| `multi` | Per account (russia, ukraine): `listener-<a>`, `media-worker-<a>`, `enrichment-telegram-<a>`, `analytics-<a>`. The four default services have 0 replicas | `TELEGRAM_MODE=multi`, `SINGLE_ACCOUNT_REPLICAS=0`, `multi-account` in `COMPOSE_PROFILES` |

Processors run in both modes (they download media inline only for default-account channels). Do not edit the lines by hand: `./scripts/switch-telegram-mode.sh single|multi` writes them (backing up `.env` first) and runs `deploy.sh`. `multi` refuses unless both accounts' credentials and sessions exist, the processors report `media_lane_enabled` on, and no default-account channel is still active. Other subcommands: `status`, `ready`, `write MODE`, `check`, `listeners`. `deploy.sh` runs `check` first and deploys nothing if the lines disagree with `TELEGRAM_MODE`.

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
| Grafana | 3000 (`GRAFANA_PORT`) | Compose default equals the frontend port; `.env.example` sets 3001 |
| Alertmanager | 9093 | |
| Loki | 3100 | |
| ntfy | 127.0.0.1:8090 | Bound to localhost |
| NocoDB | 8080 | `tools` profile |
| MkDocs | 8001 (`MKDOCS_PORT`) | `tools` profile |
| Listener metrics / health | 8001 / 8002 | Multi-account listeners metrics 8011, 8012; shadow-sync 8013 (`dr`) |
| Enrichment workers | 9099, 9196 to 9203, 9210 to 9212 | One metrics port per worker |
| Notifier | 9094 | |
| Exporters | 9100 (node), 9121 (redis), 9187 (postgres), 8081 (cAdvisor) | |
| Kratos / Oathkeeper | 4433, 4434 / 4455, 4456 | `auth` profile |
| Caddy | 80, 443, 2019 | `auth` profile |

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
