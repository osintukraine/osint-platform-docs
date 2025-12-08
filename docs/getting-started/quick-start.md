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
MINIO_ACCESS_KEY=minio_access_key_here
MINIO_SECRET_KEY=minio_secret_key_minimum_32_chars_here
JWT_SECRET_KEY=random_256_bit_key_here
```

!!! tip "Generate Strong Secrets"
    Use `openssl rand -hex 32` to generate strong random passwords for database and API credentials.

### Optional Configuration

For production deployments, also configure:

- `OLLAMA_MODEL=qwen2.5:3b` (recommended for Russian/Ukrainian content)
- `LLM_ENABLED=true` (enables AI importance classification)
- `TRANSLATION_ENABLED=true` (auto-translate messages)

See the `.env.example` file for complete configuration options.

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
1. Prompt for your phone number (must match `TELEGRAM_PHONE` in `.env`)
2. Send a verification code to your Telegram app
3. Create a session file in `./telegram_sessions/`

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
- **Optional Services**: Monitoring, enrichment workers (see profiles below)

!!! tip "Start Additional Services"
    Enable optional services using Docker Compose profiles:

    ```bash
    # Start with monitoring stack
    docker-compose --profile monitoring up -d

    # Start with enrichment workers (AI tagging, embeddings)
    docker-compose --profile enrichment up -d

    # Start everything
    docker-compose --profile monitoring --profile enrichment up -d
    ```

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
| **Frontend** | http://localhost:3000 | No auth required |
| **API Docs** | http://localhost:8000/docs | No auth required |
| **NocoDB** (optional) | http://localhost:8080 | admin@osint.local / change-this-password |
| **Grafana** (optional) | http://localhost:3001 | admin / admin |
| **MinIO Console** | http://localhost:9001 | minioadmin / minioadmin |

!!! success "Platform is Running!"
    If you can access http://localhost:3000 and see the frontend, the platform is operational.

## Step 7: Add Your First Channel

The platform uses **folder-based channel management** - no admin panel required.

### In Telegram App (Desktop or Mobile):

1. **Create a folder** named `Archive-Test` (right-click sidebar → Create Folder)
2. **Find a channel** (e.g., search for a public Ukraine news channel)
3. **Drag the channel** into the `Archive-Test` folder
4. **Wait 5 minutes** for the Listener to detect the new channel

### Verify Channel Discovery

Check the Listener logs to confirm:

```bash
docker-compose logs -f listener
```

You should see:
```
INFO: Discovered new channel in folder Archive-Test: @channel_username
INFO: Subscribing to channel @channel_username...
```

!!! tip "Folder Naming Rules"
    - `Archive-*` → Archives ALL messages (after spam filter)
    - `Monitor-*` → Archives only HIGH importance messages
    - `Discover-*` → Auto-created for forward chain discovery

    See [Channel Management](../operator-guide/channel-management.md) for details.

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
- **Configure spam filter**: Edit `spam_patterns` table in NocoDB
- **Adjust importance rules**: Edit `importance_rules` table

### Learn Core Features

- [Channel Management](../operator-guide/channel-management.md) - Folder-based workflow
- [Searching Messages](../user-guide/searching.md) - Full-text and semantic search
- [RSS Feeds](../user-guide/rss-feeds.md) - Subscribe to search queries
- [Entity Extraction](../user-guide/entities.md) - Track people, units, equipment

### Monitoring & Operations

- [Grafana Dashboards](../operator-guide/monitoring.md) - Pre-configured monitoring
- [Database Administration](../operator-guide/nocodb.md) - NocoDB usage guide
- [Backup & Recovery](../operator-guide/backup-restore.md) - Data protection

### Advanced Topics

- [Multi-Model AI](../architecture/multi-model-ai.md) - Switch LLM models at runtime
- [Enrichment Pipeline](../architecture/enrichment-pipeline.md) - Background AI tasks
- [Architecture Overview](../architecture/overview.md) - Technical deep dive

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

### Ollama Model Not Loading

**Symptom**: Importance classification fails, logs show "model not found"

**Solution**:
```bash
# Pull the model manually
docker-compose exec ollama ollama pull qwen2.5:3b

# Verify it's loaded
docker-compose exec ollama ollama list
```

### Out of Memory

**Symptom**: Services keep restarting, `docker stats` shows high memory

**Solution**:
1. Reduce Processor replicas: Edit `docker-compose.yml`, set `replicas: 1`
2. Use smaller LLM model: Set `OLLAMA_MODEL=gemma2:2b` in `.env`
3. Disable enrichment workers: Don't use `--profile enrichment`

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
- **FAQ**: Check [Frequently Asked Questions](../reference/faq.md)
- **Troubleshooting**: See [Troubleshooting Guide](../reference/troubleshooting.md)
- **GitHub Issues**: [Report bugs or request features](https://github.com/osintukraine/osint-intelligence-platform/issues)

!!! success "Congratulations!"
    You've successfully deployed the OSINT Intelligence Platform. Start adding channels and exploring the features!
