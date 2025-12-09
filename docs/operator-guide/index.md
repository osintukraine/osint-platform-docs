# Operator Guide

Complete guide for installing, configuring, monitoring, and maintaining the OSINT Intelligence Platform.

## Overview

This guide covers everything needed to deploy and operate the platform in production environments. It is designed for system administrators, DevOps engineers, and platform operators responsible for maintaining infrastructure and services.

## What You'll Learn

- [Installation](installation.md) - Deploy the platform using Docker Compose
- [Configuration](configuration.md) - Configure services and environment variables
- [Telegram Setup](telegram-setup.md) - Configure Telegram monitoring and session management
- [Monitoring & Metrics](monitoring.md) - Monitor health, performance, and metrics
- [Backup & Restore](backup-restore.md) - Backup strategies and disaster recovery
- [Troubleshooting](troubleshooting.md) - Common issues and solutions

## Who This Is For

**Platform Operators** are responsible for:

- Installing and configuring the platform
- Managing infrastructure resources (CPU, RAM, storage)
- Monitoring system health and performance
- Performing backups and disaster recovery
- Troubleshooting operational issues
- Scaling services as demand grows
- Applying security updates
- Managing Telegram sessions and channels

**Prerequisites:**

- System administration experience (Linux, Docker, networking)
- Basic security knowledge (API keys, secrets management)
- Command line proficiency (bash, docker-compose, PostgreSQL)
- Time commitment: 2-4 hours for initial setup, 1-2 hours/week for maintenance

## Architecture Overview

The platform consists of 29 containers organized into functional layers:

```
┌─────────────────────────────────────────┐
│  MONITORING (Grafana, Prometheus, ntfy) │
└─────────────────────────────────────────┘
             ↑ metrics
┌─────────────────────────────────────────┐
│  APPLICATION (Listener, Processor, API) │
└─────────────────────────────────────────┘
             ↓ ↑
┌─────────────────────────────────────────┐
│  DATA (PostgreSQL, Redis, MinIO, Ollama)│
└─────────────────────────────────────────┘
```

### Core Services (15 containers)

| Service | Purpose | Port |
|---------|---------|------|
| **listener** | Monitors 254+ Telegram channels in real-time | 9091 |
| **processor-worker** | Spam filtering, routing, entity extraction (2 replicas) | 9092 |
| **enrichment-ai-tagging** | LLM-based tag generation | 9196 |
| **enrichment-rss-validation** | RSS article validation | 9197 |
| **enrichment-router** | Priority-based message routing | 9198 |
| **enrichment-fast-pool** | CPU-bound tasks (embeddings, translation) | 9199 |
| **enrichment-telegram** | Telegram API tasks (polling, comments) | 9200 |
| **enrichment-decision** | Decision verification | 9201 |
| **enrichment-maintenance** | Hourly maintenance tasks | 9202 |
| **api** | REST API endpoints (FastAPI) | 8000 |
| **frontend** | Next.js web UI | 3000 |
| **notifier** | Alert distribution (ntfy, Discord) | 9094 |
| **rss-ingestor** | RSS feed ingestion | - |
| **opensanctions** | Entity enrichment (optional) | - |
| **entity-ingestion** | CSV entity imports (optional) | - |

### Infrastructure Services (8 containers)

| Service | Purpose | Port |
|---------|---------|------|
| **postgres** | PostgreSQL 16 + pgvector | 5432 |
| **redis** | Message queue and caching | 6379 |
| **minio** | S3-compatible media storage | 9000/9001 |
| **ollama** | Self-hosted LLM runtime (realtime) | 11434 |
| **ollama-batch** | Self-hosted LLM runtime (background) | 11435 |
| **yente** | OpenSanctions API (optional) | 8000 |
| **yente-index** | ElasticSearch for entity matching (optional) | 9200 |
| **nocodb** | Database admin UI | 8080 |

### Monitoring Services (8 containers)

| Service | Purpose | Port |
|---------|---------|------|
| **prometheus** | Metrics collection | 9090 |
| **grafana** | Visualization dashboards | 3001 |
| **alertmanager** | Alert routing | 9093 |
| **postgres-exporter** | Database metrics | 9187 |
| **redis-exporter** | Queue metrics | 9121 |
| **cadvisor** | Container resource metrics | 8081 |
| **node-exporter** | Host system metrics | 9100 |
| **ntfy** | Notification delivery | 8090 |
| **dozzle** | Real-time log viewer | 9999 |

## Resource Requirements

### Development/Testing

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 8GB | 16GB |
| CPU | 4 cores | 8 cores |
| Storage | 100GB | 500GB SSD |
| Network | 10 Mbps | 100 Mbps |

### Production

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 16GB | 32GB |
| CPU | 8 cores | 16 cores |
| Storage | 500GB SSD | 2TB NVMe |
| Network | 100 Mbps | 1 Gbps |

### Storage Growth Estimates

- **Database**: ~500MB per 100K messages (text only)
- **Media**: 12-20TB per 3 years (with effective spam filtering)
- **Logs**: ~1-2GB/week (with rotation)
- **Metrics**: ~5GB/month (30-day retention)
- **Ollama models**: ~10-15GB (6 models)

## Cost Structure

### Self-Hosted VPS (Recommended)

| Component | Monthly Cost |
|-----------|-------------|
| VPS (16GB RAM, 8 cores) | €40-80 |
| Storage (2TB SSD) | €20-40 |
| Network (unlimited) | Included |
| LLM (Ollama) | €0 |
| Translation (DeepL Free) | €0 |
| **Total** | **€60-120/month** |

**Compared to legacy system:** €300/month = 50-60% savings

**Cost Optimization Notes:**
- Spam filtering saves 73% storage costs
- Self-hosted LLM saves €50-200/month vs OpenAI
- DeepL Free API saves €20-50/month vs paid translation

### Cloud Deployment (Not Recommended)

The platform is designed for self-hosting. Cloud costs would be:

- Managed PostgreSQL: €100-200/month
- Object storage: €30-100/month
- Compute instances: €150-300/month
- **Estimated total**: €280-600/month (2-5x higher)

## Operational Workflow

### Daily Tasks (5-10 minutes)

1. Check Grafana dashboards for anomalies
2. Review ntfy notifications for critical alerts
3. Monitor disk space usage: `df -h`
4. Check service health: `docker-compose ps`

### Weekly Tasks (30-60 minutes)

1. Review spam filter effectiveness in Grafana
2. Analyze channel discovery metrics
3. Check for Docker image updates
4. Review backup status
5. Optimize database indexes if query performance degrades

### Monthly Tasks (2-4 hours)

1. Perform full database backup (validate restore)
2. Review and rotate logs
3. Update Docker images and platform code
4. Review capacity planning metrics (storage, CPU trends)
5. Test disaster recovery procedures
6. Review security updates

### Quarterly Tasks (4-8 hours)

1. Comprehensive security audit
2. Review and update monitoring alerts
3. Performance benchmarking
4. Capacity planning for next quarter
5. Update documentation

## Quick Start

### 5-Minute Health Check

```bash
# Check all services are running
docker-compose ps

# Check disk space
df -h

# Check recent errors
docker-compose logs --tail=50 | grep -i error

# Check API health
curl http://localhost:8000/health

# Check message processing (last hour)
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(*) FROM messages WHERE created_at > NOW() - INTERVAL '1 hour';"
```

### Common Management Tasks

```bash
# View logs for specific service
docker-compose logs -f --tail=100 listener

# Restart a service
docker-compose restart processor-worker

# Scale processor workers
docker-compose up -d --scale processor-worker=4

# Access database
docker-compose exec postgres psql -U osint_user -d osint_platform

# Check Redis queue depth
docker-compose exec redis redis-cli XLEN telegram_messages

# Update and restart all services
git pull origin master
docker-compose up -d --build
```

## Support Resources

- **Documentation**: [docs.osintukraine.com](https://docs.osintukraine.com)
- **GitHub Repository**: [github.com/osintukraine/osint-intelligence-platform](https://github.com/osintukraine/osint-intelligence-platform)
- **Issues**: [GitHub Issues](https://github.com/osintukraine/osint-intelligence-platform/issues)
- **Community**: [Mastodon @osintukraine](https://mastodon.social/@osintukraine)
- **Technical Guidance**: See `CLAUDE.md` in repository for AI assistant context

## Document Updates

This guide is maintained in the `osint-platform-docs` repository. To suggest improvements:

1. Fork the repository
2. Edit files in `docs/operator-guide/`
3. Submit a pull request with clear description

---

**Next Steps**: Start with [Installation](installation.md) to deploy your first instance.

<div class="grid cards" markdown>

-   :material-download:{ .lg .middle } __Installation__

    ---

    Deploy the platform with Docker Compose

    [:octicons-arrow-right-24: Install Guide](installation.md)

-   :material-cog:{ .lg .middle } __Configuration__

    ---

    Configure services and environment variables

    [:octicons-arrow-right-24: Config Guide](configuration.md)

-   :material-send:{ .lg .middle } __Telegram Setup__

    ---

    Configure Telegram monitoring and sessions

    [:octicons-arrow-right-24: Telegram Guide](telegram-setup.md)

-   :material-chart-line:{ .lg .middle } __Monitoring__

    ---

    Monitor health, performance, and metrics

    [:octicons-arrow-right-24: Monitoring Guide](monitoring.md)

-   :material-backup-restore:{ .lg .middle } __Backup & Restore__

    ---

    Backup strategies and disaster recovery

    [:octicons-arrow-right-24: Backup Guide](backup-restore.md)

-   :material-wrench:{ .lg .middle } __Troubleshooting__

    ---

    Common issues and solutions

    [:octicons-arrow-right-24: Troubleshooting Guide](troubleshooting.md)

</div>
