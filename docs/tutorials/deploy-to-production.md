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
- A domain name (optional but recommended for SSL)
- Basic knowledge of:
  - Terminal/command line
  - SSH and connecting to remote servers
  - Copy/pasting commands
- 30 minutes of uninterrupted time for initial setup
- About 500GB+ disk space available
- At least 4GB RAM (8GB recommended)

**Server Recommendations:**
- CPU: 2+ cores
- RAM: 8GB minimum (16GB recommended for 100+ channels)
- Disk: 500GB SSD minimum
- Network: Stable broadband connection

**Estimated Costs:**
- VPS: 15-30 USD/month (DigitalOcean, Hetzner, Linode)
- Domain: 10-15 USD/year (Namecheap, GoDaddy)
- SSL: FREE (Let's Encrypt via Caddy)
- **Total: ~40-50 USD/month**

---

## Architecture Overview

```
Internet → Caddy (Reverse Proxy + SSL) → Docker Services
                                          ├─ Telegram Listener
                                          ├─ Message Processor
                                          ├─ Enrichment Service
                                          ├─ API Server
                                          ├─ Frontend
                                          ├─ PostgreSQL
                                          ├─ Redis
                                          └─ Monitoring (Prometheus/Grafana)
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
# Download and run Docker installer
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose (v2)
sudo apt install -y docker-compose-plugin

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

# Allow internal Docker network (optional)
sudo ufw allow from 172.16.0.0/12

# Verify firewall rules
sudo ufw status
```

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

# Verify we're on master branch (production)
git branch -a
git checkout master
```

**Expected Output:**
```
Cloning into 'osint-intelligence-platform'...
Branch 'master' set up to track origin/master
```

---

## Step 3: Set Up Environment Variables

Create a production `.env` file with your configuration:

```bash
# Create .env file
nano /opt/osint/.env
```

Paste this configuration (modify values as needed):

```bash
# === Telegram Configuration ===
TELEGRAM_API_ID=your_api_id_from_my_telegram_org
TELEGRAM_API_HASH=your_api_hash_from_my_telegram_org
TELEGRAM_PHONE=+1234567890
TELEGRAM_SESSION_NAME=osint-listener

# === Domain and SSL ===
DOMAIN=your-domain.com
CADDY_AUTO_HTTPS=on

# === Database ===
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=osint_platform
POSTGRES_USER=osint_user
POSTGRES_PASSWORD=generate_a_strong_password_here_min_32_chars
DB_ENCRYPTION_KEY=generate_another_strong_password_here

# === Redis ===
REDIS_URL=redis://redis:6379/0
REDIS_PASSWORD=generate_strong_password_here

# === API Configuration ===
API_HOST=0.0.0.0
API_PORT=8000
API_WORKERS=4
NEXT_PUBLIC_API_URL=https://your-domain.com

# === MinIO (Media Storage) ===
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=generate_strong_password_here
MINIO_BUCKET=telegram-media
MINIO_ENDPOINT=minio:9000
MINIO_SECURE=false

# === Enrichment Services ===
OLLAMA_BASE_URL=http://ollama:11434
EMBEDDING_MODEL=all-MiniLM-L6-v2
IMPORTANCE_MODEL=qwen2.5:3b

# === Notifications ===
NTFY_ENABLED=true
NTFY_TOPIC=osint_alerts

# === Logging ===
LOG_LEVEL=INFO
SENTRY_DSN=

# === Security ===
SECRET_KEY=generate_another_strong_random_key_here
ALLOWED_HOSTS=your-domain.com,api.your-domain.com
CORS_ORIGINS=https://your-domain.com

# === Production Flags ===
DEBUG=false
ENVIRONMENT=production
POSTGRES_POOL_SIZE=20
```

!!! warning "Security: Passwords"
    Generate strong passwords (min 32 characters). Use: `openssl rand -base64 32`

```bash
# Generate strong passwords
openssl rand -base64 32  # Database password
openssl rand -base64 32  # Redis password
openssl rand -base64 32  # Secret key
```

Replace the placeholder values:
1. Press `Ctrl+X`, then `Y`, then `Enter` to save
2. Edit the file with your values:
   - Your Telegram API credentials (from https://my.telegram.org)
   - Your domain name
   - Generated passwords from above
   - Your server IP if not using a domain

---

## Step 4: Set Up Caddy for SSL/TLS

Caddy automatically manages SSL certificates and proxies traffic to your services.

Create a Caddyfile:

```bash
nano /opt/osint/Caddyfile
```

Paste this configuration:

```
# Caddy reverse proxy configuration
{
  email admin@your-domain.com
  acme_dns cloudflare {token}
}

# Main domain
your-domain.com {
  encode gzip

  # Proxy to Next.js frontend
  route / {
    reverse_proxy frontend:3000
  }

  # Proxy to FastAPI backend
  route /api/* {
    reverse_proxy api:8000
  }

  # Proxy to n8n workflows
  route /n8n/* {
    reverse_proxy n8n:5678
  }

  # Proxy to Grafana monitoring
  route /monitoring/* {
    reverse_proxy grafana:3000
  }
}

# API subdomain (optional)
api.your-domain.com {
  reverse_proxy api:8000
}
```

**If you don't have a domain yet:**

Replace the configuration with:

```
http://localhost:80 {
  reverse_proxy frontend:3000
  reverse_proxy /api/* api:8000
  reverse_proxy /n8n/* n8n:5678
}
```

You can set up SSL later when you have a domain.

---

## Step 5: Start Docker Services

Now we'll bring up all services in the correct order:

```bash
# Navigate to project directory
cd /opt/osint

# Start all services
docker compose up -d

# Wait 30 seconds for services to initialize
sleep 30

# Check service status
docker compose ps
```

**Expected Output:**
```
NAME                      STATUS        PORTS
osint-listener            Up 2 minutes
osint-processor-worker    Up 2 minutes
osint-api                 Up 2 minutes   0.0.0.0:8000->8000/tcp
osint-postgres            Up 2 minutes   5432/tcp
osint-redis               Up 2 minutes   6379/tcp
osint-minio               Up 2 minutes   9000/tcp
caddy                     Up 2 minutes   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
```

All services should show "Up".

### Check Service Logs

```bash
# View listener logs
docker compose logs -f listener | head -20

# View processor logs
docker compose logs processor | head -20

# View API logs
docker compose logs api | head -20
```

You should see initialization messages, no errors.

---

## Step 6: First-Time Telegram Authentication

The listener service needs to authenticate with Telegram. On first run, it will wait for authentication.

### 6.1: Start Interactive Authentication

```bash
# Stop the listener
docker compose stop listener

# Start listener in interactive mode
docker compose run --rm listener python -c "
from src.main import setup_telegram_client
client = setup_telegram_client()
print('Authentication successful!')
"
```

### 6.2: Follow Authentication Prompts

```
Please enter your phone number (with country code): +1234567890
Waiting for authentication code...
```

1. You'll receive a code in your Telegram app
2. Enter the code when prompted:
   ```
   Code: 12345
   ```

3. If you have 2FA enabled:
   ```
   Enter 2FA password: your_password_here
   ```

4. Once successful:
   ```
   Authentication successful!
   ```

### 6.3: Restart All Services

```bash
# Restart listener
docker compose up -d listener

# Wait for initialization
sleep 10

# Verify listener is running
docker compose logs listener | tail -5
```

**Expected Output:**
```
INFO - Telegram client initialized
INFO - Discovery cycle starting
INFO - Found 0 Telegram folders
INFO - Channel discovery complete
```

!!! warning "Session Persistence"
    The Telegram session is saved in `data/telegram_sessions/`. This persists across restarts, so you only authenticate once.

---

## Step 7: Add Your First Channel

Now let's add a channel to monitoring. This works the same way as local setup:

1. Open Telegram app (mobile or desktop)
2. Create a folder called `Archive` (or `Archive-UA` for Ukraine sources)
3. Find a public channel and add it to the folder
4. Wait 5 minutes for discovery (or restart listener)

Check if it was discovered:

```bash
# Check listener logs
docker compose logs listener | grep -i "discover\|channel"

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
# Check API health
curl -s http://localhost:8000/health | jq .

# Or on production domain
curl -s https://your-domain.com/api/health | jq .
```

**Expected Output:**
```json
{
  "status": "healthy",
  "version": "1.0",
  "timestamp": "2025-12-09T15:30:45Z"
}
```

### 8.2: List Monitored Channels

```bash
# Get channels via API
curl -s http://localhost:8000/api/channels | jq '.channels[0:3]'
```

**Expected Output:**
```json
{
  "id": 1,
  "name": "Test Channel",
  "username": "@test_channel",
  "folder": "Archive"
}
```

### 8.3: Send a Test Message

Send a test message to your monitored channel, then query for it:

```bash
# Wait 10 seconds for processing
sleep 10

# Count messages from last hour
docker compose exec -T postgres psql -U osint_user -d osint_platform -c \
  "SELECT COUNT(*) FROM messages WHERE created_at > NOW() - INTERVAL '1 hour';"
```

If your test message appeared, the pipeline is working!

---

## Step 9: Access the Frontend

Your platform is now accessible via the domain (or localhost):

```
http://localhost:3000              (local)
https://your-domain.com            (production)
```

Visit it in your browser. You should see the OSINT platform homepage.

---

## Step 10: Set Up Monitoring with Grafana

Grafana provides dashboards for monitoring platform health.

### 10.1: Access Grafana

```
http://localhost:3000/monitoring   (if using /monitoring path)
or
http://grafana-server:3000         (direct access)
```

Default credentials: `admin` / `admin`

### 10.2: Import Dashboards

Pre-configured dashboards are included:

1. Go to **Dashboards** → **Import**
2. Upload dashboards from `/infrastructure/grafana/dashboards/`:
   - `platform-overview.json`
   - `postgres-detailed.json`
   - `redis-detailed.json`
   - `application-metrics.json`

### 10.3: Set Up Alerts (Optional)

For production, set up alerts to notify you of issues:

```bash
# Configure ntfy notifications
# ntfy sends alerts to https://ntfy.sh/your-topic

# Or use your own ntfy server
NTFY_SERVER=https://ntfy.your-domain.com
NTFY_TOPIC=osint-alerts
```

---

## Step 11: Configure Backups

Production deployments should have regular backups.

### 11.1: Database Backups

```bash
# Create backup directory
mkdir -p /opt/osint/backups

# Create backup script
cat > /opt/osint/backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/opt/osint/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Backup PostgreSQL
docker compose exec -T postgres pg_dump -U osint_user osint_platform \
  | gzip > "${BACKUP_DIR}/postgres_${DATE}.sql.gz"

# Backup MinIO data (optional)
# docker compose exec -T minio mc mirror alias/bucket local-backup/

echo "Backup completed: ${BACKUP_DIR}/postgres_${DATE}.sql.gz"
EOF

# Make script executable
chmod +x /opt/osint/backup.sh

# Test backup
/opt/osint/backup.sh
```

### 11.2: Automate Backups with Cron

```bash
# Edit crontab
crontab -e

# Add this line to run backup daily at 2 AM:
0 2 * * * /opt/osint/backup.sh >> /var/log/osint_backup.log 2>&1
```

---

## Step 12: Production Checklist

Before considering your deployment complete, verify:

- [ ] All Docker services running (`docker compose ps`)
- [ ] API responding to requests (`curl http://localhost:8000/health`)
- [ ] At least one Telegram channel monitored
- [ ] Messages being archived to database
- [ ] SSL working (if domain configured)
- [ ] Monitoring dashboard accessible
- [ ] Backups running successfully
- [ ] Firewall rules in place

---

## Updating to Latest Version

To update the platform after initial deployment:

```bash
cd /opt/osint

# Pull latest changes
git pull origin master

# Restart services
docker compose down
docker compose up -d

# Check logs
docker compose logs -f api | head -20
```

---

## Troubleshooting

### "Listener can't authenticate with Telegram"

```bash
# Delete corrupted session
rm -f data/telegram_sessions/*.session

# Re-authenticate
docker compose stop listener
docker compose run --rm listener python -c "
from src.main import setup_telegram_client
client = setup_telegram_client()
print('Authentication successful!')
"
```

### "Database won't start / connection refused"

```bash
# Check database logs
docker compose logs postgres | tail -20

# Reinitialize database
docker compose down
docker volume rm osint-intelligence-platform_postgres_data
docker compose up -d postgres

# Wait 30 seconds
sleep 30

# Check it's up
docker compose logs postgres | tail -5
```

### "Services crashing repeatedly"

```bash
# Check logs for specific service
docker compose logs processor | tail -50

# If out of memory, increase Docker memory limit or reduce workers
# In docker-compose.yml, reduce PROCESSOR_WORKER_COUNT

# Restart with logs
docker compose restart processor
docker compose logs -f processor
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
2. **Set Up Team Access** - Share API URLs with your team
3. **Create RSS Feeds** - Follow the [Create Custom RSS Feed](create-custom-rss-feed.md) tutorial
4. **Set Up Discord Alerts** - Follow the [Setup Discord Alerts](setup-discord-alerts.md) tutorial
5. **Scale as Needed** - Add more channels, adjust resources
6. **Plan Maintenance** - Schedule regular updates and backups

---

## Cost Optimization Tips

- **Reduce processor workers:** Change `PROCESSOR_WORKER_COUNT=1` for low-traffic
- **Enable disk caching:** Use Redis caching for frequently accessed data
- **Archive old messages:** Move messages older than 1 year to cold storage
- **Selective archiving:** Use Monitor folders to archive only high-importance messages
- **Optimize images:** Compress media files before archiving

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
