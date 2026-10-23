# Operator Guide

Complete guide for installing, configuring, monitoring, and maintaining the OSINT Intelligence Platform.

## Overview

This guide covers everything needed to deploy and operate the platform in production environments. It is designed for system administrators, DevOps engineers, and platform operators responsible for maintaining infrastructure and services.

## What You'll Learn

- [Installation](installation.md) - Deploy the platform using Docker Compose
- [Configuration](configuration.md) - Configure services and environment variables
- [Telegram Setup](telegram-setup.md) - Configure Telegram monitoring and session management
- [Hetzner Storage](hetzner-storage.md) - Offsite storage redundancy with Hetzner Storage Box
- [Monitoring & Metrics](monitoring.md) - Monitor health, performance, and metrics
- [Scaling](scaling.md) - Scale workers and services as demand grows
- [Performance Tuning](performance-tuning.md) - Optimize Ollama, PostgreSQL, Redis
- [LLM Gateway](external-llm.md) - GPU gateway, lanes, prompt versions, Ollama alternative
- [Backlog Re-classification](backlog-reclassification.md) - Fill in pending topics and re-check spam with the LLM
- [Media Lanes](media-lanes.md) - Background media download and Telegram embeds
- [Embedding Models](model-provisioning.md) - Provision models with `fetch-models.sh`
- [Upgrades](upgrades.md) - Safely upgrade the platform and handle schema changes
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
- Command line proficiency (bash, docker compose, PostgreSQL)
- Time commitment: 2-4 hours for initial setup, 1-2 hours/week for maintenance

## Architecture Overview

The platform consists of containerized services organized into functional layers:

```
┌─────────────────────────────────────────┐
│  MONITORING (Grafana, Prometheus, Loki) │
└─────────────────────────────────────────┘
             ↑ metrics
┌─────────────────────────────────────────┐
│  APPLICATION (Listener, Processor, API) │
└─────────────────────────────────────────┘
             ↓ ↑
┌─────────────────────────────────────────┐
│  DATA (PostgreSQL, Redis, MinIO)        │
└─────────────────────────────────────────┘
             ↓ ↑
┌─────────────────────────────────────────┐
│  LLM (GPU gateway, or local Ollama)     │
└─────────────────────────────────────────┘
```

The service list, profiles and ports are maintained in one place: [Docker Services](../reference/docker-services.md). In short:

- **Always on**: postgres, redis, minio, ollama (search embeddings), listener, processor-worker, media-worker, media-sync, api, frontend, rss-ingestor.
- **Profiles** add enrichment workers (`enrichment-lite`, `-standard`, `-full`), the LLM gateway workers (`llm-gateway`), monitoring, authentication, OpenSanctions and tools.
- Configuration layers (`.env` and admin settings) are explained in [Configuration](configuration.md#three-tier-configuration).

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
- **Models**: embedding models in `./data/models` (see [Embedding Models](model-provisioning.md)); Ollama models, if used, ~10-15GB

## Cost Structure

### Self-Hosted VPS (Recommended)

| Component | Monthly Cost |
|-----------|-------------|
| VPS (16GB RAM, 8 cores) | €40-80 |
| Storage (2TB SSD) | €20-40 |
| Network (unlimited) | Included |
| LLM (local Ollama; a GPU gateway is separate hardware) | €0 on the VM |
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
4. Check service health: `docker compose ps`

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
docker compose ps

# Check disk space
df -h

# Check recent errors
docker compose logs --tail=50 | grep -i error

# Check API health
curl http://localhost:8000/health

# Check message processing (last hour)
docker compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(*) FROM messages WHERE created_at > NOW() - INTERVAL '1 hour';"
```

### Common Management Tasks

```bash
# View logs for specific service
docker compose logs -f --tail=100 listener

# Restart a service
docker compose restart processor-worker

# Scale processor workers
docker compose up -d --scale processor-worker=4

# Access database
docker compose exec postgres psql -U osint_user -d osint_platform

# Check Redis queue depth
docker compose exec redis redis-cli XLEN telegram_messages

# Update and restart all services
git pull origin master
docker compose up -d --build
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

-   :material-harddisk:{ .lg .middle } __Hetzner Storage__

    ---

    Offsite storage redundancy with Hetzner Storage Box

    [:octicons-arrow-right-24: Storage Guide](hetzner-storage.md)

-   :material-chart-line:{ .lg .middle } __Monitoring__

    ---

    Monitor health, performance, and metrics

    [:octicons-arrow-right-24: Monitoring Guide](monitoring.md)

-   :material-arrow-expand-all:{ .lg .middle } __Scaling__

    ---

    Scale workers and services as demand grows

    [:octicons-arrow-right-24: Scaling Guide](scaling.md)

-   :material-speedometer:{ .lg .middle } __Performance Tuning__

    ---

    Optimize Ollama, PostgreSQL, and Redis

    [:octicons-arrow-right-24: Performance Guide](performance-tuning.md)

-   :material-cloud-outline:{ .lg .middle } __LLM Gateway__

    ---

    GPU gateway lanes, keys and prompt versions, or local Ollama

    [:octicons-arrow-right-24: LLM Gateway Guide](external-llm.md)

-   :material-update:{ .lg .middle } __Upgrades__

    ---

    Safely upgrade platform and handle schema changes

    [:octicons-arrow-right-24: Upgrades Guide](upgrades.md)

-   :material-backup-restore:{ .lg .middle } __Backup & Restore__

    ---

    Backup strategies and disaster recovery

    [:octicons-arrow-right-24: Backup Guide](backup-restore.md)

-   :material-wrench:{ .lg .middle } __Troubleshooting__

    ---

    Common issues and solutions

    [:octicons-arrow-right-24: Troubleshooting Guide](troubleshooting.md)

</div>
