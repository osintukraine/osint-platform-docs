# API Service

> FastAPI REST API serving the frontend and external consumers

## Overview

The API service is the **primary data access layer** for the OSINT Intelligence Platform. It provides a comprehensive REST API for searching, analyzing, and retrieving enriched Telegram intelligence data, along with dynamic RSS feed generation and real-time monitoring endpoints.

**Purpose**: Unified REST API for all data access and querying
**Technologies**: FastAPI 0.127, SQLAlchemy 2.0 (async), Pydantic, Uvicorn 0.40
**Auto-generated docs**: `/docs` (Swagger UI), `/redoc` (ReDoc)

### Key Capabilities

- **40+ Router Modules** (`routers/` plus `routers/admin/`): Comprehensive endpoint coverage for all platform features
- **Dynamic RSS Feeds**: "Subscribe to any search" in RSS 2.0, Atom 1.0, and JSON Feed 1.1 formats
- **Semantic Search**: pgvector-powered similarity search with 384-dimensional embeddings
- **Social Graph Queries**: Forward chains, reply threads, engagement analytics
- **Unified Authentication**: API keys (`ak_*`), Ory Kratos sessions, Oathkeeper headers, JWT, or anonymous (one middleware)
- **Prometheus Metrics**: Request counts, latency histograms at `/metrics`
- **Export Worker**: Background CSV/JSON data export to MinIO (in-process, polls the DB)

## Architecture

### Directory Structure

```
services/api/
├── src/
│   ├── main.py                    # FastAPI app initialization
│   ├── database.py                # Session management
│   ├── feed_generator.py          # Multi-format RSS/Atom/JSON generator
│   ├── export_worker.py           # Background export worker
│   ├── schemas.py                 # Pydantic response models
│   ├── services/                  # Feed tokens/subscriptions, API keys, shadow fleet
│   ├── kratos/                    # Kratos admin client
│   ├── utils/                     # Cache, geocoding, pagination, prometheus helpers
│   ├── routers/                   # API endpoint modules
│   │   ├── __init__.py            # Router registry
│   │   ├── messages.py            # Message CRUD and search
│   │   ├── channels.py            # Channel management
│   │   ├── entities.py            # Entity profiles and search
│   │   ├── search.py              # Unified search across all sources
│   │   ├── semantic.py            # AI-powered semantic search
│   │   ├── similarity.py          # pgvector similarity queries
│   │   ├── social_graph.py        # Social network analysis
│   │   ├── rss.py                 # Dynamic feed generation
│   │   ├── analytics.py           # Aggregations and metrics
│   │   ├── timeline.py            # Per-message temporal context
│   │   ├── map.py, vessels.py     # Map GeoJSON and vessel tracking
│   │   ├── api_keys.py            # API key management
│   │   ├── channel_submissions.py # Public channel suggestions
│   │   ├── events_admin.py        # Event admin operations
│   │   ├── events.py              # Event clustering
│   │   ├── network.py             # Knowledge graph visualization
│   │   ├── flowsint_export.py     # Flowsint-compatible export
│   │   ├── validation.py          # RSS cross-validation
│   │   ├── stream.py              # Unified Telegram+RSS stream
│   │   ├── models.py              # Multi-model LLM management
│   │   ├── bookmarks.py           # User bookmarks
│   │   ├── user.py                # User preferences
│   │   ├── spam.py                # Spam management
│   │   ├── media.py               # Media gallery
│   │   ├── comments.py            # Comment translation
│   │   ├── system.py              # Health checks
│   │   ├── auth.py                # Authentication endpoints
│   │   ├── feed_tokens.py         # RSS feed tokens
│   │   ├── metrics/               # /api/metrics/* (pipeline, services, llm, overview)
│   │   ├── about.py               # Platform statistics
│   │   ├── channel_network.py     # Channel content networks
│   │   ├── news_timeline.py       # RSS+Telegram correlation
│   │   └── admin/                 # Admin-only endpoints
│   │       ├── dashboard.py       # Platform overview
│   │       ├── spam.py            # Spam review queue
│   │       ├── media.py           # Media management
│   │       ├── kanban.py          # Urgency board
│   │       ├── channels.py        # Channel admin
│   │       ├── entities.py        # Entity curation
│   │       ├── prompts.py         # LLM prompt management
│   │       ├── system.py          # Workers, audit, cache, enrichment tasks
│   │       ├── fact_check.py      # Fact-check review (see fact-check.md)
│   │       ├── users.py           # Kratos user management
│   │       ├── message_actions.py # Moderation actions
│   │       ├── role_check.py      # Oathkeeper role check
│   │       ├── feeds.py           # RSS feed config
│   │       ├── export.py          # Data export queue
│   │       ├── config.py          # System configuration
│   │       ├── stats.py           # Statistics dashboard
│   │       └── comments.py        # Comment analysis
│   ├── auth/                      # Authentication providers
│   │   ├── factory.py             # Provider selection
│   │   ├── models.py              # Auth data models
│   │   ├── none.py                # Development auth (no-op)
│   │   ├── jwt.py                 # JWT token auth
│   │   ├── cloudron.py            # Cloudron OAuth
│   │   ├── ory.py                 # Ory provider
│   │   ├── api_key.py             # API key auth
│   │   └── feed_auth.py           # RSS feed token auth
│   └── middleware/                # HTTP middleware
│       ├── auth_unified.py        # Active auth middleware (registered in main.py)
│       └── auth.py, ory_auth.py   # Earlier middleware kept in the tree
# Settings live in shared/python/config/settings.py (no per-service config module)
├── requirements.txt               # Python dependencies
├── Dockerfile                     # Container image
└── README.md                      # Service documentation
```

### Request Flow

```mermaid
graph LR
    A[Client Request] --> B[CORS Middleware]
    B --> C[Request Tracing]
    C --> D[Metrics Middleware]
    D --> E[GZip / Error Notification]
    E --> F[Auth Middleware]
    F --> G{Authenticated?}
    G -->|Yes| H[Router Handler]
    G -->|No GET| H
    G -->|No Write| I[401 Unauthorized]
    H --> J[Database Query]
    J --> K[Response Model]
    K --> L[Client Response]

    style F fill:#f9f,stroke:#333
    style H fill:#bbf,stroke:#333
    style J fill:#bfb,stroke:#333
```

### Architecture Role

The API service sits between the data layer (PostgreSQL + MinIO) and consumers (frontend + external clients):

```
PostgreSQL (data) + MinIO (media)
          |
          v
    +------------+
    | API Service|
    +------------+
          |
          v
Frontend / RSS Readers / External APIs
```

**Important**: The API is **read-only** for most operations. Write operations are limited to:
- User preferences and bookmarks
- Admin configuration changes
- Background export queue management

## Routers

### Core Data Access

#### Messages Router (`/api/messages`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/messages.py`

Message retrieval and full-text search with 15+ filters.

**Key Endpoints**:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/messages` | GET | Search messages with filters (query, channel, date, importance, entities, spam status) |
| `/api/messages/{message_id}` | GET | Get single message with full enrichment data |
| `/api/messages/{message_id}/adjacent` | GET | Get prev/next message IDs for navigation |
| `/api/messages/{message_id}/album` | GET | Get all media files in message's album (for lightbox) |
| `/api/messages/{message_id}/similar` | GET | Similar messages (`similarity.py`) |
| `/api/messages/{message_id}/timeline` | GET | Temporal context (`timeline.py`) |
| `/api/messages/{message_id}/validation` | GET | RSS corroboration (`validation.py`) |
| `/api/messages/{message_id}/network` | GET | Message network graph (`network.py`) |
| `/api/messages/{message_id}/network/export` | GET | Flowsint JSON export (`flowsint_export.py`) |

**Search Parameters**:
- `q`: Full-text search query (PostgreSQL tsvector)
- `channel_id`, `channel_username`, `channel_folder`: Channel filters
- `importance_level`: high/medium/low
- `topic`: OSINT topic classification
- `has_media`, `media_type`: Media filters
- `sentiment`, `language`: Content analysis filters
- `days`, `date_from`, `date_to`: Temporal filters
- `min_views`, `min_forwards`: Engagement filters
- `is_spam`: Spam filter (default: false)
- `page`, `page_size`: Pagination (max 100 per page)

**Response**: `MessageList` with pagination metadata

#### Channels Router (`/api/channels`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/channels.py`

Channel management and statistics.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/channels/` | GET | List all channels with folder filtering |
| `/api/channels/by-username/{username}` | GET | Channel by Telegram username |
| `/api/channels/{channel_id}` | GET | Channel details |
| `/api/channels/{channel_id}/stats` | GET | Channel statistics |
| `/api/channels/{channel_id}/backfill` | POST | Trigger historical backfill (admin) |
| `/api/channels/{channel_id}/network` | GET | Channel content network graph (`channel_network.py`) |

Messages of one channel come from `GET /api/messages?channel_id=...`.

**Features**:
- Folder-based filtering (`folder_pattern` query param)
- Message counts and engagement statistics
- Archive rule metadata

#### Entities Router (`/api/entities`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/entities.py`

Entity profiles from curated sources (ArmyGuide, Root.NK, ODIN) and OpenSanctions.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/entities/search` | GET | Search entities |
| `/api/entities/{source}/{entity_id}` | GET | Entity profile with linked content counts |
| `/api/entities/{source}/{entity_id}/messages` | GET | Messages mentioning this entity |
| `/api/entities/{source}/{entity_id}/relationships` | GET | Entity relationship graph (Wikidata SPARQL) |

**Sources**: `curated` (internal), `opensanctions` (Yente API)

**Features**:
- On-demand Wikidata SPARQL relationship fetching
- 7-day relationship cache (stored in `entity_relationships` table)
- Corporate, political, and associate connections
- Integration with OpenSanctions/Yente for sanctions data

### Search & Discovery

#### Search Router (`/api/search`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/search.py`

Unified search across all data sources (messages, events, RSS articles, entities).

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/search/` | GET | Cross-source search with grouped results |

**Parameters**:
- `q`: Search query
- `mode`: `text` or `semantic`
- `types`: Comma-separated types to search (messages, events, rss, entities)
- `limit_per_type`: Results per type (default: 5, max: 20)
- `location`, `lat`/`lng`/`radius_km`, `bounds`, `cluster_by`: geographic filters and clustering

**Response**: Grouped results with type-specific metadata and timing

#### Semantic Router (`/api/semantic`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/semantic.py`

AI-powered semantic search using vector embeddings.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/semantic/search` | GET | Find messages by meaning (not keywords) |
| `/api/semantic/similar/{message_id}` | GET | Find similar messages to a given message |
| `/api/semantic/tags` | GET | Popular tags |
| `/api/semantic/tags/search` | GET | Messages by tag (`tags`, `match_all`) |

**How It Works**:
1. Query text → embedding model from the `model_configuration` table, served by Ollama (the one place the API still calls Ollama, even when classification runs on the LLM gateway)
2. Generate 384-dimensional vector
3. pgvector cosine similarity search against `messages.content_embedding`
4. Return top N results sorted by similarity score

**Parameters**:
- `q`: Search query (semantic search)
- `threshold`: Minimum similarity score (0.0-1.0; default 0.6 for `/search`, 0.7 for `/similar`)
- `limit`: Max results (default: 20, max: 100)
- `channel_id`, `days`: Optional filters

#### Similarity Router (`/api/messages/{id}/similar`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/similarity.py`

pgvector-based message similarity queries.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/messages/{message_id}/similar` | GET | Find messages similar to given message |

### Social Graph & Analytics

#### Social Graph Router (`/api/social-graph`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/social_graph.py`

Telegram social interaction analysis.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/social-graph/messages/{message_id}` | GET | Forward chains, replies, reactions for message |
| `/api/social-graph/messages/{message_id}/engagement-timeline` | GET | Views/reactions over time |
| `/api/social-graph/messages/{message_id}/comments` | GET | Comment thread |
| `/api/social-graph/channels/{channel_id}/influence` | GET | Channel influence |
| `/api/social-graph/influence-network` | GET | Influence network across channels |
| `/api/social-graph/virality/top-forwarded` | GET | Most forwarded messages |

**Features**:
- Forward propagation tracking (who forwarded this message)
- Reply thread reconstruction
- Reaction sentiment analysis
- Author attribution (user or channel)
- Engagement timelines (views/reactions over time)

**Response Structure**:
```json
{
  "nodes": [
    {"id": "msg-123", "type": "message", "label": "...", "data": {...}},
    {"id": "user-456", "type": "author", "label": "User 456", "data": {...}}
  ],
  "edges": [
    {"id": "edge-forward-1", "source": "msg-123", "target": "msg-789", "type": "forward"}
  ]
}
```

#### Analytics Router (`/api/analytics`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/analytics.py`

Platform-wide aggregations and metrics.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/analytics/timeline` | GET | Message volume over time |
| `/api/analytics/distributions` | GET | Topic, importance and other distributions |
| `/api/analytics/heatmap` | GET | Activity heatmap |
| `/api/analytics/engagement` | GET | Engagement analytics |
| `/api/analytics/channels` | GET | Channel analytics |
| `/api/analytics/entities` | GET | Entity analytics |
| `/api/analytics/media` | GET | Media analytics |

#### Timeline Routers

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/timeline.py`

Per-message temporal context (`timeline.py`: `GET /api/messages/{message_id}/timeline`) and the news timeline correlating RSS and Telegram (`news_timeline.py`):

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/timeline` | GET | News timeline (RSS + Telegram correlations) |
| `/api/timeline/stats` | GET | News timeline statistics |

#### Events Router (`/api/events`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/events.py`

Event clustering and chronological timelines.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/events/` | GET | List clustered events |
| `/api/events/stats` | GET | Event statistics |
| `/api/events/message/{message_id}` | GET | Events containing a message |
| `/api/events/{event_id}` | GET | Event details with related messages |
| `/api/events/{event_id}/timeline` | GET | Event timeline |
| `/api/events/{event_id}/major` | PATCH | Flag as major event |
| `/api/events/{event_id}/archive` | PATCH | Archive event |

Admin operations (soft delete, source removal, audit) live in `events_admin.py` under `/api/admin/events/`.

**Event Clustering**: Groups messages by entity overlap and semantic similarity to create real-world event timelines.

### RSS & Feed Generation

#### RSS Router (`/rss`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/rss.py`

Dynamic feed generation in multiple formats.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/rss/search` | GET | Generate feed from any search query |
| `/rss/channel/{username}` | GET | Per-channel RSS feed |
| `/rss/topic/{topic}` | GET | Topic-based feed |
| `/rss/formats` | GET | Supported formats |

**Format Parameter**: `?format=rss` (RSS 2.0), `?format=atom` (Atom 1.0), `?format=json` (JSON Feed 1.1)

**Features**:
- Same filters as `/api/messages` endpoint
- Media enclosures with proper MIME types and file sizes
- Redis caching (5-15 min TTL based on popularity)
- Feed autodiscovery support
- Feed token authentication (optional)

**Feed Generator** (`feed_generator.py`):
- Three output formats with proper MIME types
- Media enclosures (RSS), attachments (JSON Feed), embedded HTML (Atom)
- Local MinIO URLs when available, Telegram fallback
- Rich metadata (importance badges, entities, sentiment)

### Data Visualization

#### Network Router

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/network.py`

Knowledge graph visualization.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/messages/{message_id}/network` | GET | Message-centric network graph |
| `/api/channels/{channel_id}/network` | GET | Channel content network |

For where each graph's data comes from (PostgreSQL and pgvector, rendered with Sigma.js), see [Architecture](../architecture.md#6-graph-data-on-postgresql-pgvector).

#### Flowsint Export Router

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/flowsint_export.py`

Export network graph data in Flowsint-compatible format for advanced OSINT investigation.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/messages/{message_id}/network/export` | GET | Export graph data in Flowsint JSON format |

### Validation & Correlation

#### Validation Router

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/validation.py`

Cross-reference Telegram claims with external RSS news sources.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/messages/{message_id}/validation` | GET | Find corroborating RSS articles for message |

**How It Works**: Semantic similarity matching between Telegram messages and RSS article embeddings.

#### Stream Router (`/api/stream`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/stream.py`

Unified intelligence stream combining Telegram messages and RSS articles.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/stream/unified` | GET | Combined Telegram+RSS feed with correlation indicators |
| `/api/stream/correlations/{message_id}` | GET | Correlations for one message |

### AI & Model Management

#### Models Router (`/api/models`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/models.py`

Multi-model LLM architecture management.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/models/registry` | GET | Model registry (`/registry/verified`, `/registry/{model_id}`) |
| `/api/models/tasks/{task}` | GET | Models for a task (`/fastest`, `/best-quality` variants) |
| `/api/models/configuration` | GET | Active configuration per task (`/configuration/{task}`) |
| `/api/models/configuration/{task}` | PUT | Update model configuration (admin); `DELETE .../{model_id}` removes one |
| `/api/models/health/{model_id}` | GET | Check model health |
| `/api/models/stats` | GET | Model usage statistics |

This registry mainly drives the embedding model used for search queries. Production classification goes through the LLM gateway (`LLM_PROVIDER`), configured in the processor, not here; see [LLM Gateway and Providers](../../operator-guide/external-llm.md).

**Runtime Switching**: Change active model without service restart via database configuration.

### User Features

#### Bookmarks Router (`/api/bookmarks`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/bookmarks.py`

User message bookmarks (requires authentication).

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/bookmarks/` | GET | List user's bookmarks |
| `/api/bookmarks/{message_id}` | POST | Add bookmark |
| `/api/bookmarks/{message_id}` | DELETE | Remove bookmark |

#### User Router (`/api/user`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/user.py`

User profile and preferences (requires authentication).

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/user/me` | GET | Get user profile |
| `/api/user/preferences` | GET/PUT | User preferences |

API keys (`ak_*`, see `api_keys.py`): `POST/GET /api/api-keys`, `GET /api/api-keys/usage/summary`, `GET/DELETE /api/api-keys/{key_id}`.

#### Feed Tokens Router (`/api/feed-tokens`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/feed_tokens.py`

RSS feed token generation for authenticated RSS feeds.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/feed-tokens` | GET | List user's feed tokens |
| `/api/feed-tokens` | POST | Generate new token |
| `/api/feed-tokens/{token_id}` | DELETE | Revoke token |
| `/api/feed-tokens/{token_id}/sign-url` | POST | Sign a feed URL |
| `/api/feed-tokens/{token_id}/subscriptions` | GET | Subscriptions (PATCH/DELETE/clone/regenerate-url on `/subscriptions/{subscription_id}`) |

**Usage**: Append `?token=YOUR_TOKEN` to RSS feed URLs for private feeds.

### Admin Endpoints

All admin routers require authentication and admin role.

#### Admin Dashboard (`/api/admin/dashboard`)

Platform overview and statistics.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/dashboard` | GET | Platform health, worker status, queue metrics |

#### Admin Spam (`/api/admin/spam`)

Spam review queue and false positive management.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/spam/` | GET | Spam review queue (`/stats` for counts) |
| `/api/admin/spam/{message_id}/review` | PUT | Review a message (`/bulk-review` for many) |
| `/api/admin/spam/{message_id}/reprocess` | POST | Reprocess through the pipeline |
| `/api/admin/spam/{message_id}` | DELETE | Delete (`/bulk-delete`, `/purge/confirmed`) |

Public-side spam actions: `POST /api/spam/messages/{message_id}/mark-not-spam` and `/confirm-spam`.

#### Admin Media (`/api/admin/media`)

Media gallery and management.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/media/` | GET | Photo/video gallery with filters |
| `/api/admin/media/stats` | GET | Storage statistics |
| `/api/admin/media/topics` | GET | Topics for gallery filters |

The public gallery is `GET /api/media/gallery` and `/api/media/stats`; `/api/media/internal/*` is the Caddy media-redirect path (bypasses auth).

#### Admin Channels (`/api/admin/channels`)

Channel management and configuration.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/channels/` | GET | All channels with metadata (`/stats`, `/folders`) |
| `/api/admin/channels/{channel_id}` | GET/PUT | Channel detail / update channel config |

#### Admin Entities (`/api/admin/entities`)

Entity curation and management.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/entities/` | GET | List curated entities (`/stats`, `/types`, `/sources`) |
| `/api/admin/entities/{entity_id}` | GET | Entity detail |

#### Admin Prompts (`/api/admin/prompts`)

LLM prompt management (for AI classification).

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/prompts/` | GET | List prompt versions (`/stats`, `/tasks`) |
| `/api/admin/prompts/task/{task}/history` | GET | Version history for a task |
| `/api/admin/prompts/` | POST | Create new prompt version |
| `/api/admin/prompts/{prompt_id}` | GET/PUT | Read / update (activation via PUT) |

**Note**: See `docs/architecture/LLM_PROMPTS.md` for prompt evolution history.

#### Admin System (`/api/admin/system`)

System management, worker logs, audit trails.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/system/workers` | GET | Worker status and logs |
| `/api/admin/system/audit` | GET | Decision audit (`/audit/stats`, `POST /audit/{decision_id}/verify`) |
| `/api/admin/system/cache/stats` | GET | Cache stats (`POST /cache/clear`) |
| `/api/admin/system/enrichment/tasks` | GET | Enrichment task status |

Also under `/api/admin/`: `fact-check/*` (see [Fact-Check](fact-check.md)), `users/*`, `config/*`, `feeds/*`, `export/*`, `stats/*`, `kanban/*`, `comments/*`, message moderation actions and `POST /api/admin/check-role` (Oathkeeper). The authoritative list is `/docs` on a running instance, or [API Endpoints](../../reference/api-endpoints.md).

### System Endpoints

#### System Router (`/api/system`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/system.py`

Health checks and service status.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Health check (Docker healthcheck) |
| `/api/health` | GET | Health plus hardware config (`/api/health/hardware`) |
| `/api/system/health` | GET | Detailed service health (`/health/llm` for the LLM gateway) |
| `/api/system/pipeline` | GET | Pipeline state |
| `/api/system/audit` | GET | Decision audit (`/audit/stats`, `POST /audit/{decision_id}/verify`) |

#### Metrics Router (`/metrics`)

**File**: `services/api/src/main.py` (Prometheus counters) and `routers/metrics/` (`/api/metrics/*`: pipeline, services, llm, overview)

Prometheus metrics.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/metrics` | GET | Prometheus metrics (request counts, latency) |

**Metrics Exposed**:
- `api_requests_total{method, endpoint, status}` - Request counts
- `api_request_duration_seconds{method, endpoint}` - Latency histogram

#### About Router (`/api/about`)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/about.py`

Public platform statistics for "About" page.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/about/stats` | GET | Platform stats (messages, channels, entities) |
| `/api/about/activity` | GET | Recent activity |

## Authentication

Authentication is handled by one middleware (`middleware/auth_unified.py`) that tries, in order: API key (`Bearer ak_*` or `?api_key=`), Oathkeeper headers from a trusted proxy, Kratos session cookie, JWT bearer token, then anonymous. `AUTH_PROVIDER` selects the provider used by the factory:

### Authentication Providers

**Configuration** (environment variables):
- `AUTH_PROVIDER`: `none` (dev), `jwt` (simple), `cloudron` (Cloudron OAuth), `ory` (Ory Kratos)
- `AUTH_REQUIRED`: `true` or `false` (default: false)

#### 1. None Provider (Development)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/auth/none.py`

No authentication required. Returns anonymous user.

**Usage**: Development and testing only.

#### 2. JWT Provider (Simple Auth)

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/auth/jwt.py`

JWT token-based authentication.

**Login Flow**:
```bash
# 1. Login to get token
curl -X POST "http://localhost:8000/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "your_password"}'

# 2. Use token in subsequent requests
curl "http://localhost:8000/api/messages" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Environment Variables**:
- `JWT_SECRET_KEY`: Secret key for signing tokens (required, min 32 chars)
- `JWT_ALGORITHM`: Algorithm (default: HS256)
- `JWT_EXPIRATION_MINUTES`: Token expiry (default: 60)
- `JWT_ADMIN_PASSWORD`: Default admin password

#### 3. Cloudron OAuth

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/auth/cloudron.py`

Integration with Cloudron OAuth proxy.

**Headers**: `X-Cloudron-User-Email`, `X-Cloudron-User-ID`

#### 4. Ory Kratos/Oathkeeper (Production)

**File**: `services/api/src/middleware/auth_unified.py` (Kratos whoami via `KRATOS_PUBLIC_URL`, default `http://kratos:4433`)

Enterprise SSO via Ory Kratos identity provider and Oathkeeper reverse proxy.

**Headers**: `X-User-ID`, `X-User-Email`, `X-User-Role`

**Setup**: See `docs/ORY_AUTHENTICATION_DEPLOYMENT.md`

### Authentication Middleware

**File**: `services/api/src/middleware/auth_unified.py` (the older `auth.py` carries the bypass lists summarized below)

Global middleware that enforces authentication rules:

**Hybrid Read/Write Model**:
- **GET requests**: Anonymous access allowed (public read)
- **POST/PUT/DELETE/PATCH**: Authentication required (when `AUTH_REQUIRED=true`; admin routes also check the admin role)

**Bypass Routes**:
- `/health`, `/docs`, `/redoc`, `/openapi.json`, `/api/admin/check-role`: No auth required
- `/rss/*`: Public RSS feeds (feed tokens when `FEED_AUTH_REQUIRED=true`, the compose default)
- `/api/media/internal/`, `/api/about/`, `/api/health`, `/api/timeline`: public
- OPTIONS requests: CORS preflight, no auth

### Feed Token Authentication

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/auth/feed_auth.py`

Optional token-based authentication for RSS feeds.

**Usage**: Append `?token=YOUR_TOKEN` to RSS URLs for private feeds.

**Generation**: POST to `/api/feed-tokens` (requires user authentication)

## Key Features

### Full-Text Search

**Implementation**: PostgreSQL `tsvector` with GIN index on `messages.content_search_vector`

**How It Works**:
1. User submits query: `q=Bakhmut artillery`
2. API converts to `ts_query`: `Bakhmut & artillery`
3. PostgreSQL matches against `content_search_vector` column
4. Results sorted by relevance rank (`ts_rank`)

**Performance**: <100ms for most queries with proper indexes

### Semantic Search

**Implementation**: pgvector cosine similarity on 384-dimensional embeddings

**Flow**:
```mermaid
graph LR
    A[User Query] --> B[Embedding Model via Ollama]
    B --> C[384-dim Vector]
    C --> D[pgvector Cosine Similarity]
    D --> E[Top N Results]

    style B fill:#f9f,stroke:#333
    style D fill:#bbf,stroke:#333
```

**Embedding Model**: configured via the `model_configuration` table (384-dimensional vectors; the enrichment embedding task must use the same model)

**Similarity Threshold**: 0.6 (`/search`) or 0.7 (`/similar`) by default, adjustable via `threshold`

**Use Case**: Find messages by meaning, not keywords. Example: "civilian casualties" matches "non-combatant losses".

### Hybrid Search

**Combine** full-text + semantic search for best results:

1. Full-text search for keyword precision
2. Semantic search for conceptual matches
3. Merge results with weighted scoring

**Endpoint**: `/api/search?mode=semantic` (unified search router)

### Dynamic RSS Feeds

**"Subscribe to Any Search"**: Convert any message search into an RSS feed.

**Example**:
```bash
# Subscribe to high-importance artillery messages in RSS reader
https://api.example.com/rss/search?q=artillery&importance_level=high&format=rss
```

**Formats**:
- `?format=rss`: RSS 2.0 XML (default)
- `?format=atom`: Atom 1.0 XML
- `?format=json`: JSON Feed 1.1

**Caching**: Redis TTL 5-15 minutes based on popularity

**Media Handling**:
- **RSS 2.0**: Media enclosures with MIME types
- **Atom 1.0**: Embedded HTML with `<img>`, `<video>` tags
- **JSON Feed**: Attachments array with size metadata

**Feed Generator** (`feed_generator.py`):
- Content-addressed media URLs (MinIO S3 storage)
- Fallback to Telegram URLs for legacy media
- Proper MIME types and file sizes
- Rich metadata (importance badges, entities, sentiment)

### Social Graph Analysis

**Track** information propagation across Telegram:

**Forward Chains**:
- Who forwarded this message?
- How many hops from original?
- Forward velocity (spread rate)

**Reply Threads**:
- Conversation structure
- Discussion participants
- Reply depth and breadth

**Engagement Metrics**:
- View counts over time
- Reaction sentiment (emoji analysis)
- Comment activity

**Graph Response**:
```json
{
  "nodes": [
    {"id": "msg-123", "type": "message", "data": {...}},
    {"id": "user-456", "type": "author", "data": {...}},
    {"id": "msg-789", "type": "forward", "data": {...}}
  ],
  "edges": [
    {"source": "user-456", "target": "msg-123", "type": "authored"},
    {"source": "msg-123", "target": "msg-789", "type": "forward"}
  ]
}
```

### Multi-Model AI Management

**Runtime Model Switching**: Change active LLM model without service restart.

**How It Works**:
1. Models configured in `model_configuration` table
2. Task assignments: classification, embedding, translation
3. API reads configuration on each request
4. Ollama serves the embedding model

**Supported Tasks**: see `ModelTask` in `shared/python/models/llm_model.py` (`/api/models/tasks/{task}`). The embedding task is the one the API depends on at request time.

**Health Checks**: `/api/models/health/{model_id}` verifies model availability. LLM gateway health is at `/api/system/health/llm`.

### Export Worker

**Background Service**: Processes CSV/JSON export requests asynchronously.

**File**: `~/code/osintukraine/osint-intelligence-platform/services/api/src/export_worker.py`

**Flow**:
1. User starts an export via `POST /api/admin/export/start`
2. Job row is created with status `pending`
3. The in-process worker polls the DB every 10 seconds and claims it
4. Generates CSV/JSON file
5. Uploads to MinIO (bucket `MINIO_BUCKET_NAME`, key prefix `exports/<user>/`)
6. Updates job status to `completed` with download URL

**Queue**: database-polled (table `export_jobs`, no Redis queue)

**Download**: `GET /api/admin/export/{job_id}/download`

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `POSTGRES_HOST` | PostgreSQL host | `postgres` |
| `POSTGRES_PORT` | PostgreSQL port | `5432` |
| `POSTGRES_DB` | Database name | `osint_platform` |
| `POSTGRES_USER` | Database user | `osint_user` |
| `POSTGRES_PASSWORD` | Database password | *required* |
| `REDIS_URL` | Redis connection URL | `redis://redis:6379/0` |
| `API_PORT` | API server port | `8000` |
| `API_CORS_ORIGINS` | Allowed CORS origins (comma-separated) | compose: `http://localhost,http://localhost:3000` |
| `FEED_AUTH_REQUIRED` | Require feed tokens on `/rss/*` | compose: `true` |
| `KRATOS_PUBLIC_URL` | Kratos public URL for session checks | `http://kratos:4433` |
| `AUTH_PROVIDER` | Auth provider (none/jwt/cloudron/ory) | `none` |
| `AUTH_REQUIRED` | Require authentication | `false` |
| `JWT_SECRET_KEY` | JWT signing secret | *required* |
| `JWT_ALGORITHM` | JWT algorithm | `HS256` |
| `JWT_EXPIRATION_MINUTES` | Token expiry | `60` |
| `MINIO_ENDPOINT` | MinIO endpoint | `minio:9000` |
| `MINIO_ACCESS_KEY` | MinIO access key | `minioadmin` |
| `MINIO_SECRET_KEY` | MinIO secret key | `minioadmin` |
| `MINIO_BUCKET_NAME` | Media bucket name | `osint-media` |
| `MINIO_PUBLIC_URL` | Public MinIO URL | `http://localhost:9000` |
| `OLLAMA_HOST` | Ollama API endpoint | `http://ollama:11434` |

### Settings Module

**File**: `shared/python/config/settings.py` (shared by all services; the API has no module of its own)

Centralized configuration management with Pydantic `BaseSettings`.

**Features**:
- Type validation
- Environment variable loading
- Default value fallbacks
- CORS origins parsing

## Performance

### Query Optimization

**Database Indexes** (from `infrastructure/postgres/init.sql`):
- `messages.content_search_vector` (GIN index for full-text search)
- `messages.content_embedding` (HNSW index `idx_messages_content_embedding`, cosine ops)
- `messages.channel_id`, `messages.telegram_date` (B-tree indexes)
- `messages.importance_level`, `messages.osint_topic` (partial indexes)

**Connection Pooling**:
- SQLAlchemy async engine (pool settings in `services/api/src/database.py` and `shared/python/database/session.py`)

**Caching Strategy**:
- RSS feeds: Redis cache, 5-15 min TTL
- Model configuration: In-memory cache per request
- Static assets: Browser cache headers

### Throughput Metrics

**Indicative performance** (compose runs a single Uvicorn process; scale replicas or add `--workers`):
- Simple queries: 200-500 req/sec
- Full-text search: 100-200 req/sec
- Semantic search: 20-50 req/sec (Ollama bottleneck)
- RSS generation: 50-100 req/sec (with Redis cache)

**Memory Usage**: ~200-500MB per worker

**Latency Targets**:
- P50: <100ms (simple queries)
- P95: <500ms (full-text search)
- P99: <2s (semantic search)

### Monitoring

**Prometheus Metrics** (`/metrics`):
```
# Request counts by endpoint and status
api_requests_total{method="GET", endpoint="/api/messages", status="200"} 12345

# Latency histogram by endpoint
api_request_duration_seconds{method="GET", endpoint="/api/messages"} {quantile="0.5"} 0.05
api_request_duration_seconds{method="GET", endpoint="/api/messages"} {quantile="0.95"} 0.2
```

**Grafana Dashboards**: `infrastructure/grafana/dashboards/` (`platform-overview.json`, `osint-pipeline.json`, `sli-slo.json`)

**Health Checks**:
```bash
curl http://localhost:8000/health
```

## Running

### Docker Compose (Recommended)

```bash
# Start API service
docker-compose up -d api

# View logs
docker-compose logs -f api

# Check health
curl http://localhost:8000/health

# Access OpenAPI docs
open http://localhost:8000/docs
```

### Development

```bash
cd services/api

# Install dependencies
pip install -r requirements.txt
pip install -r ../../shared/python/requirements.txt

# Set environment variables
export POSTGRES_HOST=localhost
export POSTGRES_PASSWORD=your_password
export REDIS_URL=redis://localhost:6379/0

# Run with hot reload
uvicorn src.main:app --reload --port 8000

# Or with multiple workers (production)
uvicorn src.main:app --host 0.0.0.0 --port 8000 --workers 4   # the Dockerfile CMD runs a single process
```

### Docker Build

```bash
# Build image
docker build -t osint-api:latest -f services/api/Dockerfile .

# Run container
docker run -p 8000:8000 \
  -e POSTGRES_HOST=postgres \
  -e POSTGRES_PASSWORD=your_password \
  osint-api:latest
```

## Troubleshooting

### "Database connection failed"

**Symptoms**: `OperationalError: could not connect to server`

**Solutions**:
```bash
# 1. Check PostgreSQL is running
docker-compose ps postgres

# 2. Test connection from API container
docker-compose exec api python -c "
from sqlalchemy import create_engine
engine = create_engine('postgresql://osint_user:password@postgres/osint_platform')
engine.connect()
print('OK')
"

# 3. Check environment variables
docker-compose exec api env | grep POSTGRES
```

### "Slow queries"

**Symptoms**: High latency, timeouts

**Diagnosis**:
```sql
-- Check missing indexes
SELECT schemaname, tablename, indexname
FROM pg_indexes
WHERE tablename = 'messages';

-- Analyze slow query
EXPLAIN ANALYZE SELECT * FROM messages WHERE content ILIKE '%keyword%';

-- Check index usage
SELECT indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE tablename = 'messages';
```

**Solutions**:
1. Ensure indexes exist (check `infrastructure/postgres/init.sql`)
2. Use full-text search (`content_search_vector`) instead of ILIKE
3. Add filters to reduce result set (channel, date range)
4. Increase connection pool size if connection exhaustion

### "CORS errors in browser"

**Symptoms**: `Access-Control-Allow-Origin` errors in browser console

**Solutions**:
```bash
# 1. Set API_CORS_ORIGINS in .env
API_CORS_ORIGINS=http://localhost:3000,https://yourdomain.com

# 2. Wildcard (*) is rejected with a security warning: list explicit origins

# 3. Check middleware order in main.py
# CORS middleware must execute BEFORE auth middleware
# (middleware executes in reverse order of registration)
```

### "Semantic search returns no results"

**Symptoms**: `/api/semantic/search` returns empty results

**Diagnosis**:
```bash
# 1. Check if embeddings exist
docker-compose exec postgres psql -U osint_user -d osint_platform -c \
  "SELECT COUNT(*) FROM messages WHERE content_embedding IS NOT NULL;"

# 2. Check Ollama is running and model loaded
curl http://localhost:11434/api/tags

# 3. Check embedding model configured
docker-compose exec postgres psql -U osint_user -d osint_platform -c \
  "SELECT * FROM model_configuration WHERE task = 'embedding';"
```

**Solutions**:
1. Ensure enrichment service has generated embeddings
2. Verify Ollama is running and the embedding model is pulled (search embeddings are the API's one Ollama dependency)
3. Check model configuration in database
4. Lower the `threshold` parameter (default 0.6)

### "RSS feed shows 'Feed not found'"

**Symptoms**: RSS readers cannot fetch feed

**Diagnosis**:
```bash
# 1. Test feed URL directly
curl -v "http://localhost:8000/rss/search?q=test"

# 2. Check feed authentication (if enabled)
curl -v "http://localhost:8000/rss/search?q=test&token=YOUR_TOKEN"

# 3. Check logs for errors
docker-compose logs api | grep rss
```

**Solutions**:
1. Verify query parameters are URL-encoded
2. Check feed token if authentication required
3. Ensure MinIO is accessible for media URLs
4. Check Redis cache isn't corrupt (clear with `FLUSHDB`)

### "Authentication always returns 401"

**Symptoms**: Cannot authenticate even with correct credentials

**Diagnosis**:
```bash
# 1. Check AUTH_PROVIDER setting
docker-compose exec api env | grep AUTH

# 2. Test JWT token generation
curl -X POST "http://localhost:8000/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "<JWT_ADMIN_PASSWORD>"}'

# 3. Check JWT_SECRET_KEY is set
docker-compose exec api env | grep JWT_SECRET_KEY
```

**Solutions**:
1. Verify `AUTH_PROVIDER` matches your setup (none/jwt/ory)
2. Ensure `JWT_SECRET_KEY` is set if using JWT auth
3. Check user exists in database for JWT auth
4. Verify token format: `Authorization: Bearer <token>`

### "Export worker not processing jobs"

**Symptoms**: Export jobs stuck in `pending` status

**Diagnosis**:
```bash
# 1. Check export worker logs
docker-compose logs api | grep export_worker

# 2. Check recent jobs (the worker polls the DB every 10 s)
docker-compose exec postgres psql -U osint_user -d osint_platform -c "SELECT id, status FROM export_jobs ORDER BY created_at DESC LIMIT 5;"
```

**Solutions**:
1. Restart API service to restart export worker
2. Ensure MinIO is running and the media bucket exists
3. Check Redis connectivity
4. Review export job error messages in database

## Related Documentation

- [Architecture Overview](../architecture.md) - Platform architecture
- [Frontend Service](frontend.md) - Next.js frontend
- [Processor Service](processor.md) - Message processing
- [Enrichment Service](enrichment.md) - Background enrichment
- [LLM Integration](../llm-integration.md) - AI classification and prompts
- [Authentication Guide](../../security-guide/authentication.md) - SSO and auth setup
- [Semantic Search Guide](../../user-guide/searching.md) - Search features and pgvector

## Files Reference

All file paths are absolute references to the source code:

**Main Files**:
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/main.py`
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/database.py`
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/feed_generator.py`
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/export_worker.py`

**Routers**:
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/__init__.py`
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/messages.py`
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/search.py`
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/semantic.py`
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/rss.py`
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/entities.py`
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/social_graph.py`

**Authentication**:
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/auth/factory.py`
- `~/code/osintukraine/osint-intelligence-platform/services/api/src/middleware/auth.py`

**Configuration**:
- `~/code/osintukraine/osint-intelligence-platform/services/api/requirements.txt`
- `~/code/osintukraine/osint-intelligence-platform/services/api/Dockerfile`
