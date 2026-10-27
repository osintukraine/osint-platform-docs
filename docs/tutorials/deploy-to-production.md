# Deploy to Production

**Time: ~30 minutes (setup) + 15 minutes (first channel monitoring)**

This is the complete walkthrough for deploying the OSINT Intelligence Platform to a production server with SSL, proper monitoring, and first-time monitoring setup. By the end, you'll have a fully operational platform running on your own domain.

---

## What You'll Learn

After this tutorial, you will be able to:

1. Provision and configure a production server
2. Set up SSL/TLS encryption with Caddy
3. Configure environment variables for production
4. Deploy all platform services with Docker Compose
5. Authenticate Telegram for the first time
6. Add and monitor your first channel
7. Monitor platform health with Grafana

---

## Prerequisites

Before starting, make sure you have:

- A VPS server (Ubuntu 22.04 LTS or Fedora 38+ recommended)
- SSH access to your server with sudo privileges
- A domain name pointing at the server (required for the HTTPS/Caddy setup below; skip Steps 4-5 and stay on `localhost` ports for a private test)
- Basic knowledge of:
  - Terminal/command line
  - SSH and connecting to remote servers
  - Copy/pasting commands
- 30 minutes of uninterrupted time for initial setup
- About 500GB+ disk space available
- At least 8GB RAM (16GB recommended); see [Installation](../operator-guide/installation.md) for full requirements

**Server Recommendations:**
- CPU: 2+ cores
- RAM: 8GB minimum (16GB recommended for 100+ channels; local Ollama needs more)
- Disk: 500GB SSD minimum
- Network: Stable broadband connection

**Estimated Costs:**
- VPS: 15-30 USD/month (DigitalOcean, Hetzner, Linode)
- Domain: 10-15 USD/year (Namecheap, GoDaddy)
- SSL: FREE (Let's Encrypt via Caddy)
- LLM: your own GPU gateway or Ollama hardware (not included in the VPS cost)
- **Total: ~40-50 USD/month**

---

## Architecture Overview

```
Internet → Caddy (HTTPS, CrowdSec bouncer)
             → Oathkeeper (auth proxy, backed by Kratos)
                 → Frontend (Next.js) / API (FastAPI)
                       ├─ Telegram Listener(s) → Redis → Processor workers → PostgreSQL + MinIO
                       ├─ Enrichment workers, RSS ingestor, Notifier (ntfy), Analytics
                       └─ Monitoring (Prometheus / Grafana / Loki)
```

All services run in Docker containers managed by Docker Compose.

---

## Step 1: Set Up the Server

Let's start with a freshly provisioned server.

### 1.1: Connect to Your Server

```bash
# SSH into your server
ssh -i /path/to/key.pem user@your-server-ip

# Or if using password auth:
ssh user@your-server-ip
```

### 1.2: Update System Packages

```bash
# Update package lists
sudo apt update

# Upgrade existing packages
sudo apt upgrade -y

# Install required packages
sudo apt install -y git curl wget nano htop
```

**Expected Output:**
```
Reading package lists... Done
Building dependency tree... Done
Processing triggers...
```

### 1.3: Install Docker and Docker Compose

```bash
# Download and run Docker installer (includes the Compose v2 plugin)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group (avoid sudo for docker commands)
sudo usermod -aG docker $USER

# Apply group change
newgrp docker
```

### 1.4: Verify Docker Installation

```bash
# Check Docker version
docker --version

# Check Docker Compose version
docker compose version

# Test Docker works
docker run hello-world
```

**Expected Output:**
```
Docker version 24.x.x
Docker Compose version v2.x.x
Hello from Docker!
```

### 1.5: Configure Firewall

```bash
# Enable UFW firewall
sudo ufw enable

# Allow SSH (critical - do this first!)
sudo ufw allow 22/tcp

# Allow HTTP and HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Verify firewall rules
sudo ufw status
```

!!! warning "Docker bypasses UFW"
    Ports published by `docker-compose.yml` (PostgreSQL 5432, Redis 6379, MinIO 9000/9001, API 8000, metrics ports, ...) are opened by Docker directly in iptables and are **not** blocked by UFW rules. With the production overlay (Step 4) only Caddy publishes 80/443 for the app; still verify from outside with `nmap` or an online port scan. See [Hardening](../security-guide/hardening.md).

**Expected Output:**
```
Status: active
     To                         Action      From
     --                         ------      ----
22/tcp                     ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```

---

## Step 2: Clone the Repository

```bash
# Create a directory for the project
mkdir -p /opt/osint
cd /opt/osint

# Clone the repository
git clone https://github.com/osintukraine/osint-intelligence-platform.git .

# Production runs the master branch
git checkout master
```

**Expected Output:**
```
Cloning into 'osint-intelligence-platform'...
Branch 'master' set up to track origin/master
```

---

## Step 3: Set Up Environment Variables

Start from the shipped template, which documents every variable:

```bash
cd /opt/osint
cp .env.example .env
chmod 600 .env
nano .env
```

Compose refuses to start if a required secret is missing. Set at least these (generate values with `openssl rand -base64 32`; hex with `openssl rand -hex 32`):

```bash
# === Telegram (from https://my.telegram.org) ===
TELEGRAM_API_ID=...
TELEGRAM_API_HASH=...
TELEGRAM_PHONE=+1234567890

# === Secrets (required by docker-compose.yml) ===
POSTGRES_USER=osint_user
POSTGRES_PASSWORD=...
REDIS_PASSWORD=...
JWT_SECRET_KEY=...            # openssl rand -hex 32
MINIO_ROOT_USER=...
MINIO_ROOT_PASSWORD=...
MINIO_ACCESS_KEY=...
MINIO_SECRET_KEY=...          # at least 32 characters
GRAFANA_ADMIN_PASSWORD=...
GRAFANA_SECRET_KEY=...
NOCODB_ADMIN_PASSWORD=...
NOCODB_JWT_SECRET=...
YENTE_UPDATE_TOKEN=...

# === Authentication (Ory Kratos) ===
AUTH_PROVIDER=ory
AUTH_REQUIRED=true
KRATOS_SECRET_COOKIE=...      # openssl rand -base64 32
KRATOS_SECRET_CIPHER=...      # openssl rand -base64 24 (exactly 32 characters)

# === Domain / production mode ===
DEPLOYMENT_MODE=production
DOMAIN=your-domain.com
NEXT_PUBLIC_API_URL=https://your-domain.com
NEXT_PUBLIC_BASE_URL=https://your-domain.com
NEXT_PUBLIC_RSS_URL=https://your-domain.com/rss
NEXT_PUBLIC_MINIO_URL=...        # public media base URL; see Caddy / Media Serving in the Operator Guide
ENVIRONMENT=production
DEBUG=false

# === Compose: production overlay + profiles ===
COMPOSE_FILE=docker-compose.yml:docker-compose.production.yml
COMPOSE_PROFILES=enrichment-standard,auth,monitoring,notifications
```

Pick your LLM provider (`LLM_PROVIDER=openai` with `LLM_GATEWAY_URL` and keys for a GPU gateway, or `ollama`, or `none`) as described in [LLM Gateway and Providers](../operator-guide/external-llm.md). Add the `llm-gateway` profile if you use a gateway. The remaining variables have working defaults; see [Configuration](../operator-guide/configuration.md).

!!! danger "Critical: Frontend Build Args"
    **`NEXT_PUBLIC_*` variables are baked into the JavaScript at BUILD time, not runtime.**

    `docker-compose.yml` already passes `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_BASE_URL` and `NEXT_PUBLIC_MINIO_URL` from `.env` as build args, so set them in `.env` **before** the first build. If you change them later, rebuild: `docker compose build frontend && docker compose up -d frontend`.

    See [Production Gotchas](../operator-guide/production-gotchas.md#frontend-build-args) for details.

!!! warning "Security: Passwords"
    Use unique, strong secrets (32+ characters). Never commit `.env`. Rotating `KRATOS_SECRET_CIPHER` later invalidates stored Kratos data, so back it up.

## Step 3b: Fetch Embedding Models

Embedding models are provisioned on the host and mounted read-only at `/models`; services never download them. Run this once, and again after upgrades that change `infrastructure/models/models.txt`:

```bash
cd /opt/osint
./scripts/fetch-models.sh
ls data/models
```

See [Embedding Models](../operator-guide/model-provisioning.md). The spam filter will not start without them. If you use an LLM gateway instead of Ollama, read [LLM Gateway and Providers](../operator-guide/external-llm.md) now and add `llm-gateway` to `COMPOSE_PROFILES`.

---

## Step 4: Set Up Caddy for SSL/TLS

The production overlay (`docker-compose.production.yml`, enabled by `COMPOSE_FILE` in Step 3) runs Caddy with the CrowdSec bouncer, using `infrastructure/caddy/Caddyfile.security`. Caddy obtains and renews Let's Encrypt certificates for `DOMAIN` automatically; you do not write a Caddyfile.

1. Point your domain's DNS A record at the server and confirm: `nslookup your-domain.com`.
2. Apply the production URLs to `.env` (the script rewrites `DEPLOYMENT_MODE`, `DOMAIN`, `FRONTEND_URL`, `API_URL`, `PROXY_URL`):

    ```bash
    ./scripts/switch-deployment-mode.sh production your-domain.com
    ```

3. Generate the CrowdSec API keys that Caddy's bouncer needs (written to `.env`):

    ```bash
    ./scripts/setup-crowdsec.sh
    ```

Routing, security headers and the CrowdSec bouncer are described in [Caddy](../operator-guide/caddy.md) and the [CrowdSec guide](../security-guide/crowdsec.md).

!!! note "No domain yet?"
    Skip this step, keep `DEPLOYMENT_MODE=development` and `AUTH_PROVIDER=none`, drop `COMPOSE_FILE` and the `auth` profile from `.env`, and use the published ports (`http://SERVER_IP:3000`, `:8000`). Do not leave a server in this state on the public internet.

---

## Step 5: Start Docker Services

```bash
cd /opt/osint

# Create the Telegram session first (Step 6), then start the stack.
# COMPOSE_FILE and COMPOSE_PROFILES from .env are picked up automatically.
docker compose up -d

# Check service status (first start builds images and can take several minutes)
docker compose ps
```

**Expected Output** (abridged; names depend on your profiles, the processor runs 2 replicas by default):
```
NAME                      STATUS
osint-postgres            Up (healthy)
osint-redis               Up (healthy)
osint-minio               Up (healthy)
osint-listener            Up (healthy)
osint-processor-worker-1  Up (healthy)
osint-api                 Up (healthy)
osint-frontend            Up (healthy)
osint-caddy-secured       Up
osint-kratos              Up (healthy)
osint-oathkeeper          Up (healthy)
```

!!! tip "Start order"
    If the stack fails at first start, bring up the infrastructure first: `docker compose up -d postgres redis minio minio-init`, wait until healthy, then `docker compose up -d`.

### Check Service Logs

```bash
docker compose logs --tail 20 listener
docker compose logs --tail 20 processor-worker
docker compose logs --tail 20 api
```

You should see initialization messages, no errors.

---

## Step 6: First-Time Telegram Authentication

The listener needs an authenticated Telegram session. Create it with the interactive script **before** (or while) the listener runs. The listener cannot prompt for a login code itself.

### 6.1: Run the Authentication Script

```bash
cd /opt/osint
python3 -m venv .venv && . .venv/bin/activate
pip install -r services/listener/requirements.txt python-dotenv   # telethon + shared settings deps

python3 scripts/telegram_auth.py
```

### 6.2: Follow the Prompts

1. Enter your phone number if asked (it defaults to `TELEGRAM_PHONE`).
2. Enter the login code Telegram sends to your app.
3. Enter your 2FA password if enabled.

The script writes `telegram_sessions/osint_platform.session` and copies it to the per-service session files (listener, discovery, enrichment, analytics). The directory `./telegram_sessions` is mounted into the containers at `/app/sessions`.

For several Telegram accounts, run `python3 scripts/telegram_auth.py --account russia` (and `--account ukraine`) then switch with `./scripts/switch-telegram-mode.sh multi` (sets `TELEGRAM_MODE=multi`; unset means single-account); see [Telegram Setup](../operator-guide/telegram-setup.md).

### 6.3: Start or Restart the Listener

```bash
docker compose up -d listener
docker compose logs --tail 20 listener
```

You should see the Telegram client connect and the folder sync start; there will be 0 channels until Step 7.

!!! warning "Session Persistence and Safety"
    Sessions persist across restarts, so you authenticate once. Treat `telegram_sessions/` like a password (it grants full access to the account), back it up, and never run the same session file in two places: Telegram revokes it (`AuthKeyDuplicated`).

---

## Step 7: Add Your First Channel

Now let's add a channel to monitoring. This works the same way as local setup:

1. Open Telegram app (mobile or desktop)
2. Create a folder called `Archive` (or `Archive-UA` for Ukraine sources)
3. Find a public channel and add it to the folder
4. Wait up to 5 minutes for discovery (or `docker compose restart listener`)

Check if it was discovered:

```bash
# Check listener logs
docker compose logs listener | grep -i "folder\|channel"

# Or query the database
docker compose exec -T postgres psql -U osint_user -d osint_platform -c \
  "SELECT id, name, username, folder, rule FROM channels LIMIT 5;"
```

**Expected Output:**
```
 id |      name       |   username    | folder | rule
----+-----------------+---------------+--------+---------
  1 | Test Channel    | test_channel  | Archive | archive_all
```

---

## Step 8: Verify Platform is Working

Let's verify all services are operational:

### 8.1: Check API Endpoint

```bash
# Through Caddy on your domain
curl -s https://your-domain.com/api/health | jq .

# Or from the server itself (the API container's own health endpoint)
docker compose exec -T api curl -s http://localhost:8000/health
```

**Expected Output:**
```json
{
  "status": "healthy",
  "service": "api",
  "version": "..."
}
```

### 8.2: List Monitored Channels

```bash
# Get channels via API (returns a JSON array)
curl -s https://your-domain.com/api/channels | jq '.[0:3]'
```

**Expected Output:**
```json
[
  {
    "id": 1,
    "name": "Test Channel",
    "username": "test_channel",
    "folder": "Archive"
  }
]
```

### 8.3: Send a Test Message

Send a test message to your monitored channel, then query for it:

```bash
# Wait 10 seconds for processing
sleep 10

# Count messages from last hour
docker compose exec -T postgres psql -U osint_user -d osint_platform -c \
  "SELECT COUNT(*) FROM messages WHERE telegram_date > NOW() - INTERVAL '1 hour';"
```

If your test message appeared, the pipeline is working!

---

## Step 9: Access the Frontend

Your platform is now accessible via the domain (or localhost):

```
https://your-domain.com
```

Visit it in your browser. You should see the OSINT platform homepage. With `AUTH_PROVIDER=ory`, create the first account via the registration page, then see [Authentication](../security-guide/authentication.md) and [Authorization](../security-guide/authorization.md) to grant it the admin role.

---

## Step 10: Set Up Monitoring with Grafana

Grafana provides dashboards for monitoring platform health.

### 10.1: Access Grafana

Grafana is part of the `monitoring` profile. It publishes `GRAFANA_PORT` (3001 in `.env.example`, 3000 if unset) on the host, which should stay firewalled; reach it through an SSH tunnel:

```bash
ssh -L 3001:localhost:3001 user@your-server-ip
# then open http://localhost:3001
```

Login: `GRAFANA_ADMIN_USER` (default `admin`) with the `GRAFANA_ADMIN_PASSWORD` you set in `.env`.

### 10.2: Dashboards

Dashboards and datasources are provisioned automatically from `infrastructure/grafana/` (for example `platform-overview`, `osint-pipeline`, `postgres`, `redis`, `enrichment-performance`, `service-logs`). No manual import is needed.

### 10.3: Set Up Alerts (Optional)

Alertmanager routes alerts through the `notifier` service to the bundled ntfy server (profiles `monitoring` / `notifications`). Set `NTFY_TOPIC_PREFIX` in `.env` (default `osint-platform`) and subscribe to the topics with the ntfy app; see [Notifications](../user-guide/notifications.md) and [Monitoring](../operator-guide/monitoring.md).

---

## Step 11: Configure Backups

Use the shipped backup script. It dumps PostgreSQL, copies the Telegram sessions, a Redis snapshot and the configuration files into `backups/<timestamp>/`:

```bash
cd /opt/osint
./scripts/backup.sh --no-ollama      # add --include-media to also rsync media (slow, large)
ls backups/
```

Automate it with cron (daily at 2 AM):

```bash
crontab -e
# add:
0 2 * * * cd /opt/osint && ./scripts/backup.sh --no-ollama >> /var/log/osint_backup.log 2>&1
```

Media in MinIO is not part of the default dump; see [Backup & Restore](../operator-guide/backup-restore.md) and [Hetzner Storage](../operator-guide/hetzner-storage.md) for the media strategy and restore procedure.

---

## Step 12: Production Checklist

Before considering your deployment complete, verify:

- [ ] All Docker services running (`docker compose ps`)
- [ ] API responding to requests (`curl https://your-domain.com/api/health`)
- [ ] At least one Telegram channel monitored
- [ ] Messages being archived to database
- [ ] SSL working (if domain configured)
- [ ] Monitoring dashboard accessible
- [ ] Backups running successfully
- [ ] Firewall rules in place and internal ports (5432, 6379, 9000, 8000, ...) not reachable from outside
- [ ] Default/placeholder secrets all replaced

---

## Updating to Latest Version

Use the deploy script. It pulls, applies pending database migrations, restarts services and runs a health check:

```bash
cd /opt/osint

./scripts/deploy.sh --dry-run    # preview
./scripts/deploy.sh              # pull, migrate, restart
./scripts/deploy.sh --build      # same, and rebuild images first
./scripts/deploy.sh --quick      # pull and restart, skip migrations
```

Two steps are not part of the script:

```bash
./scripts/fetch-models.sh        # when infrastructure/models/models.txt changed
./scripts/migrate.sh --status    # check migration state at any time
```

`./scripts/migrate.sh --dry-run` previews pending migrations and `./scripts/migrate.sh` applies them. See [Upgrades](../operator-guide/upgrades.md).

---

## Troubleshooting

### "Listener can't authenticate with Telegram"

```bash
# Check which session files exist
python3 scripts/telegram_auth.py --list

# Recreate the sessions (stop the listener first)
docker compose stop listener
mkdir -p telegram_sessions/old && mv telegram_sessions/*.session* telegram_sessions/old/
python3 scripts/telegram_auth.py
docker compose up -d listener
```

### "Database won't start / connection refused"

```bash
# Check database logs
docker compose logs postgres | tail -20
```

!!! danger "Never delete the database volume on a system with data"
    Removing the PostgreSQL volume (`docker volume rm ...`) destroys every archived message. Only do that on a brand-new install, and restore from a backup otherwise ([Backup & Restore](../operator-guide/backup-restore.md)). Check `docker volume ls | grep postgres` for the real volume name first.

A password mismatch after editing `POSTGRES_PASSWORD` in `.env` is a common cause: the password is only applied when the volume is first created.

### "Services crashing repeatedly"

```bash
# Check logs for specific service
docker compose logs processor-worker | tail -50

# If out of memory, increase the server's memory or reduce replicas:
# set PROCESSOR_WORKER_REPLICAS=1 in .env, then
docker compose up -d processor-worker
docker compose logs -f processor-worker
```

### "SSL certificate not working"

```bash
# Check Caddy logs
docker compose logs caddy

# Caddy auto-renews certificates, but you can force:
docker compose restart caddy

# Verify DNS is pointing to server:
nslookup your-domain.com
# Should show your server IP
```

### "Frontend calls wrong API URL (localhost)"

The browser's Network tab shows API calls going to `localhost:8000` instead of your production URL.

**Cause**: `NEXT_PUBLIC_*` variables are baked at build time, not runtime.

**Fix**:

```bash
# 1. Set NEXT_PUBLIC_API_URL, NEXT_PUBLIC_BASE_URL, NEXT_PUBLIC_MINIO_URL in .env
# 2. Rebuild and restart the frontend
docker compose build frontend
docker compose up -d frontend
```

See [Production Gotchas](../operator-guide/production-gotchas.md#frontend-build-args) for details.

---

## What You Learned

Congratulations! You now understand:

1. **Server provisioning** - How to set up a production Linux server
2. **Docker deployment** - How to run services in containers
3. **Environment configuration** - How to configure the platform for production
4. **SSL/TLS setup** - How to secure traffic with Caddy
5. **Telegram authentication** - How to authenticate once on the server
6. **Monitoring setup** - How to monitor platform health
7. **Backup strategy** - How to back up critical data
8. **Troubleshooting** - How to diagnose and fix common issues

---

## Next Steps

Now that you have a production deployment:

1. **Monitor Daily** - Check Grafana dashboards regularly
2. **Set Up Team Access** - Create accounts and roles; see the [Security Guide](../security-guide/index.md)
3. **Create RSS Feeds** - Follow the [Create Custom RSS Feed](create-custom-rss-feed.md) tutorial
4. **Set Up Discord Alerts** - Follow the [Setup Discord Alerts](setup-discord-alerts.md) tutorial
5. **Scale as Needed** - Add more channels, adjust resources
6. **Plan Maintenance** - Schedule regular updates and backups

---

## Cost Optimization Tips

- **Reduce processor workers:** set `PROCESSOR_WORKER_REPLICAS=1` for low traffic
- **Pick a lighter enrichment profile:** `enrichment-lite` (CPU only) instead of `enrichment-standard`
- **Selective archiving:** use `Monitor-*` folders to archive only medium and high importance messages
- **Cheaper media storage:** see [Hetzner Storage](../operator-guide/hetzner-storage.md) and [Multi-Storage](../operator-guide/multi-storage-setup.md)

---

## Key Takeaways

| Step | Time | Key Action |
|------|------|-----------|
| **Server Setup** | 10 min | Install Docker, configure firewall |
| **Deploy Platform** | 5 min | Clone repo, start services |
| **Authenticate** | 5 min | Set up Telegram session |
| **Add Channels** | 5 min | Create folder, add channels |
| **Verify** | 5 min | Check logs, test API |

---

**Your OSINT Intelligence Platform is now live in production!**
