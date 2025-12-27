# Docker Services Reference

Complete reference for all Docker services in the OSINT Intelligence Platform.

**Source**: `/osint-intelligence-platform/docker-compose.yml`

**Last Updated**: 2025-12-27

---

## Overview

The platform's containers are organized into functional categories:

| Category | Purpose |
|----------|---------|
| **Core Infrastructure** | Database, cache, storage, LLM |
| **Application Services** | Data ingestion, processing, enrichment, API, frontend |
| **Monitoring Stack** | Metrics, logs, alerts, dashboards |
| **Infrastructure Exporters** | PostgreSQL and Redis metrics |
| **Container Management** | Resource monitoring and updates |
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

## LLM Layer (Self-Hosted AI)

### ollama

| Field | Value |
|-------|-------|
| **Image** | `ollama/ollama:latest` |
| **Container Name** | `osint-ollama` |
| **Purpose** | Realtime LLM inference for processor and API (classification, semantic search) |
| **Ports** | `11434:11434` |
| **Profile** | (none - essential) |
| **Dependencies** | None |
| **Volumes** | `./data/ollama:/root/.ollama` (bind mount) |
| **Environment** | `OLLAMA_NUM_PARALLEL`: 1<br>`OLLAMA_MAX_LOADED_MODELS`: 2<br>`OLLAMA_CPU_THREADS`: 6<br>`OLLAMA_KEEP_ALIVE`: 5m |
| **Resources** | CPU Limit: 6.0 cores<br>Memory Limit: 8G |
| **Healthcheck** | `ollama list` every 30s |

### ollama-init

| Field | Value |
|-------|-------|
| **Build** | `infrastructure/ollama/Dockerfile.init` |
| **Container Name** | `osint-ollama-init` |
| **Purpose** | Pulls and initializes models from database configuration |
| **Dependencies** | `ollama` (healthy), `postgres` (healthy) |
| **Lifecycle** | Restarts on failure (max 3 times) |

### ollama-batch

| Field | Value |
|-------|-------|
| **Image** | `ollama/ollama:latest` |
| **Container Name** | `osint-ollama-batch` |
| **Purpose** | Batch LLM processing for enrichment tasks (non-blocking) |
| **Ports** | `11435:11434` |
| **Profile** | `enrichment` |
| **Dependencies** | None |
| **Volumes** | `./data/ollama:/root/.ollama` (shared models) |
| **Environment** | `OLLAMA_NUM_PARALLEL`: 1<br>`OLLAMA_MAX_LOADED_MODELS`: 2<br>`OLLAMA_CPU_THREADS`: 4<br>`OLLAMA_KEEP_ALIVE`: 30m |
| **Resources** | CPU Limit: 4.0 cores<br>Memory Limit: 6G |

### ollama-batch-init

| Field | Value |
|-------|-------|
| **Build** | `infrastructure/ollama/Dockerfile.init` |
| **Container Name** | `osint-ollama-batch-init` |
| **Purpose** | Initializes models for batch Ollama instance |
| **Profile** | `enrichment` |
| **Dependencies** | `ollama-batch` (healthy), `postgres` (healthy) |

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
| **Container Name** | Multiple (replicas) |
| **Purpose** | Real-time message processing: spam filter, entity extraction, LLM classification, media archival |
| **Profile** | (none - essential) |
| **Dependencies** | `postgres` (healthy), `redis` (healthy), `ollama` (healthy) |
| **Volumes** | `./services/processor:/app/services/processor`<br>`./shared/python:/app/shared/python`<br>`./telegram_sessions:/app/sessions`<br>`./config:/config:ro`<br>`./data/huggingface:/app/.cache/huggingface` |
| **Environment** | `LLM_ENABLED`: true<br>`TRANSLATION_ENABLED`: true<br>`PROCESSOR_BATCH_SIZE`: 10<br>`PROCESSOR_WORKERS`: 4 |
| **Replicas** | 2 (configurable via `PROCESSOR_REPLICAS`) |
| **Healthcheck** | `curl http://localhost:8002/metrics` every 30s |

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
| **Profile** | `enrichment` |
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
| **Profile** | `dev` |
| **Dependencies** | `postgres` (healthy) |
| **Volumes** | `nocodb_data:/usr/app/data` |
| **Environment** | `NC_ADMIN_EMAIL`: admin@osint.local<br>`NC_DISABLE_TELE`: true |
| **Healthcheck** | `curl http://localhost:8080/api/v1/health` every 30s |

---

## Enrichment Workers

All enrichment workers use the `enrichment` profile and have similar patterns. They process messages in batches with time budgets.

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
| **Purpose** | LLM-based article validation for RSS correlation |
| **Ports** | `9197:9197` (Prometheus metrics) |
| **Dependencies** | `postgres` (healthy), `ollama-batch` (healthy) |
| **Environment** | `RSS_VALIDATION_MODEL`: granite3-dense:2b<br>`BATCH_SIZE`: 2<br>`TIME_BUDGET_SECONDS`: 60 |

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

### dozzle

| Field | Value |
|-------|-------|
| **Image** | `amir20/dozzle:latest` |
| **Container Name** | `osint-dozzle` |
| **Purpose** | Real-time Docker log viewer (web UI) |
| **Ports** | `9999:8080` |
| **Profile** | `monitoring` |
| **Volumes** | `/var/run/docker.sock:/var/run/docker.sock:ro` |
| **Environment** | `DOZZLE_FILTER`: name=osint<br>`DOZZLE_TAILSIZE`: 300 |

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

## Container Management

### watchtower

| Field | Value |
|-------|-------|
| **Image** | `containrrr/watchtower:latest` |
| **Container Name** | `osint-watchtower` |
| **Purpose** | Automatic container updates (checks daily) |
| **Profile** | `utilities` |
| **Volumes** | `/var/run/docker.sock:/var/run/docker.sock` |
| **Environment** | `WATCHTOWER_CLEANUP`: true<br>`WATCHTOWER_POLL_INTERVAL`: 86400<br>`DOCKER_API_VERSION`: 1.44 |

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

### mailslurper

| Field | Value |
|-------|-------|
| **Image** | `oryd/mailslurper:latest-smtps` |
| **Container Name** | `osint-mailslurper` |
| **Purpose** | Email testing for development (catches outbound SMTP) |
| **Ports** | `4436:4436` (Web UI)<br>`4437:4437` (SMTP)<br>`1025:1025` (SMTP alt) |
| **Profile** | `dev` |

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

## Dashboard & Documentation

### dashy

| Field | Value |
|-------|-------|
| **Image** | `lissy93/dashy:latest` |
| **Container Name** | `osint-dashy` |
| **Purpose** | Platform landing page and service dashboard |
| **Ports** | `4000:8080` |
| **Profile** | `dev` |
| **Volumes** | `./infrastructure/dashy/conf.yml:/app/user-data/conf.yml` |
| **Environment** | `NODE_ENV`: production |
| **Healthcheck** | Node healthcheck service every 90s |

### mkdocs

| Field | Value |
|-------|-------|
| **Image** | `squidfunk/mkdocs-material:latest` |
| **Container Name** | `osint-mkdocs` |
| **Purpose** | Documentation site with Material theme |
| **Ports** | `8001:8000` |
| **Profile** | `dev` |
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

Docker Compose profiles control which service groups are started:

| Profile | Services | Use Case |
|---------|----------|----------|
| (none) | Core + Application | Production runtime (15 containers) |
| `enrichment` | All enrichment workers | Full enrichment pipeline (12 workers) |
| `enrichment-standard` | LLM workers | AI tagging, geolocation, cluster validation, event detection |
| `enrichment-full` | All workers | Standard + advanced features |
| `monitoring` | Prometheus stack | Metrics, logs, alerts (8 services) |
| `opensanctions` | Entity matching | Sanctions data enrichment (4 services) |
| `multi-account` | Russia/Ukraine listeners | Multi-account Telegram monitoring (2 services) |
| `auth` | Kratos + Oathkeeper | Authentication and access control (4 services) |
| `dev` | NocoDB, Dashy, MkDocs, Mailslurper | Development tools (4 services) |
| `utilities` | Watchtower | Container updates (1 service) |

**Start with profile:**
```bash
docker-compose --profile monitoring --profile enrichment up -d
```

**Check active profiles:**
```bash
docker-compose ps --all
```

---

## Port Reference

Quick reference for commonly accessed services:

| Service | Port | URL |
|---------|------|-----|
| Frontend | 3000 | http://localhost:3000 |
| API | 8000 | http://localhost:8000 |
| API Docs | 8000 | http://localhost:8000/docs |
| Dashy Dashboard | 4000 | http://localhost:4000 |
| NocoDB | 8080 | http://localhost:8080 |
| Grafana | 3001 | http://localhost:3001 |
| Prometheus | 9090 | http://localhost:9090 |
| ntfy | 8090 | http://localhost:8090 |
| MinIO Console | 9001 | http://localhost:9001 |
| Dozzle Logs | 9999 | http://localhost:9999 |
| AlertManager | 9093 | http://localhost:9093 |
| MkDocs | 8001 | http://localhost:8001 |

---

## Resource Limits

Services with explicit resource constraints:

| Service | CPU Limit | Memory Limit | Notes |
|---------|-----------|--------------|-------|
| ollama | 6.0 cores | 8G | Realtime LLM (was 2.0 - bottleneck!) |
| ollama-batch | 4.0 cores | 6G | Background LLM processing |

Host system: AMD Ryzen 9 7940HS (8 cores / 16 threads, 54GB RAM)

---

## Dependencies Graph

Critical path (must be healthy):

```
postgres → listener → redis → processor → minio → api → frontend
         ↘ ollama ↗
```

Enrichment dependency:
```
postgres + redis + ollama-batch → enrichment workers
```

Monitoring dependency:
```
prometheus → grafana
           ↘ alertmanager → notifier → ntfy
```

---

## Environment Variables

Key environment variables (see `.env.example` for complete list):

| Variable | Default | Services | Purpose |
|----------|---------|----------|---------|
| `POSTGRES_USER` | postgres | All | Database username |
| `POSTGRES_PASSWORD` | postgres | All | Database password |
| `POSTGRES_DB` | osint_platform | All | Database name |
| `TELEGRAM_API_ID` | (required) | listener, processor, analytics | Telegram API credentials |
| `TELEGRAM_API_HASH` | (required) | listener, processor, analytics | Telegram API credentials |
| `DEEPL_API_KEY` | (required) | processor, enrichment-fast-pool | Translation (free tier) |
| `OLLAMA_MODEL` | qwen2.5:3b | processor, api | Default LLM model |
| `AI_TAGGING_MODEL` | qwen2.5:1.5b | enrichment-ai-tagging | Tagging model (3x faster) |
| `RSS_VALIDATION_MODEL` | granite3-dense:2b | enrichment-rss-validation | Validation model (100% accuracy) |
| `PROCESSOR_REPLICAS` | 2 | processor-worker | Number of processor instances |
| `AUTH_PROVIDER` | none | api, frontend | Authentication method (none/kratos) |
| `LOG_LEVEL` | INFO | All | Logging verbosity |

---

## Healthcheck Summary

All services with healthchecks use:
- **Interval**: 30s (60s for non-critical)
- **Timeout**: 10s
- **Retries**: 3
- **Start Period**: 10-60s (300s for yente)

Healthcheck methods:
- HTTP endpoints: `curl`, `wget`
- Database: `pg_isready`, `redis-cli ping`
- Process: Python process checks (slim images without `pgrep`)

---

## Related Documentation

- [Architecture Overview](../developer-guide/architecture.md)
- [Installation Guide](../operator-guide/installation.md)
- [Monitoring Setup](../operator-guide/monitoring.md)

---

**File Path**: `~/code/osintukraine/osint-platform-docs/docs/reference/docker-services.md`
