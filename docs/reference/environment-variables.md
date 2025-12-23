# Environment Variables Reference

Complete reference for all environment variables used across the OSINT Intelligence Platform.

## Table of Contents

- [Core Infrastructure](#core-infrastructure)
- [Database Configuration](#database-configuration)
- [Redis Configuration](#redis-configuration)
- [Authentication & Security](#authentication--security)
- [API Service](#api-service)
- [Processor Service](#processor-service)
- [Enrichment Service](#enrichment-service)
- [Listener Service](#listener-service)
- [Media Services](#media-services)
- [Telegram Configuration](#telegram-configuration)
- [LLM & AI Services](#llm--ai-services)
- [Geolocation & Mapping](#geolocation--mapping)
- [Event Detection & Clustering](#event-detection--clustering)
- [OpenSanctions Integration](#opensanctions-integration)
- [Metrics & Monitoring](#metrics--monitoring)

---

## Core Infrastructure

### Deployment Mode

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `DEPLOYMENT_MODE` | `development` | Deployment mode: `development`, `production`, `staging` | No |

---

## Database Configuration

PostgreSQL database connection settings.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `POSTGRES_HOST` | `localhost` | PostgreSQL server hostname | No |
| `POSTGRES_PORT` | `5432` | PostgreSQL server port | No |
| `POSTGRES_DB` | `osint_platform` | Database name | No |
| `POSTGRES_USER` | `osint_user` | Database username | No |
| `POSTGRES_PASSWORD` | _(none)_ | Database password | **Yes** |
| `DATABASE_URL` | _(auto-generated)_ | Full PostgreSQL connection URL (overrides individual settings if set) | No |

**Example:**
```bash
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=osint_platform
POSTGRES_USER=osint_user
POSTGRES_PASSWORD=your_secure_password_here
```

---

## Redis Configuration

Redis connection for queues, caching, and pub/sub.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `REDIS_URL` | `redis://redis:6379` | Redis connection URL | No |

**Example:**
```bash
REDIS_URL=redis://redis:6379/0
```

---

## Authentication & Security

### Authentication Provider

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `AUTH_PROVIDER` | `none` | Authentication provider: `none`, `jwt`, `cloudron`, `ory` | No |
| `AUTH_REQUIRED` | `false` | Require authentication for all endpoints | No |

### JWT Authentication

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `JWT_SECRET_KEY` | _(none)_ | Secret key for JWT token signing (min 32 chars). Generate with: `openssl rand -hex 64` | **Yes** (if `AUTH_PROVIDER=jwt`) |
| `JWT_ALGORITHM` | `HS256` | JWT signing algorithm | No |
| `JWT_EXPIRATION_MINUTES` | `60` | JWT token expiration time in minutes | No |
| `JWT_ADMIN_PASSWORD` | _(none)_ | Default admin user password (hashed with bcrypt) | **Yes** (if `AUTH_PROVIDER=jwt`) |

### Ory Kratos/Oathkeeper

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `KRATOS_PUBLIC_URL` | `http://kratos:4433` | Ory Kratos public API URL | No |
| `KRATOS_ADMIN_URL` | `http://kratos:4434` | Ory Kratos admin API URL | No |
| `ORY_OATHKEEPER_URL` | _(none)_ | Ory Oathkeeper URL for pre-authenticated requests | No |

### CORS & WebSocket Security

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `API_CORS_ORIGINS` | `http://localhost,http://localhost:3000,http://localhost:8000` | Comma-separated list of allowed CORS origins | No |
| `ALLOWED_ORIGINS` | _(falls back to `API_CORS_ORIGINS`)_ | Allowed origins for WebSocket connections | No |
| `FRONTEND_URL` | _(none)_ | Production frontend URL (automatically added to allowed origins) | No |
| `WEBSOCKET_ALLOW_NO_ORIGIN` | `false` | Allow WebSocket connections without Origin header (not recommended) | No |
| `WEBSOCKET_MAX_CONNECTIONS_PER_IP` | `10` | Maximum concurrent WebSocket connections per IP address | No |
| `WEBSOCKET_RATE_LIMIT` | `10` | Maximum WebSocket messages per second per connection | No |

**Example:**
```bash
AUTH_PROVIDER=jwt
JWT_SECRET_KEY=your_64_char_hex_key_here
JWT_ADMIN_PASSWORD=secure_admin_password
FRONTEND_URL=https://v2.osintukraine.com
```

---

## API Service

### General API Settings

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `NEXT_PUBLIC_API_URL` | `http://localhost:8000` | API base URL for frontend client-side API calls | No |
| `OLLAMA_HOST` | `http://ollama:11434` | Ollama LLM server URL for semantic search and system checks | No |

### Rate Limiting

Redis-backed sliding window rate limiting for map endpoints.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `MAP_MESSAGES_RATE_LIMIT` | `120` | Requests per minute for `/api/map/messages` endpoint | No |
| `MAP_CLUSTERS_RATE_LIMIT` | `120` | Requests per minute for `/api/map/clusters` endpoint | No |
| `MAP_EVENTS_RATE_LIMIT` | `60` | Requests per minute for `/api/map/events` endpoint | No |
| `MAP_HEATMAP_RATE_LIMIT` | `60` | Requests per minute for `/api/map/heatmap` endpoint | No |
| `MAP_SUGGEST_RATE_LIMIT` | `60` | Requests per minute for `/api/map/locations/suggest` endpoint | No |
| `MAP_REVERSE_RATE_LIMIT` | `60` | Requests per minute for `/api/map/locations/reverse` endpoint | No |
| `MAP_CLUSTER_MESSAGES_RATE_LIMIT` | `60` | Requests per minute for `/api/map/clusters/{id}/messages` endpoint | No |

### Map Cache TTLs

Cache time-to-live in seconds for map endpoints.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `MAP_CACHE_TTL_MESSAGES` | `60` | Cache TTL for individual message points (seconds) | No |
| `MAP_CACHE_TTL_CLUSTERS` | `300` | Cache TTL for event clusters (seconds) | No |
| `MAP_CACHE_TTL_EVENTS` | `300` | Cache TTL for events endpoint (seconds) | No |
| `MAP_CACHE_TTL_HEATMAP` | `300` | Cache TTL for heatmap aggregations (seconds) | No |
| `MAP_CACHE_TTL_SUGGESTIONS` | `600` | Cache TTL for location autocomplete (seconds) | No |

**Rationale:**
- **Messages (60s)**: Real-time feel important; WebSocket provides true real-time updates
- **Clusters/Events (300s)**: Tier changes infrequent (require multiple sources), 5min acceptable
- **Heatmap (300s)**: Aggregate data; individual additions have minimal visual impact
- **Suggestions (600s)**: Gazetteer data is static; only changes on manual GeoNames import

**Example:**
```bash
MAP_MESSAGES_RATE_LIMIT=120
MAP_CACHE_TTL_MESSAGES=60
MAP_CACHE_TTL_CLUSTERS=300
```

---

## Processor Service

Real-time message processing (spam filter, entity extraction, media archival).

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `MEDIA_BUFFER_PATH` | `/var/cache/osint-media-buffer` | Local buffer path for media files before sync to Hetzner | No |

---

## Enrichment Service

Background enrichment tasks (translation, embeddings, AI tagging, geolocation, event detection).

### General Enrichment Settings

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `ENRICHMENT_TASKS` | `translation,entity_matching` | Comma-separated list of enabled enrichment tasks | No |
| `ENRICHMENT_INTERVAL` | `3600` | Time between enrichment cycles in seconds | No |
| `ENRICHMENT_MAX_CONCURRENT` | `2` | Maximum concurrent enrichment tasks | No |
| `ENRICHMENT_BATCH_SIZE` | `100` | Default messages per batch | No |

**Available Tasks:**
- `translation` - DeepL/Google Translate message translation
- `entity_matching` - Match entities against OpenSanctions/Wikidata
- `embedding` - Generate semantic embeddings with sentence-transformers
- `ai_tagging` - Extract entities/topics with LLM
- `social_graph_extraction` - Build channel relationship graph
- `engagement_polling` - Poll view/forward/comment counts
- `geolocation` - Extract coordinates from location names
- `event_detection` - Detect and cluster related events

### Translation Settings

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `DEEPL_API_KEY` | _(none)_ | DeepL API key for translation (falls back to Google Translate if not set) | No |
| `TRANSLATION_BATCH_SIZE` | `50` | Messages per translation batch | No |

### Entity Matching

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `ENTITY_MATCHING_THRESHOLD` | `0.75` | Minimum similarity score for entity matching | No |
| `ENTITY_MATCHING_BATCH_SIZE` | `100` | Messages per entity matching batch | No |

### Embedding Generation

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Sentence-transformers model for embeddings | No |
| `EMBEDDING_BATCH_SIZE` | `100` | Messages per embedding batch | No |

### AI Tagging

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `OLLAMA_HOST` | `http://ollama:11434` | Ollama LLM server URL | No |
| `AI_TAGGING_MODEL` | `qwen2.5:3b` | LLM model for AI tagging | No |
| `AI_TAGGING_BATCH_SIZE` | `50` | Messages per AI tagging batch | No |
| `AI_TAGGING_TIMEOUT` | `300.0` | Request timeout for LLM calls in seconds | No |

### Wikidata Enrichment

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `WIKIDATA_BATCH_SIZE` | `50` | Entities per Wikidata enrichment batch | No |
| `WIKIDATA_MIN_CONFIDENCE` | `0.90` | Minimum confidence for Wikidata lookups | No |

### Event Detection

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `EVENT_ENTITY_OVERLAP_THRESHOLD` | `1` | Minimum shared entities to link events | No |
| `EVENT_EMBEDDING_SIMILARITY_THRESHOLD` | `0.85` | Minimum embedding similarity for event matching | No |
| `EVENT_TIME_WINDOW_HOURS` | `72` | Time window for event correlation in hours | No |
| `EVENT_MATCH_LOOKBACK_HOURS` | `168` | How far back to look for matching telegram clusters (7 days) | No |

**Example:**
```bash
ENRICHMENT_TASKS=translation,embedding,ai_tagging,geolocation,event_detection
DEEPL_API_KEY=your_deepl_api_key_here
AI_TAGGING_MODEL=qwen2.5:3b
EMBEDDING_MODEL=all-MiniLM-L6-v2
```

---

## Listener Service

Telegram channel monitoring and message ingestion.

Telegram credentials are configured per-account (see [Telegram Configuration](#telegram-configuration) section).

---

## Media Services

### Media Sync Worker

Background worker that syncs media from local buffer to Hetzner/MinIO storage.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `MEDIA_BUFFER_PATH` | `/var/cache/osint-media-buffer` | Local buffer path for media files | No |
| `MINIO_ENDPOINT` | `localhost:9000` | MinIO S3 endpoint | No |
| `MINIO_ACCESS_KEY` | `minioadmin` | MinIO access key | No |
| `MINIO_SECRET_KEY` | `minioadmin` | MinIO secret key | No |
| `MINIO_BUCKET` | `osint-media` | MinIO bucket name | No |
| `MINIO_SECURE` | `false` | Use HTTPS for MinIO connection | No |
| `MINIO_PUBLIC_URL` | `http://localhost:9000` | Public URL for MinIO (used for fallback media serving) | No |
| `HETZNER_STORAGE_URL` | _(none)_ | Hetzner storage box URL (if using Hetzner instead of MinIO) | No |
| `WORKER_ID` | `sync-worker-{PID}` | Unique worker identifier | No |

**Storage Routing:**
- If `HETZNER_STORAGE_URL` is set, media is synced to Hetzner storage box
- Otherwise, media is synced to MinIO (fallback)

**Example:**
```bash
MINIO_ENDPOINT=minio:9000
MINIO_ACCESS_KEY=admin
MINIO_SECRET_KEY=your_secure_key
MINIO_BUCKET=osint-media
HETZNER_STORAGE_URL=https://your-storage-box.com/media
```

---

## Telegram Configuration

### Multiple Account Support

The platform supports multiple Telegram accounts using suffix-based configuration. Each account can have its own API credentials and phone number.

**Configuration Pattern:**
- Default (no suffix): `TELEGRAM_API_ID`, `TELEGRAM_API_HASH`, `TELEGRAM_PHONE`
- Russia account: `TELEGRAM_API_ID_RUSSIA`, `TELEGRAM_API_HASH_RUSSIA`, `TELEGRAM_PHONE_RUSSIA`
- Ukraine account: `TELEGRAM_API_ID_UKRAINE`, `TELEGRAM_API_HASH_UKRAINE`, `TELEGRAM_PHONE_UKRAINE`

### Telegram Credentials

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `TELEGRAM_API_ID` | `0` | Telegram API ID from https://my.telegram.org | **Yes** (for Telegram features) |
| `TELEGRAM_API_HASH` | _(none)_ | Telegram API hash from https://my.telegram.org | **Yes** (for Telegram features) |
| `TELEGRAM_PHONE` | _(none)_ | Phone number for Telegram account (format: +1234567890) | **Yes** (for Telegram features) |
| `TELEGRAM_SESSION_PATH` | `/app/sessions/discovery.session` | Path to Telegram session file | No |

### Multi-Account Example

```bash
# Default account (Russia)
TELEGRAM_API_ID=12345678
TELEGRAM_API_HASH=abcdef1234567890abcdef1234567890
TELEGRAM_PHONE=+79991234567

# Ukraine account
TELEGRAM_API_ID_UKRAINE=87654321
TELEGRAM_API_HASH_UKRAINE=fedcba0987654321fedcba0987654321
TELEGRAM_PHONE_UKRAINE=+380991234567

# Additional accounts can be added with any suffix:
# TELEGRAM_API_ID_BACKUP=...
# TELEGRAM_API_HASH_BACKUP=...
# TELEGRAM_PHONE_BACKUP=...
```

---

## LLM & AI Services

### Ollama Configuration

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `OLLAMA_HOST` | `http://ollama:11434` | Ollama LLM server URL | No |

### Worker-Specific Settings

Different enrichment workers have specialized LLM configurations:

#### AI Tagging Worker

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `TIME_BUDGET_SECONDS` | `120` | Maximum time per worker cycle | No |
| `BATCH_SIZE` | `10` | Messages per batch | No |
| `CYCLE_INTERVAL_SECONDS` | `30` | Time between worker cycles | No |
| `AI_TAGGING_MODEL` | `qwen2.5:3b` | LLM model for AI tagging | No |

#### Geolocation LLM Worker

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `TIME_BUDGET_SECONDS` | `300` | Maximum time per worker cycle | No |
| `BATCH_SIZE` | `5` | Messages per batch (LLM geolocation is slower) | No |
| `CYCLE_INTERVAL_SECONDS` | `60` | Time between worker cycles | No |
| `GEOLOCATION_MODEL` | `qwen2.5:3b` | LLM model for geolocation extraction | No |
| `GEOLOCATION_TIMEOUT` | `120` | Request timeout for geolocation LLM calls | No |

#### Decision Worker

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `TIME_BUDGET_SECONDS` | `60` | Maximum time per worker cycle | No |
| `BATCH_SIZE` | `50` | Messages per batch | No |
| `CYCLE_INTERVAL_SECONDS` | `30` | Time between worker cycles | No |

#### RSS Validation Worker

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `TIME_BUDGET_SECONDS` | `60` | Maximum time per worker cycle | No |
| `BATCH_SIZE` | `5` | RSS items per batch | No |
| `CYCLE_INTERVAL_SECONDS` | `30` | Time between worker cycles | No |
| `RSS_VALIDATION_MODEL` | _(AI tagging model)_ | LLM model for RSS validation | No |
| `MIN_SIMILARITY` | `0.5` | Minimum similarity for RSS item matching | No |

---

## Geolocation & Mapping

### Geolocation Pipeline

4-stage pipeline for extracting coordinates from location names:

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `NOMINATIM_URL` | `https://nominatim.openstreetmap.org` | Nominatim API endpoint for geocoding | No |
| `NOMINATIM_RATE_LIMIT` | `1.0` | Requests per second to Nominatim API | No |
| `NOMINATIM_TIMEOUT` | `10` | Request timeout in seconds | No |
| `NOMINATIM_MAX_RETRIES` | `3` | Maximum retry attempts on failure | No |

**Pipeline Stages:**
1. **Gazetteer Match** (offline, 0.95 confidence)
2. **LLM Relative Location** ("10km north of X", 0.75 confidence)
3. **Nominatim API** (OSM, 0.85 confidence)
4. **Mark Unresolved** (manual review queue)

---

## Event Detection & Clustering

### Cluster Detection

Velocity-based event cluster detection.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `CLUSTER_VELOCITY_THRESHOLD` | `2.0` | Messages per hour to form cluster | No |
| `CLUSTER_TIME_WINDOW_HOURS` | `2` | Sliding time window for velocity calculation | No |
| `CLUSTER_SIMILARITY_THRESHOLD` | `0.80` | Minimum embedding similarity for clustering | No |
| `MIN_MESSAGES_FOR_CLUSTER` | `3` | Minimum messages to form a cluster | No |
| `CLUSTER_RUMOR_TTL_HOURS` | `24` | Hours before archiving unconfirmed rumors | No |

**Tier Progression (Automatic):**
- **1 channel** → rumor (red)
- **2-3 channels, same affiliation** → unconfirmed (yellow)
- **3+ channels, cross-affiliation** → confirmed (orange)
- **Human verified** → verified (green)

### Comment Polling

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `COMMENT_BACKFILL_MAX_AGE_DAYS` | `30` | Maximum message age for comment backfill | No |
| `COMMENT_BACKFILL_BATCH_SIZE` | `50` | Messages per backfill batch | No |
| `COMMENT_TIER_HOT_INTERVAL` | `4` | Polling interval for hot messages (hours) | No |
| `COMMENT_TIER_WARM_INTERVAL` | `24` | Polling interval for warm messages (hours) | No |
| `COMMENT_TIER_COOL_INTERVAL` | `168` | Polling interval for cool messages (hours, 7 days) | No |
| `COMMENT_AUTO_TRANSLATE` | `false` | Auto-translate comments | No |

### Engagement Metrics

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `VIRAL_VIEWS_MULTIPLIER` | `3` | Channel median views multiplier for viral threshold | No |
| `VIRAL_FORWARDS_MIN` | `50` | Minimum forwards to consider viral | No |
| `VIRAL_COMMENTS_MIN` | `20` | Minimum comments to consider viral | No |
| `VIRAL_VELOCITY_VIEWS_PER_HOUR` | `1000` | Views per hour threshold for viral velocity | No |

**Example:**
```bash
CLUSTER_VELOCITY_THRESHOLD=2.0
CLUSTER_TIME_WINDOW_HOURS=2
CLUSTER_SIMILARITY_THRESHOLD=0.80
MIN_MESSAGES_FOR_CLUSTER=3
```

---

## OpenSanctions Integration

Entity matching against sanctions/PEP databases.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `OPENSANCTIONS_BATCH_SIZE` | `10` | Entities per batch | No |
| `OPENSANCTIONS_POLL_INTERVAL` | `60` | Polling interval in seconds | No |

---

## Entity Ingestion

CSV-based entity import with auto-embedding generation.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `ENTITY_INGESTION_ENABLED` | `false` | Enable automatic CSV entity ingestion | No |
| `ENTITY_CSV_DIR` | `/data/entities/csv/` | Directory to scan for entity CSV files | No |
| `ENTITY_PROCESSED_FILE` | `/data/entities/processed/status.json` | Status file tracking processed CSVs | No |
| `ENTITY_SCAN_INTERVAL` | `300` | CSV directory scan interval in seconds | No |
| `ENTITY_BATCH_SIZE` | `100` | Entities per processing batch | No |
| `ENTITY_GENERATE_EMBEDDINGS` | `true` | Auto-generate embeddings for entities | No |
| `ENTITY_EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Embedding model for entities | No |

**Example:**
```bash
ENTITY_INGESTION_ENABLED=true
ENTITY_CSV_DIR=/data/entities/csv/
ENTITY_BATCH_SIZE=100
```

---

## Metrics & Monitoring

Prometheus metrics exposure.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `METRICS_PORT` | Service-specific | Port for Prometheus metrics endpoint | No |

**Default Ports by Service:**
- Enrichment Router: `9198`
- Fast Worker: `9199`
- Geolocation LLM Worker: `9200`
- Decision Worker: `9201`
- Maintenance Worker: `9202`
- AI Tagging Worker: `9096`
- RSS Validation Worker: `9097`
- Cluster Detection Worker: `9211`
- Enrichment Service: `9095`

---

## Worker-Specific Settings

### Enrichment Router

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `ROUTER_POLL_INTERVAL` | `30` | Queue polling interval in seconds | No |
| `ROUTER_BATCH_SIZE` | `100` | Messages per routing batch | No |

### Fast Worker

Translation and entity extraction worker.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `TIME_BUDGET_SECONDS` | `60` | Maximum time per worker cycle | No |
| `BATCH_SIZE` | `50` | Messages per batch | No |
| `CYCLE_INTERVAL_SECONDS` | `30` | Time between worker cycles | No |

### Telegram Worker

Engagement polling, comment fetching, social graph extraction.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `TIME_BUDGET_SECONDS` | `120` | Maximum time per worker cycle | No |
| `BATCH_SIZE` | `20` | Messages per batch | No |
| `CYCLE_INTERVAL_SECONDS` | `30` | Time between worker cycles | No |
| `RATE_LIMIT_PER_SECOND` | `20` | Telegram API rate limit (requests/second) | No |

### Maintenance Worker

Database maintenance and cleanup tasks.

| Variable | Default | Description | Required |
|----------|---------|-------------|----------|
| `TIME_BUDGET_SECONDS` | `120` | Maximum time per worker cycle | No |
| `BATCH_SIZE` | `100` | Records per batch | No |
| `CYCLE_INTERVAL_SECONDS` | `300` | Time between worker cycles (5 minutes) | No |

---

## Complete Example Configuration

Here's a complete example `.env` file for production deployment:

```bash
# ============================================================================
# Core Infrastructure
# ============================================================================
DEPLOYMENT_MODE=production

# ============================================================================
# Database
# ============================================================================
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=osint_platform
POSTGRES_USER=osint_user
POSTGRES_PASSWORD=your_secure_db_password_here

# ============================================================================
# Redis
# ============================================================================
REDIS_URL=redis://redis:6379/0

# ============================================================================
# Authentication
# ============================================================================
AUTH_PROVIDER=jwt
JWT_SECRET_KEY=your_64_char_hex_secret_key_generated_with_openssl_rand_hex_64
JWT_ALGORITHM=HS256
JWT_EXPIRATION_MINUTES=60
JWT_ADMIN_PASSWORD=your_secure_admin_password

# ============================================================================
# Frontend
# ============================================================================
NEXT_PUBLIC_API_URL=https://api.osintukraine.com
FRONTEND_URL=https://v2.osintukraine.com

# ============================================================================
# Telegram Accounts
# ============================================================================
# Russia account (default)
TELEGRAM_API_ID=12345678
TELEGRAM_API_HASH=abcdef1234567890abcdef1234567890
TELEGRAM_PHONE=+79991234567

# Ukraine account
TELEGRAM_API_ID_UKRAINE=87654321
TELEGRAM_API_HASH_UKRAINE=fedcba0987654321fedcba0987654321
TELEGRAM_PHONE_UKRAINE=+380991234567

# ============================================================================
# Media Storage
# ============================================================================
MEDIA_BUFFER_PATH=/var/cache/osint-media-buffer
HETZNER_STORAGE_URL=https://your-storage-box.com/media
MINIO_ENDPOINT=minio:9000
MINIO_ACCESS_KEY=admin
MINIO_SECRET_KEY=your_minio_secret_key
MINIO_BUCKET=osint-media
MINIO_PUBLIC_URL=https://minio.osintukraine.com

# ============================================================================
# Enrichment Services
# ============================================================================
ENRICHMENT_TASKS=translation,embedding,ai_tagging,geolocation,event_detection
DEEPL_API_KEY=your_deepl_api_key_here
OLLAMA_HOST=http://ollama:11434
AI_TAGGING_MODEL=qwen2.5:3b
EMBEDDING_MODEL=all-MiniLM-L6-v2

# ============================================================================
# Map API Rate Limiting & Caching
# ============================================================================
MAP_MESSAGES_RATE_LIMIT=120
MAP_CLUSTERS_RATE_LIMIT=120
MAP_EVENTS_RATE_LIMIT=60
MAP_HEATMAP_RATE_LIMIT=60

MAP_CACHE_TTL_MESSAGES=60
MAP_CACHE_TTL_CLUSTERS=300
MAP_CACHE_TTL_EVENTS=300
MAP_CACHE_TTL_HEATMAP=300

# ============================================================================
# Event Detection & Clustering
# ============================================================================
CLUSTER_VELOCITY_THRESHOLD=2.0
CLUSTER_TIME_WINDOW_HOURS=2
CLUSTER_SIMILARITY_THRESHOLD=0.80
MIN_MESSAGES_FOR_CLUSTER=3

# ============================================================================
# Geolocation
# ============================================================================
NOMINATIM_URL=https://nominatim.openstreetmap.org
NOMINATIM_RATE_LIMIT=1.0

# ============================================================================
# WebSocket Security
# ============================================================================
WEBSOCKET_MAX_CONNECTIONS_PER_IP=10
WEBSOCKET_RATE_LIMIT=10
WEBSOCKET_ALLOW_NO_ORIGIN=false
```

---

## Environment-Specific Recommendations

### Development

```bash
DEPLOYMENT_MODE=development
AUTH_PROVIDER=none
MAP_CACHE_TTL_MESSAGES=10  # Shorter cache for faster iteration
OLLAMA_HOST=http://localhost:11434
```

### Staging

```bash
DEPLOYMENT_MODE=staging
AUTH_PROVIDER=jwt
MAP_MESSAGES_RATE_LIMIT=60  # More restrictive than dev
```

### Production

```bash
DEPLOYMENT_MODE=production
AUTH_PROVIDER=jwt
JWT_SECRET_KEY=<strong 64-char key>
MAP_MESSAGES_RATE_LIMIT=120
WEBSOCKET_MAX_CONNECTIONS_PER_IP=10
WEBSOCKET_ALLOW_NO_ORIGIN=false
```

---

## Security Best Practices

### Required for Production

1. **Set strong `JWT_SECRET_KEY`**: Generate with `openssl rand -hex 64`
2. **Set `POSTGRES_PASSWORD`**: Use strong random password
3. **Configure `FRONTEND_URL`**: Match your production domain
4. **Restrict CORS origins**: Only allow trusted domains in `API_CORS_ORIGINS`
5. **Enable authentication**: Set `AUTH_PROVIDER=jwt` and `AUTH_REQUIRED=true`

### Recommended

- Store secrets in `.env` file (excluded from git via `.gitignore`)
- Use Docker secrets or environment variable injection for production
- Rotate `JWT_SECRET_KEY` periodically
- Monitor rate limits and adjust based on legitimate traffic patterns
- Enable HTTPS for `MINIO_ENDPOINT` and `HETZNER_STORAGE_URL`

---

## Troubleshooting

### Common Issues

**JWT authentication not working:**
- Verify `JWT_SECRET_KEY` is at least 32 characters
- Check `JWT_ADMIN_PASSWORD` is set
- Ensure frontend sends `Authorization: Bearer <token>` header

**Map endpoints returning 429 (rate limited):**
- Increase `MAP_*_RATE_LIMIT` values
- Check if rate limit is per-IP (legitimate users behind NAT may share IP)

**Media not syncing to Hetzner:**
- Verify `HETZNER_STORAGE_URL` is accessible
- Check `MEDIA_BUFFER_PATH` has write permissions
- Monitor media-sync worker logs

**Geolocation not working:**
- Verify `NOMINATIM_URL` is accessible
- Check `NOMINATIM_RATE_LIMIT` compliance (OSM default: 1 req/sec)
- Ensure gazetteer table is populated

**Telegram features disabled:**
- Verify `TELEGRAM_API_ID` and `TELEGRAM_API_HASH` are set
- Check session file exists at `TELEGRAM_SESSION_PATH`
- Confirm phone number format: `+1234567890` (no spaces)

---

## See Also

- [Architecture Documentation](~/code/osintukraine/osint-platform-docs/docs/architecture/)
- [Deployment Guide](~/code/osintukraine/osint-platform-docs/docs/deployment/)
- [API Reference](~/code/osintukraine/osint-platform-docs/docs/api/)
