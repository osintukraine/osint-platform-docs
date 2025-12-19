# API Endpoints Reference

Comprehensive documentation of all REST API endpoints in the OSINT Intelligence Platform.

**Base URL**: `http://localhost:8000/api` (development) | `https://your-domain.com/api` (production)

**Authentication**: Most endpoints require Bearer token authentication. See [Authentication](#authentication) section.

---

## Table of Contents

- [Authentication](#authentication)
- [Messages](#messages)
- [Channels](#channels)
- [Entities](#entities)
- [Search](#search)
- [Events](#events)
- [Comments](#comments)
- [Analytics](#analytics)
- [Network & Social Graph](#network-social-graph)
- [Metrics](#metrics)
- [About](#about)
- [Admin Endpoints](#admin-endpoints)
- [Channel Network](#channel-network)
- [News Timeline](#news-timeline)
- [Stream (RSS Intelligence)](#stream-rss-intelligence)
- [Validation (RSS Fact-Checking)](#validation-rss-fact-checking)
- [Admin Dashboard](#admin-dashboard)
- [Admin Kanban](#admin-kanban)
- [Admin Configuration](#admin-configuration)
- [Media Storage](#media-storage)

---

## Authentication

### GET /auth/info

Get authentication configuration info.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/auth/info` | None | Get auth provider and requirements |

**Response Example**:
```json
{
  "provider": "jwt",
  "required": true,
  "login_endpoint": "/api/auth/login"
}
```

### POST /auth/login

Login with username and password (JWT provider only).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/auth/login` | None | Login and get JWT token |

**Request Body**:
```json
{
  "username": "admin",
  "password": "your-password"
}
```

**Response**:
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "token_type": "bearer",
  "expires_in": 43200
}
```

### GET /auth/users/me

Get current authenticated user information.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/auth/users/me` | Required | Get current user info |

**Response**:
```json
{
  "id": "user-123",
  "username": "admin",
  "email": "admin@example.com",
  "display_name": "Admin User",
  "roles": ["admin", "viewer"]
}
```

### POST /auth/users

Create a new user (admin only, JWT provider only).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/auth/users` | Required (admin) | Create new user |

### GET /auth/users

List all users (admin only, JWT provider only).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/auth/users` | Required (admin) | List all users |

### POST /auth/users/me/password

Change current user's password (JWT provider only).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/auth/users/me/password` | Required | Change password |

---

## Messages

### GET /messages

Search messages with filters and pagination.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/messages` | See below | Search messages with filters |

**Query Parameters**:

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `q` | string | Full-text search query | `q=Bakhmut` |
| `channel_id` | integer | Filter by channel ID | `channel_id=123` |
| `channel_username` | string | Filter by channel username | `channel_username=ukraine_news` |
| `channel_folder` | string | Filter by folder pattern | `channel_folder=%UA` (Ukrainian sources) |
| `topic` | string | Filter by OSINT topic | `topic=combat` |
| `has_media` | boolean | Filter messages with media | `has_media=true` |
| `media_type` | string | Filter by media type | `media_type=photo` |
| `is_spam` | boolean | Include spam messages | `is_spam=false` |
| `spam_type` | string | Filter by spam type | `spam_type=financial` |
| `importance_level` | string | Filter by importance | `importance_level=high` |
| `sentiment` | string | Filter by sentiment | `sentiment=urgent` |
| `language` | string | Filter by language | `language=uk` |
| `needs_human_review` | boolean | Flagged for review | `needs_human_review=true` |
| `has_comments` | boolean | Has discussion threads | `has_comments=true` |
| `min_views` | integer | Minimum view count | `min_views=1000` |
| `min_forwards` | integer | Minimum forward count | `min_forwards=100` |
| `date_from` | datetime | Start date | `date_from=2024-01-01T00:00:00Z` |
| `date_to` | datetime | End date | `date_to=2024-12-31T23:59:59Z` |
| `days` | integer | Last N days | `days=7` |
| `page` | integer | Page number (1-indexed) | `page=2` |
| `page_size` | integer | Items per page (1-100) | `page_size=50` |
| `sort_by` | string | Sort field | `sort_by=created_at` |
| `sort_order` | string | Sort order | `sort_order=desc` |

**Example Request**:
```bash
GET /messages?importance_level=high&days=7&page=1&page_size=20
```

### GET /messages/{message_id}

Get a single message by ID.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/messages/{message_id}` | Optional | Get message details |

**Response includes**:
- Message content (original and translated)
- Media URLs and items
- AI-generated tags
- Curated entity matches (from knowledge graph)
- OpenSanctions entity matches (sanctions/PEPs)
- Channel information

### GET /messages/{message_id}/adjacent

Get previous and next message IDs for navigation.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/messages/{message_id}/adjacent` | Optional | Get prev/next message IDs |

**Response**:
```json
{
  "current_id": 12345,
  "prev_id": 12344,
  "next_id": 12346
}
```

### GET /messages/{message_id}/album

Get all media files for a message's album (Telegram grouped media).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/messages/{message_id}/album` | Optional | Get album media for lightbox |

**Response**:
```json
{
  "grouped_id": "67890",
  "album_size": 5,
  "current_index": 2,
  "media": [
    {
      "message_id": 12345,
      "media_id": 1,
      "media_url": "http://localhost:9000/telegram-archive/...",
      "media_type": "photo",
      "mime_type": "image/jpeg",
      "file_size": 245678,
      "content": "Photo caption",
      "telegram_date": "2024-12-09T10:30:00"
    }
  ]
}
```

### GET /messages/{message_id}/network

Build entity relationship graph for a message (Flowsint-style visualization).

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/messages/{message_id}/network` | `include_similar`, `similarity_threshold`, `max_similar` | Get network graph |

**Query Parameters**:
- `include_similar` (boolean, default: true): Include semantically similar messages
- `similarity_threshold` (float, 0.5-1.0, default: 0.8): Minimum similarity score
- `max_similar` (integer, 1-10, default: 5): Max similar messages

**Response includes**:
- Nodes: message, curated entities, AI tags, OpenSanctions entities, similar messages
- Edges: relationships with confidence scores
- Metadata: counts and statistics

---

## Channels

### GET /channels

List channels with filters.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/channels` | See below | List channels |

**Query Parameters**:
- `active_only` (boolean, default: true): Show only active channels
- `rule` (string): Filter by processing rule (archive_all, selective_archive, test, staging)
- `folder` (string): Filter by Telegram folder name
- `verified_only` (boolean, default: false): Only verified channels
- `limit` (integer, 1-500, default: 100): Maximum channels to return

### GET /channels/{channel_id}

Get detailed information about a specific channel.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/channels/{channel_id}` | Optional | Get channel details |

### GET /channels/{channel_id}/stats

Get statistics for a specific channel.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/channels/{channel_id}/stats` | Optional | Get channel statistics |

**Response**:
```json
{
  "total_messages": 12450,
  "spam_messages": 340,
  "archived_messages": 12110,
  "high_importance_count": 1205,
  "messages_by_topic": {
    "combat": 5000,
    "civilian": 3000,
    "equipment": 2000
  },
  "first_message_at": "2024-01-01T00:00:00",
  "last_message_at": "2024-12-09T12:00:00"
}
```

### POST /channels/{channel_id}/backfill

Trigger manual historical backfill for a channel.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/channels/{channel_id}/backfill` | Required | Trigger backfill |

**Request Body**:
```json
{
  "from_date": "2024-01-01T00:00:00Z"
}
```

---

## Entities

### GET /entities/search

Search entities with fuzzy matching (curated + OpenSanctions).

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/entities/search` | `q`, `source`, `entity_type`, `limit` | Search entities |

**Query Parameters**:
- `q` (string, required): Search query
- `source` (string, optional): Filter by source (curated, opensanctions)
- `entity_type` (string, optional): Filter by entity type
- `limit` (integer, 1-100, default: 20): Max results

**Response**:
```json
{
  "items": [
    {
      "id": "123",
      "source": "curated",
      "name": "T-90M Proryv",
      "entity_type": "military_vehicle",
      "description": "Russian main battle tank",
      "score": 0.95
    }
  ],
  "total": 42
}
```

### GET /entities/{source}/{entity_id}

Get entity details by source and ID.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/entities/{source}/{entity_id}` | Optional | Get entity details |

**Path Parameters**:
- `source`: `curated` or `opensanctions`
- `entity_id`: Numeric ID (curated) or Wikidata QID (opensanctions, e.g., "Q3874799")

**Query Parameters**:
- `include_linked` (boolean, default: true): Include linked content counts

### GET /entities/{source}/{entity_id}/messages

Get messages linked to an entity.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/entities/{source}/{entity_id}/messages` | `limit`, `offset` | Get entity messages |

### GET /entities/{source}/{entity_id}/relationships

Get entity relationship graph data (Wikidata SPARQL + OpenSanctions enrichment).

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/entities/{source}/{entity_id}/relationships` | `refresh` | Get relationships |

**Query Parameters**:
- `refresh` (boolean, default: false): Force refresh from Wikidata SPARQL

**Response**:
```json
{
  "entity_id": "Q20850503",
  "entity_name": "Vladimir Putin",
  "cached": true,
  "fetched_at": "2024-12-09T10:00:00Z",
  "expires_at": "2024-12-16T10:00:00Z",
  "corporate": [
    {
      "type": "employer",
      "entity_id": "Q1065",
      "name": "Gazprom",
      "start": "2000",
      "end": "2008"
    }
  ],
  "political": [],
  "associates": [],
  "sources": ["wikidata", "opensanctions"]
}
```

---

## Search

### GET /search

Unified search across all platform data sources.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/search` | See below | Unified search |

**Query Parameters**:
- `q` (string, required): Search query
- `mode` (string, default: "text"): Search mode (text, semantic)
- `types` (string, default: "messages,events,rss,entities"): Comma-separated types
- `limit_per_type` (integer, 1-20, default: 5): Results per type
- `channel_username` (string, optional): Filter by channel
- `channel_folder` (string, optional): Filter by folder
- `days` (integer, optional): Last N days
- `entity_type` (string, optional): Filter entities by type

**Response**:
```json
{
  "query": "Bakhmut",
  "mode": "text",
  "results": {
    "messages": {
      "items": [...],
      "total": 5,
      "has_more": true
    },
    "events": {
      "items": [...],
      "total": 2,
      "has_more": false
    },
    "rss": {
      "items": [...],
      "total": 3,
      "has_more": false
    },
    "entities": {
      "items": [...],
      "total": 1,
      "has_more": false
    }
  },
  "timing_ms": 125
}
```

---

## Events

### GET /events

List events with pagination and filtering.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/events` | See below | List events |

**Query Parameters**:
- `page` (integer, default: 1): Page number
- `page_size` (integer, 1-100, default: 20): Items per page
- `tab` (string, default: "active"): Filter (active, major, archived, all)
- `event_type` (string, optional): Filter by event type
- `tier_status` (string, optional): Filter by tier (breaking, developing, confirmed, verified)
- `search` (string, optional): Text search
- `search_mode` (string, default: "text"): Search mode (text, semantic)
- `similarity_threshold` (float, 0.0-1.0, default: 0.5): Min similarity for semantic search

### GET /events/stats

Get event statistics for dashboard.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/events/stats` | Optional | Get event statistics |

**Response**:
```json
{
  "active": 45,
  "major": 12,
  "archived": 230,
  "by_tier": {
    "breaking": 5,
    "developing": 18,
    "confirmed": 22,
    "verified": 0
  },
  "total": 275
}
```

### GET /events/message/{message_id}

Get all events that a message is linked to.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/events/message/{message_id}` | Optional | Get events for message |

### GET /events/{event_id}

Get event details with sources and linked messages.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/events/{event_id}` | `include_messages`, `include_sources`, `message_limit` | Get event details |

### PATCH /events/{event_id}/major

Toggle major event status.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| PATCH | `/events/{event_id}/major` | `is_major` | Set major status |

### PATCH /events/{event_id}/archive

Archive or unarchive an event.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| PATCH | `/events/{event_id}/archive` | `archive` | Archive/unarchive event |

### GET /events/{event_id}/timeline

Get chronological timeline of all sources for an event (RSS + Telegram).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/events/{event_id}/timeline` | Optional | Get event timeline |

---

## Comments

### GET /comments/{comment_id}

Get a single comment with its translation status.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/comments/{comment_id}` | Optional | Get comment |

### POST /comments/{comment_id}/translate

Translate a comment on-demand.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/comments/{comment_id}/translate` | Optional | Translate comment |

**Response**:
```json
{
  "comment_id": 12345,
  "original_content": "Дякую за інформацію",
  "translated_content": "Thank you for the information",
  "original_language": "uk",
  "translation_method": "google_free",
  "cached": false
}
```

---

## Analytics

### GET /analytics/timeline

Get time-series message statistics for visualizations.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/analytics/timeline` | See below | Get timeline stats |

**Query Parameters**:
- `granularity` (string, default: "day"): Time bucket (hour, day, week, month, year)
- `channel_id` (integer, optional): Filter by channel
- `topic` (string, optional): Filter by topic
- `importance_level` (string, optional): Filter by importance
- `date_from` (datetime, optional): Start date
- `date_to` (datetime, optional): End date
- `days` (integer, optional): Last N days

**Response** (60s cache TTL):
```json
{
  "granularity": "day",
  "buckets": [
    {
      "timestamp": "2024-12-01T00:00:00",
      "message_count": 1250,
      "media_count": 340
    }
  ],
  "total_buckets": 30
}
```

### GET /analytics/distributions

Get statistical distributions for visualizations.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/analytics/distributions` | `metrics`, filters | Get distributions |

**Query Parameters**:
- `metrics` (list[string], default: all): Which distributions (importance_level, topics, channels, media_types, languages)
- Plus filter parameters (channel_id, date_from, date_to, days)

**Response** (5min cache TTL):
```json
{
  "importance_level": {
    "high": 1200,
    "medium": 5400,
    "low": 3400
  },
  "topics": {
    "combat": 4000,
    "civilian": 2000
  },
  "channels": {
    "123": 1500,
    "456": 1200
  }
}
```

### GET /analytics/heatmap

Get activity heatmap for calendar visualization (day of week × hour of day).

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/analytics/heatmap` | `channel_id`, `days` | Get activity heatmap |

**Response** (5min cache TTL):
```json
{
  "heatmap": [
    [45, 23, 12, ...],  // Sunday, 24 hours
    [67, 34, 18, ...]   // Monday, 24 hours
  ],
  "days": ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"],
  "hours": [0, 1, 2, ..., 23],
  "total_messages": 12450
}
```

### GET /analytics/channels

Get channel performance analytics using materialized view.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/analytics/channels` | `channel_id`, `limit`, `order_by`, `days` | Get channel analytics |

**Query Parameters**:
- `channel_id` (integer, optional): Specific channel
- `limit` (integer, 1-100, default: 20): Number of channels
- `order_by` (string, default: "messages"): Sort field (messages, spam_rate, importance)
- `days` (integer, 1-365, default: 30): Last N days

### GET /analytics/entities

Get entity mention analytics.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/analytics/entities` | `entity_type`, `limit`, `days` | Get entity analytics |

### GET /analytics/media

Get media archival analytics.

| Method | Path | Query Parameters | Description |
|--------|------|------|-------------|
| GET | `/analytics/media` | `days` | Get media statistics |

**Response**:
```json
{
  "total_files": 45678,
  "total_size_bytes": 45678901234,
  "total_size_human": "42.5 GB",
  "by_type": [
    {
      "media_type": "image",
      "count": 30000,
      "total_size_bytes": 20000000000,
      "total_size_human": "18.6 GB",
      "percentage": 43.8
    }
  ],
  "deduplication_savings_bytes": 5000000000,
  "deduplication_savings_human": "4.7 GB"
}
```

---

## Network & Social Graph

### GET /social-graph/messages/{message_id}

Get social graph for a message (forwards, replies, reactions, author).

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/social-graph/messages/{message_id}` | `include_forwards`, `include_replies`, `max_depth`, `max_comments` | Get message social graph |

**Query Parameters**:
- `include_forwards` (boolean, default: true): Include forward chain
- `include_replies` (boolean, default: true): Include reply thread
- `max_depth` (integer, 1-10, default: 3): Max graph depth
- `max_comments` (integer, 1-200, default: 50): Max comments to include

**Response includes**:
- Nodes: message, author, forwards, replies, reactions, comments
- Edges: relationships (authored, forwarded_to, replied_to, reacted, commented_on)
- Reactions: emoji sentiment data
- Metadata: engagement metrics (views, forwards, virality, reach)

### GET /social-graph/channels/{channel_id}/influence

Get channel influence network (who forwards from/to this channel).

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/social-graph/channels/{channel_id}/influence` | `limit`, `min_forward_count` | Get channel influence |

### GET /social-graph/influence-network

Get platform-wide channel influence network.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/social-graph/influence-network` | `min_forward_count`, `limit` | Get influence network |

**Uses** `channel_influence_network` materialized view for performance.

### GET /social-graph/messages/{message_id}/engagement-timeline

Get engagement timeline (views, forwards, reactions over time).

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/social-graph/messages/{message_id}/engagement-timeline` | `granularity`, `time_range_hours` | Get engagement timeline |

### GET /social-graph/messages/{message_id}/comments

Get comment thread for a message.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/social-graph/messages/{message_id}/comments` | `limit`, `offset`, `sort`, `include_replies` | Get comments |

### GET /social-graph/virality/top-forwarded

Get most-forwarded messages (virality leaderboard).

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/social-graph/virality/top-forwarded` | `limit`, `time_range_days` | Get top viral messages |

**Uses** `top_forwarded_messages` materialized view.

---

## Metrics

All metrics endpoints are cached in Redis for 15 seconds.

### GET /metrics/overview

Get high-level platform operational metrics.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/metrics/overview` | Optional | Get platform overview metrics |

**Response**:
```json
{
  "timestamp": "2024-12-09T12:00:00Z",
  "messages_per_second": 2.5,
  "messages_archived_per_second": 2.3,
  "messages_skipped_per_second": 0.2,
  "queue_depth": 45,
  "enrichment_queue_depth": 120,
  "queue_lag_seconds": 5.2,
  "llm_requests_per_minute": 45.6,
  "llm_avg_latency_seconds": 1.2,
  "llm_success_rate_percent": 99.8,
  "database_connections": 25,
  "redis_memory_mb": 128.5,
  "enrichment_error_rate": 0.002,
  "spam_rate_percent": 2.8,
  "services_healthy": 4,
  "services_total": 4,
  "prometheus_available": true,
  "cached": false,
  "cache_ttl_seconds": 15
}
```

### GET /metrics/llm

Get LLM (Ollama) performance metrics.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/metrics/llm` | Optional | Get LLM metrics |

**Response**:
```json
{
  "timestamp": "2024-12-09T12:00:00Z",
  "requests_per_minute": 45.6,
  "requests_total": 125430,
  "avg_latency_seconds": 1.2,
  "p50_latency_seconds": 1.1,
  "p95_latency_seconds": 2.3,
  "p99_latency_seconds": 3.5,
  "success_rate_percent": 99.8,
  "error_count": 25,
  "avg_batch_size": 10.5,
  "total_batches": 11945,
  "active_model": "qwen2.5:3b",
  "model_loaded": true,
  "prometheus_available": true
}
```

### GET /metrics/pipeline

Get real-time pipeline metrics for architecture visualization.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/metrics/pipeline` | Optional | Get pipeline metrics |

**Response includes**:
- Overall status (healthy, degraded, down)
- Stages: listener, redis-queue, processor, postgres, enrichment, api
- Enrichment workers detail (per task)
- KPIs: messages/sec, archive rate, queue depth, LLM performance

### GET /metrics/services

Get per-service health and performance metrics.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/metrics/services` | Optional | Get service metrics |

**Response**:
```json
{
  "timestamp": "2024-12-09T12:00:00Z",
  "total_services": 11,
  "healthy_count": 10,
  "degraded_count": 1,
  "down_count": 0,
  "services": [
    {
      "name": "listener",
      "display_name": "Telegram Listener",
      "status": "healthy",
      "category": "core",
      "requests_per_second": 2.5,
      "up": true
    }
  ]
}
```

---

## About

### GET /about/stats

Get platform statistics for the About page.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/about/stats` | None | Get public platform stats |

**Response**:
```json
{
  "channels": 254,
  "messages": 1254000,
  "messages_formatted": "1.3M",
  "media_size_bytes": 45678901234,
  "media_size_formatted": "42.5 GB",
  "entities": 1425,
  "spam_blocked": 34520,
  "spam_blocked_formatted": "34.5K",
  "sanctions_matches": 1234,
  "timestamp": "2024-12-09T12:00:00Z"
}
```

---

## Admin Endpoints

All admin endpoints require authentication.

### LLM Prompts

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/admin/prompts` | `page`, `page_size`, `task`, `is_active`, `prompt_type`, `search` | List prompts |
| GET | `/admin/prompts/stats` | None | Get prompt statistics |
| GET | `/admin/prompts/tasks` | None | Get distinct task names |
| GET | `/admin/prompts/{prompt_id}` | None | Get prompt details |
| GET | `/admin/prompts/task/{task}/history` | None | Get version history |
| POST | `/admin/prompts` | None | Create new prompt version |
| PUT | `/admin/prompts/{prompt_id}` | None | Update prompt metadata |

### Comments (Admin)

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| POST | `/admin/comments/fetch` | None | Fetch comments on-demand |
| GET | `/admin/comments/stats` | None | Get comment statistics |
| GET | `/admin/comments/viral` | `active_only`, `limit` | Get viral posts |

### Export

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/admin/export/profiles` | None | Get export profiles |
| POST | `/admin/export/estimate` | None | Estimate export size |
| POST | `/admin/export/start` | None | Start export job |
| GET | `/admin/export/jobs` | `page`, `page_size`, `status` | List export jobs |
| GET | `/admin/export/{job_id}` | None | Get export job status |
| GET | `/admin/export/{job_id}/download` | `token` | Download export file |
| DELETE | `/admin/export/{job_id}` | None | Cancel/delete export job |
| GET | `/admin/export/stats/summary` | None | Get export statistics |

### Statistics

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/admin/stats/overview` | None | Platform overview (30s cache) |
| GET | `/admin/stats/quality` | None | Data quality metrics (60s cache) |
| GET | `/admin/stats/processing` | `hours` | Processing metrics |
| GET | `/admin/stats/storage` | None | Storage usage metrics |

### Spam Management

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/admin/spam` | `page`, `page_size`, `status`, `spam_type`, `channel` | Get spam queue |
| GET | `/admin/spam/stats` | None | Get spam statistics |
| PUT | `/admin/spam/{message_id}/review` | `status` | Review spam message |
| POST | `/admin/spam/bulk-review` | None | Bulk review spam |
| POST | `/admin/spam/{message_id}/reprocess` | None | Reprocess message |
| DELETE | `/admin/spam/{message_id}` | None | Delete spam message |
| POST | `/admin/spam/bulk-delete` | None | Bulk delete spam |
| DELETE | `/admin/spam/purge/confirmed` | None | Purge confirmed spam |

### System Management

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/admin/system/workers` | None | Get worker status |
| GET | `/admin/system/workers/stats` | None | Get worker statistics |
| GET | `/admin/system/audit` | `page`, `page_size`, `decision_type`, `verification_status`, `channel_id` | Get audit log |
| GET | `/admin/system/audit/stats` | None | Get audit statistics |
| POST | `/admin/system/audit/{decision_id}/verify` | `status`, `notes` | Verify audit decision |
| GET | `/admin/system/cache/stats` | None | Get Redis cache stats |
| POST | `/admin/system/cache/clear` | `pattern` | Clear cache keys |
| GET | `/admin/system/enrichment/tasks` | None | Get enrichment task status |

### RSS Feeds Management

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/admin/feeds/rss` | `page`, `page_size`, `search`, `category`, `trust_level`, `active`, `sort_by`, `sort_order` | List RSS feeds |
| GET | `/admin/feeds/rss/stats` | None | Get feed statistics |
| GET | `/admin/feeds/rss/categories` | None | Get feed categories |
| GET | `/admin/feeds/rss/{feed_id}` | None | Get feed details |
| POST | `/admin/feeds/rss` | None | Create feed |
| PUT | `/admin/feeds/rss/{feed_id}` | None | Update feed |
| DELETE | `/admin/feeds/rss/{feed_id}` | None | Delete feed |
| POST | `/admin/feeds/rss/test` | `url` | Test feed URL |
| POST | `/admin/feeds/rss/{feed_id}/poll` | None | Trigger feed poll |
| POST | `/admin/feeds/rss/batch/activate` | None | Batch activate/deactivate |

---

## Channel Network

### GET /channel-network/{channel_id}/network

Get entity and content network graph for a specific channel.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/channel-network/{channel_id}/network` | See below | Get channel content network |

**Query Parameters**:
- `min_similarity` (float, 0.0-1.0, default: 0.7): Minimum similarity score for connections
- `max_entities` (integer, 1-100, default: 50): Maximum entities to include
- `days` (integer, 1-365, default: 30): Time window for analysis

**Response includes**:
- Nodes: channel, entities, topics, key messages
- Edges: content relationships with similarity scores
- Statistics: entity distribution, topic breakdown

---

## News Timeline

### GET /news-timeline

Get unified timeline of RSS articles + Telegram messages with correlations.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/news-timeline` | See below | Get news timeline |

**Query Parameters**:
- `page` (integer, default: 1): Page number
- `page_size` (integer, 1-100, default: 20): Items per page
- `source_type` (string, optional): Filter by source (rss, telegram, both)
- `category` (string, optional): Filter by news category
- `trust_level` (integer, 1-5, optional): Filter by trust level
- `days` (integer, 1-365, default: 7): Time window
- `search` (string, optional): Text search

**Response**:
```json
{
  "items": [
    {
      "type": "rss",
      "id": 12345,
      "title": "Article headline",
      "published_at": "2024-12-09T10:00:00Z",
      "source_name": "Kyiv Independent",
      "trust_level": 5,
      "correlation_count": 3,
      "correlations": [...]
    },
    {
      "type": "telegram",
      "id": 67890,
      "content": "Message content...",
      "channel_name": "Ukraine News",
      "telegram_date": "2024-12-09T10:05:00Z",
      "validation_status": "confirmed"
    }
  ],
  "total": 150,
  "page": 1
}
```

### GET /news-timeline/stats

Get news timeline statistics.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/news-timeline/stats` | Optional | Get timeline statistics |

**Response**:
```json
{
  "rss_articles_24h": 245,
  "telegram_messages_24h": 1250,
  "correlations_24h": 180,
  "avg_correlation_confidence": 0.78,
  "by_category": {
    "military": 450,
    "political": 200
  }
}
```

---

## Stream (RSS Intelligence)

### GET /stream/unified

Get unified intelligence stream (RSS + Telegram correlated content).

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/stream/unified` | See below | Get unified stream |

**Query Parameters**:
- `limit` (integer, 1-100, default: 50): Maximum items
- `source_filter` (string, optional): Filter sources (rss, telegram, all)
- `hours` (integer, 1-168, default: 24): Time window
- `min_importance` (string, optional): Minimum importance level

**Response includes**:
- RSS articles with telegram correlations
- Telegram messages with RSS validations
- Validation status (confirms, contradicts, context)
- Confidence scores

### GET /stream/correlations/{message_id}

Get RSS correlations for a specific Telegram message.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/stream/correlations/{message_id}` | `min_similarity`, `limit` | Get message correlations |

**Query Parameters**:
- `min_similarity` (float, 0.0-1.0, default: 0.5): Minimum similarity score
- `limit` (integer, 1-20, default: 10): Maximum correlations

**Response**:
```json
{
  "message_id": 12345,
  "correlations": [
    {
      "article_id": 6789,
      "title": "Related article headline",
      "url": "https://example.com/article",
      "published_at": "2024-12-09T09:00:00Z",
      "similarity_score": 0.82,
      "validation_type": "confirms",
      "confidence": 0.85,
      "source_trust_level": 4
    }
  ],
  "total_correlations": 3
}
```

---

## Validation (RSS Fact-Checking)

### GET /validation/{message_id}/validation

Get LLM-powered validation summary for a message.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/validation/{message_id}/validation` | `refresh`, `min_similarity` | Get validation summary |

**Query Parameters**:
- `refresh` (boolean, default: false): Force regenerate validation (ignore cache)
- `min_similarity` (float, 0.3-1.0, default: 0.5): Minimum article similarity

**Response**:
```json
{
  "message_id": 12345,
  "validation_summary": "This message's claims about the strike in Kharkiv are confirmed by 3 independent news sources including Reuters and Kyiv Independent.",
  "confidence_score": 0.89,
  "total_articles_found": 5,
  "articles_analyzed": 3,
  "validation_breakdown": {
    "confirms": 3,
    "contradicts": 0,
    "provides_context": 2
  },
  "cached": true,
  "cached_at": "2024-12-09T10:00:00Z",
  "expires_at": "2024-12-09T22:00:00Z"
}
```

**Notes**:
- Validation summaries are cached for 12 hours
- LLM generates human-readable synthesis of correlated articles
- Uses RSS correlation + LLM analysis for classification

---

## Admin Dashboard

### GET /admin/dashboard

Get comprehensive admin dashboard data.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/admin/dashboard` | Required (admin) | Get dashboard data |

**Response**:
```json
{
  "platform_status": "healthy",
  "overview": {
    "total_messages": 1250000,
    "total_channels": 254,
    "active_channels": 245,
    "messages_today": 12500,
    "messages_this_hour": 520
  },
  "processing": {
    "queue_depth": 45,
    "messages_per_second": 2.5,
    "spam_rate_percent": 2.8,
    "llm_latency_ms": 1200
  },
  "enrichment": {
    "pending_embedding": 150,
    "pending_translation": 80,
    "pending_tagging": 200
  },
  "storage": {
    "database_size_gb": 45.6,
    "media_size_gb": 42.5,
    "redis_memory_mb": 128
  }
}
```

### POST /admin/actions/{action}

Trigger admin actions (clear cache, restart services, etc.).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/admin/actions/{action}` | Required (admin) | Trigger admin action |

**Actions**:
- `clear_cache`: Clear Redis cache
- `refresh_materialized_views`: Refresh all materialized views
- `trigger_enrichment`: Manually trigger enrichment cycle

---

## Admin Kanban

### GET /admin/kanban

Get urgency-based kanban board for message prioritization.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/admin/kanban` | See below | Get kanban board |

**Query Parameters**:
- `channel_id` (integer, optional): Filter by channel
- `days` (integer, 1-30, default: 7): Time window

**Response**:
```json
{
  "columns": {
    "critical": {
      "messages": [...],
      "count": 5
    },
    "high": {
      "messages": [...],
      "count": 12
    },
    "medium": {
      "messages": [...],
      "count": 45
    },
    "low": {
      "messages": [...],
      "count": 200
    }
  },
  "total": 262
}
```

### GET /admin/kanban/stats

Get kanban statistics.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/admin/kanban/stats` | Required (admin) | Get kanban stats |

---

## Admin Configuration

### GET /admin/config

List all platform configuration values.

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/admin/config` | `category`, `search` | List config values |

**Query Parameters**:
- `category` (string, optional): Filter by category (system, features, thresholds, etc.)
- `search` (string, optional): Search in key/description

### GET /admin/config/categories

Get all configuration categories.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/admin/config/categories` | Required (admin) | Get categories |

### GET /admin/config/{key}

Get a specific configuration value.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/admin/config/{key}` | Required (admin) | Get config value |

### PUT /admin/config/{key}

Update a configuration value.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| PUT | `/admin/config/{key}` | Required (admin) | Update config |

**Request Body**:
```json
{
  "value": "new_value"
}
```

### PUT /admin/config/bulk/update

Bulk update multiple configuration values.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| PUT | `/admin/config/bulk/update` | Required (admin) | Bulk update |

**Request Body**:
```json
{
  "updates": [
    {"key": "features.spam_filter_enabled", "value": "true"},
    {"key": "thresholds.min_importance", "value": "medium"}
  ]
}
```

### Model Configuration

| Method | Path | Query Parameters | Description |
|--------|------|------------------|-------------|
| GET | `/admin/config/models` | None | List model configurations |
| GET | `/admin/config/models/tasks` | None | Get available tasks |
| PUT | `/admin/config/models/{config_id}` | None | Update model config |
| POST | `/admin/config/models` | None | Create model config |
| DELETE | `/admin/config/models/{config_id}` | None | Delete model config |

---

## Map API

The Map API provides GeoJSON endpoints for the map interface, supporting real-time geolocation visualization and event detection. Part of Event Detection V3.

**Performance Features**:
- Server-side point clustering (zoom < 12)
- Redis caching: 60s messages, 300s clusters, 180s heatmap
- Spatial index usage for bbox queries
- Real-time WebSocket updates

**Important**: Database stores `(latitude, longitude)` but GeoJSON uses `[longitude, latitude]`.

### GET /api/map/messages

Get geocoded messages within bounding box as GeoJSON FeatureCollection.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/map/messages` | Optional | Get geocoded messages |

**Query Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `south` | float | Required | South boundary latitude |
| `west` | float | Required | West boundary longitude |
| `north` | float | Required | North boundary latitude |
| `east` | float | Required | East boundary longitude |
| `zoom` | integer | None | Map zoom level (0-22) |
| `cluster` | boolean | `false` | Enable server-side clustering |
| `limit` | integer | `500` | Maximum messages (1-2000) |
| `min_confidence` | float | `0.5` | Minimum confidence (0.0-1.0) |
| `start_date` | datetime | None | Start date (ISO 8601) |
| `end_date` | datetime | None | End date (ISO 8601) |

**Example**:
```bash
curl "http://localhost:8000/api/map/messages?south=48&west=35&north=50&east=40&limit=500"
```

### GET /api/map/clusters

Get event clusters within bounding box as GeoJSON FeatureCollection.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/map/clusters` | Optional | Get event clusters by tier |

**Query Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `south` | float | Required | South boundary latitude |
| `west` | float | Required | West boundary longitude |
| `north` | float | Required | North boundary latitude |
| `east` | float | Required | East boundary longitude |
| `tier` | string | None | Filter: rumor, unconfirmed, confirmed, verified |
| `limit` | integer | `200` | Maximum clusters (1-1000) |

**Tier Colors**: rumor (red), unconfirmed (yellow), confirmed (orange), verified (green)

### GET /api/map/clusters/{cluster_id}/messages

Get messages for a specific cluster (expansion data).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/map/clusters/{id}/messages` | Optional | Get cluster messages |

### GET /api/map/heatmap

Get aggregated heatmap data for message density visualization.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/map/heatmap` | Optional | Get density aggregation |

**Query Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `south` | float | Required | South boundary |
| `west` | float | Required | West boundary |
| `north` | float | Required | North boundary |
| `east` | float | Required | East boundary |
| `grid_size` | float | `0.1` | Grid cell size in degrees |

### GET /api/map/locations/suggest

Location autocomplete for frontend search.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/map/locations/suggest` | Optional | Location autocomplete |

**Query Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `q` | string | Required | Location name prefix (min 2 chars) |
| `limit` | integer | `10` | Maximum suggestions (1-50) |

### GET /api/map/locations/reverse

Reverse geocoding - find nearest location to coordinates.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/map/locations/reverse` | Optional | Reverse geocode |

**Query Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `lat` | float | Required | Latitude |
| `lng` | float | Required | Longitude |

### WS /api/map/ws/map/live

WebSocket endpoint for real-time map updates.

| Method | Path | Description |
|--------|------|-------------|
| WS | `/api/map/ws/map/live` | Real-time location updates |

**Query Parameters**: `south`, `west`, `north`, `east` (bounding box)

**Message Types**:
- `feature`: New geocoded message (GeoJSON)
- `heartbeat`: Keep-alive (every 30s)

---

## Common Response Patterns

### Pagination

```json
{
  "items": [...],
  "total": 1250,
  "page": 2,
  "page_size": 50,
  "total_pages": 25,
  "has_next": true,
  "has_prev": true
}
```

### Error Response

```json
{
  "detail": "Message not found",
  "status_code": 404
}
```

### Cache Headers

```http
Cache-Control: public, max-age=15
X-Cached: true
X-Cache-TTL: 15
X-Prometheus-Available: true
```

---

## Media Storage

Internal API endpoints for media routing and caching. These endpoints are used by Caddy for media delivery.

### GET /media/internal/media-redirect/{file_hash:path}

Route a media request to the appropriate storage location.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/media/internal/media-redirect/{file_hash}` | Internal | Route media to storage box |

**Path Parameters**:

| Name | Type | Description |
|------|------|-------------|
| `file_hash` | string | Media file path (e.g., `ab/cd/abcd1234.jpg`) |

**Response**: HTTP 302 redirect to storage location

**Response Headers**:
```http
HTTP/1.1 302 Found
Location: /storage/default/media/ab/cd/abcd1234.jpg
X-Media-Source: storage-box
X-Cache-Status: hit
```

**Cache Status Values**:

| Value | Description |
|-------|-------------|
| `hit` | Route found in Redis cache |
| `miss` | Route not cached, queried from database |
| `populated` | Cache miss, now populated |

**Error Response** (404):
```json
{
  "detail": "Media file not found"
}
```

---

### POST /media/internal/media-invalidate/{file_hash}

Invalidate the Redis cache entry for a media file.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/media/internal/media-invalidate/{file_hash}` | Internal | Invalidate cache entry |

**Path Parameters**:

| Name | Type | Description |
|------|------|-------------|
| `file_hash` | string | SHA-256 hash of the media file |

**Response** (200):
```json
{
  "status": "ok",
  "sha256": "abcd1234...",
  "cache_deleted": true
}
```

---

### GET /media/internal/media-stats

Get media cache statistics.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/media/internal/media-stats` | Internal | Get cache statistics |

**Response** (200):
```json
{
  "cache_size": 15234,
  "estimated_hit_rate": 0.994,
  "cache_ttl_seconds": 86400
}
```

**Response Fields**:

| Field | Type | Description |
|-------|------|-------------|
| `cache_size` | integer | Number of entries in Redis cache |
| `estimated_hit_rate` | float | Estimated cache hit rate (0.0-1.0) |
| `cache_ttl_seconds` | integer | Cache entry TTL in seconds |

---

### Media Delivery Flow

The media delivery system uses a tiered approach:

```
Browser Request: /media/ab/cd/abcd1234.jpg
         │
         ▼
    ┌─────────┐
    │  Caddy  │
    └─────────┘
         │
    1. Check local buffer (/var/cache/osint-media-buffer)
         │
    ┌────┴────┐
   HIT       MISS
    │         │
    ▼         ▼
  Serve    Call API: /media/internal/media-redirect/ab/cd/abcd1234.jpg
  from          │
  SSD           ▼
           Redis Cache → Database → 302 Redirect
```

**Related Documentation**:

- [Operator Guide: Hetzner Storage](../operator-guide/hetzner-storage.md)
- [Developer Guide: Media Storage](../developer-guide/media-storage.md)

---

## Rate Limiting

Currently no rate limiting is enforced. Future implementations may add:
- 100 requests/minute for authenticated users
- 20 requests/minute for unauthenticated users
- Higher limits for admin endpoints

---

## CORS

CORS is enabled for `http://localhost:3000` (frontend) in development.

Production requires explicit CORS configuration via environment variables.

---

## Changelog

### 2024-12-09
- Added Wikidata relationship enrichment endpoint
- Added on-demand comment translation
- Added export job management endpoints
- Enhanced metrics endpoints with 15s caching
- Added social graph and engagement timeline endpoints

---

**Generated**: 2024-12-09 | **API Version**: 1.0 | **Platform**: OSINT Intelligence Platform
