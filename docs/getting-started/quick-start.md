# Quick Start

Get the OSINT Intelligence Platform running locally in under 10 minutes.

!!! info "Time Estimate"
    **~5-10 minutes** total setup time (excluding Docker image downloads)

## Prerequisites

Before you begin, ensure you have:

- **Docker 20.10+** and **Docker Compose 2.0+** installed
- **8GB+ RAM** (16GB recommended for AI models)
- **50GB+ free disk space** (plus media storage)
- **Telegram API credentials** from [my.telegram.org/apps](https://my.telegram.org/apps)

!!! warning "Telegram API Credentials Required"
    You need a Telegram account and API credentials to monitor channels. Visit [my.telegram.org/apps](https://my.telegram.org/apps) to create an application and get your `api_id` and `api_hash`.

## Step 1: Clone the Repository

```bash
git clone https://github.com/osintukraine/osint-intelligence-platform.git
cd osint-intelligence-platform
```

## Step 2: Configure Environment

Copy the example environment file and edit it with your credentials:

```bash
cp .env.example .env
nano .env  # or use your preferred editor
```

### Required Configuration

At minimum, you must set these variables in `.env`:

```bash
# Telegram API (get from https://my.telegram.org/apps)
TELEGRAM_API_ID=YOUR_API_ID_HERE
TELEGRAM_API_HASH=YOUR_API_HASH_HERE
TELEGRAM_PHONE=+1234567890  # Your phone with country code

# DeepL Translation (get free API key from https://www.deepl.com/pro-api)
DEEPL_API_KEY=YOUR_DEEPL_API_KEY_HERE

# Change all CHANGE_ME passwords
POSTGRES_PASSWORD=strong_random_password_here
REDIS_PASSWORD=another_strong_password_here
MINIO_ROOT_PASSWORD=strong_minio_admin_password
MINIO_ACCESS_KEY=minio_access_key_here
MINIO_SECRET_KEY=minio_secret_key_minimum_32_chars_here
JWT_SECRET_KEY=random_256_bit_key_here
```

!!! tip "Generate Strong Secrets"
    Use `openssl rand -hex 32` to generate strong random passwords for database and API credentials.

### Optional Configuration

For production deployments, also configure:

- `LLM_PROVIDER` (`ollama` for a local model, `openai` for a GPU gateway, `none` for no LLM) and, for Ollama, `OLLAMA_MODEL`; see [LLM Gateway and Providers](../operator-guide/external-llm.md)
- `LLM_ENABLED=true` (enables AI classification)
- `TRANSLATION_ENABLED=true` (auto-translate messages)
- `COMPOSE_PROFILES` (which optional services start, see Step 5)

See the `.env.example` file for complete configuration options.

Before starting the stack, fetch the embedding models once: `./scripts/fetch-models.sh` (see [Embedding Models](../operator-guide/model-provisioning.md)).

## Step 3: Build PyTorch Base Image

The platform uses a shared PyTorch base image for AI services. Build it once:

```bash
./scripts/build-pytorch-services.sh
```

This creates the `osint-platform-pytorch-cpu:latest` image (~2-3 minutes).

## Step 4: Create Telegram Session

Authenticate your Telegram account (required for the Listener service):

```bash
python3 scripts/telegram_auth.py
```

This will:
1. Prompt for your phone number (with country code)
2. Send a verification code to your Telegram app
3. Create the session files in `./telegram_sessions/`

For a second account (`--account russia` / `--account ukraine`) or to list existing sessions (`--list`), see [Telegram Setup](../operator-guide/telegram-setup.md).

!!! warning "Session File Security"
    The session file (`telegram_sessions/*.session`) contains authentication credentials. **Never commit it to version control.** It's already in `.gitignore`.

## Step 5: Start All Services

Start the entire platform with one command:

```bash
docker-compose up -d
```

This will start:

- **Core Infrastructure**: PostgreSQL, Redis, MinIO, Ollama
- **Application Services**: Listener, Processor, API, Frontend
- **Optional Services**: Monitoring, enrichment workers, authentication and more (profiles, see below)

!!! tip "Start Additional Services"
    Enable optional services with Docker Compose profiles. Set them once in `.env` so every `docker-compose` command sees them:

    ```bash
    # .env
    COMPOSE_PROFILES=enrichment-standard,monitoring
    ```

    Available profiles include `enrichment-lite`, `enrichment-standard`, `enrichment-full` (increasing numbers of background enrichment workers), `monitoring`, `auth`, `opensanctions`, `notifications`, `backfill`, `llm-gateway` and `multi-account`. `.env.example` lists them all.

    `enrichment-lite` runs the core background workers (fast pool, router, Telegram and decision tasks, maintenance, cluster detection). `enrichment-standard` adds AI tagging, event detection, fact-checking, LLM geolocation and the entity/sanctions services. Without any enrichment profile, messages are still archived but background enrichment does not run.

    If you turn on historical backfill (`BACKFILL_ENABLED`), also add the `backfill` profile, or backfilled messages are never stored.

## Step 6: Verify Installation

### Check Service Health

```bash
docker-compose ps
```

All services should show `healthy` status within 2-3 minutes. If any are `unhealthy`, check logs:

```bash
docker-compose logs -f <service_name>
```

### Access Web Interfaces

| Service | URL | Credentials |
|---------|-----|-------------|
| **Frontend** | http://localhost:3000 | No auth required (unless the `auth` profile is on) |
| **API Docs** | http://localhost:8000/docs | No auth required |
| **NocoDB** | http://localhost:8080 | `NOCODB_ADMIN_EMAIL` / `NOCODB_ADMIN_PASSWORD` from `.env` |
| **Grafana** (`monitoring` profile) | http://localhost:3001 | `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD` from `.env` |
| **MinIO Console** | http://localhost:9001 | `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` from `.env` |

!!! success "Platform is Running!"
    If you can access http://localhost:3000 and see the frontend, the platform is operational.

## Step 7: Add Your First Channel

The platform uses **folder-based channel management** - no admin panel required.

### In Telegram App (Desktop or Mobile):

1. **Create a folder** named `Archive` (right-click sidebar → Create Folder)
2. **Find a channel** (e.g., search for a public Ukraine news channel)
3. **Drag the channel** into the `Archive` folder
4. **Wait 5 minutes** for the Listener to detect the new channel

!!! note "Folder Naming Examples"
    - `Archive` - Basic archive folder (7 chars) ✅
    - `Archive-UA` - Ukraine sources (10 chars) ✅
    - `Archive-RU` - Russia sources (10 chars) ✅
    - `Monitor-UA` - Strict filtering for Ukraine (10 chars) ✅

### Verify Channel Discovery

Check the Listener logs to confirm:

```bash
docker-compose logs -f listener
```

You should see:
```
INFO: Discovered: Channel Name (@channel_username) ...
INFO: Added new channel: Channel Name (@channel_username)
```

!!! tip "Folder Naming Rules"
    - `Archive-*` → LLM archives most content (lenient mode)
    - `Monitor-*` → LLM archives only more important content; low importance is never archived (strict mode)
    - `Discover-*` → Very strict; new channels are evaluated for about a week after 30+ messages, then promoted or dropped

    **Important**: Telegram limits folder names to **12 characters**. Use `-UA` or `-RU` suffixes, not full country names.

    See [Add a Telegram Channel](../tutorials/add-telegram-channel.md) for details.

## Step 8: View Your First Message

Once a message arrives in your monitored channel:

1. **Check Processor logs** to see message processing:
   ```bash
   docker-compose logs -f processor-worker
   ```

2. **Open the Frontend** at http://localhost:3000

3. **View messages** in the message list (sorted by date, newest first)

4. **Try searching** using the search bar (full-text and semantic search)

## Next Steps

### Essential Configuration

- **Add more channels**: Drag channels into Telegram folders
- **Tune classification and spam thresholds**: admin settings (Admin → Config in the frontend, or `scripts/admin-config.sh`); see [Configuration](../operator-guide/configuration.md)
- **Review rejected or spam-flagged messages**: Admin → Spam in the frontend

### Learn Core Features

- [Channel Management](../tutorials/add-telegram-channel.md) - Folder-based workflow
- [Searching Messages](../user-guide/searching.md) - Full-text and semantic search
- [RSS Feeds](../user-guide/rss-feeds.md) - Subscribe to search queries
- [Entity Extraction](../user-guide/entities.md) - Track people, units, equipment

### Monitoring & Operations

- [Grafana Dashboards](../operator-guide/monitoring.md) - Pre-configured monitoring
- [Database Administration](../operator-guide/index.md) - NocoDB and admin tools
- [Backup & Recovery](../operator-guide/backup-restore.md) - Data protection

### Advanced Topics

- [LLM Integration](../developer-guide/llm-integration.md) - How classification uses the LLM
- [Enrichment Service](../developer-guide/services/enrichment.md) - Background AI tasks
- [Architecture Overview](../developer-guide/architecture.md) - Technical deep dive

## Common Issues

### Listener Not Connecting

**Symptom**: `docker-compose logs listener` shows connection errors

**Solution**:
1. Verify Telegram API credentials in `.env`
2. Re-run `python3 scripts/telegram_auth.py`
3. Check session file exists: `ls telegram_sessions/`

### Processor Not Processing Messages

**Symptom**: Messages arrive but aren't saved to database

**Solution**:
1. Check Redis connection: `docker-compose logs redis`
2. Check PostgreSQL health: `docker-compose exec postgres pg_isready`
3. Verify Processor logs: `docker-compose logs processor-worker`

### Ollama Model Not Loading (`LLM_PROVIDER=ollama`)

**Symptom**: Importance classification fails, logs show "model not found"

**Solution**:
```bash
# Pull the model manually
docker-compose exec ollama ollama pull <your OLLAMA_MODEL>

# Verify it's loaded
docker-compose exec ollama ollama list
```

### Out of Memory

**Symptom**: Services keep restarting, `docker stats` shows high memory

**Solution**:
1. Reduce Processor replicas: set `PROCESSOR_WORKER_REPLICAS=1` in `.env`
2. Use smaller LLM model: set a smaller `OLLAMA_MODEL` in `.env`
3. Run fewer enrichment workers: use `COMPOSE_PROFILES=enrichment-lite` or none

### Port Conflicts

**Symptom**: `Error: bind: address already in use`

**Solution**:
1. Check which service is using the port: `sudo lsof -i :PORT`
2. Change port in `.env` (e.g., `API_PORT=8001`)
3. Restart services: `docker-compose down && docker-compose up -d`

## Helpful Commands

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f listener

# Last 100 lines
docker-compose logs --tail=100 processor-worker
```

### Restart Services

```bash
# Restart single service
docker-compose restart listener

# Restart all services
docker-compose restart

# Full rebuild (after code changes)
docker-compose down && docker-compose build --no-cache && docker-compose up -d
```

### Database Access

```bash
# Connect to PostgreSQL
docker-compose exec postgres psql -U osint_user -d osint_platform

# Run SQL query
docker-compose exec postgres psql -U osint_user -d osint_platform -c "SELECT COUNT(*) FROM messages;"
```

### Clean Up

```bash
# Stop all services
docker-compose down

# Stop and remove volumes (⚠️ deletes all data)
docker-compose down -v

# Remove all platform images
docker images | grep osint | awk '{print $3}' | xargs docker rmi
```

## Getting Help

- **Documentation**: Browse the [User Guide](../user-guide/index.md) and [Operator Guide](../operator-guide/index.md)
- **Troubleshooting**: See [Troubleshooting Guide](../operator-guide/troubleshooting.md)
- **GitHub Issues**: [Report bugs or request features](https://github.com/osintukraine/osint-intelligence-platform/issues)

!!! success "Congratulations!"
    You've successfully deployed the OSINT Intelligence Platform. Start adding channels and exploring the features!
