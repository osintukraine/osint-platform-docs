# Monitoring Guide

**OSINT Intelligence Platform - Production Monitoring & Observability**

Complete guide to monitoring stack health, investigating issues, and optimizing performance using Prometheus, Grafana, Loki, and Dozzle.

---

## Table of Contents

- [Overview](#overview)
- [Monitoring Stack Architecture](#monitoring-stack-architecture)
- [Accessing Monitoring Tools](#accessing-monitoring-tools)
- [Grafana Dashboards](#grafana-dashboards)
- [Key Metrics Reference](#key-metrics-reference)
- [Alert Rules](#alert-rules)
- [Health Check Endpoints](#health-check-endpoints)
- [Log Analysis](#log-analysis)
- [Performance Monitoring](#performance-monitoring)
- [Troubleshooting Monitoring Stack](#troubleshooting-monitoring-stack)

---

## Overview

The OSINT Intelligence Platform includes a comprehensive monitoring solution based on industry-standard tools:

- **Prometheus**: Time-series metrics database (scrapes every 15 seconds, 30-day retention)
- **Grafana**: Visualization dashboards (11 pre-configured dashboards)
- **AlertManager**: Alert routing and notification delivery
- **Loki**: Centralized log aggregation (optional, for advanced deployments)
- **Dozzle**: Real-time container log viewer (lightweight alternative to Loki)
- **Exporters**: PostgreSQL, Redis, MinIO, cAdvisor, node-exporter

**What We Monitor:**

- Message processing pipeline (ingestion rate, spam detection, archival)
- LLM performance (response time, success rate, cost estimation)
- Database health (connections, query performance, cache hit ratio)
- Queue metrics (Redis depth, consumer lag, backpressure)
- Media archival (storage usage, deduplication efficiency)
- API performance (request rate, error rate, latency)
- System resources (CPU, memory, disk, network)

---

## Monitoring Stack Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Services                      │
├──────────┬──────────┬──────────┬──────────┬─────────────────┤
│ Listener │ Processor│   API    │Enrichment│  Infrastructure │
│ :8001    │ :8002    │ :8000    │ :9095    │  (PG,Redis,S3)  │
└────┬─────┴────┬─────┴────┬─────┴────┬─────┴────┬────────────┘
     │          │          │          │          │
     │ /metrics │ /metrics │ /metrics │ /metrics │ Exporters
     │          │          │          │          │ (:9187,:9121)
     └──────────┴──────────┴──────────┴──────────┘
                           │
                           ▼
               ┌───────────────────────┐
               │      Prometheus       │
               │  Metrics Collection   │
               │      :9090            │
               └─────────┬─────────────┘
                         │
                         ├─────────────────┐
                         │                 │
                         ▼                 ▼
               ┌──────────────┐  ┌──────────────────┐
               │  Grafana     │  │  AlertManager    │
               │ Dashboards   │  │  Alert Routing   │
               │    :3001     │  │      :9093       │
               └──────────────┘  └─────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │  ntfy Notifier  │
                                  │  Push Delivery  │
                                  │     :8090       │
                                  └─────────────────┘

          ┌──────────────────────────────────────┐
          │           Log Pipeline               │
          ├────────────────┬─────────────────────┤
          │     Dozzle     │    Loki (Optional)  │
          │  Container Logs│  Centralized Logs   │
          │     :8888      │      :3100          │
          └────────────────┴─────────────────────┘
```

---

## Accessing Monitoring Tools

### Grafana (Primary Dashboard UI)

```bash
# URL
http://localhost:3001

# Default credentials
Username: admin
Password: admin

# IMPORTANT: Change password on first login!
```

**First Login Checklist:**

1. Login with default credentials
2. Navigate to **Profile → Change Password**
3. Set strong password (minimum 12 characters)
4. Navigate to **Dashboards → Browse** to view pre-configured dashboards
5. Verify Prometheus datasource: **Configuration → Data Sources → Prometheus → Test**

### Prometheus (Metrics Query UI)

```bash
# URL
http://localhost:9090

# Access
- No authentication required (internal network only)

# Common tasks
- Query metrics: Click "Graph" → enter PromQL expression
- View targets: Status → Targets
- Check alerts: Alerts → View all configured alerts
- View configuration: Status → Configuration
```

### Dozzle (Real-Time Container Logs)

```bash
# URL
http://localhost:8888

# Access
- No authentication required (internal network only)

# Features
- Real-time log streaming for all containers
- Search and filter logs by container, time range, or keyword
- Multi-container view (compare logs side-by-side)
- No storage overhead (reads directly from Docker daemon)
```

### AlertManager (Alert Management)

```bash
# URL
http://localhost:9093

# Access
- No authentication required (internal network only)

# Features
- View active alerts with severity and status
- Silence alerts temporarily
- View alert history and routing configuration
```

### Metrics Endpoints (Raw Prometheus Format)

```bash
# Application services
curl http://localhost:8001/metrics   # Listener
curl http://localhost:8002/metrics   # Processor Worker 1
curl http://localhost:8003/metrics   # Processor Worker 2
curl http://localhost:8000/metrics   # API
curl http://localhost:9095/metrics   # Enrichment service

# Infrastructure exporters
curl http://localhost:9187/metrics   # PostgreSQL Exporter
curl http://localhost:9121/metrics   # Redis Exporter
curl http://localhost:8080/metrics   # cAdvisor (container stats)
curl http://localhost:9100/metrics   # Node Exporter (host stats)

# Monitoring stack self-monitoring
curl http://localhost:9090/metrics   # Prometheus
curl http://localhost:3001/metrics   # Grafana
```

---

## Grafana Dashboards

The platform includes **11 pre-configured dashboards** located in `/infrastructure/grafana/dashboards/`:

### 1. Platform Overview (`platform-overview.json`)

**Purpose**: Executive dashboard showing overall system health at a glance.

**Key Panels:**

- **Service Status Grid**: Up/down status of all critical services
- **Message Processing Rate**: Real-time throughput (messages/second)
- **Spam Detection Rate**: Percentage of messages flagged as spam
- **OSINT Score Distribution**: Histogram of message relevance scores (0-100)
- **Active Telegram Channels**: Count of channels being monitored
- **Redis Queue Depth**: Messages waiting to be processed
- **Database Size**: PostgreSQL storage usage
- **MinIO Storage**: Media archive capacity

**Refresh Rate**: 30 seconds

**Use Cases:**

- Daily operations monitoring
- Quick health check before deployments
- Executive reporting for stakeholders

### 2. OSINT Pipeline (`osint-pipeline.json`)

**Purpose**: Detailed view of message processing pipeline from ingestion to archival.

**Key Panels:**

- **Pipeline Stages**: Sankey diagram showing message flow
- **Ingestion Rate**: Messages received from Telegram (messages/second)
- **Spam Filter Performance**: Filter effectiveness and false positive rate
- **Rule Coverage**: Percentage of messages matched by OSINT rules
- **LLM Classification**: Success rate and response time
- **Entity Extraction**: Entities extracted per second (by type)
- **Media Archival**: Files archived (photos, videos, documents)
- **Database Writes**: Messages persisted to PostgreSQL

**Refresh Rate**: 15 seconds

**Use Cases:**

- Pipeline performance optimization
- Identifying bottlenecks in processing flow
- Validating spam filter effectiveness

### 3. SLI/SLO Dashboard (`sli-slo.json`)

**Purpose**: Service Level Indicators and Objectives tracking with burn rate alerts.

**Key Panels:**

- **API Availability SLO**: 99.9% uptime target (43.2 minutes/month error budget)
- **Message Processing SLO**: 99.5% success rate
- **API Latency SLO**: 95% of requests < 500ms (p95 target)
- **Media Archival SLO**: 99% success rate
- **Error Budget Consumption**: Remaining budget for current month
- **Burn Rate Alerts**: Fast burn (1h/5m) and slow burn (6h/30m) indicators
- **SLO Compliance History**: 30-day trend of SLO adherence

**Refresh Rate**: 15 seconds

**Use Cases:**

- Reliability engineering and SRE workflows
- Error budget tracking for feature vs stability prioritization
- Production readiness assessments

**Google SRE Multi-Window Burn Rate Alerting:**

- **Fast burn (critical)**: Burn rate > 14.4x = consuming entire monthly error budget in ~2.5 hours
- **Slow burn (warning)**: Burn rate > 6x = sustained degradation over 6 hours

### 4. Enrichment Performance (`enrichment-performance.json`)

**Purpose**: Batch enrichment pipeline monitoring (background tasks).

**Key Panels:**

- **Queue Depth by Task**: Messages pending for each enrichment type
- **Queue Lag**: Time oldest message has been waiting (seconds)
- **Backpressure Ratio**: Queue growth vs processing rate (>1 = falling behind)
- **Task Execution Metrics**: Success rate, duration, throughput
- **LLM Worker Performance**: AI tagging and RSS validation
- **Cycle Duration**: Time to complete one enrichment cycle (target: <5 minutes)
- **Task Stalled Alerts**: Tasks making no progress despite pending work

**Refresh Rate**: 30 seconds

**Use Cases:**

- Enrichment pipeline capacity planning
- Identifying slow or stuck tasks
- Batch processing performance tuning

### 5. Service Logs (`service-logs.json`)

**Purpose**: Centralized log analysis with Loki integration (optional feature).

**Key Panels:**

- **Log Volume by Service**: Log lines per second
- **Error Rate by Service**: Error-level logs per second
- **Recent Errors**: Table of last 100 error messages
- **Log Search**: Full-text search across all services
- **Slow Query Logs**: PostgreSQL queries > 100ms
- **LLM Request Logs**: Ollama request/response details

**Refresh Rate**: 10 seconds

**Note**: Requires Loki deployment. For lightweight alternative, use Dozzle (http://localhost:8888).

### 6. PostgreSQL Dashboard (`postgres.json`)

**Purpose**: Database health and performance monitoring.

**Key Panels:**

- **Active Connections**: Current vs max connections
- **Query Performance**: Average query duration and slow queries (>100ms)
- **Cache Hit Ratio**: Percentage of queries served from memory (target: >90%)
- **Transaction Rate**: Commits and rollbacks per second
- **Database Size**: Storage usage by table
- **Table Bloat**: Tables needing VACUUM
- **Index Usage**: Unused indexes wasting space
- **Deadlocks**: Deadlock detection and frequency
- **Replication Lag**: Lag if using PostgreSQL replication

**Refresh Rate**: 30 seconds

**Use Cases:**

- Database performance tuning
- Identifying slow queries needing optimization
- Capacity planning for storage growth

### 7. Redis Dashboard (`redis.json`)

**Purpose**: Message queue health monitoring.

**Key Panels:**

- **Queue Depth**: Messages in `telegram_messages` stream
- **Consumer Lag**: Processing delay (seconds)
- **Memory Usage**: Current vs max memory (512MB limit)
- **Eviction Rate**: Keys evicted due to memory pressure
- **Connection Count**: Active clients
- **Command Rate**: Commands per second
- **Hit Rate**: Cache hit percentage
- **Persistence Status**: AOF and RDB snapshot status

**Refresh Rate**: 15 seconds

**Use Cases:**

- Queue backlog monitoring
- Memory pressure detection
- Consumer throughput optimization

### 8. Node Exporter Dashboard (`node-exporter.json`)

**Purpose**: Host system resource monitoring.

**Key Panels:**

- **CPU Usage**: Overall CPU utilization and per-core breakdown
- **Memory Usage**: RAM usage and available memory
- **Disk I/O**: Read/write throughput and I/O wait time
- **Disk Space**: Filesystem usage by mount point
- **Network I/O**: Bytes in/out per interface
- **System Load**: 1/5/15 minute load averages
- **Network Errors**: Receive/transmit errors per interface
- **TCP Connections**: Connection state distribution

**Refresh Rate**: 15 seconds

**Use Cases:**

- Host capacity planning
- Identifying resource bottlenecks (CPU, memory, disk, network)
- Hardware performance troubleshooting

### 9. cAdvisor Dashboard (`cadvisor.json`)

**Purpose**: Container-level resource monitoring.

**Key Panels:**

- **Container CPU Usage**: CPU percentage per container
- **Container Memory Usage**: RAM usage per container (MB/GB)
- **Container Network I/O**: Bytes sent/received per container
- **Container Disk I/O**: Disk read/write per container
- **Container Restart Count**: Restarts in last 24 hours
- **Container Status**: Running/stopped/crashed status
- **Top Containers by CPU**: Highest CPU consumers
- **Top Containers by Memory**: Highest memory consumers

**Refresh Rate**: 15 seconds

**Use Cases:**

- Identifying resource-intensive containers
- Container scaling decisions
- Resource limit tuning

### 10. MinIO Dashboard (`minio.json`)

**Purpose**: Object storage monitoring.

**Key Panels:**

- **Storage Capacity**: Used vs total capacity
- **Bucket Size**: Size of `osint-media` bucket
- **API Request Rate**: Requests per second
- **API Latency**: Average response time
- **Bandwidth Usage**: Bytes uploaded/downloaded
- **Error Rate**: Failed API requests
- **Object Count**: Total objects stored
- **Multipart Uploads**: In-progress uploads

**Refresh Rate**: 30 seconds

**Use Cases:**

- Storage capacity planning
- Media archival performance monitoring
- Identifying storage bottlenecks

### 11. AlertManager Dashboard (`alertmanager.json`)

**Purpose**: Alert routing and notification tracking.

**Key Panels:**

- **Active Alerts**: Currently firing alerts by severity
- **Alert History**: Alerts fired in last 24 hours
- **Notification Rate**: Notifications sent per hour
- **Notification Success Rate**: Successful vs failed deliveries
- **Silenced Alerts**: Temporarily silenced alerts
- **Alert Group Distribution**: Alerts by component (processor, llm, database, etc.)

**Refresh Rate**: 30 seconds

**Use Cases:**

- Alert fatigue analysis
- Notification delivery verification
- Alert routing configuration validation

---

## Key Metrics Reference

### Message Processing Metrics

**Source**: Listener (`listener:8001/metrics`) and Processor (`processor:8002/metrics`)

```promql
# Ingestion rate
rate(osint_messages_processed_total[5m])

# Spam detection rate (percentage)
osint:spam_rate:5m

# Message skip rate (low OSINT score)
rate(osint_messages_skipped_total[5m]) / rate(osint_messages_processed_total[5m])

# Archival rate
rate(osint_messages_archived_total[5m])

# Queue depth
osint_queue_messages_pending

# Consumer lag
osint_queue_consumer_lag_seconds
```

**Key Thresholds:**

- Spam rate: 40-70% normal, >95% investigate filter, <20% verify filter working
- Skip rate: <90% normal, >90% for 15m triggers alert
- Queue depth: <10,000 normal, >10,000 for 15m triggers alert
- Consumer lag: <60s normal, >300s investigate processor capacity

### LLM Performance Metrics

**Source**: Processor (`processor:8002/metrics`)

```promql
# LLM success rate
osint:llm_success_rate:5m

# Average response time
osint:llm_response:avg_duration_seconds

# LLM error rate
rate(osint_llm_requests_total{status!="success"}[5m]) / rate(osint_llm_requests_total[5m])

# OSINT score distribution
osint_score_distribution_bucket

# Topic classification
osint_topics_total
```

**Key Thresholds:**

- Success rate: >90% normal, <90% investigate Ollama health
- Response time: <5s target, <15s acceptable, >15s for 15m triggers alert
- Error rate: <10% normal, >10% for 10m triggers alert

### Database Metrics

**Source**: PostgreSQL Exporter (`postgres-exporter:9187/metrics`)

```promql
# Active connections
pg_stat_activity_count

# Cache hit ratio
sum(pg_stat_database_blks_hit) / (sum(pg_stat_database_blks_hit) + sum(pg_stat_database_blks_read))

# Average query duration
osint:database_query:avg_duration_seconds

# Database size
pg_database_size_bytes{datname="osint_platform"}

# Table sizes (top 20)
pg_table_total_bytes

# Deadlocks
rate(pg_stat_database_deadlocks[5m])
```

**Key Thresholds:**

- Active connections: <80 normal, >80 for 10m triggers warning, >90 critical
- Cache hit ratio: >90% target, <90% for 30m triggers warning
- Query duration: <100ms target, >1s for 15m triggers warning
- Deadlocks: 0 normal, >0 investigate transaction logic

### Queue Metrics

**Source**: Redis Exporter (`redis-exporter:9121/metrics`)

```promql
# Queue depth
redis_stream_length{stream="telegram_messages"}

# Memory usage
redis_memory_used_bytes

# Eviction rate
rate(redis_evicted_keys_total[5m])

# Connection count
redis_connected_clients
```

**Key Thresholds:**

- Queue depth: See message processing metrics
- Memory usage: <90% of max (512MB), >90% for 10m triggers warning
- Eviction rate: 0 normal, >10/s for 10m triggers warning
- Connections: <80% of max, >80% for 10m triggers warning

### Media Archival Metrics

**Source**: Processor (`processor:8002/metrics`)

```promql
# Media archived by type
rate(osint_media_archived_total[5m])

# Storage usage
osint_media_storage_bytes_total

# Deduplication efficiency
osint:media_deduplication:efficiency_percentage

# Media errors
rate(osint_media_errors_total[5m])
```

**Key Thresholds:**

- Deduplication efficiency: 30-40% expected (varies by content)
- Error rate: <1/sec normal, >1/sec for 15m triggers warning
- Storage growth: Monitor capacity, plan expansion at 85% full

### API Performance Metrics

**Source**: API (`api:8000/metrics`)

```promql
# Request rate
rate(osint_api_requests_total[5m])

# Success rate
osint:api_success_rate:5m

# Error rate
osint:api_error_rate:5m

# Average response time
osint:api_request:avg_duration_seconds

# P95 latency
histogram_quantile(0.95, rate(api_request_duration_seconds_bucket[5m]))
```

**Key Thresholds:**

- Success rate: >95% target
- Error rate: <5% normal, >5% for 10m triggers warning
- Response time: <100ms target, <2s acceptable, >2s for 15m triggers warning
- P95 latency: <500ms target (SLO)

### Enrichment Pipeline Metrics

**Source**: Enrichment Service (`enrichment:9095/metrics`)

```promql
# Queue depth by task
enrichment_queue_depth{task="ai_tagging"}

# Queue lag (oldest message waiting)
enrichment_queue_lag_seconds

# Backpressure ratio (queue growth vs processing)
enrichment_backpressure_ratio

# Cycle duration
histogram_quantile(0.95, rate(enrichment_cycle_duration_seconds_bucket[5m]))

# Task stalled (no progress despite pending work)
increase(enrichment_messages_processed_total[30m]) == 0 and enrichment_queue_depth > 0
```

**Key Thresholds:**

- AI tagging queue depth: <500 normal, >500 warning, >1000 critical
- Queue lag: <1800s (30min) normal, >1800s warning, >3600s critical
- Backpressure ratio: <1 normal (keeping up), >5 warning, >10 critical
- Cycle duration: <300s (5min) target, >300s warning, >600s critical

---

## Alert Rules

Alert rules are defined in `/infrastructure/prometheus/rules/alerting_rules.yml`. **75 total alerts** across 12 categories.

### Viewing Active Alerts

**In Grafana:**

1. Navigate to **Alerting → Alert rules**
2. Filter by severity: `severity=critical` or `severity=warning`
3. View alert details, thresholds, and current values

**In Prometheus:**

1. Open http://localhost:9090/alerts
2. View all configured alerts and their state (Inactive/Pending/Firing)
3. Click alert name to see PromQL expression and evaluation

**In AlertManager:**

1. Open http://localhost:9093
2. View active alerts with grouping by component
3. Silence alerts temporarily if needed

See full alert documentation in `/infrastructure/prometheus/rules/alerting_rules.yml`.

---

## Health Check Endpoints

All services expose health check endpoints:

```bash
# Application services
curl http://localhost:8001/health   # Listener
curl http://localhost:8002/health   # Processor Worker
curl http://localhost:8000/health   # API
curl http://localhost:9095/health   # Enrichment

# Infrastructure services
docker exec osint-postgres pg_isready -U postgres -d osint_platform
docker exec osint-redis redis-cli ping
curl http://localhost:9000/minio/health/live
curl http://localhost:11434/api/tags  # Ollama

# Monitoring stack
curl http://localhost:9090/-/healthy  # Prometheus
curl http://localhost:3001/api/health # Grafana
curl http://localhost:9093/-/healthy  # AlertManager
```

---

## Log Analysis

### Dozzle (Real-Time Container Logs)

**Access**: http://localhost:8888

**Features:**

- Real-time log streaming for all containers
- Search and filter logs by keyword, regex, or time range
- Multi-container view (compare logs side-by-side)
- Export logs for offline analysis
- No storage overhead

### Docker Compose Logs (CLI)

```bash
# View all logs
docker-compose logs -f

# View specific service
docker-compose logs -f processor-worker

# View last 100 lines
docker-compose logs --tail=100 processor-worker

# Search logs
docker-compose logs processor-worker | grep "ERROR"

# Export logs to file
docker-compose logs --no-color processor-worker > processor-$(date +%Y%m%d).log
```

### Structured JSON Logging

All services use structured JSON logging:

```json
{
  "timestamp": "2025-12-09T12:34:56.789Z",
  "level": "INFO",
  "service": "processor",
  "worker_id": "worker-1",
  "trace_id": "abc123def456",
  "message": "Message processed successfully",
  "channel_id": 1234567890,
  "message_id": 987654,
  "osint_score": 87,
  "spam": false,
  "duration_ms": 234
}
```

**Parsing JSON logs:**

```bash
# Extract ERROR-level logs
docker-compose logs processor-worker | jq 'select(.level == "ERROR")'

# Find slow processing (>1000ms)
docker-compose logs processor-worker | jq 'select(.duration_ms > 1000)'

# Get average processing time
docker-compose logs processor-worker | jq -s '[.[].duration_ms] | add / length'
```

---

## Performance Monitoring

### Identifying Bottlenecks

**1. Message Processing Pipeline**

```bash
# Check queue depth
curl -s http://localhost:8002/metrics | grep osint_queue_messages_pending
# If >10,000: Scale up processor workers

# Check consumer lag
curl -s http://localhost:8002/metrics | grep osint_queue_consumer_lag_seconds
# If >300s: Processor can't keep up with ingestion
```

**2. LLM Performance**

```bash
# Check LLM response time
curl -s http://localhost:8002/metrics | grep osint_llm_response_duration
# If >15s: Ollama may be overloaded

# Check LLM error rate
# Grafana → AI/ML Processing → LLM Success Rate
# If <90%: Check Ollama logs for errors
```

**3. Database Performance**

```bash
# Check cache hit ratio
curl -s http://localhost:9187/metrics | grep pg_stat_database_blks
# If <90%: Increase shared_buffers in postgresql.conf

# Check slow queries
docker exec osint-postgres psql -U postgres -d osint_platform -c "
SELECT calls, mean_exec_time, query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;"
```

### Tuning Recommendations

**Scale Processor Workers:**

```bash
# If queue depth >10,000 or consumer lag >300s:
docker-compose up -d --scale processor-worker=4
```

**PostgreSQL Tuning:**

```bash
# Edit infrastructure/postgres/postgresql.conf
shared_buffers = 512MB  # Increase from 256MB if cache hit ratio <90%
work_mem = 16MB         # Increase from 4MB if complex queries slow
max_connections = 200   # Increase from 100 if >80 connections frequently

# Restart PostgreSQL
docker-compose restart postgres
```

**Redis Tuning:**

```bash
# Edit docker-compose.yml redis command
command: redis-server --appendonly yes --maxmemory 1gb --maxmemory-policy allkeys-lru

# Restart Redis
docker-compose restart redis
```

---

## Troubleshooting Monitoring Stack

### Issue: Grafana shows "No data"

```bash
# 1. Verify Prometheus datasource
# Grafana → Configuration → Data sources → Prometheus → Save & test

# 2. Check if metrics exist
curl -s http://localhost:9090/api/v1/query?query=up | jq

# 3. Check time range (ensure it includes when services were running)

# 4. Check Grafana logs
docker logs osint-grafana | tail -50
```

### Issue: Prometheus targets showing "down"

```bash
# 1. Check service is exposing metrics
curl http://localhost:8001/metrics  # Should return Prometheus format metrics

# 2. Check Docker network
docker network inspect osint-intelligence-platform_backend

# 3. Check Prometheus logs
docker logs osint-prometheus | grep -i error

# 4. Verify Prometheus config
docker exec osint-prometheus promtool check config /etc/prometheus/prometheus.yml
```

### Issue: High Prometheus memory usage

```bash
# 1. Check current retention
docker exec osint-prometheus promtool tsdb analyze /prometheus

# 2. Reduce retention period (edit docker-compose.yml)
command:
  - '--storage.tsdb.retention.time=7d'  # Reduce from 30d

# 3. Restart Prometheus
docker-compose restart prometheus
```

### Issue: Alerts not firing

```bash
# 1. Check alerting rules syntax
docker exec osint-prometheus promtool check rules /etc/prometheus/rules/alerting_rules.yml

# 2. Verify alert is configured
curl http://localhost:9090/api/v1/rules | jq

# 3. Check if condition is met (query the alert expression)
```

---

## Production Checklist

Before deploying monitoring to production:

- [ ] Change Grafana admin password from default `admin/admin`
- [ ] Set strong secrets in `.env`
- [ ] Disable Grafana anonymous auth
- [ ] Enable HTTPS with reverse proxy (Caddy/Nginx)
- [ ] Configure AlertManager with production notification channels
- [ ] Test alert routing end-to-end
- [ ] Set appropriate Prometheus retention (default 30d)
- [ ] Schedule backups of Grafana and Prometheus data volumes
- [ ] Document runbooks for critical alerts
- [ ] Set up log rotation for Docker containers
- [ ] Restrict monitoring ports with firewall rules
- [ ] Enable audit logging in Grafana

---

## Additional Resources

### Documentation

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [PromQL Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [AlertManager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)

### Project Files

- Prometheus config: `/infrastructure/prometheus/prometheus.yml`
- Recording rules: `/infrastructure/prometheus/rules/recording_rules.yml`
- Alerting rules: `/infrastructure/prometheus/rules/alerting_rules.yml`
- Grafana provisioning: `/infrastructure/grafana/provisioning/`
- Dashboard JSONs: `/infrastructure/grafana/dashboards/`
- Metrics module: `/shared/python/observability/metrics.py`
- Logging module: `/shared/python/observability/logging.py`

---

**Last Updated**: 2025-12-09
**Version**: 1.0
**Platform Version**: 1.0 (Production-ready)
