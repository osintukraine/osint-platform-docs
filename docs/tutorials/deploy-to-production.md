# Tutorial: Deploy to Production

Complete guide for deploying the OSINT Intelligence Platform to production.

## Learning Objectives

By the end of this tutorial, you will:

- Deploy the platform to a production server
- Configure SSL/TLS encryption
- Set up monitoring and alerting
- Configure backups
- Implement security hardening
- Verify production readiness

## Prerequisites

- Linux server (Ubuntu 22.04+ or Fedora 38+ recommended)
- Root or sudo access
- Domain name (optional but recommended)
- SSL certificate (Let's Encrypt recommended)
- At least 16GB RAM and 500GB disk

## Estimated Time

2-4 hours

## Step 1: Prepare Production Server

**TODO: Content to be generated from codebase analysis**

### Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo apt install docker-compose-plugin -y

# Add user to docker group
sudo usermod -aG docker $USER
```

### Configure Firewall

**TODO: Add firewall configuration:**

```bash
# TODO: Add ufw/iptables rules
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable
```

## Step 2: Clone Repository

**TODO: Add deployment-specific instructions:**

```bash
# Clone repository
git clone https://github.com/osintukraine/osint-intelligence-platform.git
cd osint-intelligence-platform

# Checkout master branch (production)
git checkout master
```

## Step 3: Configure Environment Variables

**TODO: Document production environment configuration:**

```bash
# Copy example environment file
cp .env.example .env

# Edit production configuration
nano .env
```

### Critical Environment Variables

**TODO: List production-critical variables:**

- Database passwords (strong, random)
- API secrets
- Domain name
- Email configuration
- Backup configuration

## Step 4: Configure SSL/TLS

**TODO: Document SSL setup with Let's Encrypt:**

### Install Certbot

```bash
# TODO: Add certbot installation
sudo apt install certbot python3-certbot-nginx -y
```

### Obtain Certificate

```bash
# TODO: Add certificate generation
sudo certbot certonly --standalone -d your-domain.com
```

### Configure Nginx

**TODO: Add Nginx SSL configuration**

## Step 5: Initialize Database

**TODO: Document database initialization:**

```bash
# Start database
docker-compose up -d postgres

# Verify initialization
docker-compose logs postgres

# Check tables were created
docker-compose exec postgres psql -U osint_user -d osint_platform -c "\dt"
```

## Step 6: Start Services

**TODO: Document service startup order:**

```bash
# Start infrastructure services
docker-compose up -d postgres redis minio

# Wait for health checks
sleep 10

# Start application services
docker-compose up -d

# Scale workers
docker-compose up -d --scale processor-worker=4
docker-compose up -d --scale enrichment-worker=2
```

## Step 7: Configure Telegram Session

**TODO: Document Telegram session setup:**

```bash
# Create session interactively
docker-compose run --rm listener python -m create_session

# Verify session
docker-compose logs listener
```

## Step 8: Set Up Monitoring

**TODO: Document monitoring stack configuration:**

### Access Monitoring Tools

- Grafana: https://your-domain.com:3001
- Prometheus: https://your-domain.com:9090

### Import Dashboards

**TODO: Add dashboard import instructions**

### Configure Alerts

**TODO: Add alerting configuration**

## Step 9: Configure Backups

**TODO: Document backup configuration:**

### Database Backups

```bash
# TODO: Add backup script
# Create backup cron job
crontab -e

# Add daily backup at 2 AM
0 2 * * * /path/to/backup-script.sh
```

### Media Backups

**TODO: Add MinIO backup configuration**

## Step 10: Security Hardening

**TODO: Reference security guide sections:**

- [ ] Enable CrowdSec
- [ ] Configure fail2ban
- [ ] Disable unnecessary ports
- [ ] Enable audit logging
- [ ] Configure RBAC
- [ ] Review secret management

## Step 11: Performance Tuning

**TODO: Document performance optimization:**

### Database Tuning

```sql
-- TODO: Add PostgreSQL tuning recommendations
-- Adjust shared_buffers
-- Configure work_mem
-- Optimize maintenance_work_mem
```

### Worker Scaling

**TODO: Document how to determine optimal worker count**

## Step 12: Verify Deployment

**TODO: Add comprehensive verification checklist:**

### Service Health Checks

```bash
# Check all services are running
docker-compose ps

# Check service health
curl https://your-domain.com/health
```

### Functionality Tests

- [ ] Frontend accessible
- [ ] Login works
- [ ] Search functionality
- [ ] Message archival working
- [ ] Enrichment running
- [ ] RSS feeds working
- [ ] Notifications working

## Step 13: Configure DNS

**TODO: Document DNS configuration:**

```
# A record
your-domain.com    → your-server-ip

# Optional CNAME
www                → your-domain.com
```

## Step 14: Set Up Monitoring Alerts

**TODO: Document alert configuration:**

### Critical Alerts

- Service down
- Disk space < 10%
- Database connection failures
- High error rate

### Warning Alerts

- CPU > 80%
- Memory > 90%
- Queue backlog growing

## Step 15: Create Admin User

**TODO: Document admin user creation:**

```bash
# TODO: Add user creation command
docker-compose exec api python -m create_admin_user
```

## Post-Deployment Tasks

**TODO: Document ongoing maintenance:**

### Daily Tasks

- Check service health
- Review error logs
- Monitor disk space

### Weekly Tasks

- Review backup success
- Check for security updates
- Analyze performance metrics

### Monthly Tasks

- Rotate logs
- Review user access
- Update dependencies
- Test disaster recovery

## Production Checklist

**TODO: Comprehensive production readiness checklist:**

### Infrastructure

- [ ] SSL/TLS configured
- [ ] Firewall configured
- [ ] DNS configured
- [ ] Reverse proxy configured
- [ ] Monitoring enabled

### Security

- [ ] Strong passwords set
- [ ] Secrets not in git
- [ ] CrowdSec enabled
- [ ] Audit logging enabled
- [ ] RBAC configured

### Operations

- [ ] Backups configured
- [ ] Backup restoration tested
- [ ] Monitoring alerts configured
- [ ] Documentation updated
- [ ] Runbook created

### Application

- [ ] All services healthy
- [ ] Database initialized
- [ ] Telegram session active
- [ ] Message archival working
- [ ] Search working
- [ ] Admin user created

## Troubleshooting Production Issues

**TODO: Common production issues:**

### Services Won't Start

**TODO: Add diagnostics**

### Performance Issues

**TODO: Add performance troubleshooting**

### SSL/TLS Issues

**TODO: Add SSL troubleshooting**

## Rollback Procedure

**TODO: Document rollback process:**

```bash
# Backup current state
# ...

# Checkout previous version
git checkout <previous-tag>

# Restart services
docker-compose down
docker-compose up -d
```

## Next Steps

After deployment:

- Monitor system for 24-48 hours
- Add more channels to monitoring
- Configure additional integrations
- Train users on the platform
- Set up regular maintenance schedule

---

!!! danger "Critical"
    Always test the disaster recovery procedure before going live! A backup that hasn't been tested is not a real backup.

!!! warning "Security"
    Never expose database or Redis ports directly to the internet. Always use SSL/TLS for production.

!!! note "Documentation Status"
    This page is a placeholder. Content will be generated from deployment scripts and production documentation.
