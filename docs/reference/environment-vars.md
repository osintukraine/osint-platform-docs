# Environment Variables Reference

Complete reference for all environment variables in the OSINT Intelligence Platform.

**Source File**: `.env.example`
**Setup**: Copy `.env.example` to `.env` and customize for your deployment

## Quick Start Checklist

Before starting the platform:

1. Copy `.env.example` to `.env`
2. Change ALL passwords marked with `CHANGE_ME`
3. Fill in Telegram API credentials
4. Fill in DeepL API key (free tier available)
5. Generate Kratos secrets (production only)
6. Review and adjust settings for your environment

## Table of Contents

- [Environment & Logging](#environment--logging)
- [PostgreSQL Database](#postgresql-database)
- [Redis Configuration](#redis-configuration)
- [MinIO (S3 Storage)](#minio-s3-storage)
- [Telegram Configuration](#telegram-configuration)
- [Comment Polling](#comment-polling)
- [Viral Post Detection](#viral-post-detection)
- [Historical Backfill](#historical-backfill)
- [Ollama (Local LLM)](#ollama-local-llm)
- [Folder-Based Channel Management](#folder-based-channel-management)
- [Ntfy (Notifications)](#ntfy-notifications)
- [Translation](#translation)
- [Worker Configuration](#worker-configuration)
- [OpenSanctions Entity Intelligence](#opensanctions-entity-intelligence)
- [API Configuration](#api-configuration)
- [Authentication](#authentication)
- [Frontend Configuration](#frontend-configuration)
- [Monitoring](#monitoring)
- [Deployment & Domain](#deployment--domain)
- [Production-Only Settings](#production-only-settings)
- [NocoDB](#nocodb)
- [RSS Intelligence Layer](#rss-intelligence-layer)
- [Entity Ingestion](#entity-ingestion)
- [Development Helpers](#development-helpers)
- [Backup & Maintenance](#backup--maintenance)

---

## Environment & Logging

General platform environment configuration.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `ENVIRONMENT` | Environment mode | `development` | Yes |
| `DEBUG` | Enable debug mode (NEVER in production) | `true` | Yes |
| `LOG_LEVEL` | Logging level | `INFO` | Yes |
| `LOG_FORMAT` | Log format | `json` | Yes |

**Values**:
- `ENVIRONMENT`: `development`, `staging`, `production`
- `DEBUG`: `true`, `false`
- `LOG_LEVEL`: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`
- `LOG_FORMAT`: `json`, `text`

---

## PostgreSQL Database

PostgreSQL connection and pooling configuration.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `POSTGRES_HOST` | PostgreSQL hostname | `postgres` | Yes |
| `POSTGRES_PORT` | PostgreSQL port | `5432` | Yes |
| `POSTGRES_DB` | Database name | `osint_platform` | Yes |
| `POSTGRES_USER` | Database user | `osint_user` | Yes |
| `POSTGRES_PASSWORD` | Database password | `CHANGE_ME_STRONG_PASSWORD_HERE` | Yes |
| `DATABASE_URL` | Constructed database URL (auto-generated) | `postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}` | Auto |
| `POSTGRES_POOL_SIZE` | Connection pool size | `20` | No |
| `POSTGRES_MAX_OVERFLOW` | Max overflow connections | `10` | No |
| `POSTGRES_POOL_TIMEOUT` | Pool timeout (seconds) | `30` | No |
| `POSTGRES_POOL_RECYCLE` | Pool recycle time (seconds) | `3600` | No |

**Security**:
- Change `POSTGRES_PASSWORD` to a strong random password (min 32 characters)
- Never commit `.env` to version control

**Performance Tuning**:
- Development: `POSTGRES_POOL_SIZE=5-10`
- Production: `POSTGRES_POOL_SIZE=20-50`

---

## Redis Configuration

Redis connection and streams configuration.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `REDIS_HOST` | Redis hostname | `redis` | Yes |
| `REDIS_PORT` | Redis port | `6379` | Yes |
| `REDIS_PASSWORD` | Redis password | `CHANGE_ME_STRONG_PASSWORD_HERE` | Yes |
| `REDIS_DB` | Redis database number | `0` | Yes |
| `REDIS_URL` | Constructed Redis URL (auto-generated) | `redis://:${REDIS_PASSWORD}@${REDIS_HOST}:${REDIS_PORT}/${REDIS_DB}` | Auto |
| `REDIS_STREAM_NAME` | Stream name for messages | `telegram_messages` | No |
| `REDIS_CONSUMER_GROUP` | Consumer group name | `processor_group` | No |
| `REDIS_MAX_STREAM_LENGTH` | Maximum stream length | `100000` | No |

**Security**:
- Change `REDIS_PASSWORD` to a strong random password

**Memory Usage**:
- `REDIS_MAX_STREAM_LENGTH=100000` ≈ 1GB memory

---

## MinIO (S3 Storage)

MinIO object storage for media archival.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `MINIO_ENDPOINT` | MinIO endpoint | `minio:9000` | Yes |
| `MINIO_ACCESS_KEY` | MinIO access key | `CHANGE_ME_ACCESS_KEY` | Yes |
| `MINIO_SECRET_KEY` | MinIO secret key | `CHANGE_ME_SECRET_KEY_AT_LEAST_32_CHARS` | Yes |
| `MINIO_BUCKET_NAME` | S3 bucket name | `telegram-archive` | Yes |
| `MINIO_SECURE` | Use HTTPS | `false` | Yes |
| `MINIO_REGION` | S3 region | `us-east-1` | No |
| `MINIO_API_PORT` | MinIO API port | `9000` | No |
| `MINIO_CONSOLE_PORT` | MinIO console port | `9001` | No |
| `MINIO_PUBLIC_URL` | Public URL for serving media | `http://localhost:9000` | No |
| `MINIO_DOMAIN` | Domain for virtual-host-style requests | `localhost` | No |

**Security**:
- Change `MINIO_ACCESS_KEY` and `MINIO_SECRET_KEY`
- Set `MINIO_SECURE=true` in production with HTTPS

**Production**:
- Set `MINIO_PUBLIC_URL` to your public domain (e.g., `https://media.osintukraine.com`)

---

## Telegram Configuration

Telegram API credentials and session management.

### Single Account Mode (Default)

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `TELEGRAM_API_ID` | Telegram API ID | `YOUR_API_ID_HERE` | Yes |
| `TELEGRAM_API_HASH` | Telegram API hash | `YOUR_API_HASH_HERE` | Yes |
| `TELEGRAM_PHONE` | Phone number with country code | `+1234567890` | Yes |

**Get credentials**: https://my.telegram.org/apps

### Multi-Account Mode (Optional)

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `TELEGRAM_API_ID_RUSSIA` | Russia account API ID | - | No |
| `TELEGRAM_API_HASH_RUSSIA` | Russia account API hash | - | No |
| `TELEGRAM_PHONE_RUSSIA` | Russia account phone | - | No |
| `TELEGRAM_API_ID_UKRAINE` | Ukraine account API ID | - | No |
| `TELEGRAM_API_HASH_UKRAINE` | Ukraine account API hash | - | No |
| `TELEGRAM_PHONE_UKRAINE` | Ukraine account phone | - | No |

**Multi-Account Setup**:
1. Fill in country-specific credentials
2. Authenticate: `python3 scripts/telegram_auth.py --account russia`
3. Start listeners: `docker-compose --profile multi-account up -d listener-russia listener-ukraine`
4. (Optional) Stop default listener: `docker-compose stop listener`

**Folder Convention**:
- Russia account: `Archive-RU-*`, `Monitor-RU-*`, `Discover-RU`
- Ukraine account: `Archive-UA-*`, `Monitor-UA-*`, `Discover-UA`
- Default account: `Archive-*`, `Monitor-*`, `Discover-*`

### Session & Rate Limiting

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `TELEGRAM_SESSION_PATH` | Session storage path | `/data/sessions` | No |
| `TELEGRAM_SESSION_NAME` | Session file name | `osint_platform` | No |
| `TELEGRAM_RATE_LIMIT_PER_CHANNEL` | Messages per minute per channel | `20` | No |
| `TELEGRAM_FLOOD_WAIT_MULTIPLIER` | Backoff multiplier on flood-wait | `2` | No |

---

## Comment Polling

Tiered polling intervals for comment refresh.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `COMMENT_TIER_HOT_INTERVAL` | Hot tier polling interval (hours) | `4` | No |
| `COMMENT_TIER_WARM_INTERVAL` | Warm tier polling interval (hours) | `24` | No |
| `COMMENT_TIER_COOL_INTERVAL` | Cool tier polling interval (hours) | `168` | No |
| `COMMENT_BACKFILL_MAX_AGE_DAYS` | Max age for backfill (days) | `30` | No |
| `COMMENT_BACKFILL_BATCH_SIZE` | Messages per backfill batch | `50` | No |
| `COMMENT_AUTO_TRANSLATE` | Auto-translate all comments | `false` | No |

**Tiers**:
- Hot: 0-24h old messages, polled every 4 hours
- Warm: 1-7 days old messages, polled every 24 hours
- Cool: 7-30 days old messages, polled weekly

**Translation**:
- `COMMENT_AUTO_TRANSLATE=false`: On-demand translation (saves quota)
- `COMMENT_AUTO_TRANSLATE=true`: Auto-translate during fetch

---

## Viral Post Detection

Detect viral posts for enhanced polling.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `VIRAL_VIEWS_MULTIPLIER` | Views multiplier vs channel average | `3` | No |
| `VIRAL_FORWARDS_MIN` | Minimum forwards for viral status | `50` | No |
| `VIRAL_COMMENTS_MIN` | Minimum comments for viral status | `20` | No |
| `VIRAL_VELOCITY_VIEWS_PER_HOUR` | Views per hour threshold | `1000` | No |

**Logic**: Multiple conditions (OR logic). Any condition triggers viral status.

**Effect**: Viral posts get hot tier polling regardless of age.

---

## Historical Backfill

Automatic backfill of historical messages.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `BACKFILL_ENABLED` | Enable automatic backfill | `false` | Yes |
| `BACKFILL_START_DATE` | Backfill start date (YYYY-MM-DD) | `2024-01-01` | No |
| `BACKFILL_MODE` | Backfill mode | `manual` | No |
| `BACKFILL_BATCH_SIZE` | Messages per batch | `100` | No |
| `BACKFILL_DELAY_MS` | Delay between batches (ms) | `1000` | No |
| `BACKFILL_MEDIA_STRATEGY` | Media handling strategy | `download_available` | No |
| `BACKFILL_PRIORITY` | Processing priority | `lower` | No |

**Modes**:
- `manual`: Only backfill when triggered via API (RECOMMENDED for testing)
- `on_discovery`: Auto-backfill when channel added to folder (RECOMMENDED for production)
- `scheduled`: Backfill during off-peak hours (future)

**Media Strategies**:
- `download_available`: Try to download, mark as unavailable if 404 (RECOMMENDED)
- `skip`: Don't download media (faster, saves storage)
- `download_all`: Download and fail if unavailable (strict)

**Start Dates**:
- `2022-02-24`: Russian invasion of Ukraine (full war archive)
- `2024-01-01`: Start of current year
- Current date: No backfill, only live messages

**Warning**: Set `BACKFILL_ENABLED=false` until ready. Backfilling can take hours and trigger rate limits.

---

## Ollama (Local LLM)

Local LLM configuration for AI analysis.

### Core Configuration

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `LLM_ENABLED` | Enable LLM features | `true` | Yes |
| `LLM_PROVIDER` | LLM provider | `ollama` | Yes |
| `OLLAMA_BASE_URL` | Ollama server URL | `http://ollama:11434` | Yes |
| `OLLAMA_PORT` | Ollama port | `11434` | Yes |
| `OLLAMA_MODEL` | Ollama model name | `qwen2.5:3b` | Yes |
| `OLLAMA_TIMEOUT` | Request timeout (seconds) | `30` | No |
| `LLM_FALLBACK_TO_RULES` | Use rule-based scoring if LLM fails | `true` | No |

**Model Recommendations**:

**Production (Russian/Ukrainian OSINT)**:
- `qwen2.5:3b`: 2.4GB RAM, superior RU/UK support, 32k context (RECOMMENDED)
- `llama3.2:3b-instruct`: 2.5GB RAM, excellent multilingual, proven reliability

**Development (Laptop)**:
- `gemma2:2b`: 1.5GB RAM, 50-100% CPU, fast responses
- `granite3.0:2b`: 1.8GB RAM, 80-120% CPU, business-focused

**Specialized**:
- `phi3.5:3.8b`: 3GB RAM, best reasoning, slower, English-only
- `all-minilm`: 200MB RAM, embeddings only (semantic search)

**Performance Comparison**:
- `gemma2:2b`: ~50% CPU, ~75% quality, 30-35 tok/sec (fast development)
- `qwen2.5:3b`: ~180% CPU, ~87% quality, 18-25 tok/sec (production RU/UK)
- `llama3.2:3b`: ~200% CPU, ~85% quality, 20-25 tok/sec (fallback)

### Development Mode Tuning

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `DEVELOPMENT_MODE` | Enable development optimizations | `true` | No |
| `OLLAMA_NUM_PARALLEL` | Parallel requests | `1` | No |
| `OLLAMA_MAX_LOADED_MODELS` | Max models in memory | `3` | No |
| `OLLAMA_CPU_THREADS` | CPU threads limit | `2` | No |
| `OLLAMA_KEEP_ALIVE` | Model unload timeout | `2m` | No |
| `OLLAMA_FLASH_ATTENTION` | Enable flash attention | `false` | No |
| `OLLAMA_NUM_PREDICT` | Max tokens to generate | `250` | No |
| `AI_TAGGING_MODEL` | AI tagging model | `qwen2.5:3b` | No |
| `DEVELOPMENT_PREFERRED_MODEL` | Development model preference | `qwen2.5:3b` | No |

**Development Mode** (`DEVELOPMENT_MODE=true`):
- Prevents CPU spikes on laptops
- Limits parallel processing
- Faster model unloading

**Production Mode** (`DEVELOPMENT_MODE=false`):
- No resource limits
- Full parallelization
- Better throughput

---

## Folder-Based Channel Management

Telegram folder patterns for automatic channel discovery.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `FOLDER_SYNC_INTERVAL` | Folder sync interval (seconds) | `300` | No |
| `MONITORING_OSINT_THRESHOLD` | OSINT threshold for selective archive | `70` | No |

**Folder Patterns** (hardcoded in `channel_discovery.py`):
- `Archive*`: `archive_all` rule (store all non-spam)
- `Monitor*`: `selective_archive` rule (only high importance)
- `Discover*`: `discovery` rule (auto-joined, 14-day probation)

**Note**: To add new patterns, edit `FOLDER_RULES` in `services/listener/src/channel_discovery.py`

---

## Ntfy (Notifications)

Self-hosted notification system for real-time events.

### Core Configuration

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `NTFY_URL` | Ntfy server URL | `http://ntfy:80` | Yes |
| `NTFY_PORT` | Ntfy port | `8090` | Yes |
| `NTFY_BASE_URL` | Ntfy base URL | `http://localhost:8090` | Yes |
| `NTFY_TOPIC_PREFIX` | Topic prefix | `osint-platform` | No |
| `NTFY_CACHE_DURATION` | Message retention | `168h` | No |
| `NTFY_ENABLE_LOGIN` | Enable login | `false` | No |
| `NTFY_ENABLE_SIGNUP` | Enable signup | `false` | No |
| `NTFY_RATE_LIMIT_EVENTS` | Max events per minute per topic | `100` | No |

**Web UI**: http://localhost:8090

### Aggregator

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `AGGREGATOR_ENABLED` | Enable notification aggregator | `true` | No |
| `AGGREGATOR_BATCH_INTERVAL` | Batch interval (seconds) | `300` | No |
| `AGGREGATOR_MAX_BATCH_SIZE` | Max notifications per batch | `50` | No |

### Feature Flags

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `NOTIFY_MESSAGE_ARCHIVED` | Notify on message archived | `true` | No |
| `NOTIFY_SPAM_DETECTED` | Notify on spam detected | `true` | No |
| `NOTIFY_OSINT_HIGH` | Notify on high OSINT scores (≥70) | `true` | No |
| `NOTIFY_OSINT_CRITICAL` | Notify on critical OSINT scores (≥90) | `true` | No |
| `NOTIFY_LLM_ACTIVITY` | Notify on LLM inference | `true` | No |
| `NOTIFY_CHANNEL_DISCOVERY` | Notify on channel discovery | `true` | No |
| `NOTIFY_SYSTEM_HEALTH` | Notify on container health changes | `true` | No |
| `NOTIFY_API_ERRORS` | Notify on API errors | `true` | No |

**Tip**: Disable flags to reduce notification noise.

---

## Translation

Translation service configuration.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `TRANSLATION_ENABLED` | Enable translation | `true` | Yes |
| `TRANSLATION_PROVIDER` | Translation provider | `deepl` | Yes |
| `TRANSLATION_TARGET_LANG` | Target language (ISO 639-1) | `en` | Yes |
| `TRANSLATION_FROM_LANGUAGES` | Source languages (comma-separated) | `ru,uk` | No |
| `DEEPL_API_KEY` | DeepL API key | `YOUR_DEEPL_API_KEY_HERE` | Yes |
| `DEEPL_API_URL` | DeepL API URL | `https://api-free.deepl.com/v2` | Yes |
| `GOOGLE_TRANSLATE_API_KEY` | Google Translate API key (fallback) | `YOUR_GOOGLE_API_KEY_HERE_OPTIONAL` | No |
| `TRANSLATION_DAILY_BUDGET_USD` | Daily budget (USD) | `999999.0` | No |

**Providers**:
- `deepl`: DeepL API (primary, free Pro account available)
- `google`: Google Translate (fallback)
- `none`: Disable translation

**DeepL API**:
- Free tier available: https://www.deepl.com/pro-api
- Free API: `https://api-free.deepl.com/v2`
- Paid API: `https://api.deepl.com/v2`

**Budget**: Set to `999999.0` for effectively unlimited (free DeepL)

---

## Worker Configuration

Processor worker configuration.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `WORKER_COUNT` | Number of processor workers | `4` | No |
| `WORKER_BATCH_SIZE` | Batch size for processing | `50` | No |
| `WORKER_TIMEOUT` | Worker timeout (seconds) | `300` | No |
| `SPAM_FILTER_ENABLED` | Enable spam filtering | `true` | No |
| `ENTITY_EXTRACTION_ENABLED` | Enable entity extraction | `true` | No |
| `TRANSLATION_ENABLED_IN_WORKER` | Enable translation in worker | `true` | No |
| `LLM_SCORING_ENABLED` | Enable LLM scoring | `true` | No |

**Performance Tuning**:
- Development: `WORKER_COUNT=1-2`
- Production: `WORKER_COUNT=4-8` (match CPU cores)

---

## OpenSanctions Entity Intelligence

Entity verification via OpenSanctions (sanctions, PEPs, criminals).

### Core Configuration

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `OPENSANCTIONS_ENABLED` | Enable OpenSanctions enrichment | `false` | Yes |
| `OPENSANCTIONS_BACKEND` | Backend mode | `api` | Yes |
| `OPENSANCTIONS_API_KEY` | OpenSanctions API key | - | Conditional |
| `OPENSANCTIONS_BASE_URL` | API endpoint (auto-configured) | - | No |
| `YENTE_UPDATE_TOKEN` | Yente update token | `unsafe-default-token-change-in-production` | No |

**Backend Modes**:
- `api`: Public API (simpler, rate-limited, requires API key)
- `yente`: Self-hosted (no limits, faster, requires ElasticSearch)

**API Key**:
- API mode: REQUIRED - Get from https://www.opensanctions.org/api/
- Yente mode: OPTIONAL - Enables bulk data access

**Start Yente**: `docker-compose --profile opensanctions up -d`

### Dataset & Matching

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `OPENSANCTIONS_DATASET` | Dataset to match against | `default` | No |
| `OPENSANCTIONS_MATCH_THRESHOLD` | Minimum confidence threshold (0-100) | `85` | No |

**Datasets**:
- `default`: All datasets (sanctions, PEPs, crime, corporate)
- `sanctions`: Only sanctions lists (OFAC, UN, EU)
- `peps`: Only Politically Exposed Persons

**Threshold**: 85 recommended for high-quality matches

### Processing

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `OPENSANCTIONS_BATCH_SIZE` | Messages per batch | `10` | No |
| `OPENSANCTIONS_POLL_INTERVAL` | Poll interval (seconds) | `60` | No |

### Embeddings

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `OPENSANCTIONS_GENERATE_EMBEDDINGS` | Generate entity embeddings | `true` | No |
| `OPENSANCTIONS_EMBEDDING_MODEL` | Embedding model | `all-MiniLM-L6-v2` | No |

**Embedding Models**:
- `all-MiniLM-L6-v2`: 384 dims, fast, good quality (RECOMMENDED)
- `all-mpnet-base-v2`: 768 dims, slower, better quality

**Purpose**: Enables "find similar entities" and entity clustering

---

## API Configuration

API server settings.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `API_HOST` | API host | `0.0.0.0` | Yes |
| `API_PORT` | API port | `8000` | Yes |
| `API_WORKERS` | API workers | `4` | No |
| `API_RELOAD` | Auto-reload (development) | `true` | No |
| `API_RATE_LIMIT_PER_MINUTE` | Rate limit per minute | `60` | No |
| `API_RATE_LIMIT_BURST` | Rate limit burst | `100` | No |
| `FEED_AUTH_REQUIRED` | Require auth for RSS feeds | `false` | No |
| `API_CORS_ORIGINS` | CORS origins (comma-separated) | `http://localhost:3000,http://localhost:5173` | Yes |

**Production**:
- Set `API_RELOAD=false`
- Increase `API_WORKERS=4-8`
- Set `API_CORS_ORIGINS` to your domains

**CORS Examples**:
- Development: `http://localhost:3000,http://localhost:5173`
- Production: `https://osintukraine.com,https://www.osintukraine.com`

### Service Ports

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `LISTENER_PORT` | Listener port | `8001` | No |
| `LISTENER_METRICS_PORT` | Listener metrics port | `9091` | No |
| `PROCESSOR_PORT` | Processor port | `8002` | No |
| `PROCESSOR_METRICS_PORT` | Processor metrics port | `9092` | No |
| `API_METRICS_PORT` | API metrics port | `9093` | No |

---

## Authentication

Multi-provider authentication system.

### Core Configuration

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `AUTH_PROVIDER` | Authentication provider | `none` | Yes |
| `AUTH_REQUIRED` | Reject unauthenticated requests | `false` | No |

**Providers**:
- `none`: No authentication (development, private VPN)
- `jwt`: Simple JWT authentication (small teams)
- `cloudron`: Cloudron OAuth (Cloudron deployments)
- `ory`: Ory Kratos/Oathkeeper (production, zero-trust)

### JWT Authentication

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `JWT_SECRET_KEY` | JWT secret key | `CHANGE_ME_RANDOM_256_BIT_KEY_HERE` | Conditional |
| `JWT_ALGORITHM` | JWT algorithm | `HS256` | No |
| `JWT_EXPIRATION_MINUTES` | Access token expiration | `60` | No |
| `JWT_REFRESH_EXPIRATION_DAYS` | Refresh token expiration | `30` | No |

**Generate Secret**: `openssl rand -hex 32`

### Ory Authentication

#### Service URLs

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `ORY_KRATOS_PUBLIC_URL` | Kratos public URL | `http://ory-kratos:4433` | No |
| `ORY_KRATOS_ADMIN_URL` | Kratos admin URL | `http://ory-kratos:4434` | No |
| `ORY_OATHKEEPER_URL` | Oathkeeper URL | `http://ory-oathkeeper:4455` | No |

#### Kratos Secrets

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `KRATOS_SECRET_COOKIE` | Session cookie secret (any length) | `CHANGE_ME_GENERATE_WITH_OPENSSL_RAND_BASE64_32` | Yes |
| `KRATOS_SECRET_CIPHER` | Data encryption key (MUST be 32 chars) | `CHANGE_ME_EXACTLY_32_CHARACTERS!` | Yes |

**Security Critical**: Generate strong random secrets!

**Generate Secrets**:
```bash
# Cookie secret (any length)
openssl rand -base64 32

# Cipher secret (EXACTLY 32 characters)
openssl rand -base64 24
```

**One-liner**:
```bash
echo "KRATOS_SECRET_COOKIE=$(openssl rand -base64 32)" && echo "KRATOS_SECRET_CIPHER=$(openssl rand -base64 24)"
```

#### Social Login (Optional)

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `GOOGLE_CLIENT_ID` | Google OAuth client ID | - | No |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret | - | No |
| `GITHUB_CLIENT_ID` | GitHub OAuth client ID | - | No |
| `GITHUB_CLIENT_SECRET` | GitHub OAuth client secret | - | No |

**Setup**:
- Google: https://console.cloud.google.com/apis/credentials
- GitHub: https://github.com/settings/developers

#### Email Configuration

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `SMTP_CONNECTION_URI` | SMTP connection URI | `smtp://mailslurper:1025/?skip_ssl_verify=true` | Yes |
| `SMTP_FROM_ADDRESS` | From email address | `noreply@localhost` | Yes |
| `SMTP_FROM_NAME` | From name | `OSINT Platform` | Yes |

**Development**: Uses MailSlurper (included in dev profile)

**Production Examples**:

SendGrid:
```
SMTP_CONNECTION_URI=smtps://apikey:YOUR_SENDGRID_API_KEY@smtp.sendgrid.net:465
SMTP_FROM_ADDRESS=noreply@osintukraine.com
SMTP_FROM_NAME=OSINT Ukraine Platform
```

AWS SES:
```
SMTP_CONNECTION_URI=smtps://YOUR_SMTP_USERNAME:YOUR_SMTP_PASSWORD@email-smtp.us-east-1.amazonaws.com:465
SMTP_FROM_ADDRESS=noreply@osintukraine.com
SMTP_FROM_NAME=OSINT Ukraine Platform
```

Mailgun:
```
SMTP_CONNECTION_URI=smtps://YOUR_SMTP_USERNAME:YOUR_SMTP_PASSWORD@smtp.mailgun.org:465
SMTP_FROM_ADDRESS=noreply@osintukraine.com
SMTP_FROM_NAME=OSINT Ukraine Platform
```

---

## Frontend Configuration

Next.js frontend configuration.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `FRONTEND_PORT` | Frontend dev server port | `3000` | Yes |
| `NEXT_PUBLIC_API_URL` | API URL (client-side) | `http://localhost:8000` | Yes |
| `NEXT_PUBLIC_RSS_URL` | RSS URL (client-side) | `http://localhost:8000/rss` | Yes |
| `NEXT_PUBLIC_BASE_URL` | Base URL for meta tags | `http://localhost:3000` | Yes |
| `NODE_ENV` | Node environment | `development` | Yes |

**Important**: Use `NEXT_PUBLIC_API_URL` for all client-side API calls. Never use relative paths or Next.js rewrites in Docker.

**Production**:
- `NEXT_PUBLIC_API_URL=https://api.osintukraine.com`
- `NEXT_PUBLIC_BASE_URL=https://osintukraine.com`
- `NODE_ENV=production`

**Reference**: See `services/frontend-nextjs/app/admin/page.tsx` for the established pattern.

---

## Monitoring

Prometheus and Grafana configuration.

### Prometheus

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `PROMETHEUS_PORT` | Prometheus port | `9090` | Yes |

### Grafana

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `GRAFANA_PORT` | Grafana port | `3001` | Yes |
| `GRAFANA_ADMIN_USER` | Grafana admin user | `admin` | Yes |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin password | `CHANGE_ME_STRONG_PASSWORD_HERE` | Yes |
| `GRAFANA_ROOT_URL` | Grafana root URL | `http://localhost:3001` | Yes |

**Security**: Change `GRAFANA_ADMIN_PASSWORD` to a strong password

### Grafana SMTP (Optional)

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `GRAFANA_SMTP_ENABLED` | Enable SMTP alerts | `false` | No |
| `GRAFANA_SMTP_HOST` | SMTP host | `smtp.example.com:587` | No |
| `GRAFANA_SMTP_USER` | SMTP user | `alerts@example.com` | No |
| `GRAFANA_SMTP_PASSWORD` | SMTP password | `smtp_password_here` | No |
| `GRAFANA_SMTP_FROM` | From address | `alerts@example.com` | No |

---

## Deployment & Domain

Deployment mode and domain configuration.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `DEPLOYMENT_MODE` | Deployment mode | `development` | Yes |
| `DOMAIN` | Production domain | `osint.example.com` | No |
| `FRONTEND_URL` | Frontend URL | `http://localhost:3000` | Yes |
| `API_URL` | API URL (internal) | `http://localhost:8000` | Yes |
| `PROXY_URL` | Oathkeeper proxy URL | `http://localhost:8000` | Yes |

**Deployment Modes**:
- `development`: Direct access to services (no Oathkeeper proxy)
- `production`: All traffic routed through Oathkeeper + Caddy with HTTPS

**Production Example**:
```
DEPLOYMENT_MODE=production
DOMAIN=v2.osintukraine.com
FRONTEND_URL=https://v2.osintukraine.com
API_URL=https://v2.osintukraine.com/api
PROXY_URL=https://v2.osintukraine.com
```

---

## Production-Only Settings

Settings only used in `docker-compose.production.yml`.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `VERSION` | Docker image version tag | `latest` | No |

**Note**: These are placeholders for future production configuration.

---

## NocoDB

Airtable-like UI over PostgreSQL.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `NOCODB_PORT` | NocoDB port | `8080` | Yes |
| `NOCODB_PUBLIC_URL` | NocoDB public URL | `http://localhost:8080` | Yes |
| `NOCODB_JWT_SECRET` | NocoDB JWT secret | `change-this-secret-in-production-min-32-chars` | Yes |
| `NOCODB_ADMIN_EMAIL` | Initial admin email | `admin@osint.local` | Yes |
| `NOCODB_ADMIN_PASSWORD` | Initial admin password | `change-this-password` | Yes |

**Security**: Change `NOCODB_JWT_SECRET` and `NOCODB_ADMIN_PASSWORD` in production

### NocoDB SMTP (Optional)

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `NOCODB_SMTP_FROM` | From address | - | No |
| `NOCODB_SMTP_HOST` | SMTP host | - | No |
| `NOCODB_SMTP_PORT` | SMTP port | - | No |
| `NOCODB_SMTP_USERNAME` | SMTP username | - | No |
| `NOCODB_SMTP_PASSWORD` | SMTP password | - | No |
| `NOCODB_SMTP_SECURE` | Use TLS | `true` | No |

**Usage**:
```bash
docker-compose -f docker-compose.phase1.yml -f docker-compose.nocodb.yml up -d
```

**Features**:
- Public data portal (anonymous access)
- Admin backend (channel management, rules)
- Project management (development tasks)

**See**: `docs/NOCODB_CLOUDRON_STRATEGY.md`

---

## RSS Intelligence Layer

Optional RSS ingestion for cross-validation and fact-checking.

### Core Configuration

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `RSS_INGESTION_ENABLED` | Master toggle for RSS ingestion | `true` | Yes |
| `RSS_INGESTION_INTERVAL_MINUTES` | Feed poll interval (minutes) | `5` | No |

### Feed Limits

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `RSS_MAX_FEEDS` | Maximum feeds allowed | `50` | No |
| `RSS_MAX_ARTICLES_PER_POLL` | Max articles per poll | `50` | No |
| `RSS_RETENTION_DAYS` | Article retention (days) | `90` | No |

### Cross-Correlation

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `RSS_CORRELATION_ENABLED` | Enable RSS-Telegram correlation | `true` | No |
| `RSS_CORRELATION_SIMILARITY_THRESHOLD` | Similarity threshold (0.0-1.0) | `0.40` | No |
| `RSS_CORRELATION_TIME_WINDOW_HOURS` | Time window (±hours) | `6` | No |
| `RSS_CORRELATION_MAX_PER_MESSAGE` | Max correlations per message | `10` | No |

**Similarity Thresholds**:
- `0.55+`: Very similar (same event, high confidence)
- `0.45-0.55`: Similar (related topic, medium confidence)
- `0.40-0.45`: Somewhat similar (loosely related, low confidence)
- `0.40`: RECOMMENDED for cross-lingual (RU/UK Telegram → EN RSS)
- `0.65+`: RECOMMENDED for same language

**Time Window**: Telegram often arrives 1-6 hours BEFORE RSS news

### Fact-Checking

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `RSS_FACT_CHECK_ENABLED` | Enable fact-checking | `true` | No |
| `RSS_FACT_CHECK_MIN_OSINT_SCORE` | Minimum OSINT score for fact-check | `75` | No |
| `RSS_FACT_CHECK_MIN_CORRELATION` | Minimum correlation for display | `80` | No |
| `RSS_SHOW_ALTERNATIVE_VIEWPOINTS` | Show different perspectives | `true` | No |

**Purpose**:
- Cross-validate high-OSINT Telegram messages with external news
- Show "verified by external sources" or "contradictory reports" flags
- Display alternative viewpoints (Ukraine vs Russia sources)

### Performance

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `RSS_WORKER_THREADS` | Parallel fetching threads | `2` | No |
| `RSS_EMBEDDING_BATCH_SIZE` | Embedding batch size | `10` | No |

### Outbound RSS

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `RSS_OUTBOUND_ENABLED` | Enable `/rss/*` API endpoints | `true` | No |
| `RSS_CACHE_MINUTES` | Cache duration (minutes) | `10` | No |

**Note**: Outbound RSS is separate from RSS ingestion. This controls RSS feeds you PROVIDE.

**Pre-configured Feeds**: 12 feeds seeded on first launch (Ukraine, Russia, Neutral sources)

**See**: `docs/PHASE2D_RSS_INTELLIGENCE_LAYER.md`

---

## Entity Ingestion

CSV-based entity knowledge graph ingestion.

### Core Configuration

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `ENTITY_INGESTION_ENABLED` | Enable entity ingestion | `true` | Yes |
| `ENTITY_SCAN_INTERVAL` | CSV scan interval (seconds) | `300` | No |
| `ENTITY_BATCH_SIZE` | Entities per batch | `100` | No |

**Performance**:
- `ENTITY_BATCH_SIZE=100`: Good balance (default)
- `ENTITY_BATCH_SIZE=500-1000`: Better performance
- `ENTITY_BATCH_SIZE=50`: Lower memory usage

### Entity Embeddings

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `ENTITY_GENERATE_EMBEDDINGS` | Generate embeddings for semantic search | `true` | No |
| `ENTITY_EMBEDDING_MODEL` | Embedding model | `all-MiniLM-L6-v2` | No |
| `ENTITY_MATCHING_THRESHOLD` | Similarity threshold | `0.65` | No |

**Embedding Models**:
- `all-MiniLM-L6-v2`: 384 dims, fast, 80MB model (RECOMMENDED)
- `all-mpnet-base-v2`: 768 dims, slower, better quality, 420MB
- `paraphrase-multilingual-MiniLM-L12-v2`: 384 dims, multilingual

**Matching Thresholds**:
- `0.65`: More matches, potential false positives
- `0.75`: Balanced (RECOMMENDED)
- `0.85`: Stricter, fewer matches

**Usage**:
```bash
# Start entity-ingestion service
docker-compose --profile entity-ingestion up -d

# View logs
docker-compose logs -f entity-ingestion

# Check status
cat data/entities/processed/status.json
```

**See**: Entity ingestion service documentation

---

## Development Helpers

Development-only convenience settings.

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `SQL_ECHO` | Enable SQL query logging (verbose) | - | No |
| `SHOW_ERROR_DETAILS` | Enable detailed error traces | - | No |

**Note**: Commented out by default. Uncomment in `.env` to enable.

---

## Backup & Maintenance

Settings for backup scripts (not used by Docker Compose).

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `BACKUP_RETENTION_DAYS` | Backup retention (days) | - | No |
| `BACKUP_PATH` | Backup destination path | - | No |
| `BACKUP_ENCRYPTION_KEY` | Backup encryption key | - | No |

**Note**: Commented out by default. Configure when using backup scripts.

---

## Security Checklist

Before deploying:

- [ ] Changed all `CHANGE_ME` placeholders
- [ ] Used strong random passwords (min 32 characters)
- [ ] Generated `JWT_SECRET_KEY` (if using JWT): `openssl rand -hex 32`
- [ ] Generated `KRATOS_SECRET_COOKIE`: `openssl rand -base64 32`
- [ ] Generated `KRATOS_SECRET_CIPHER` (MUST be 32 chars): `openssl rand -base64 24`
- [ ] Configured production SMTP (not MailSlurper)
- [ ] Verified `.env` is in `.gitignore`
- [ ] Never shared `.env` file or credentials

---

## Production Checklist

Before going live:

- [ ] Set `ENVIRONMENT=production`
- [ ] Set `DEBUG=false`
- [ ] Set `API_RELOAD=false`
- [ ] Set `MINIO_SECURE=true` (with HTTPS)
- [ ] Configure `MINIO_PUBLIC_URL` to your domain
- [ ] Configure `API_CORS_ORIGINS` to your domain(s)
- [ ] Set strong unique passwords for all services
- [ ] Enable HTTPS (Caddy handles automatically)
- [ ] Configure Grafana SMTP for alerts (optional)

---

## Common Issues

### Connection Refused
- Check service names match `docker-compose.yml`

### Authentication Failed
- Verify passwords in `.env` match

### Permission Denied
- Check volume permissions

### Rate Limit Exceeded
- Increase `TELEGRAM_RATE_LIMIT_PER_CHANNEL`

### Out of Memory
- Increase Docker memory limits
- Reduce `WORKER_COUNT`
- Use lighter LLM model (`gemma2:2b`)

---

## Performance Tuning

### Development (Laptop)

| Variable | Recommended Value |
|----------|-------------------|
| `WORKER_COUNT` | `1-2` |
| `API_WORKERS` | `2` |
| `POSTGRES_POOL_SIZE` | `5-10` |
| `OLLAMA_MODEL` | `gemma2:2b` |
| `DEVELOPMENT_MODE` | `true` |

### Production (VPS)

| Variable | Recommended Value |
|----------|-------------------|
| `WORKER_COUNT` | `4-8` (match CPU cores) |
| `API_WORKERS` | `4-8` (match CPU cores) |
| `POSTGRES_POOL_SIZE` | `20-50` |
| `OLLAMA_MODEL` | `qwen2.5:3b` |
| `DEVELOPMENT_MODE` | `false` |

---

## Support

- **Documentation**: `docs/`
- **Issues**: https://github.com/osintukraine/osint-intelligence-platform/issues
- **Stack Manager**: `./scripts/stack-manager.sh`

---

**Last Updated**: 2025-12-09
**Platform Version**: 1.0
**Source**: `.env.example` from osint-intelligence-platform
