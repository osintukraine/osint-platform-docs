# API Reference

Complete technical reference for the OSINT Intelligence Platform.

## Overview

This reference documentation provides comprehensive details about:

- **[API Endpoints](api-endpoints.md)** - All REST API endpoints with parameters and responses
- **[Environment Variables](environment-vars.md)** - Complete configuration reference
- **[Database Tables](database-tables.md)** - PostgreSQL schema documentation
- **[Docker Services](docker-services.md)** - Container architecture

## Quick Facts

- **Database**: PostgreSQL 16 + pgvector for semantic search
- **API**: FastAPI with OpenAPI/Swagger documentation
- **Message Queue**: Redis Streams for async processing
- **Object Storage**: MinIO (S3-compatible) for media
- **AI**: Ollama for self-hosted LLM inference
- **Tech Stack**: Python 3.11+, Next.js 14, Bun

## Architecture Patterns

**Message Processing Flow**:
```
Telegram → Listener → Redis → Processor → PostgreSQL/MinIO → Enrichment → API → Frontend
```

**Service Tiers**:
- **Real-time** (Processor): <1s per message, LLM classification, spam filtering
- **Background** (Enrichment): Hours OK, embeddings, tagging, translation
- **Maintenance**: Hourly tasks, channel cleanup, discovery evaluation

## API Conventions

### Base URL

- **Development**: `http://localhost:8000`
- **Production**: `https://your-domain.com`

### Authentication

```http
GET /api/messages
Authorization: Bearer YOUR_JWT_TOKEN
```

Authentication is optional (configurable via `AUTH_PROVIDER`):
- `none` - No authentication (private VPN deployments)
- `jwt` - Simple JWT tokens
- `ory` - Ory Kratos/Oathkeeper (zero-trust)

### Pagination

All list endpoints support pagination:

```http
GET /api/messages?page=1&page_size=50
```

**Response**:
```json
{
  "items": [...],
  "total": 1000,
  "page": 1,
  "page_size": 50,
  "total_pages": 20,
  "has_next": true,
  "has_prev": false
}
```

### Filtering

Common query parameters:
- `q` - Full-text search query
- `days` - Filter by last N days
- `channel_id` - Filter by channel
- `topic` - Filter by OSINT topic
- `importance_level` - Filter by importance (high/medium/low)

### Error Responses

```json
{
  "detail": "Error message"
}
```

HTTP status codes:
- `400` - Bad Request
- `401` - Unauthorized
- `404` - Not Found
- `422` - Validation Error
- `500` - Internal Server Error

## Data Types Reference

### PostgreSQL Types

- **BIGINT**: 64-bit integer (Telegram IDs)
- **INTEGER**: 32-bit integer (database IDs)
- **TEXT**: Unlimited text
- **VARCHAR(N)**: Limited string (e.g., VARCHAR(100))
- **JSONB**: Binary JSON with indexing support
- **TIMESTAMP WITH TIME ZONE**: UTC timestamps
- **vector(384)**: 384-dimensional embedding vector (pgvector)
- **TSVECTOR**: Full-text search document
- **TEXT[]**: Array of text strings

### API Response Types

- **MessageDetail**: Full message with media, tags, entities
- **MessageList**: Message summary for lists
- **ChannelDetail**: Channel with stats and metadata
- **EntityDetail**: Entity with relationships and mentions
- **EventDetail**: Event with linked messages and RSS sources

## Database Patterns

### Content Hashing

All media uses SHA-256 content-addressed storage:
```
media/{hash[:2]}/{hash[2:4]}/{hash}.{ext}
```

Deduplication: Same file = same hash = single storage.

### Vector Embeddings

**Model**: all-MiniLM-L6-v2 (384 dimensions)

**Tables with embeddings**:
- `messages.content_embedding` - Message semantic search
- `events.content_embedding` - Event similarity
- `curated_entities.entity_embedding` - Entity matching
- `opensanctions_entities.entity_embedding` - Sanctions entities
- `external_news.embedding` - RSS article search

**Search pattern**:
```sql
SELECT * FROM messages
WHERE 1 - (content_embedding <=> query_embedding) >= 0.7
ORDER BY content_embedding <=> query_embedding
LIMIT 20;
```

### Full-Text Search

All tables with `search_vector` column support full-text search:

```sql
SELECT * FROM messages
WHERE search_vector @@ plainto_tsquery('english', 'query')
ORDER BY ts_rank(search_vector, plainto_tsquery('english', 'query')) DESC;
```

## Docker Profiles

Services are grouped by profile:

- **Default**: Core services (listener, processor, API, frontend)
- **enrichment**: Background enrichment workers
- **monitoring**: Prometheus, Grafana, exporters
- **opensanctions**: Entity intelligence (Yente + ElasticSearch)
- **auth**: Ory Kratos/Oathkeeper authentication
- **dev**: Development tools (Dashy, MailSlurper, MkDocs)

**Start specific profiles**:
```bash
docker-compose --profile enrichment --profile monitoring up -d
```

## Configuration Hierarchy

Configuration precedence (highest to lowest):

1. Environment variables
2. `.env` file
3. `docker-compose.yml` defaults
4. Database `platform_config` table (runtime config)
5. Code defaults

**Runtime configuration** (no restart needed):
- LLM prompts (`llm_prompts` table)
- Model selection (`model_configuration` table)
- Platform settings (`platform_config` table)

## Resource Requirements

### Minimum (Development)

- **CPU**: 4 cores
- **RAM**: 8GB
- **Disk**: 50GB

### Recommended (Production)

- **CPU**: 8-16 cores
- **RAM**: 32GB
- **Disk**: 500GB SSD

### Storage Breakdown

- **PostgreSQL**: ~100GB for 1M messages
- **MinIO**: ~300GB for media archive
- **Ollama**: ~20GB for models (qwen2.5:3b, granite3-dense:2b)
- **Monitoring**: ~50GB for metrics (30 days retention)

## Version Information

- **Platform Version**: 1.0
- **API Version**: v1 (implicit in `/api/*` paths)
- **PostgreSQL**: 16+ (required for pgvector)
- **Python**: 3.11+
- **Node.js**: 20+ (frontend)
- **Docker**: 27+
- **Docker Compose**: 2.20+

## Schema Management

**No Alembic migrations**. Schema is managed via `init.sql`:

1. Make changes to `~/code/osintukraine/osint-intelligence-platform/infrastructure/postgres/init.sql`
2. Rebuild database:
   ```bash
   docker-compose down
   docker volume rm osint-intelligence-platform_postgres_data
   docker-compose up -d postgres
   ```

## Additional Resources

- [Architecture Documentation](../developer-guide/architecture.md)
- [Developer Guide](../developer-guide/index.md)
- [Deployment Guide](../tutorials/deploy-to-production.md)
- [Monitoring Guide](../operator-guide/monitoring.md)

## Support

- **Documentation**: `~/code/osintukraine/osint-intelligence-platform/docs/`
- **Issues**: GitHub Issues
- **Source Code**: `~/code/osintukraine/osint-intelligence-platform/`
