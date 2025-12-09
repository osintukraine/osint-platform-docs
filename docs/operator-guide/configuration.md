# Configuration

Complete reference for configuring the OSINT Intelligence Platform services and components.

## Overview

The platform uses environment variables for all configuration, centralized in a `.env` file. Configuration is grouped by functional area with sensible defaults for development.

**Configuration Philosophy**:

- Environment variables for all secrets and deployment-specific values
- No hardcoded paths or credentials in code
- Validation on startup with clear error messages
- Separate development and production configurations

## Quick Start

```bash
# Copy example configuration
cp .env.example .env

# Edit configuration (minimum required changes)
nano .env

# Required: Change these before starting
POSTGRES_PASSWORD=...           # Strong password
REDIS_PASSWORD=...              # Strong password
MINIO_ACCESS_KEY=...            # Access key
MINIO_SECRET_KEY=...            # Secret key (32+ chars)
TELEGRAM_API_ID=...             # From my.telegram.org
TELEGRAM_API_HASH=...           # From my.telegram.org
TELEGRAM_PHONE=...              # Your phone number
DEEPL_API_KEY=...               # From deepl.com
JWT_SECRET_KEY=...              # openssl rand -hex 32

# For production authentication (Ory Kratos)
KRATOS_SECRET_COOKIE=...        # openssl rand -base64 32
KRATOS_SECRET_CIPHER=...        # openssl rand -base64 24 (MUST be 32 chars)

# Verify configuration
docker-compose config
```

## Environment Variables Reference

### Core Configuration

#### Environment Settings

```bash
# Environment: development, staging, production
ENVIRONMENT=development

# Debug mode (NEVER enable in production)
DEBUG=true

# Logging
LOG_LEVEL=INFO          # DEBUG, INFO, WARNING, ERROR, CRITICAL
LOG_FORMAT=json         # json or text
```

**Production Settings**:
- `ENVIRONMENT=production`
- `DEBUG=false`
- `LOG_LEVEL=INFO` or `WARNING`
- `LOG_FORMAT=json` (for structured logging)

### PostgreSQL Configuration

```bash
# Database connection
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=osint_platform
POSTGRES_USER=osint_user
POSTGRES_PASSWORD=CHANGE_ME_STRONG_PASSWORD_HERE

# Connection pooling
POSTGRES_POOL_SIZE=20           # Concurrent connections per worker
POSTGRES_MAX_OVERFLOW=10        # Additional connections under load
POSTGRES_POOL_TIMEOUT=30        # Seconds to wait for connection
POSTGRES_POOL_RECYCLE=3600      # Recycle connections after 1 hour
```

**Tuning for Scale**:

- **Laptop (dev)**: `POOL_SIZE=5-10`
- **VPS (production)**: `POOL_SIZE=20-50`
- **Monitor**: `pg_stat_activity` for connection usage

**Connection String** (auto-constructed):
```bash
DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}
```

### Redis Configuration

```bash
# Redis connection
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=CHANGE_ME_STRONG_PASSWORD_HERE
REDIS_DB=0

# Redis Streams configuration
REDIS_STREAM_NAME=telegram_messages
REDIS_CONSUMER_GROUP=processor_group
REDIS_MAX_STREAM_LENGTH=100000          # ~1GB memory usage
```

**Tuning Redis**:

- `MAX_STREAM_LENGTH=100000`: Standard (1GB RAM)
- `MAX_STREAM_LENGTH=50000`: Constrained memory (512MB RAM)
- Monitor with: `docker-compose exec redis redis-cli INFO memory`

### MinIO (S3) Configuration

```bash
# MinIO connection
MINIO_ENDPOINT=minio:9000
MINIO_ACCESS_KEY=CHANGE_ME_ACCESS_KEY
MINIO_SECRET_KEY=CHANGE_ME_SECRET_KEY_AT_LEAST_32_CHARS
MINIO_BUCKET_NAME=telegram-archive
MINIO_SECURE=false              # Set true for production with HTTPS
MINIO_REGION=us-east-1

# MinIO ports
MINIO_API_PORT=9000
MINIO_CONSOLE_PORT=9001

# Public URL for serving media
MINIO_PUBLIC_URL=http://localhost:9000  # Production: https://media.osintukraine.com

# Domain for virtual-host-style requests (optional)
MINIO_DOMAIN=localhost
```

**Production Configuration**:
```bash
MINIO_SECURE=true
MINIO_PUBLIC_URL=https://media.osintukraine.com
MINIO_DOMAIN=media.osintukraine.com
```

**Content-Addressed Storage**: All media uses SHA-256 hashing for automatic deduplication (30-40% storage savings).

### Telegram Configuration

#### Single Account Mode (Default)

```bash
# Telegram API credentials (from https://my.telegram.org/apps)
TELEGRAM_API_ID=YOUR_API_ID_HERE
TELEGRAM_API_HASH=YOUR_API_HASH_HERE
TELEGRAM_PHONE=+1234567890              # With country code

# Session management
TELEGRAM_SESSION_PATH=/data/sessions
TELEGRAM_SESSION_NAME=osint_platform

# Rate limiting (per channel)
TELEGRAM_RATE_LIMIT_PER_CHANNEL=20      # Messages per minute
TELEGRAM_FLOOD_WAIT_MULTIPLIER=2        # Backoff multiplier
```

#### Multi-Account Mode (Optional)

For scaled deployments with separate Russia/Ukraine accounts:

```bash
# Russia account (monitors Archive-RU-*, Monitor-RU-* folders)
TELEGRAM_API_ID_RUSSIA=YOUR_RUSSIA_API_ID_HERE
TELEGRAM_API_HASH_RUSSIA=YOUR_RUSSIA_API_HASH_HERE
TELEGRAM_PHONE_RUSSIA=+1234567890

# Ukraine account (monitors Archive-UA-*, Monitor-UA-* folders)
TELEGRAM_API_ID_UKRAINE=YOUR_UKRAINE_API_ID_HERE
TELEGRAM_API_HASH_UKRAINE=YOUR_UKRAINE_API_HASH_HERE
TELEGRAM_PHONE_UKRAINE=+0987654321
```

**Enable Multi-Account**:
```bash
# Authenticate each account
python3 scripts/telegram_auth.py --account russia
python3 scripts/telegram_auth.py --account ukraine

# Start multi-account listeners
docker-compose --profile multi-account up -d listener-russia listener-ukraine

# Stop default listener (optional)
docker-compose stop listener
```

### Historical Backfill Configuration

```bash
# Enable automatic backfill of historical messages
BACKFILL_ENABLED=false                  # Set true when ready

# Start date for historical backfill (ISO format: YYYY-MM-DD)
BACKFILL_START_DATE=2024-01-01         # Examples:
                                        # 2022-02-24 = Russian invasion start
                                        # 2024-01-01 = Current year
                                        # 2025-10-01 = Last 30 days

# Backfill mode
BACKFILL_MODE=manual                    # on_discovery, manual, scheduled

# Messages per batch (rate-limit friendly)
BACKFILL_BATCH_SIZE=100                 # Safe for Telegram API

# Delay between batches (milliseconds)
BACKFILL_DELAY_MS=1000                  # 1s = ~6000 msgs/hour safely

# Media handling strategy
BACKFILL_MEDIA_STRATEGY=download_available  # download_available, skip, download_all
```

**Backfill Strategies**:

- `download_available`: Try download, mark unavailable if 404 (RECOMMENDED)
- `skip`: Don't download media (faster, text-only)
- `download_all`: Fail if media unavailable (strict mode)

### Ollama (Local LLM) Configuration

```bash
# Enable/disable LLM features
LLM_ENABLED=true
LLM_PROVIDER=ollama                     # Currently only "ollama" or "none"

# Ollama connection
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_PORT=11434

# Model selection (RECOMMENDED: qwen2.5:3b for Russian/Ukrainian OSINT)
OLLAMA_MODEL=qwen2.5:3b
OLLAMA_TIMEOUT=30

# Fallback behavior
LLM_FALLBACK_TO_RULES=true              # Use rule-based scoring if LLM fails
```

**Model Recommendations**:

| Use Case | Model | RAM | CPU | Quality |
|----------|-------|-----|-----|---------|
| **Production (RU/UK OSINT)** | `qwen2.5:3b` | 2.4GB | 180% | 87% |
| **Development (fast)** | `gemma2:2b` | 1.5GB | 50% | 75% |
| **Production (fallback)** | `llama3.2:3b` | 2.5GB | 200% | 85% |
| **Specialized (reasoning)** | `phi3.5:3.8b` | 3GB | 250% | 90% |

**Performance Tuning (Development)**:

```bash
DEVELOPMENT_MODE=true                   # Optimize for laptop

# Ollama resource limits
OLLAMA_NUM_PARALLEL=1                   # Process 1 request at a time
OLLAMA_MAX_LOADED_MODELS=3              # Keep max 3 models in memory
OLLAMA_CPU_THREADS=2                    # Limit CPU threads
OLLAMA_KEEP_ALIVE=2m                    # Unload after 2 min idle
OLLAMA_FLASH_ATTENTION=false            # Disable CPU-intensive feature
OLLAMA_NUM_PREDICT=250                  # Max tokens (faster)
```

**Production Settings**:
```bash
DEVELOPMENT_MODE=false
OLLAMA_NUM_PARALLEL=2                   # Allow 2 concurrent requests
OLLAMA_MAX_LOADED_MODELS=5              # More models
OLLAMA_CPU_THREADS=6                    # Use more cores
OLLAMA_KEEP_ALIVE=30m                   # Keep models loaded
```

### Folder-Based Channel Management

Channels are auto-discovered based on Telegram folder names:

```bash
# Folder patterns (HARDCODED in services/listener/src/channel_discovery.py)
# Archive* → archive_all (store all non-spam messages)
# Monitor* → selective_archive (only high importance)
# Discover* → discovery (auto-joined, 14-day probation)

# Sync interval
FOLDER_SYNC_INTERVAL=300                # 5 minutes

# OSINT score threshold for selective_archive
MONITORING_OSINT_THRESHOLD=70           # Messages below this not archived
```

**To change folder patterns**: Edit `services/listener/src/channel_discovery.py` and rebuild.

### Translation Configuration

```bash
# Enable/disable translation
TRANSLATION_ENABLED=true
TRANSLATION_PROVIDER=deepl              # deepl (primary), google (fallback), none
TRANSLATION_TARGET_LANG=en

# Source languages (comma-separated, empty = all)
TRANSLATION_FROM_LANGUAGES=ru,uk

# DeepL API (free Pro account available)
DEEPL_API_KEY=YOUR_DEEPL_API_KEY_HERE
DEEPL_API_URL=https://api-free.deepl.com/v2  # or https://api.deepl.com/v2 for paid

# Google Translate API (optional fallback)
GOOGLE_TRANSLATE_API_KEY=YOUR_GOOGLE_API_KEY_HERE_OPTIONAL

# Translation budget tracking
TRANSLATION_DAILY_BUDGET_USD=999999.0   # Effectively unlimited for free DeepL
```

**Cost Optimization**: Translation moved from listener to processor for 80-90% cost savings (translate only non-spam messages).

### Worker Configuration

```bash
# Number of processor workers
WORKER_COUNT=4                          # Match CPU cores

# Batch size for processing
WORKER_BATCH_SIZE=50

# Worker timeout (seconds)
WORKER_TIMEOUT=300

# Processing feature flags
SPAM_FILTER_ENABLED=true
ENTITY_EXTRACTION_ENABLED=true
TRANSLATION_ENABLED_IN_WORKER=true
LLM_SCORING_ENABLED=true
```

**Scaling Guidelines**:

- **Laptop**: `WORKER_COUNT=1-2`
- **VPS (4 cores)**: `WORKER_COUNT=4`
- **VPS (8 cores)**: `WORKER_COUNT=8`
- Monitor with: `docker stats osint-processor-worker*`

### Comment Polling Configuration

```bash
# Tiered polling intervals (hours)
COMMENT_TIER_HOT_INTERVAL=4             # 0-24h old messages
COMMENT_TIER_WARM_INTERVAL=24           # 1-7 days old
COMMENT_TIER_COOL_INTERVAL=168          # 7-30 days old (weekly)

# Comment backfill settings
COMMENT_BACKFILL_MAX_AGE_DAYS=30
COMMENT_BACKFILL_BATCH_SIZE=50

# Auto-translate comments (default: false, users translate on-demand)
COMMENT_AUTO_TRANSLATE=false
```

### Viral Post Detection

```bash
# Viral posts get enhanced polling regardless of age

# Views threshold (message is viral if views > channel_avg * multiplier)
VIRAL_VIEWS_MULTIPLIER=3

# Forward threshold
VIRAL_FORWARDS_MIN=50

# Comments threshold
VIRAL_COMMENTS_MIN=20

# Velocity threshold (views per hour)
VIRAL_VELOCITY_VIEWS_PER_HOUR=1000
```

### API Configuration

```bash
# API server settings
API_HOST=0.0.0.0
API_PORT=8000
API_WORKERS=4
API_RELOAD=true                         # Set false for production

# Authentication provider
AUTH_PROVIDER=none                      # none, jwt, cloudron, ory
AUTH_REQUIRED=false                     # If true, reject unauthenticated

# JWT authentication (if AUTH_PROVIDER=jwt)
JWT_SECRET_KEY=CHANGE_ME_RANDOM_256_BIT_KEY_HERE  # openssl rand -hex 32
JWT_ALGORITHM=HS256
JWT_EXPIRATION_MINUTES=60
JWT_REFRESH_EXPIRATION_DAYS=30

# Rate limiting (per user/IP)
API_RATE_LIMIT_PER_MINUTE=60
API_RATE_LIMIT_BURST=100

# Feed authentication
FEED_AUTH_REQUIRED=false                # Require auth for RSS/Atom/JSON feeds

# CORS origins (comma-separated)
API_CORS_ORIGINS=http://localhost:3000,http://localhost:5173
```

**Production API Settings**:
```bash
API_RELOAD=false
AUTH_PROVIDER=ory
AUTH_REQUIRED=true
API_CORS_ORIGINS=https://osintukraine.com,https://www.osintukraine.com
```

### Ory Authentication (Production)

```bash
# ============================================================================
# ORY KRATOS (Identity Management)
# ============================================================================

# Service URLs (internal Docker network)
ORY_KRATOS_PUBLIC_URL=http://ory-kratos:4433
ORY_KRATOS_ADMIN_URL=http://ory-kratos:4434
ORY_OATHKEEPER_URL=http://ory-oathkeeper:4455

# Secrets (REQUIRED - MUST be strong random values)
# CRITICAL: KRATOS_SECRET_CIPHER must be EXACTLY 32 characters
KRATOS_SECRET_COOKIE=CHANGE_ME_GENERATE_WITH_OPENSSL_RAND_BASE64_32
KRATOS_SECRET_CIPHER=CHANGE_ME_EXACTLY_32_CHARACTERS!

# Generate secrets:
# KRATOS_SECRET_COOKIE=$(openssl rand -base64 32)
# KRATOS_SECRET_CIPHER=$(openssl rand -base64 24)  # Produces exactly 32 chars

# Social login (optional)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=

# Email configuration (REQUIRED for production)
# Development: Uses MailSlurper (included)
SMTP_CONNECTION_URI=smtp://mailslurper:1025/?skip_ssl_verify=true
SMTP_FROM_ADDRESS=noreply@localhost
SMTP_FROM_NAME=OSINT Platform

# Production examples:
# SendGrid:
#   SMTP_CONNECTION_URI=smtps://apikey:YOUR_SENDGRID_API_KEY@smtp.sendgrid.net:465
#   SMTP_FROM_ADDRESS=noreply@osintukraine.com
#   SMTP_FROM_NAME=OSINT Ukraine Platform
```

### Frontend Configuration

```bash
# Frontend dev server port
FRONTEND_PORT=3000

# API URLs (for frontend to call backend)
NEXT_PUBLIC_API_URL=http://localhost:8000          # Client-side (browser)
NEXT_PUBLIC_RSS_URL=http://localhost:8000/rss

# Base URL for OpenGraph meta tags
NEXT_PUBLIC_BASE_URL=http://localhost:3000         # Production: https://osintukraine.com

# Node environment
NODE_ENV=development                                # production for builds
```

**CRITICAL**: Use `NEXT_PUBLIC_API_URL` for all client-side API calls. Never use relative paths (Docker networking doesn't support rewrites).

### Monitoring Configuration

```bash
# Prometheus
PROMETHEUS_PORT=9090

# Grafana
GRAFANA_PORT=3001
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=CHANGE_ME_STRONG_PASSWORD_HERE
GRAFANA_ROOT_URL=http://localhost:3001

# Grafana SMTP (optional - for email alerts)
GRAFANA_SMTP_ENABLED=false
GRAFANA_SMTP_HOST=smtp.example.com:587
GRAFANA_SMTP_USER=alerts@example.com
GRAFANA_SMTP_PASSWORD=smtp_password_here
GRAFANA_SMTP_FROM=alerts@example.com
```

### Notification System (ntfy)

```bash
# ntfy server connection
NTFY_URL=http://ntfy:80
NTFY_PORT=8090
NTFY_BASE_URL=http://localhost:8090

# Notification topic prefix
NTFY_TOPIC_PREFIX=osint-platform

# Message retention
NTFY_CACHE_DURATION=168h                            # 7 days

# Authentication (disable for local dev)
NTFY_ENABLE_LOGIN=false
NTFY_ENABLE_SIGNUP=false

# Rate limiting
NTFY_RATE_LIMIT_EVENTS=100                          # Max events/min per topic

# Aggregator service
AGGREGATOR_ENABLED=true
AGGREGATOR_BATCH_INTERVAL=300                       # 5 minutes
AGGREGATOR_MAX_BATCH_SIZE=50

# Feature flags (disable to reduce noise)
NOTIFY_MESSAGE_ARCHIVED=true
NOTIFY_SPAM_DETECTED=true
NOTIFY_OSINT_HIGH=true
NOTIFY_OSINT_CRITICAL=true
NOTIFY_LLM_ACTIVITY=true
NOTIFY_CHANNEL_DISCOVERY=true
NOTIFY_SYSTEM_HEALTH=true
NOTIFY_API_ERRORS=true
```

### RSS Intelligence Layer (Optional)

```bash
# Master toggle
RSS_INGESTION_ENABLED=true

# Poll interval
RSS_INGESTION_INTERVAL_MINUTES=5                    # 12 polls/hour

# Feed limits
RSS_MAX_FEEDS=50
RSS_MAX_ARTICLES_PER_POLL=50

# Retention
RSS_RETENTION_DAYS=90                               # 3 months

# Cross-correlation settings
RSS_CORRELATION_ENABLED=true
RSS_CORRELATION_SIMILARITY_THRESHOLD=0.40           # Cross-lingual RU/UK → EN
RSS_CORRELATION_TIME_WINDOW_HOURS=6                 # ±6h window
RSS_CORRELATION_MAX_PER_MESSAGE=10

# Fact-checking
RSS_FACT_CHECK_ENABLED=true
RSS_FACT_CHECK_MIN_OSINT_SCORE=75                   # Only check high-value msgs
RSS_FACT_CHECK_MIN_CORRELATION=80                   # 80% confidence

# Alternative viewpoints
RSS_SHOW_ALTERNATIVE_VIEWPOINTS=true

# Performance tuning
RSS_WORKER_THREADS=2
RSS_EMBEDDING_BATCH_SIZE=10

# Outbound RSS (existing feature)
RSS_OUTBOUND_ENABLED=true
RSS_CACHE_MINUTES=10
```

### OpenSanctions Entity Intelligence (Optional)

```bash
# Master toggle
OPENSANCTIONS_ENABLED=false

# Backend mode: "api" (public API) or "yente" (self-hosted)
OPENSANCTIONS_BACKEND=yente                         # RECOMMENDED for production

# API key (required for API mode, optional for yente bulk data)
OPENSANCTIONS_API_KEY=

# Dataset selection
OPENSANCTIONS_DATASET=default                       # default, sanctions, peps

# Matching configuration
OPENSANCTIONS_MATCH_THRESHOLD=85                    # 0-100, higher = fewer matches

# Processing
OPENSANCTIONS_BATCH_SIZE=10
OPENSANCTIONS_POLL_INTERVAL=60                      # seconds

# Embeddings (semantic search)
OPENSANCTIONS_GENERATE_EMBEDDINGS=true
OPENSANCTIONS_EMBEDDING_MODEL=all-MiniLM-L6-v2

# Yente configuration (if using yente backend)
YENTE_UPDATE_TOKEN=unsafe-default-token-change-in-production
```

**Start OpenSanctions stack**:
```bash
docker-compose --profile opensanctions up -d
```

### Entity Ingestion Service (CSV Import)

```bash
# Master toggle
ENTITY_INGESTION_ENABLED=true

# Scan interval (how often to check for new CSVs)
ENTITY_SCAN_INTERVAL=300                            # 5 minutes

# Batch size for database inserts
ENTITY_BATCH_SIZE=100                               # Balance speed/memory

# Embeddings
ENTITY_GENERATE_EMBEDDINGS=true
ENTITY_EMBEDDING_MODEL=all-MiniLM-L6-v2

# Matching threshold
ENTITY_MATCHING_THRESHOLD=0.65                      # 0.75 recommended
```

### NocoDB Configuration (Optional)

```bash
# NocoDB port
NOCODB_PORT=8080

# Public URL (for shared views)
NOCODB_PUBLIC_URL=http://localhost:8080

# Authentication
NOCODB_JWT_SECRET=change-this-secret-in-production-min-32-chars

# Initial admin account
NOCODB_ADMIN_EMAIL=admin@osint.local
NOCODB_ADMIN_PASSWORD=change-this-password

# SMTP (optional - for email notifications)
NOCODB_SMTP_FROM=
NOCODB_SMTP_HOST=
NOCODB_SMTP_PORT=
NOCODB_SMTP_USERNAME=
NOCODB_SMTP_PASSWORD=
NOCODB_SMTP_SECURE=true
```

**Start NocoDB**:
```bash
docker-compose --profile dev up -d nocodb
```

### Deployment Mode & Domain

```bash
# Deployment mode
DEPLOYMENT_MODE=development                         # development or production

# Production domain
DOMAIN=osint.example.com

# Service URLs (auto-configured based on DEPLOYMENT_MODE)
FRONTEND_URL=http://localhost:3000
API_URL=http://localhost:8000
PROXY_URL=http://localhost:8000
```

**Production Settings**:
```bash
DEPLOYMENT_MODE=production
DOMAIN=osintukraine.com
FRONTEND_URL=https://osintukraine.com
API_URL=https://osintukraine.com/api
PROXY_URL=https://osintukraine.com
```

## Docker Compose Profiles

The platform uses Docker Compose profiles to organize services:

```bash
# Start core services only (listener, processor, api, frontend)
docker-compose up -d

# Start with monitoring stack
docker-compose --profile monitoring up -d

# Start with enrichment workers
docker-compose --profile enrichment up -d

# Start with OpenSanctions integration
docker-compose --profile opensanctions up -d

# Start with authentication (Ory Kratos/Oathkeeper)
docker-compose --profile auth up -d

# Start with development tools (NocoDB, Dashy, MailSlurper)
docker-compose --profile dev up -d

# Start multi-account listeners
docker-compose --profile multi-account up -d

# Combine profiles
docker-compose --profile monitoring --profile enrichment up -d
```

### Available Profiles

| Profile | Services | Use Case |
|---------|----------|----------|
| `(none)` | listener, processor, api, frontend, postgres, redis, minio, ollama | Core platform |
| `monitoring` | prometheus, grafana, alertmanager, node-exporter, postgres-exporter, redis-exporter, cadvisor, dozzle, ntfy, notifier | Operational monitoring |
| `enrichment` | ai-tagging, event-detection, router, rss-validation, fast-pool, telegram-worker, decision-worker, maintenance, analytics, ollama-batch | Background enrichment |
| `opensanctions` | yente, yente-index, opensanctions, entity-ingestion | Entity intelligence |
| `auth` | kratos, oathkeeper, caddy, mailslurper | Authentication |
| `dev` | nocodb, dashy, mkdocs, mailslurper | Development tools |
| `multi-account` | listener-russia, listener-ukraine | Multi-account Telegram |

## Scaling Configuration

### Horizontal Scaling (Replicas)

```bash
# Scale processor workers
docker-compose up -d --scale processor-worker=4

# Scale specific enrichment workers
docker-compose --profile enrichment up -d --scale enrichment-fast-pool=2
```

**Environment Variable Control**:
```bash
# In .env file
PROCESSOR_REPLICAS=2                    # Number of processor worker replicas
```

### Vertical Scaling (Resources)

Edit `docker-compose.yml` to adjust CPU/memory limits:

```yaml
services:
  ollama:
    deploy:
      resources:
        limits:
          cpus: '6.0'                   # 6 cores for realtime
          memory: 8G
        reservations:
          cpus: '2.0'
          memory: 4G
```

## Configuration Validation

### Pre-flight Checks

```bash
# Validate docker-compose configuration
docker-compose config

# Check for syntax errors
docker-compose config --quiet

# Verify environment variables are loaded
docker-compose config | grep TELEGRAM_API_ID
```

### Startup Validation

The platform validates configuration on startup:

- Database connectivity
- Redis connectivity
- MinIO bucket existence
- Telegram session validity
- Required secrets present
- Model availability (Ollama)

**View validation logs**:
```bash
docker-compose logs listener processor api | grep -i "error\|warning"
```

## Security Checklist

Before deploying to production:

- [ ] Changed all `CHANGE_ME` placeholders
- [ ] Used strong random passwords (32+ characters)
- [ ] Generated new `JWT_SECRET_KEY` (openssl rand -hex 32)
- [ ] Generated `KRATOS_SECRET_COOKIE` (openssl rand -base64 32)
- [ ] Generated `KRATOS_SECRET_CIPHER` (openssl rand -base64 24) - MUST be 32 chars
- [ ] Configured production SMTP (not MailSlurper)
- [ ] Set `DEBUG=false` and `ENVIRONMENT=production`
- [ ] Set `API_RELOAD=false`
- [ ] Set `MINIO_SECURE=true` (with HTTPS)
- [ ] Configured `MINIO_PUBLIC_URL` to production domain
- [ ] Configured `API_CORS_ORIGINS` to production domains
- [ ] Verified `.env` is in `.gitignore`
- [ ] Never shared `.env` file or credentials

## Performance Tuning

### Development (Laptop)

```bash
PROCESSOR_REPLICAS=1
WORKER_COUNT=1-2
API_WORKERS=2
POSTGRES_POOL_SIZE=5-10
OLLAMA_MODEL=gemma2:2b                  # Fast, low CPU
DEVELOPMENT_MODE=true
```

### Production (VPS)

```bash
PROCESSOR_REPLICAS=2-4                  # Match CPU cores
WORKER_COUNT=4-8
API_WORKERS=4-8
POSTGRES_POOL_SIZE=20-50
OLLAMA_MODEL=qwen2.5:3b                 # Best for RU/UK
DEVELOPMENT_MODE=false
REDIS_MAX_STREAM_LENGTH=100000          # ~1GB memory
```

## Configuration Files

Beyond environment variables, some configuration is in files:

### Intelligence Rules

**File**: `/config/osint_rules.yml`

```yaml
rules:
  - name: "Combat Equipment"
    conditions:
      - type: "keyword"
        values: ["tank", "artillery", "drone"]
    score: 85
    topic: "equipment"
```

**Reload rules**: Restart processor workers
```bash
docker-compose restart processor-worker
```

### Spam Filter Patterns

**File**: `services/processor/src/spam_filter.py`

Hardcoded patterns for:
- Financial spam (donation scams)
- Off-topic content (Israel/Gaza, US politics)

**To customize**: Edit file and rebuild
```bash
docker-compose build processor-worker
docker-compose up -d processor-worker
```

## Troubleshooting Configuration

### Configuration Not Loading

```bash
# Check .env file exists
ls -la .env

# Check syntax (no spaces around =)
cat .env | grep "="

# Verify Docker Compose sees it
docker-compose config | grep YOUR_VARIABLE
```

### Service Can't Connect to Another Service

```bash
# Check network configuration
docker-compose ps
docker network ls
docker network inspect osint-intelligence-platform_backend

# Verify service names match
docker-compose config | grep "container_name"
```

### Environment Variable Not Applied

```bash
# Rebuild service
docker-compose build service_name

# Restart with fresh environment
docker-compose up -d --force-recreate service_name

# Verify environment inside container
docker-compose exec service_name env | grep VARIABLE_NAME
```

## Next Steps

- [Telegram Setup](telegram-setup.md) - Configure Telegram monitoring
- [Monitoring](monitoring.md) - Set up monitoring and alerts
- [Backup & Restore](backup-restore.md) - Implement backup strategies
- [Troubleshooting](troubleshooting.md) - Resolve common issues

## References

- `.env.example` - Complete environment variable template
- `docker-compose.yml` - Service configuration
- `infrastructure/postgres/postgresql.conf` - PostgreSQL tuning
- `CLAUDE.md` - Configuration rules and patterns
