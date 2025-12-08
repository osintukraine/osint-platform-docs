# Environment Variables

Complete reference for all configuration environment variables.

## Overview

**TODO: Content to be generated from codebase analysis**

All services are configured via environment variables defined in `.env` file.

## Core Configuration

### Database

**TODO: Document from .env.example:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `POSTGRES_USER` | PostgreSQL username | `osint_user` | Yes |
| `POSTGRES_PASSWORD` | PostgreSQL password | - | Yes |
| `POSTGRES_DB` | Database name | `osint_platform` | Yes |
| `POSTGRES_HOST` | Database host | `postgres` | Yes |
| `POSTGRES_PORT` | Database port | `5432` | Yes |
| `DATABASE_URL` | Full connection string | Auto-generated | No |

### Redis

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `REDIS_HOST` | Redis hostname | `redis` | Yes |
| `REDIS_PORT` | Redis port | `6379` | Yes |
| `REDIS_URL` | Full Redis URL | `redis://redis:6379` | Yes |

### MinIO

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `MINIO_ROOT_USER` | MinIO access key | - | Yes |
| `MINIO_ROOT_PASSWORD` | MinIO secret key | - | Yes |
| `MINIO_ENDPOINT` | MinIO endpoint | `minio:9000` | Yes |
| `MINIO_BUCKET` | Storage bucket name | `osint-media` | Yes |

## Telegram Configuration

**TODO: Document Telegram variables:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `TELEGRAM_API_ID` | Telegram API ID | - | Yes |
| `TELEGRAM_API_HASH` | Telegram API hash | - | Yes |
| `TELEGRAM_PHONE` | Phone number | - | Yes |
| `TELEGRAM_SESSION_PATH` | Session file path | `/app/sessions` | No |

## LLM Configuration

**TODO: Document Ollama variables:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `OLLAMA_BASE_URL` | Ollama API URL | `http://ollama:11434` | Yes |
| `OLLAMA_MODEL` | Default model | `qwen2.5:3b` | Yes |
| `LLM_TEMPERATURE` | Model temperature | `0.7` | No |

## API Configuration

**TODO: Document API variables:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `API_HOST` | API bind host | `0.0.0.0` | No |
| `API_PORT` | API port | `8000` | No |
| `API_SECRET_KEY` | JWT secret | - | Yes |
| `API_CORS_ORIGINS` | CORS origins | `*` | No |

## Frontend Configuration

**TODO: Document Next.js variables:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `NEXT_PUBLIC_API_URL` | Public API URL | `http://localhost:8000` | Yes |
| `NEXTAUTH_SECRET` | NextAuth secret | - | Yes |
| `NEXTAUTH_URL` | NextAuth callback URL | - | Yes |

## Worker Configuration

**TODO: Document worker variables:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `PROCESSOR_WORKERS` | Processor worker count | `2` | No |
| `ENRICHMENT_WORKERS` | Enrichment worker count | `1` | No |
| `WORKER_CONCURRENCY` | Tasks per worker | `4` | No |

## Enrichment Task Configuration

**TODO: Document enrichment task toggles:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `ENABLE_EMBEDDING_TASK` | Enable embeddings | `true` | No |
| `ENABLE_AI_TAGGING_TASK` | Enable AI tagging | `true` | No |
| `ENABLE_SOCIAL_GRAPH_TASK` | Enable social graph | `true` | No |
| `ENABLE_WIKIDATA_TASK` | Enable Wikidata | `true` | No |

## Monitoring Configuration

**TODO: Document monitoring variables:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `GRAFANA_ADMIN_PASSWORD` | Grafana password | `admin` | Yes |
| `PROMETHEUS_RETENTION` | Metrics retention | `15d` | No |

## Notification Configuration

**TODO: Document notification variables:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `DISCORD_WEBHOOK_URL` | Discord webhook | - | No |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token | - | No |
| `SMTP_HOST` | Email SMTP host | - | No |
| `SMTP_PORT` | Email SMTP port | `587` | No |

## Security Configuration

**TODO: Document security variables:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `CROWDSEC_API_KEY` | CrowdSec API key | - | No |
| `ENABLE_CROWDSEC` | Enable CrowdSec | `false` | No |
| `SESSION_TIMEOUT` | Session timeout (seconds) | `3600` | No |

## Logging Configuration

**TODO: Document logging variables:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `LOG_LEVEL` | Logging level | `INFO` | No |
| `LOG_FORMAT` | Log format | `json` | No |
| `LOG_FILE` | Log file path | - | No |

## Feature Flags

**TODO: Document feature flags:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `ENABLE_SPAM_FILTER` | Enable spam filtering | `true` | No |
| `ENABLE_MEDIA_ARCHIVAL` | Archive media | `true` | No |
| `ENABLE_DEDUPLICATION` | Deduplicate content | `true` | No |

## Development Configuration

**TODO: Document dev-specific variables:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `DEBUG` | Debug mode | `false` | No |
| `RELOAD` | Hot reload | `false` | No |
| `TESTING` | Test mode | `false` | No |

## Environment-Specific Configs

### Development (.env.development)

**TODO: Document development defaults**

### Production (.env.production)

**TODO: Document production defaults**

## Configuration Validation

**TODO: Document validation:**

```bash
# Validate configuration
docker-compose config

# Check for missing required variables
./scripts/validate-env.sh
```

## Security Best Practices

**TODO: Document security practices:**

- Never commit `.env` files to git
- Use strong random passwords
- Rotate secrets regularly
- Use secret management in production

---

!!! warning "Security"
    Always use strong, randomly generated passwords for production. Never use default values!

!!! note "Documentation Status"
    This page is a placeholder. Content will be generated from .env.example and service configuration code.
