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
- [ ] **Firewall configured** - Block direct access to PostgreSQL, Redis, MinIO
- [ ] **CrowdSec deployed** - Intrusion prevention enabled (production)
- [ ] **Docker security** - Containers run as non-root users
- [ ] **Backup encryption** - Database backups are GPG-encrypted
- [ ] **Monitoring enabled** - Prometheus + Grafana monitoring security events

### Environment Variable Security

- [ ] `POSTGRES_PASSWORD` - Random 32+ character password
- [ ] `REDIS_PASSWORD` - Random 32+ character password
- [ ] `JWT_SECRET_KEY` - Random 256-bit key (`openssl rand -hex 32`)
- [ ] `KRATOS_SECRET_COOKIE` - Random base64 secret
- [ ] `KRATOS_SECRET_CIPHER` - Exactly 32 characters
- [ ] `MINIO_ACCESS_KEY` - Strong random string
- [ ] `MINIO_SECRET_KEY` - Minimum 32 characters
- [ ] `SECRET_KEY` - Random 256-bit key for general encryption

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
- [ ] **Non-root users** - All containers run as unprivileged users
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

# Block direct database access from internet
sudo ufw deny 5432/tcp  # PostgreSQL
sudo ufw deny 6379/tcp  # Redis
sudo ufw deny 9000/tcp  # MinIO API
sudo ufw deny 9001/tcp  # MinIO Console

# Enable firewall
sudo ufw enable

# Verify rules
sudo ufw status verbose
```

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

# Block database ports from internet
sudo iptables -A INPUT -p tcp --dport 5432 -j DROP  # PostgreSQL
sudo iptables -A INPUT -p tcp --dport 6379 -j DROP  # Redis
sudo iptables -A INPUT -p tcp --dport 9000 -j DROP  # MinIO

# Save rules
sudo apt-get install iptables-persistent
sudo netfilter-persistent save
```

### Network Segmentation

**Docker Networks:**

The platform uses isolated Docker networks:

```yaml
# docker-compose.yml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
```

**Network Isolation:**

- **Frontend network** - Public-facing services (Caddy, Frontend)
- **Backend network** - Internal services (PostgreSQL, Redis, API)
- **Monitoring network** - Prometheus, Grafana (isolated)

**Service Placement:**

```yaml
# Caddy - both networks (reverse proxy)
networks:
  - frontend
  - backend

# API - backend only
networks:
  - backend

# PostgreSQL - backend only
networks:
  - backend
```

### Port Exposure

**Production Port Mapping:**

Only expose essential ports to the internet:

```yaml
# EXPOSE TO INTERNET
ports:
  - "80:80"      # HTTP (redirects to HTTPS)
  - "443:443"    # HTTPS

# DO NOT EXPOSE TO INTERNET
# Remove these from production docker-compose:
#   - "5432:5432"  # PostgreSQL
#   - "6379:6379"  # Redis
#   - "9000:9000"  # MinIO API
#   - "8000:8000"  # API (use Caddy reverse proxy)
#   - "3000:3000"  # Frontend (use Caddy reverse proxy)
```

**Access Internal Services:**

Use SSH tunneling or VPN to access internal services:

```bash
# SSH tunnel to PostgreSQL
ssh -L 5432:localhost:5432 user@your-server

# SSH tunnel to Grafana
ssh -L 3001:localhost:3001 user@your-server
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

**Caddyfile Example:**

```caddyfile
{
    email admin@osint.example.com
}

osint.example.com {
    # Automatic HTTPS
    tls {
        protocols tls1.2 tls1.3
    }

    # Security headers
    header {
        Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "SAMEORIGIN"
        X-XSS-Protection "1; mode=block"
        Referrer-Policy "strict-origin-when-cross-origin"
    }

    # Reverse proxy to frontend
    reverse_proxy frontend:3000
}
```

### Manual TLS Certificates

**Using Commercial Certificates:**

```bash
# Place certificates in infrastructure/caddy/certs/
mkdir -p infrastructure/caddy/certs
cp your-domain.crt infrastructure/caddy/certs/
cp your-domain.key infrastructure/caddy/certs/

# Update Caddyfile
osint.example.com {
    tls /etc/caddy/certs/your-domain.crt /etc/caddy/certs/your-domain.key
}
```

### TLS Best Practices

**Strong Cipher Suites:**

```caddyfile
tls {
    protocols tls1.2 tls1.3
    ciphers TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 \
            TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 \
            TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256 \
            TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
}
```

**HSTS Configuration:**

```caddyfile
header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
```

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

# General secret key
SECRET_KEY=$(openssl rand -hex 32)
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
SECRET_KEY=$SECRET_KEY
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

### Docker Secrets (Production)

For enhanced security, use Docker secrets instead of environment variables:

```bash
# Create secret files
echo "your-postgres-password" | docker secret create postgres_password -
echo "your-redis-password" | docker secret create redis_password -

# Update docker-compose.yml
services:
  postgres:
    secrets:
      - postgres_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password

secrets:
  postgres_password:
    external: true
  redis_password:
    external: true
```

### Secrets Rotation

**Rotate Credentials Quarterly:**

```bash
# 1. Generate new password
NEW_PASSWORD=$(openssl rand -base64 32)

# 2. Update database password
docker-compose exec postgres psql -U postgres -c "ALTER USER osint_user PASSWORD '$NEW_PASSWORD';"

# 3. Update .env
sed -i "s/POSTGRES_PASSWORD=.*/POSTGRES_PASSWORD=$NEW_PASSWORD/" .env

# 4. Restart services
docker-compose restart api processor-worker enrichment-*
```

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

```dockerfile
# Create unprivileged user
RUN useradd --system --create-home --shell /bin/bash osint

# Switch to non-root user
USER osint

# Set working directory
WORKDIR /home/osint
```

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
    # Require password authentication
    POSTGRES_HOST_AUTH_METHOD: md5
  command:
    - postgres
    - -c
    - ssl=on
    - -c
    - ssl_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
    - -c
    - ssl_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
```

**User Permissions:**

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

**Encrypted Backups with GPG:**

```bash
#!/bin/bash
# backup-encrypted.sh

# Configuration
BACKUP_DIR="/backups"
DATE=$(date +%Y-%m-%d)
GPG_RECIPIENT="admin@osint.example.com"

# Create backup
docker-compose exec -T postgres pg_dump \
  -U osint_user \
  -d osint_platform \
  --format=custom \
  | gpg --encrypt --recipient "$GPG_RECIPIENT" \
  > "$BACKUP_DIR/osint-platform-$DATE.pgdump.gpg"

# Verify backup
gpg --list-packets "$BACKUP_DIR/osint-platform-$DATE.pgdump.gpg"

# Retention (keep 30 days)
find "$BACKUP_DIR" -name "*.pgdump.gpg" -mtime +30 -delete

echo "Encrypted backup created: $BACKUP_DIR/osint-platform-$DATE.pgdump.gpg"
```

**Restore Encrypted Backup:**

```bash
# Decrypt and restore
gpg --decrypt /backups/osint-platform-2025-01-01.pgdump.gpg \
  | docker-compose exec -T postgres pg_restore \
    -U osint_user \
    -d osint_platform \
    --clean \
    --if-exists
```

**Automated Backups:**

```bash
# /etc/cron.d/osint-backup
0 2 * * * root /usr/local/bin/backup-encrypted.sh >> /var/log/osint-backup.log 2>&1
```

## Media Storage Security (MinIO/S3)

### Overview

The platform stores archived media files (photos, videos, documents) in MinIO, an S3-compatible object storage. Proper security configuration prevents unauthorized access to sensitive media.

### Pre-Signed URL Security

Pre-signed URLs provide secure, time-limited access to media files without exposing MinIO directly.

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

**Docker Compose Configuration:**

```yaml
services:
  minio:
    # ❌ WRONG - exposes to internet
    # ports:
    #   - "9000:9000"

    # ✅ CORRECT - internal only
    expose:
      - "9000"
    networks:
      - backend  # Internal network only
```

**Caddy Proxy Configuration:**

```caddyfile
# Media endpoint with rate limiting
handle /media/* {
    # Rate limit: 60 requests per minute per IP
    rate_limit {
        zone media_zone {
            key {remote_host}
            events 60
            window 1m
        }
    }

    # Proxy to internal MinIO
    reverse_proxy minio:9000 {
        header_up Host {upstream_hostport}
    }
}
```

### Bucket Access Control

**Default Bucket Policy (Private):**

```bash
# Create bucket with private access
docker-compose exec minio mc mb local/osint-media

# Verify bucket is private (no public access)
docker-compose exec minio mc policy get local/osint-media
# Should output: none
```

**Never Use Public Buckets:**

```bash
# ❌ NEVER do this in production
mc policy set public local/osint-media

# ✅ CORRECT - keep private, use pre-signed URLs
mc policy set none local/osint-media
```

### MinIO Credentials

**Strong Credentials:**

```bash
# Generate strong MinIO credentials
MINIO_ACCESS_KEY=$(openssl rand -hex 16)
MINIO_SECRET_KEY=$(openssl rand -base64 48)

# Add to .env
echo "MINIO_ACCESS_KEY=$MINIO_ACCESS_KEY" >> .env
echo "MINIO_SECRET_KEY=$MINIO_SECRET_KEY" >> .env
```

**Credential Rotation:**

```bash
# 1. Generate new credentials
NEW_ACCESS_KEY=$(openssl rand -hex 16)
NEW_SECRET_KEY=$(openssl rand -base64 48)

# 2. Add new credentials to MinIO
docker-compose exec minio mc admin user add local $NEW_ACCESS_KEY $NEW_SECRET_KEY

# 3. Update services to use new credentials
sed -i "s/MINIO_ACCESS_KEY=.*/MINIO_ACCESS_KEY=$NEW_ACCESS_KEY/" .env
sed -i "s/MINIO_SECRET_KEY=.*/MINIO_SECRET_KEY=$NEW_SECRET_KEY/" .env

# 4. Restart services
docker-compose restart api processor-worker

# 5. Remove old credentials from MinIO
docker-compose exec minio mc admin user remove local $OLD_ACCESS_KEY
```

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
- [ ] **Bucket is private** - No public access policy
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

# View failed login attempts
docker-compose logs api | grep "401 Unauthorized"
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
# View PostgreSQL connection logs
docker-compose exec postgres cat /var/lib/postgresql/data/log/postgresql-*.log | grep "connection"
```

### Prometheus Alerts

**Security Alert Rules:**

Create `infrastructure/prometheus/rules/security.yml`:

```yaml
groups:
  - name: security_alerts
    interval: 1m
    rules:
      - alert: HighFailedLoginRate
        expr: rate(http_requests_total{status="401",endpoint="/api/auth/login"}[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High rate of failed login attempts"
          description: "{{ $value }} failed logins per second"

      - alert: UnauthorizedAdminAccess
        expr: rate(http_requests_total{status="403",path=~"/api/admin.*"}[5m]) > 5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Unauthorized admin access attempts"

      - alert: DatabaseConnectionSpike
        expr: pg_stat_activity_count > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Unusual number of database connections"
```

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

# Disable compromised user
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "UPDATE users SET is_active = false WHERE email = 'compromised@example.com';"
```

**3. Investigate:**

```bash
# Review access logs
docker-compose logs api | grep <suspicious-ip>

# Check database for unauthorized changes
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT * FROM audit_log WHERE user_id = <compromised-user-id> ORDER BY created_at DESC LIMIT 100;"
```

**4. Recover:**

```bash
# Restore from backup if needed
./scripts/restore-backup.sh /backups/osint-platform-2025-01-01.pgdump.gpg

# Rotate all credentials
./scripts/rotate-secrets.sh
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
