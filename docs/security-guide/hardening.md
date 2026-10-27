# Security Hardening

Production security best practices and deployment checklist for the OSINT Intelligence Platform.

## Overview

This guide provides comprehensive security hardening recommendations for production deployments. Following these practices protects against common attack vectors and ensures the platform meets enterprise security standards.

## Production Security Checklist

Use this checklist before deploying to production:

### Critical Security Items

- [ ] **Authentication enabled** - `AUTH_PROVIDER=ory` or `jwt` (never "none")
- [ ] **AUTH_REQUIRED=true** - Enforce authentication for all endpoints
- [ ] **HTTPS enabled** - Configure TLS with valid certificates
- [ ] **Strong passwords** - All database and service passwords are random 256-bit secrets
- [ ] **Secrets not in git** - Verify `.env` is gitignored and never committed
- [ ] **Published ports closed** - `docker-compose.yml` publishes PostgreSQL, Redis, MinIO, Kratos admin (4434) and Caddy admin (2019) on all interfaces; Docker bypasses UFW (see Port Exposure)
- [ ] **CrowdSec deployed** - Intrusion prevention enabled (production)
- [ ] **Docker security** - Only the frontend image drops root today; review the others
- [ ] **Backup encryption** - `scripts/backup.sh` output is unencrypted; encrypt before off-site copy
- [ ] **Monitoring enabled** - Prometheus + Grafana monitoring security events

### Environment Variable Security

- [ ] `POSTGRES_PASSWORD` - Random 32+ character password
- [ ] `REDIS_PASSWORD` - must be set (compose requires it) but the shipped Redis service does **not** enforce a password, so Redis must never be reachable from outside
- [ ] `JWT_SECRET_KEY` - Random 256-bit key (`openssl rand -hex 32`)
- [ ] `KRATOS_SECRET_COOKIE` - Random base64 secret
- [ ] `KRATOS_SECRET_CIPHER` - Exactly 32 characters
- [ ] `MINIO_ACCESS_KEY` - Strong random string
- [ ] `MINIO_SECRET_KEY` - Minimum 32 characters
- [ ] `MINIO_ROOT_PASSWORD`, `GRAFANA_ADMIN_PASSWORD`, `NOCODB_ADMIN_PASSWORD` - strong, unique

### Network Security

- [ ] **Firewall rules** - UFW or iptables configured
- [ ] **Port exposure** - Only 80/443 exposed to internet
- [ ] **Internal services** - PostgreSQL/Redis only accessible from Docker network
- [ ] **VPN access** - Admin interfaces accessible via VPN only (optional)
- [ ] **DDoS protection** - Cloudflare or similar CDN (optional)

### TLS/SSL Configuration

- [ ] **TLS certificates** - Let's Encrypt or commercial certificates installed
- [ ] **TLS 1.2+** - Disable TLS 1.0 and 1.1
- [ ] **Strong ciphers** - Modern cipher suites only
- [ ] **HSTS enabled** - HTTP Strict Transport Security configured
- [ ] **Certificate renewal** - Auto-renewal configured

### Container Security

- [ ] **Image scanning** - Vulnerability scanning enabled
- [ ] **Non-root users** - Only `frontend-nextjs` sets `USER`; other service Dockerfiles run as root
- [ ] **Resource limits** - CPU/memory limits configured
- [ ] **Read-only filesystems** - Where applicable
- [ ] **Minimal images** - Using slim/alpine base images

## Network Security

### Firewall Configuration

**UFW (Ubuntu/Debian):**

```bash
# Install UFW
sudo apt-get install ufw

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (important - don't lock yourself out!)
sudo ufw allow ssh
sudo ufw allow 22/tcp

# Allow HTTP/HTTPS for Caddy
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable

# Verify rules
sudo ufw status verbose
```

!!! danger "Docker bypasses UFW"
    Ports published by Compose (`ports:`) are added to iptables by Docker ahead of UFW's rules, so `ufw deny 5432` does **not** protect a published PostgreSQL. Either do not publish the port (bind it to `127.0.0.1`), or filter in the `DOCKER-USER` iptables chain. Verify from a different machine with a port scan.

**iptables (Advanced):**

```bash
# Flush existing rules
sudo iptables -F
sudo iptables -X

# Default policies
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Allow established connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP/HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Docker-published ports are filtered in the DOCKER-USER chain, not INPUT:
sudo iptables -I DOCKER-USER -i eth0 -p tcp -m multiport --dports 5432,6379,9000,9001,4433,4434,4455,4456,2019,3000,8000 -j DROP
# (replace eth0 with your public interface; allow 80/443 only to Caddy)

# Save rules
sudo apt-get install iptables-persistent
sudo netfilter-persistent save
```

### Network Segmentation

**Docker Networks:**

`docker-compose.yml` defines a single `backend` bridge network. `docker-compose.production.yml` adds a `frontend` network that Caddy, CrowdSec and the frontend join; the API is moved to `backend` only and loses its published port.

### Port Exposure

With only `docker-compose.yml`, these services publish ports on **all host interfaces**:

| Ports | Service | Risk |
|-------|---------|------|
| 5432 | PostgreSQL | database access |
| 6379 | Redis | **no password configured** by the shipped service |
| 9000/9001 | MinIO API/console | object store |
| 4433/4434 | Kratos public/**admin** | admin API controls identities |
| 4455/4456 | Oathkeeper proxy/API | |
| 2019 | Caddy admin API (`auth` profile base file) | reconfigure proxy |
| 3000, 8000/8003 | Frontend, API/metrics | bypass Caddy and CrowdSec |
| 11434/11435 | Ollama | unauthenticated LLM |
| 9xxx, 8001-8013 | service metrics | information disclosure |
| 8080 | NocoDB (`tools` profile) | |

Only `ntfy` is bound to `127.0.0.1` by default. For production, reduce this surface with a `docker-compose.override.yml` (Compose 2.24+ supports `!reset`/`!override`):

```yaml
services:
  postgres:
    ports: !override
      - "127.0.0.1:5432:5432"
  redis:
    ports: !override
      - "127.0.0.1:6379:6379"
  minio:
    ports: !override
      - "127.0.0.1:9000:9000"
      - "127.0.0.1:9001:9001"
  kratos:
    ports: !override
      - "127.0.0.1:4433:4433"
      - "127.0.0.1:4434:4434"
  caddy:
    ports: !override
      - "80:80"
      - "443:443"
```

Repeat for every other service whose port you do not need from outside, then confirm with `ss -tlnp` on the host and an external scan.

**Access internal services:** use SSH tunnels or a VPN:

```bash
ssh -L 5432:localhost:5432 user@your-server
ssh -L 3001:localhost:3001 user@your-server   # Grafana (GRAFANA_PORT)
```

## TLS/SSL Configuration

### Let's Encrypt (Caddy)

Caddy automatically obtains TLS certificates from Let's Encrypt:

**Configuration:**

```bash
# Set your domain in .env
DOMAIN=osint.example.com

# Caddy will automatically:
# 1. Obtain TLS certificate from Let's Encrypt
# 2. Renew certificates before expiration
# 3. Redirect HTTP → HTTPS
# 4. Enable HSTS
```

**Shipped configuration:** `infrastructure/caddy/Caddyfile.security` (used by the production overlay) already sets HSTS (`max-age=63072000; includeSubDomains; preload`), `X-Frame-Options`, a Content-Security-Policy, rate limits and the CrowdSec bouncer. Caddy's defaults negotiate TLS 1.2+ with modern ciphers. Review the CSP (it allows `'unsafe-inline'` and `'unsafe-eval'` scripts) if your threat model requires stricter rules.

### Manual TLS Certificates

Caddy can use your own certificate with a `tls /path/cert /path/key` directive in the site block. The shipped compose files do not mount a certificate directory, so add a volume for it in an override file and edit the Caddyfile accordingly.

### TLS Best Practices

Leave Caddy's default cipher and protocol selection unless you have a compliance requirement; restricting ciphers by hand is easy to get wrong.

**Test TLS Configuration:**

```bash
# Test with SSL Labs
https://www.ssllabs.com/ssltest/analyze.html?d=osint.example.com

# Test with testssl.sh
./testssl.sh osint.example.com
```

## Secrets Management

### Environment Variables

**Generate Strong Secrets:**

```bash
# PostgreSQL password (32+ characters)
POSTGRES_PASSWORD=$(openssl rand -base64 32)

# Redis password (32+ characters)
REDIS_PASSWORD=$(openssl rand -base64 32)

# JWT secret (256-bit)
JWT_SECRET_KEY=$(openssl rand -hex 32)

# Kratos secrets
KRATOS_SECRET_COOKIE=$(openssl rand -base64 32)
KRATOS_SECRET_CIPHER=$(openssl rand -base64 24)  # Exactly 32 chars

# MinIO credentials
MINIO_ACCESS_KEY=$(openssl rand -hex 16)
MINIO_SECRET_KEY=$(openssl rand -base64 32)

```

**Store in .env:**

```bash
# Add to .env (gitignored!)
cat >> .env << EOF
POSTGRES_PASSWORD=$POSTGRES_PASSWORD
REDIS_PASSWORD=$REDIS_PASSWORD
JWT_SECRET_KEY=$JWT_SECRET_KEY
KRATOS_SECRET_COOKIE=$KRATOS_SECRET_COOKIE
KRATOS_SECRET_CIPHER=$KRATOS_SECRET_CIPHER
MINIO_ACCESS_KEY=$MINIO_ACCESS_KEY
MINIO_SECRET_KEY=$MINIO_SECRET_KEY
EOF

# Secure .env file permissions
chmod 600 .env
```

**Verify .env is Gitignored:**

```bash
# Check .gitignore
grep "\.env" .gitignore
# Should output: .env

# Verify .env is not tracked
git status --ignored | grep .env
# Should show: .env (ignored)
```

### Docker Secrets

`docker secret` requires Swarm mode and the compose files do not use `*_FILE` variables, so secrets live in `.env` (mode 600, owner-only, excluded from backups sent off-site unencrypted).

### Secrets Rotation

**Rotate credentials quarterly** (example: database password):

```bash
# 1. Change the role password (POSTGRES_USER is the superuser, osint_user by default)
docker compose exec postgres psql -U osint_user -d osint_platform -c "ALTER USER osint_user PASSWORD '<new>';"

# 2. Update POSTGRES_PASSWORD in .env
# 3. Recreate services so they read the new .env (restart does NOT re-read .env)
docker compose up -d
```

Kratos also builds its DSN from `POSTGRES_PASSWORD`; recreate `kratos` too (the `up -d` above does). Do not rotate `KRATOS_SECRET_CIPHER` casually: it invalidates data Kratos has encrypted.

## Container Security

### Image Security

**Use Official Base Images:**

```dockerfile
# ✅ Good - official Python slim image
FROM python:3.11-slim

# ❌ Bad - unverified community image
FROM random-user/python:latest
```

**Scan for Vulnerabilities:**

```bash
# Install Trivy scanner
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -

# Scan Docker images
trivy image python:3.11-slim
trivy image osint-api:latest

# Scan for high/critical vulnerabilities only
trivy image --severity HIGH,CRITICAL osint-api:latest
```

**Regular Image Updates:**

```bash
# Pull latest base images monthly
docker-compose pull

# Rebuild custom images
docker-compose build --no-cache

# Restart services
docker-compose up -d
```

### Runtime Security

**Non-Root Containers:**

Only the frontend image (`services/frontend-nextjs/Dockerfile`) creates and uses an unprivileged user. Other service images run as root inside the container. Adding `USER` to a Dockerfile is straightforward but must be tested per service (bind-mounted `./telegram_sessions`, `./data/models` and log directories need matching ownership).

**Resource Limits:**

```yaml
# docker-compose.yml
services:
  processor-worker:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '0.5'
          memory: 1G
```

**Read-Only Root Filesystem:**

```yaml
services:
  api:
    read_only: true
    tmpfs:
      - /tmp
    volumes:
      - ./logs:/app/logs  # Only logs directory is writable
```

**Drop Capabilities:**

```yaml
services:
  api:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Only if binding to ports <1024
```

## Database Security

### PostgreSQL Hardening

**Connection Security:**

```yaml
# docker-compose.yml
postgres:
  environment:
    POSTGRES_HOST_AUTH_METHOD: scram-sha-256
```

Traffic between containers stays on the Docker network; enable PostgreSQL TLS only if the database is reached across hosts (supply your own certificate files).

**User Permissions:**

All services connect as `POSTGRES_USER` (a superuser created by the image), and migrations need DDL rights. A least-privilege role is possible but must be configured per service and kept out of the migration path:

```sql
-- Create limited user for application
CREATE USER osint_app WITH PASSWORD 'strong-password';

-- Grant only necessary permissions
GRANT CONNECT ON DATABASE osint_platform TO osint_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO osint_app;

-- Revoke dangerous permissions
REVOKE CREATE ON SCHEMA public FROM osint_app;
REVOKE ALL ON ALL FUNCTIONS IN SCHEMA public FROM osint_app;
```

**Connection Pooling:**

```bash
# .env configuration
POSTGRES_POOL_SIZE=20
POSTGRES_MAX_OVERFLOW=10
POSTGRES_POOL_TIMEOUT=30
POSTGRES_POOL_RECYCLE=3600
```

**Audit Logging:**

```sql
-- Enable audit logging
ALTER SYSTEM SET log_statement = 'mod';  # Log all modifications
ALTER SYSTEM SET log_connections = 'on';
ALTER SYSTEM SET log_disconnections = 'on';
SELECT pg_reload_conf();
```

### Backup Encryption

**Encrypted backups:** `./scripts/backup.sh` writes an unencrypted directory `backups/<timestamp>/` containing the PostgreSQL dump, Telegram sessions (full account access), a Redis snapshot and configuration. Encrypt it before it leaves the server:

```bash
tar -C backups -c <timestamp> | gpg --encrypt --recipient admin@example.com > backups/<timestamp>.tar.gpg
```

**Restore:**

```bash
gpg --decrypt backups/<timestamp>.tar.gpg | tar -C backups -x
./scripts/restore.sh backups/<timestamp> --dry-run   # preview first
./scripts/restore.sh backups/<timestamp>
```

`restore.sh` overwrites existing data. Schedule `backup.sh` with cron (see [Backup & Restore](../operator-guide/backup-restore.md)).

## Media Storage Security (MinIO/S3)

### Overview

The platform stores archived media files (photos, videos, documents) in MinIO, an S3-compatible object storage. Proper security configuration prevents unauthorized access to sensitive media.

### Pre-Signed URL Security

Pre-signed URLs provide time-limited access to media files. They are **off by default** (`USE_PRESIGNED_URLS=false`); without them media URLs are plain public URLs (`MINIO_PUBLIC_URL`).

**Benefits:**

- **Time-limited**: URLs expire after configured period (default 4 hours)
- **Cryptographically signed**: Cannot be tampered with or guessed
- **No public bucket**: MinIO doesn't need to be publicly accessible
- **Audit trail**: Can track who requested access

**Configuration:**

Add to your `.env` file:

```bash
# Enable pre-signed URLs (RECOMMENDED for production)
USE_PRESIGNED_URLS=true

# URL expiry in hours (1-24, default 4)
PRESIGNED_URL_EXPIRY_HOURS=4

# Public URL base (browser-accessible, through Caddy proxy)
MINIO_PUBLIC_URL=https://yourdomain.com
```

**How It Works:**

1. Client requests media via API endpoint
2. API generates pre-signed URL with expiry signature
3. Client downloads directly from MinIO with signed URL
4. URL expires after configured period

**Example Pre-Signed URL:**

```
https://yourdomain.com/osint-media/media/ab/cd/abcd1234.jpg?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...&X-Amz-Date=20250127T120000Z&X-Amz-Expires=14400&X-Amz-SignedHeaders=host&X-Amz-Signature=...
```

### MinIO Network Isolation

**CRITICAL**: MinIO should never be directly exposed to the internet.

**Architecture:**

```
Internet → Caddy (443) → /media/* → MinIO (internal:9000)
                        ↓
                    Rate limiting
                    Pre-signed validation
```

**Compose:** the base file publishes MinIO on `9000/9001` (all interfaces). Rebind to `127.0.0.1` or remove the mapping with an override (see Port Exposure); Caddy and the services reach it as `minio:9000` on the `backend` network. Media is served to browsers by Caddy (local buffer / storage routes with an API redirect fallback); see [Caddy](../operator-guide/caddy.md).

### Bucket Access Control

!!! warning "The media bucket is anonymously readable by default"
    `minio-init` runs `mc anonymous set download` on the bucket (`MINIO_BUCKET_NAME`, default `osint-media`). Anyone who can reach MinIO and knows or guesses an object key can download it. Keys are content hashes (`media/ab/cd/<sha256>.ext`), so they are unguessable but not secret. Pre-signed URLs only add expiry if you also make the bucket private.

To make the bucket private (then `USE_PRESIGNED_URLS=true` is required for direct access), change the `minio-init` command in your override file to `mc anonymous set none`, apply it once, and verify media still loads in the UI. Test on a staging copy first, because Caddy's media routes may rely on anonymous reads.

Never expose the MinIO console (9001) publicly; the root credentials (`MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD`) grant full control.

### MinIO Credentials

**Strong credentials:** generate with `openssl rand -hex 16` / `openssl rand -base64 48` and store in `.env` (`MINIO_ROOT_PASSWORD`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`).

**Credential rotation:** the services use `MINIO_ACCESS_KEY`/`MINIO_SECRET_KEY`; MinIO itself uses `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD`. Create the new user via the MinIO console or `mc admin user add` from a temporary `mc` container, update `.env`, run `docker compose up -d` (recreate, not restart) and then remove the old user.

### RSS Feed Media URLs

**Note**: RSS feeds require stable URLs that don't expire. For RSS feeds:

- Pre-signed URLs are **not used** (RSS readers may cache feed URLs)
- Media is accessed through the Caddy proxy
- Rate limiting provides anti-leeching protection

**RSS Media Configuration:**

```bash
# RSS uses the public URL directly (through Caddy)
MINIO_PUBLIC_URL=https://yourdomain.com

# Pre-signed URLs only affect API responses, not RSS
USE_PRESIGNED_URLS=true
```

### Security Checklist for Media Storage

- [ ] **MinIO not exposed to internet** - Internal Docker network only
- [ ] **Pre-signed URLs enabled** - `USE_PRESIGNED_URLS=true`
- [ ] **Bucket policy reviewed** - default is anonymous download (see above)
- [ ] **Strong credentials** - Random 32+ character secrets
- [ ] **Caddy proxy configured** - Rate limiting enabled
- [ ] **HTTPS only** - All media accessed via HTTPS
- [ ] **Credential rotation** - Rotate quarterly

## Monitoring and Auditing

### Security Event Logging

**Authentication Events:**

```bash
# View authentication logs
docker-compose logs -f api | grep -E "(login|logout|auth)"

# View failed login attempts (Kratos flows are in the kratos log)
docker-compose logs api | grep " 401 "
```

**Authorization Failures:**

```bash
# View 403 Forbidden events
docker-compose logs -f api | grep "403 Forbidden"

# View admin access attempts
docker-compose logs api | grep "/api/admin"
```

**Database Access:**

```bash
# View PostgreSQL logs
docker-compose logs postgres | grep -i "connection"
```

### Prometheus Alerts

**Security alert rules:** Prometheus loads `infrastructure/prometheus/alerts.yml` and `infrastructure/prometheus/rules/*.yml` (service down, queue depth, spam rate, Telegram flood waits, resource usage). Add security-specific rules (for example on CrowdSec metrics from port 6060) to a new file under `rules/` and reload Prometheus. Alerts are delivered through Alertmanager, the notifier and ntfy; see [Monitoring](../operator-guide/monitoring.md).

### Log Retention

**Configure Log Rotation:**

```bash
# /etc/logrotate.d/osint-platform
/var/lib/docker/containers/*/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

## Operating System Hardening

### System Updates

**Automatic Security Updates (Ubuntu/Debian):**

```bash
# Install unattended-upgrades
sudo apt-get install unattended-upgrades apt-listchanges

# Configure automatic updates
sudo dpkg-reconfigure -plow unattended-upgrades

# Edit /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```

**Manual Updates:**

```bash
# Update system monthly
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
sudo apt-get autoremove -y
sudo reboot
```

### SSH Hardening

**Disable Password Authentication:**

```bash
# Edit /etc/ssh/sshd_config
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
AllowUsers your-username

# Restart SSH
sudo systemctl restart sshd
```

**Use SSH Keys Only:**

```bash
# Generate SSH key (on your local machine)
ssh-keygen -t ed25519 -C "admin@osint-platform"

# Copy to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@your-server
```

### Fail2Ban

**Install Fail2Ban:**

```bash
# Install
sudo apt-get install fail2ban

# Configure
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# Edit /etc/fail2ban/jail.local
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log

# Restart
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

## Security Checklist for Go-Live

### Pre-Production

- [ ] All passwords changed from defaults
- [ ] `.env` file secured (permissions 600, gitignored)
- [ ] TLS certificates configured (Let's Encrypt or commercial)
- [ ] Firewall rules tested (only 80/443 exposed)
- [ ] Database backups tested (encrypt + restore)
- [ ] Monitoring configured (Prometheus + Grafana)
- [ ] CrowdSec installed and tested
- [ ] SSH hardened (keys only, no root)
- [ ] System updates automated
- [ ] Security scan completed (Trivy or similar)

### Post-Deployment

- [ ] Monitor logs for 48 hours
- [ ] Review CrowdSec decisions (check for false positives)
- [ ] Verify backups are working
- [ ] Test disaster recovery procedure
- [ ] Document security configuration
- [ ] Schedule quarterly security review

### Ongoing Maintenance

- [ ] **Weekly**: Review security logs
- [ ] **Monthly**: System updates, rotate secrets
- [ ] **Quarterly**: Security audit, penetration testing
- [ ] **Annually**: Disaster recovery drill, threat model review

## Incident Response

### Security Incident Procedure

**1. Detect:**

- Monitor Prometheus alerts
- Review CrowdSec ban decisions
- Check authentication logs

**2. Contain:**

```bash
# Block suspicious IP immediately
docker-compose exec crowdsec cscli decisions add \
  --ip <suspicious-ip> \
  --duration 24h \
  --reason "Security incident investigation"

# Revoke a compromised user's sessions and demote/delete them (admin API; user ids come from GET /api/admin/users)
#   DELETE /api/admin/users/<user_id>/sessions
#   PUT    /api/admin/users/<user_id>   {"role": "viewer"}
#   DELETE /api/admin/users/<user_id>
```

**3. Investigate:**

```bash
# Review access logs
docker-compose logs api | grep <suspicious-ip>

# Check database for unauthorized changes
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT * FROM admin_audit_log WHERE kratos_identity_id = '<compromised-identity-uuid>' ORDER BY created_at DESC LIMIT 100;"
```

**4. Recover:**

```bash
# Restore from backup if needed
./scripts/restore.sh backups/<timestamp>

# Rotate all credentials by hand (no helper script exists): .env secrets,
# Kratos cookie secret (invalidates sessions), MinIO and database passwords,
# then recreate services with `docker compose up -d`
```

**5. Document:**

Create incident report with:

- Timeline of events
- Root cause analysis
- Actions taken
- Prevention measures

## Related Documentation

- [Authentication Guide](authentication.md) - User authentication setup
- [Authorization Guide](authorization.md) - Role-based access control
- [CrowdSec Integration](crowdsec.md) - Intrusion prevention
- [Monitoring Guide](../operator-guide/monitoring.md) - Security monitoring

---

!!! danger "Critical Security Reminder"
    Security is an ongoing process, not a one-time configuration. Regularly review logs, update systems, and adapt to new threats.

!!! tip "Start Small, Scale Gradually"
    Implement security measures incrementally. Start with authentication and firewalls, then add CrowdSec and monitoring.

!!! info "Security by Design"
    The OSINT Intelligence Platform is designed with security in mind, but requires proper configuration and ongoing maintenance to remain secure.
