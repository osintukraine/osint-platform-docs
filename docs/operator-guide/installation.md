# Installation

Deploy the OSINT Intelligence Platform using Docker Compose.

## Overview

This guide walks through installing the platform from scratch on a fresh server. The installation process takes 2-4 hours depending on your server specifications and network speed.

**What you'll accomplish:**

- Install Docker and Docker Compose
- Clone the repository and configure environment variables
- Build required base images
- Create Telegram authentication session
- Start all services
- Verify successful deployment

## System Requirements

### Minimum Requirements (Development/Testing)

- **Operating System**: Linux (Ubuntu 22.04 LTS or Debian 12 recommended)
- **CPU**: 4 cores (8 threads)
- **RAM**: 8GB
- **Storage**: 100GB SSD
- **Network**: 10 Mbps internet connection
- **Software**: Docker 20.10+, Docker Compose 2.0+

### Recommended Requirements (Production)

- **Operating System**: Linux (Ubuntu 22.04 LTS or Debian 12)
- **CPU**: 8+ cores (16+ threads preferred)
- **RAM**: 16-32GB
- **Storage**: 500GB-2TB NVMe SSD
- **Network**: 100 Mbps+ internet connection, 1 Gbps preferred
- **Software**: Docker 24.0+, Docker Compose 2.20+

### Storage Breakdown

Plan for the following storage allocation:

| Component | Space Required |
|-----------|----------------|
| Docker images | ~20GB |
| PostgreSQL database | 500MB per 100K messages |
| MinIO media storage | 12-20TB over 3 years (with spam filter) |
| Ollama models | 10-15GB (6 models) |
| Logs | 1-2GB/week |
| Metrics | 5GB/month (30-day retention) |
| OS and system | 20-30GB |

## Pre-Installation Checklist

Before you begin, ensure you have:

- [ ] Root or sudo access to Linux server
- [ ] Server accessible via SSH
- [ ] Domain name (optional, for production HTTPS)
- [ ] Telegram account for monitoring
- [ ] Telegram API credentials from [my.telegram.org](https://my.telegram.org)
- [ ] DeepL API key (free, from [deepl.com/pro-api](https://www.deepl.com/pro-api))
- [ ] Backup storage solution configured (optional but recommended)
- [ ] Firewall rules configured (see Network Requirements below)

### Network Requirements

**Inbound Ports (if exposing services):**

| Port | Service | Required |
|------|---------|----------|
| 80 | HTTP (Caddy reverse proxy) | Optional |
| 443 | HTTPS (Caddy reverse proxy) | Optional |
| 3000 | Frontend (Next.js) | Development only |
| 8000 | API (FastAPI) | Development only |

**Outbound Ports (required for operation):**

| Port | Service | Purpose |
|------|---------|---------|
| 443 | HTTPS | Telegram API, DeepL API, RSS feeds |
| 80 | HTTP | Package downloads, RSS feeds |

## Installation Steps

### Step 1: Install Docker and Docker Compose

#### Ubuntu/Debian

```bash
# Update package index
sudo apt-get update

# Install prerequisites
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add your user to docker group (logout/login required)
sudo usermod -aG docker $USER
```

#### Verification

```bash
# Verify Docker installation
docker --version  # Should show Docker version 20.10+
docker compose version  # Should show Docker Compose version 2.0+

# Test Docker (should download and run hello-world)
docker run hello-world
```

### Step 2: Clone Repository

```bash
# Navigate to your preferred installation directory
cd /opt  # or ~/projects for development

# Clone the repository
git clone https://github.com/osintukraine/osint-intelligence-platform.git
cd osint-intelligence-platform

# Checkout master branch (production) or develop (latest features)
git checkout master  # For production
# OR
git checkout develop  # For latest features (may be unstable)
```

### Step 3: Configure Environment Variables

```bash
# Copy example environment file
cp .env.example .env

# Edit with your favorite editor
nano .env  # or vim, vi, etc.
```

**Critical variables to configure:**

```bash
# Database credentials (CHANGE THESE!)
POSTGRES_PASSWORD=your_strong_password_here
POSTGRES_USER=osint_user

# Redis credentials (CHANGE THIS!)
REDIS_PASSWORD=your_redis_password_here

# MinIO credentials (CHANGE THESE!)
MINIO_ROOT_USER=your_minio_user
MINIO_ROOT_PASSWORD=your_strong_minio_password_32_chars

# JWT secret (CHANGE THIS!)
JWT_SECRET_KEY=$(openssl rand -hex 32)

# Telegram API credentials (from https://my.telegram.org/apps)
TELEGRAM_API_ID=your_api_id_from_telegram
TELEGRAM_API_HASH=your_api_hash_from_telegram
TELEGRAM_PHONE=+1234567890  # Your phone number with country code

# DeepL API key (free from https://www.deepl.com/pro-api)
DEEPL_API_KEY=your_deepl_api_key_here

# Grafana admin password (CHANGE THIS!)
GRAFANA_ADMIN_PASSWORD=your_grafana_password

# NocoDB admin credentials (CHANGE THESE!)
NOCODB_ADMIN_EMAIL=admin@yourdomain.com
NOCODB_ADMIN_PASSWORD=your_nocodb_password
NOCODB_JWT_SECRET=$(openssl rand -hex 32)
```

**Generate secure secrets:**

```bash
# Generate random passwords and secrets
openssl rand -hex 32  # For JWT_SECRET_KEY
openssl rand -base64 32  # For database passwords
openssl rand -base64 24  # For KRATOS_SECRET_CIPHER (must be exactly 32 chars)
```

**Optional production settings:**

```bash
# For production with domain name
DOMAIN=osint.yourdomain.com
DEPLOYMENT_MODE=production
AUTH_PROVIDER=ory  # Enable Ory Kratos authentication

# For development
DEPLOYMENT_MODE=development
AUTH_PROVIDER=none  # No authentication required
```

### Step 4: Build PyTorch Base Image

The platform uses a shared PyTorch base image for AI services to save disk space and build time.

```bash
# Build the shared PyTorch CPU base image (takes 5-10 minutes)
./scripts/build-pytorch-services.sh

# Verify the image was created
docker images | grep pytorch-cpu
# Should show: osint-platform-pytorch-cpu    latest    ...
```

### Step 5: Create Telegram Session

The listener service requires an authenticated Telegram session to monitor channels.

```bash
# Run interactive authentication script
python3 scripts/telegram_auth.py

# Follow the prompts:
# 1. Enter your phone number (with country code: +1234567890)
# 2. Enter verification code sent to Telegram app
# 3. Session files are created automatically

# Verify session files were created
ls -lh telegram_sessions/
# Should show:
# osint_platform.session
# listener.session
# enrichment.session
```

**For multi-account setup** (optional, for rate limit scaling):

```bash
# Authenticate Russia account
python3 scripts/telegram_auth.py --account russia

# Authenticate Ukraine account
python3 scripts/telegram_auth.py --account ukraine

# Check status
python3 scripts/telegram_auth.py --status
```

### Step 6: Initialize Ollama Models

Download required LLM models before starting services (prevents delays during first message processing).

```bash
# Create Ollama data directory
mkdir -p data/ollama

# Download models (takes 10-20 minutes, ~10GB total)
docker run --rm \
  -v ./data/ollama:/root/.ollama \
  ollama/ollama pull all-minilm

docker run --rm \
  -v ./data/ollama:/root/.ollama \
  ollama/ollama pull qwen2.5:3b

docker run --rm \
  -v ./data/ollama:/root/.ollama \
  ollama/ollama pull llama3.2:3b

docker run --rm \
  -v ./data/ollama:/root/.ollama \
  ollama/ollama pull granite3-dense:2b

# Verify models
docker run --rm \
  -v ./data/ollama:/root/.ollama \
  ollama/ollama list
```

### Step 7: Start Infrastructure Services

Start core infrastructure first to ensure databases are ready.

```bash
# Start PostgreSQL, Redis, MinIO
docker-compose up -d postgres redis minio minio-init

# Wait for health checks (30-60 seconds)
watch docker-compose ps
# Wait until postgres, redis, minio show "healthy"
# Press Ctrl+C when all are healthy
```

**Verify infrastructure:**

```bash
# Check PostgreSQL
docker-compose exec postgres psql -U osint_user -d osint_platform -c "\dt"
# Should show list of tables (messages, channels, etc.)

# Check Redis
docker-compose exec redis redis-cli ping
# Should return: PONG

# Check MinIO
curl http://localhost:9000/minio/health/live
# Should return: OK
```

### Step 8: Start Application Services

Start all application services (listener, processor, API, frontend).

```bash
# Start all services
docker-compose up -d

# Monitor startup (wait 2-3 minutes)
docker-compose logs -f
# Press Ctrl+C when you see "Application startup complete" from API service
```

**Start with specific profiles:**

```bash
# Essential services only (no monitoring)
docker-compose up -d

# With monitoring (Prometheus, Grafana)
docker-compose --profile monitoring up -d

# With enrichment workers
docker-compose --profile enrichment up -d

# With OpenSanctions entity matching
docker-compose --profile opensanctions up -d

# All profiles (full production stack)
docker-compose --profile monitoring --profile enrichment --profile opensanctions up -d
```

### Step 9: Initialize Ollama Models in Containers

The ollama-init container automatically pulls models defined in database configuration.

```bash
# Check ollama-init logs
docker-compose logs ollama-init

# Should show:
# "Successfully pulled model: qwen2.5:3b"
# "Successfully pulled model: all-minilm"
# etc.

# Verify models are loaded in Ollama
docker-compose exec ollama ollama list
```

## Post-Installation Verification

### Step 1: Check Service Health

```bash
# All services should show "Up" and "healthy"
docker-compose ps

# Expected output (example):
# NAME                  STATUS
# osint-postgres        Up (healthy)
# osint-redis           Up (healthy)
# osint-minio           Up (healthy)
# osint-listener        Up (healthy)
# osint-processor-...   Up (healthy)
# osint-api             Up (healthy)
# osint-frontend        Up (healthy)
```

### Step 2: Verify Database

```bash
# Connect to database
docker-compose exec postgres psql -U osint_user -d osint_platform

# Check schema
\dt

# Should show tables:
# channels, messages, media_files, message_tags, etc.

# Check extensions
\dx

# Should show:
# vector (pgvector for embeddings)
# pg_trgm (similarity search)
# btree_gin (compound indexes)

# Exit
\q
```

### Step 3: Access Web Interfaces

**Frontend**: [http://localhost:3000](http://localhost:3000)
- Should show Next.js search interface
- No messages yet (expected until channels are added)

**API Docs**: [http://localhost:8000/docs](http://localhost:8000/docs)
- Should show Swagger UI with API endpoints
- Try GET /health endpoint - should return {"status": "healthy"}

**Grafana** (if monitoring profile enabled): [http://localhost:3001](http://localhost:3001)
- Default credentials: admin/admin (change on first login)
- Dashboards should be pre-configured

**NocoDB** (if dev profile enabled): [http://localhost:8080](http://localhost:8080)
- Create account on first access
- Connect to PostgreSQL database manually

### Step 4: Test Telegram Connection

```bash
# Check listener logs
docker-compose logs listener | tail -50

# Should show:
# "Successfully connected to Telegram"
# "Discovering channels from folders..."
# "Found X channels in Archive-* folders"
```

### Step 5: Verify Message Processing Pipeline

```bash
# Check Redis queue
docker-compose exec redis redis-cli XLEN telegram_messages
# Should return: 0 (no messages yet)

# Check processor logs
docker-compose logs processor-worker | tail -50

# Should show:
# "Waiting for messages..."
# "Connected to PostgreSQL"
# "Connected to Ollama"
```

## Initial Configuration

### Step 1: Add Telegram Channels

Channels are managed through Telegram folders (not through web UI).

**In your Telegram app:**

1. Create a folder named "Archive-Test"
2. Add a test channel to this folder
3. Wait 5 minutes for folder sync

**Verify channel was discovered:**

```bash
# Check listener logs
docker-compose logs listener | grep "Discovered channel"

# Check database
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT id, name, username, folder_name FROM channels;"
```

### Step 2: Wait for First Message

Once a channel posts a message:

1. Listener receives it via Telegram API
2. Pushes to Redis queue
3. Processor pulls from queue and processes
4. Message appears in database and frontend

**Monitor the pipeline:**

```bash
# Watch all services in real-time
docker-compose logs -f listener processor-worker api

# Check message count
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(*) FROM messages;"
```

### Step 3: Configure Monitoring Alerts (Optional)

If using monitoring profile:

```bash
# Edit Prometheus alert rules
nano infrastructure/prometheus/rules/alerting_rules.yml

# Reload Prometheus configuration
curl -X POST http://localhost:9090/-/reload

# Configure ntfy subscriptions (for mobile notifications)
# Open http://localhost:8090 and subscribe to topics:
# - osint-platform-critical
# - osint-platform-warnings
# - osint-platform-info
```

## Troubleshooting Installation Issues

### Services Won't Start

**Check logs:**

```bash
docker-compose logs service_name

# Common issues:
# - Missing environment variables
# - Port conflicts
# - Insufficient disk space
# - Docker daemon not running
```

**Common fixes:**

```bash
# Restart specific service
docker-compose restart service_name

# Full restart
docker-compose down
docker-compose up -d

# Check Docker daemon
sudo systemctl status docker
sudo systemctl restart docker
```

### Database Initialization Failed

```bash
# Check PostgreSQL logs
docker-compose logs postgres

# Reset database (WARNING: deletes all data)
docker-compose down
docker volume rm osint-intelligence-platform_postgres_data
docker-compose up -d postgres

# Wait for init.sql to run
docker-compose logs postgres | grep "database system is ready"
```

### Telegram Session Issues

```bash
# Re-authenticate
python3 scripts/telegram_auth.py

# Check session files exist
ls -lh telegram_sessions/

# Check listener can connect
docker-compose logs listener | grep -i telegram
```

### Ollama Model Download Failures

```bash
# Check Ollama logs
docker-compose logs ollama ollama-init

# Manually download model
docker-compose exec ollama ollama pull qwen2.5:3b

# Check available models
docker-compose exec ollama ollama list
```

### Out of Memory

```bash
# Check memory usage
docker stats

# Reduce services:
# - Stop enrichment profile
# - Reduce processor replicas to 1
# - Limit Ollama memory in docker-compose.yml
```

## Next Steps

After successful installation:

1. **[Configure Services](configuration.md)**: Fine-tune environment variables and service settings
2. **[Setup Telegram Monitoring](telegram-setup.md)**: Organize channels into folders and configure rules
3. **[Configure Monitoring](monitoring.md)**: Set up Grafana dashboards and alerts
4. **[Setup Backups](backup-restore.md)**: Implement automated backup procedures

## Production Deployment Checklist

Before deploying to production:

- [ ] Change all default passwords in .env
- [ ] Configure HTTPS with valid SSL certificate
- [ ] Enable authentication (AUTH_PROVIDER=ory)
- [ ] Configure firewall rules
- [ ] Set up automated backups
- [ ] Configure monitoring and alerting
- [ ] Test disaster recovery procedures
- [ ] Document custom configurations
- [ ] Review security best practices

---

**Installation complete!** Your platform is ready to monitor Telegram channels and perform OSINT analysis.
