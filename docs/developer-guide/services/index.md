# Services Deep Dive

Detailed documentation of each microservice in the platform.

## Overview

**TODO: Content to be generated from codebase analysis**

The platform consists of 15 application services organized by function.

## Core Services

### Listener Service

**TODO: Document listener service:**

- Telegram client management
- Session handling
- Folder monitoring
- Channel discovery
- Message streaming to Redis

**Code**: `/services/listener/`

### Processor Service

**TODO: Document processor service:**

- Real-time message processing (<1s target)
- Spam filtering
- Entity extraction
- Media archival
- Message persistence
- Intelligence rule evaluation

**Code**: `/services/processor/`

### Enrichment Service

**TODO: Document enrichment service:**

- Background task system
- Embedding generation
- AI tagging
- Social graph analysis
- Engagement polling
- Wikidata enrichment
- OpenSanctions enrichment

**Code**: `/services/enrichment/`

### API Service

**TODO: Document API service:**

- FastAPI REST API
- Authentication
- Search endpoints
- Entity endpoints
- Admin endpoints
- WebSocket support

**Code**: `/services/api/`

### Frontend Service

**TODO: Document frontend service:**

- Next.js 14 application
- Server-side rendering
- Client components
- API integration
- Search UI
- Entity explorer

**Code**: `/services/frontend-nextjs/`

## Supporting Services

### RSS Ingestor

**TODO: Document RSS ingestor service**

**Code**: `/services/rss-ingestor/`

### OpenSanctions Service

**TODO: Document OpenSanctions enrichment**

**Code**: `/services/opensanctions/`

### Migration Service

**TODO: Document legacy data migration**

**Code**: `/services/migration/`

## Infrastructure Services

### PostgreSQL

**TODO: Document PostgreSQL setup and extensions**

- pgvector for embeddings
- Full-text search
- JSON support
- Schema in `init.sql`

### Redis

**TODO: Document Redis usage**

- Streams for message queue
- Pub/Sub for notifications
- Caching layer

### MinIO

**TODO: Document MinIO setup**

- S3-compatible storage
- Bucket configuration
- Content-addressed paths

### Ollama

**TODO: Document Ollama setup**

- Model management
- API integration
- Performance tuning

## Service Communication

**TODO: Describe inter-service communication:**

- Redis Streams for async messaging
- Direct HTTP for sync API calls
- Database as shared state
- Event-driven patterns

## Service Scaling

**TODO: Document scaling strategies for each service:**

```bash
# Example scaling
docker-compose up -d --scale processor-worker=4
docker-compose up -d --scale enrichment-worker=2
```

---

!!! note "Documentation Status"
    This page is a placeholder. Content will be generated from service code and README files.
