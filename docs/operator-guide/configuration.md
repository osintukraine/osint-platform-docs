# Configuration

Complete reference for configuring all platform services and components.

## Overview

**TODO: Content to be generated from codebase analysis**

This page will cover:

- Environment variable reference
- Service-specific configuration
- Docker Compose configuration
- Network and port configuration
- Storage configuration
- LLM model configuration
- Authentication configuration
- Logging configuration

## Environment Variables

### Core Configuration

**TODO: Document core environment variables**

```bash
# Database
POSTGRES_USER=osint_user
POSTGRES_PASSWORD=...
POSTGRES_DB=osint_platform

# Redis
REDIS_URL=redis://redis:6379

# MinIO
MINIO_ROOT_USER=...
MINIO_ROOT_PASSWORD=...
```

### Service Configuration

**TODO: Document service-specific environment variables:**

- Listener configuration
- Processor configuration
- Enrichment configuration
- API configuration
- Frontend configuration

## Docker Compose Configuration

**TODO: Explain docker-compose.yml structure and customization**

### Scaling Services

```bash
# Scale processor workers
docker-compose up -d --scale processor-worker=4

# Scale enrichment workers
docker-compose up -d --scale enrichment-worker=2
```

## Network Configuration

**TODO: Document network configuration:**

- Internal Docker networks
- Port mappings
- Reverse proxy setup
- SSL/TLS configuration

## Storage Configuration

### PostgreSQL

**TODO: Document PostgreSQL configuration and tuning**

### MinIO

**TODO: Document MinIO bucket configuration and storage paths**

### Redis

**TODO: Document Redis configuration and persistence**

## LLM Configuration

**TODO: Document Ollama model configuration:**

- Model selection (qwen2.5:3b)
- Model download
- Memory limits
- Performance tuning

## Authentication Configuration

**TODO: Document authentication setup:**

- OAuth2 providers
- API keys
- User management
- Role-based access control

## Logging Configuration

**TODO: Document logging configuration:**

- Log levels
- Log aggregation
- Log rotation
- Structured logging

## Advanced Configuration

**TODO: Document advanced topics:**

- Custom spam filters
- Custom intelligence rules
- Custom enrichment tasks
- Performance tuning

---

!!! note "Documentation Status"
    This page is a placeholder. Content will be generated from .env.example, docker-compose.yml, and service configuration files.
