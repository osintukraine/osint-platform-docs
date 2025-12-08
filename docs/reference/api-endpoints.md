# API Endpoints

Complete REST API reference for the OSINT Intelligence Platform.

## Overview

**TODO: Content to be generated from codebase analysis**

Base URL: `http://localhost:8000` (development) or `https://your-domain.com` (production)

## Authentication

**TODO: Document authentication methods:**

### Bearer Token

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
  http://localhost:8000/api/messages
```

### API Key

**TODO: Add API key authentication if supported**

## Messages API

### List Messages

**TODO: Document from API code:**

```http
GET /api/messages
```

Query Parameters:

- `q` - Search query
- `channel_id` - Filter by channel
- `entity_id` - Filter by entity
- `tags` - Filter by AI tags
- `from_date` - Start date
- `to_date` - End date
- `limit` - Results per page (default: 50)
- `offset` - Pagination offset

Response:

```json
{
  "total": 1000,
  "limit": 50,
  "offset": 0,
  "messages": [
    {
      "id": 1,
      "text": "Message content",
      "channel_id": 123,
      "created_at": "2024-01-15T10:30:00Z",
      "entities": [...],
      "tags": [...]
    }
  ]
}
```

### Get Message

**TODO: Document endpoint:**

```http
GET /api/messages/{message_id}
```

### Search Messages

**TODO: Document search endpoint:**

```http
POST /api/messages/search
```

## Channels API

### List Channels

**TODO: Document from API code:**

```http
GET /api/channels
```

### Get Channel

```http
GET /api/channels/{channel_id}
```

### Get Channel Statistics

**TODO: Document stats endpoint:**

```http
GET /api/channels/{channel_id}/stats
```

## Entities API

### List Entities

**TODO: Document from API code:**

```http
GET /api/entities
```

Query Parameters:

- `type` - Entity type (person, organization, location)
- `source` - Entity source (armyguide, rootnk, odin, wikidata)
- `q` - Search query

### Get Entity

```http
GET /api/entities/{entity_id}
```

### Get Entity Mentions

**TODO: Document mentions endpoint:**

```http
GET /api/entities/{entity_id}/mentions
```

## RSS Feeds API

### List Feeds

**TODO: Document from RSS ingestor:**

```http
GET /api/rss/feeds
```

### Create Feed

```http
POST /api/rss/feeds
```

Request Body:

```json
{
  "name": "Feed name",
  "description": "Feed description",
  "filters": {
    "channels": [1, 2, 3],
    "entities": [10, 20],
    "tags": ["military", "political"],
    "keywords": ["keyword1", "keyword2"]
  }
}
```

### Get Feed

```http
GET /api/rss/feeds/{feed_id}
```

### Update Feed

```http
PUT /api/rss/feeds/{feed_id}
```

### Delete Feed

```http
DELETE /api/rss/feeds/{feed_id}
```

### Get Feed XML

```http
GET /api/rss/feeds/{feed_id}/xml
```

## Search API

### Full-Text Search

**TODO: Document search API:**

```http
POST /api/search
```

### Semantic Search

**TODO: Document vector search:**

```http
POST /api/search/semantic
```

Request Body:

```json
{
  "query": "natural language query",
  "limit": 20
}
```

## Admin API

### List Users

**TODO: Document admin endpoints:**

```http
GET /api/admin/users
```

### Channel Management

```http
GET /api/admin/channels
POST /api/admin/channels
PUT /api/admin/channels/{channel_id}
DELETE /api/admin/channels/{channel_id}
```

### System Stats

**TODO: Document system statistics:**

```http
GET /api/admin/stats
```

## Health & Status

### Health Check

```http
GET /health
```

Response:

```json
{
  "status": "healthy",
  "version": "1.0.0",
  "services": {
    "database": "healthy",
    "redis": "healthy",
    "minio": "healthy"
  }
}
```

### Metrics

**TODO: Document metrics endpoint if available:**

```http
GET /metrics
```

## WebSocket API

**TODO: Document WebSocket endpoints if available:**

### Real-time Messages

```
ws://localhost:8000/ws/messages
```

## Error Responses

**TODO: Document error response format:**

```json
{
  "error": "Error message",
  "code": "ERROR_CODE",
  "details": {}
}
```

### Common Error Codes

- `400` - Bad Request
- `401` - Unauthorized
- `403` - Forbidden
- `404` - Not Found
- `422` - Validation Error
- `500` - Internal Server Error

## Rate Limiting

**TODO: Document rate limiting:**

- Limit: X requests per minute
- Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`

## Pagination

**TODO: Document pagination pattern:**

All list endpoints support pagination:

- `limit` - Items per page (default: 50, max: 250)
- `offset` - Offset for pagination

## Examples

**TODO: Provide comprehensive API usage examples:**

### Python

```python
# TODO: Add Python examples
import requests

response = requests.get(
    "http://localhost:8000/api/messages",
    headers={"Authorization": f"Bearer {token}"},
    params={"limit": 10}
)
messages = response.json()
```

### cURL

```bash
# TODO: Add cURL examples
```

### JavaScript

```javascript
// TODO: Add JavaScript examples
```

---

!!! note "Documentation Status"
    This page is a placeholder. Content will be generated from FastAPI service code and OpenAPI schema.
