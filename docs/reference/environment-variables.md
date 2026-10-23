# Environment Variables Reference

[`.env.example`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/.env.example) in the platform repo is the **canonical reference**: every variable, its default and a comment explaining it. This page does not repeat it. It explains how the file is organised, which values you must set, and where the October 2026 LLM variables live.

**Setup**: copy `.env.example` to `.env` and edit. For how `.env` relates to admin settings, see [Configuration](../operator-guide/configuration.md#three-tier-configuration).

## Required secrets

`docker-compose.yml` refuses to start unless these are set in `.env` (the error names the missing variable; there are no built-in fallbacks):

`POSTGRES_PASSWORD`, `REDIS_PASSWORD`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`, `MINIO_ROOT_PASSWORD`, `JWT_SECRET_KEY`, `NOCODB_JWT_SECRET`, `NOCODB_ADMIN_PASSWORD`, `GRAFANA_ADMIN_PASSWORD`, `GRAFANA_SECRET_KEY`, `YENTE_UPDATE_TOKEN`.

You also need Telegram API credentials and, for translation, `DEEPL_API_KEY`. Run `./scripts/preflight-check.sh` to catch missing or `CHANGE_ME` values and `.env` variables that `.env.example` does not document.

## Groups in `.env.example`

| Section | What it covers |
|---|---|
| Platform identity, environment | Name, `ENVIRONMENT`, `DEBUG`, logging |
| `COMPOSE_PROFILES` | Which service groups start. See [Docker Services](docker-services.md#profiles) |
| Hardware-adaptive configuration | Tier and LLM sizing written by `./scripts/bootstrap-hardware.sh` |
| PostgreSQL, Redis | Connections, credentials, memory |
| Telegram, comment polling, viral posts, backfill, gap detection | Listener behaviour; per-account variables for multi-account setups |
| Embedding-based fast classification | `EMBEDDING_SPAM_*`, `EMBEDDING_OFF_TOPIC_THRESHOLD`, `BACKFILL_FAST_MODE` |
| LLM configuration | Provider, gateway and Ollama variables (below) |
| Folder-based channel management | `FOLDER_SYNC_INTERVAL` |
| ntfy (notifications) | Notifier and ntfy |
| Translation | DeepL |
| Worker configuration | Processor and enrichment replicas, batch sizes |
| OpenSanctions, shadow fleet | Optional entity services |
| API, Ory authentication, CORS, WebSocket, rate limits, cache TTLs, feed authentication | API and security |
| Frontend, monitoring, deployment mode | Public URLs, Prometheus and Grafana, domain |
| Backup, NocoDB, timeouts, disaster recovery | Operations and optional tools |

## LLM variables

Details and examples: [LLM Gateway and Providers](../operator-guide/external-llm.md).

| Variable | Purpose |
|---|---|
| `LLM_ENABLED`, `LLM_PROVIDER` | `openai` (gateway), `ollama` or `none` |
| `LLM_GATEWAY_URL`, `LLM_GATEWAY_HOST`, `LLM_GATEWAY_IP` | Gateway address and optional hosts pin. Keep real values in `.env` only |
| `LLM_GATEWAY_API_KEY`, `LLM_GATEWAY_AUDIT_API_KEY`, `LLM_GATEWAY_BATCH_API_KEY` | One key per lane (audit and batch fall back to the first) |
| `LLM_GATEWAY_MODEL`, `LLM_GATEWAY_BATCH_MODEL` | Realtime and batch aliases |
| `LLM_GATEWAY_TIMEOUT` | Seconds per request |
| `FACT_CHECK_MODEL` and other `FACT_CHECK_*` | Audit lane alias, sampling, thresholds and pacing. Read by `docker-compose.yml` with defaults |
| `RSS_VALIDATION_MODEL` and other `RSS_VALIDATION_*` | Batch lane alias and pacing for RSS validation. Read by `docker-compose.yml` with defaults |
| `OLLAMA_BASE_URL`, `OLLAMA_MODEL`, `OLLAMA_API_KEY`, `OLLAMA_EXTERNAL_MODE` | Ollama option |
| `OLLAMA_BATCH_URL`, `FACT_CHECK_OLLAMA_HOST` | Endpoints for the CPU-era batch workers |
| `LLM_CLASSIFIER_MODE`, `REALTIME_USE_EMBEDDING` | Classifier mode and the embedding-only path (also admin settings) |

## Backlog, media and models

| Variable | Purpose |
|---|---|
| `BACKLOG_RECLASSIFY_ENABLED`, `BACKLOG_CONCURRENCY` | Fallbacks for the admin settings of the [backlog worker](../operator-guide/backlog-reclassification.md) |
| `MEDIA_WORKER_REPLICAS`, `MEDIA_DOWNLOAD_TIMEOUT_S` | [Media lane](../operator-guide/media-lanes.md) worker; the lane itself is switched with the `media_lane_enabled` admin setting |
| `EMBEDDING_SPAM_MODEL` | Path under `/models`, see [Embedding Models](../operator-guide/model-provisioning.md) |
| `EMBEDDING_SPAM_THRESHOLD`, `EMBEDDING_OFF_TOPIC_THRESHOLD` | Spam and off-topic similarity (0.75 and 0.70) |

## Variables that admin settings override

Some variables are only the fallback for an admin setting stored in the database (for example `EMBEDDING_SPAM_THRESHOLD` and `LLM_ENABLED`). The authoritative list, with each setting's `.env` variable, default and whether it applies live, is [`runtime_keys.py`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/shared/python/config/runtime_keys.py). The admin page **Admin, Config, Platform Settings** shows the effective value per service.

## Security notes

- Never commit `.env`; protect the `telegram_sessions` directory the same way.
- Use long random values for every secret (`openssl rand -base64 32`). `KRATOS_SECRET_CIPHER` must be exactly 32 characters.
- Set `ENVIRONMENT=production`, restrict `API_CORS_ORIGINS` to your domain, and keep `DEBUG=false`.
- Keep the LLM gateway URL, hostname, address and keys out of version control and logs.

## See Also

- [Configuration](../operator-guide/configuration.md)
- [Docker Services](docker-services.md)
- [Installation Guide](../operator-guide/installation.md)
- [Security Hardening](../security-guide/hardening.md)
