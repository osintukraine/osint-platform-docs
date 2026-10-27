# Configuration

How to configure the OSINT Intelligence Platform: where each kind of setting lives and how to change it.

## Overview

Configuration comes from three tiers, from most static to most dynamic. Knowing which tier a setting lives in tells you how to change it and whether a restart is needed. Two reference pages are canonical, so this guide does not repeat them:

- [`.env.example`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/.env.example) is the **canonical list of every environment variable**, with defaults and comments. This guide does not repeat it; see [Environment Variables](../reference/environment-variables.md) for an overview.
- [Docker Services](../reference/docker-services.md) is the canonical list of services, profiles and ports.

## Three-tier configuration

| Tier | Where | Change it with | Applies |
|---|---|---|---|
| 1. Infrastructure and secrets | `.env` | Edit `.env`, then recreate the affected services | After a restart. Shown read-only in **Admin, Config, Environment** (secrets masked) |
| 2. Runtime settings | `platform_config` table | **Admin, Config, Platform Settings** | Live, within about 30 seconds; no restart |
| 3. UI and per-user state | Database only | The application itself | Immediately |

For a tier-2 setting the precedence is: **admin override > `.env` variable > built-in default**. **Reset** in the admin page removes the override, so the service falls back to `.env`. Each setting shows whether it applies live or needs a service restart, which `.env` variable it overrides, and the value each running service instance actually uses (with a warning when a service cannot apply the configured value, for example translation on without `DEEPL_API_KEY`).

Only settings registered in [`runtime_keys.py`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/shared/python/config/runtime_keys.py) are wired to a service; a stored row that is not in that registry is shown as "not applied". Examples of tier-2 settings: `llm_classification_enabled`, `realtime_use_embedding`, `embedding_spam_threshold`, `llm_prefilter_spam_similarity`, `media_lane_enabled`, `backlog_reclassify_enabled`, `backlog_spam_first`, `backlog_concurrency`, and the RSS ingestion, correlation and validation settings.

Folder-based channel management (`Archive-*`, `Monitor-*`, `Discover-*` Telegram folders) is fixed in code; only the sync interval (`FOLDER_SYNC_INTERVAL`, seconds, minimum 30) is configurable.

## Quick Start

```bash
# Copy the template and edit it
cp .env.example .env
nano .env

# Fetch the pinned embedding models (host-side, one time and after upgrades)
./scripts/fetch-models.sh

# Validate the result
./scripts/preflight-check.sh
docker compose config --quiet
```

At minimum set the secrets listed in [Environment Variables](../reference/environment-variables.md#required-secrets), your Telegram API credentials and `DEEPL_API_KEY`. Choose an LLM provider (`LLM_PROVIDER`): see [LLM Gateway and Providers](external-llm.md). For production authentication (Ory Kratos), generate `KRATOS_SECRET_COOKIE` (`openssl rand -base64 32`) and `KRATOS_SECRET_CIPHER` (exactly 32 characters).

## Docker Compose Profiles

Profiles choose which service groups start. Set them once in `.env`:

```bash
COMPOSE_PROFILES=enrichment-lite,monitoring,auth,llm-gateway
```

or pass them per command (`docker compose --profile monitoring up -d`). The full profile table (`enrichment-lite`, `enrichment-standard`, `enrichment-full`, `llm-gateway`, `monitoring`, `auth`, `opensanctions`, `notifications`, `tools`, `multi-account`, `dr`, `backfill`, `cloudflare`) is in [Docker Services](../reference/docker-services.md#profiles).

`TELEGRAM_MODE` (`single` or `multi`, unset means `single`) selects the Telegram services. Do not edit it by hand: `./scripts/switch-telegram-mode.sh single|multi` writes it together with `SINGLE_ACCOUNT_REPLICAS` and the `multi-account` entry in `COMPOSE_PROFILES`, and `deploy.sh` refuses to run if they disagree.

## Scaling Configuration

Replica counts are `.env` variables read by `docker-compose.yml`:

```bash
PROCESSOR_WORKER_REPLICAS=2     # processor-worker
MEDIA_WORKER_REPLICAS=1         # media-worker
ENRICHMENT_WORKER_REPLICAS=1    # enrichment workers that use it
```

Apply with `docker compose up -d`. See [Scaling](scaling.md) and [Performance Tuning](performance-tuning.md) for sizing advice. The API runs a single uvicorn worker.

## Configuration Validation

### Pre-flight Checks

```bash
# Validate docker compose configuration
docker compose config

# Check for syntax errors
docker compose config --quiet

# Verify environment variables are loaded
docker compose config | grep TELEGRAM_API_ID
```

### Startup Validation

The platform validates configuration on startup:

- Database connectivity
- Redis connectivity
- MinIO bucket existence
- Telegram session validity
- Required secrets present
- Embedding model files under `/models` (see [Embedding Models](model-provisioning.md))

**View validation logs**:
```bash
docker compose logs listener processor api | grep -i "error\|warning"
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
- [ ] Set `MINIO_SECURE=true` (with HTTPS)
- [ ] Configured `MINIO_PUBLIC_URL` to production domain
- [ ] Configured `API_CORS_ORIGINS` to production domains
- [ ] Verified `.env` is in `.gitignore`
- [ ] Never shared `.env` file or credentials

## Configuration Files

- `infrastructure/postgres/postgresql.conf`: PostgreSQL tuning.
- `infrastructure/caddy/`: reverse-proxy configuration.
- Prompts and spam examples live in the database and are managed from the admin pages and SQL migrations, not in files. Prompt versions: [LLM Gateway and Providers](external-llm.md#prompt-versions).

## Troubleshooting Configuration

### Configuration Not Loading

```bash
# Check .env file exists
ls -la .env

# Check syntax (no spaces around =)
cat .env | grep "="

# Verify Docker Compose sees it
docker compose config | grep YOUR_VARIABLE
```

### Service Can't Connect to Another Service

```bash
# Check network configuration
docker compose ps
docker network ls
docker network inspect osint-intelligence-platform_backend

# Verify service names match
docker compose config | grep "container_name"
```

### Environment Variable Not Applied

```bash
# Rebuild service
docker compose build service_name

# Restart with fresh environment
docker compose up -d --force-recreate service_name

# Verify environment inside container
docker compose exec service_name env | grep VARIABLE_NAME
```

## Next Steps

- [Telegram Setup](telegram-setup.md) - Configure Telegram monitoring
- [Monitoring](monitoring.md) - Set up monitoring and alerts
- [Backup & Restore](backup-restore.md) - Implement backup strategies
- [Troubleshooting](troubleshooting.md) - Resolve common issues

## References

- [`.env.example`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/.env.example) - Complete environment variable template
- `docker-compose.yml` - Service configuration
- `infrastructure/postgres/postgresql.conf` - PostgreSQL tuning
- `CLAUDE.md` - Configuration rules and patterns
