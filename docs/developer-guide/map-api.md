# Map API Reference

This guide provides complete documentation for the Map API endpoints used to power the interactive map interface in the OSINT Intelligence Platform.

## Overview

The Map API provides GeoJSON endpoints for rendering geocoded messages, event clusters, and real-time location updates on MapLibre GL maps. It is part of Event Detection V3 - the geolocation and cluster detection pipeline.

**Key Features:**

- GeoJSON FeatureCollection responses for MapLibre compatibility
- Real-time updates via WebSocket subscription
- Server-side point clustering at low zoom levels
- Redis caching with automatic invalidation
- Rate limiting to prevent abuse
- Spatial indexing for fast bounding box queries
- Polygon and timeline filtering support

**Base URL:** `http://localhost:8000/api/map` (development)

**Related Documentation:**
- [Architecture Overview](./architecture.md) - System architecture including geolocation pipeline
- [Map Interface User Guide](../user-guide/map-interface.md) - End-user guide for map features
- [API Service](services/api.md)

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                     MAP API REQUEST FLOW                           │
└────────────────────────────────────────────────────────────────────┘

Frontend Map Component
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     GET /api/map/messages                           │
│                                                                     │
│  1. Validate bounding box (south < north)                          │
│  2. Generate cache key (bbox + filters)                            │
│  3. Try Redis cache (60s TTL)                                      │
│     └── HIT? → Return cached GeoJSON                               │
│                                                                     │
│  4. Query PostgreSQL with spatial index:                           │
│     - Zoom < 12 + cluster=true → Server-side clustering            │
│     - Otherwise → Individual points                                │
│                                                                     │
│  5. Build GeoJSON FeatureCollection                                │
│  6. Cache result in Redis                                          │
│  7. Add rate limit headers                                         │
│  8. Return GeoJSON                                                 │
└─────────────────────────────────────────────────────────────────────┘
         │
         ▼
MapLibre GL renders features
```

```
┌────────────────────────────────────────────────────────────────────┐
│                     WEBSOCKET REAL-TIME FLOW                       │
└────────────────────────────────────────────────────────────────────┘

Enrichment Service
  (Geolocation Task)
         │
         ▼
Redis pub/sub: map:new_location
         │
         ▼
WebSocket Handler
  (/ws/map/live)
         │
         ├── Filter by bounding box
         ├── Rate limit (10 msg/s)
         ├── Build GeoJSON Feature
         ▼
Frontend (subscribed clients)
  → Update map in real-time
```

## Performance Features

### Caching Strategy

All Map API endpoints use Redis caching with configurable TTLs:

| Endpoint | Default TTL | Env Variable | Rationale |
|----------|-------------|--------------|-----------|
| /messages | 60s | MAP_CACHE_TTL_MESSAGES | Real-time feel, WebSocket for immediacy |
| /clusters | 300s | MAP_CACHE_TTL_CLUSTERS | Tier changes are infrequent |
| /events | 300s | MAP_CACHE_TTL_EVENTS | Curated data, changes slowly |
| /heatmap | 300s | MAP_CACHE_TTL_HEATMAP | Aggregate data, minimal visual impact per update |
| /locations/suggest | 600s | MAP_CACHE_TTL_SUGGESTIONS | Static gazetteer data |
| /locations/reverse | 600s | MAP_CACHE_TTL_SUGGESTIONS | Static gazetteer data |

**Cache Invalidation:**
- Messages endpoint: Auto-invalidated when new locations published to Redis `map:new_location`
- Clusters/Events: Manual invalidation or TTL expiry
- Heatmap: TTL expiry only

### Server-Side Clustering

When `zoom < 12` and `cluster=true`, the API performs grid-based aggregation:

```python
# Calculate grid size based on zoom level
def calculate_grid_size(zoom: int) -> float:
    # Zoom 0-5: 1.0° cells (~111km)
    # Zoom 6-8: 0.5° cells (~55km)
    # Zoom 9-11: 0.1° cells (~11km)
    if zoom <= 5:
        return 1.0
    elif zoom <= 8:
        return 0.5
    else:
        return 0.1
```

**Benefits:**
- Reduces payload size (thousands of points → hundreds of clusters)
- Faster rendering on client
- Prevents browser performance issues

**Query Example:**
```sql
SELECT
    FLOOR(ml.latitude / :grid_size) as lat_bucket,
    FLOOR(ml.longitude / :grid_size) as lng_bucket,
    COUNT(*) as point_count,
    AVG(ml.latitude) as center_lat,
    AVG(ml.longitude) as center_lng,
    MAX(m.telegram_date) as latest_date
FROM messages m
JOIN message_locations ml ON m.id = ml.message_id
WHERE ml.latitude BETWEEN :south AND :north
  AND ml.longitude BETWEEN :west AND :east
GROUP BY lat_bucket, lng_bucket
LIMIT 1000
```

### Rate Limiting

Redis-backed sliding window rate limiter (per IP):

| Endpoint | Default Limit | Env Variable | Status Code |
|----------|---------------|--------------|-------------|
| /messages | 30/min | MAP_MESSAGES_RATE_LIMIT | 429 |
| /clusters | 30/min | MAP_CLUSTERS_RATE_LIMIT | 429 |
| /events | 30/min | MAP_EVENTS_RATE_LIMIT | 429 |
| /heatmap | 20/min | MAP_HEATMAP_RATE_LIMIT | 429 |
| /locations/suggest | 60/min | MAP_SUGGEST_RATE_LIMIT | 429 |
| /locations/reverse | 60/min | MAP_REVERSE_RATE_LIMIT | 429 |
| /clusters/{id}/messages | 60/min | MAP_CLUSTER_MESSAGES_RATE_LIMIT | 429 |

**Response Headers (on success):**
```
X-RateLimit-Limit: 30
X-RateLimit-Remaining: 28
X-RateLimit-Reset: 1702992000
```

**429 Response:**
```json
{
  "detail": "Rate limit exceeded: 30 requests per minute"
}
```

## Authentication

**Current Status:** No authentication required (public read-only endpoints)

**Future Consideration:** Optional API key authentication for high-volume integrations.

## Endpoints Reference

### GET /api/map/messages

Get geocoded messages within a bounding box as GeoJSON.

**URL:** `/api/map/messages`

**Method:** `GET`

**Query Parameters:**

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| south | float | Yes | -90 to 90 | South boundary latitude |
| west | float | Yes | -180 to 180 | West boundary longitude |
| north | float | Yes | -90 to 90 | North boundary latitude |
| east | float | Yes | -180 to 180 | East boundary longitude |
| zoom | int | No | 0 to 22 | Map zoom level (affects clustering) |
| cluster | bool | No | - | Enable server-side clustering (default: false) |
| limit | int | No | 1 to 2000 | Maximum messages to return (default: 500) |
| days | int | No | - | Filter to last N days |
| channel_id | int | No | - | Filter by channel ID |
| min_confidence | float | No | 0 to 1 | Minimum location confidence (default: 0.5) |
| start_date | datetime | No | ISO 8601 | Start date for timeline filter |
| end_date | datetime | No | ISO 8601 | End date for timeline filter |
| polygon | string | No | JSON array | Polygon filter as JSON array of [lng, lat] pairs |

**Response:** `200 OK`

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [37.6173, 55.7558]
      },
      "properties": {
        "message_id": 12345,
        "content": "Russian forces report...",
        "content_translated": "Russian forces report...",
        "telegram_date": "2025-12-18T14:30:00Z",
        "channel_name": "Intel Slava Z",
        "channel_username": "intelslava",
        "channel_affiliation": "ru",
        "location_name": "Moscow",
        "location_hierarchy": "RU-MOW",
        "confidence": 0.95,
        "extraction_method": "gazetteer",
        "precision_level": "medium",
        "population": 12500000,
        "media_count": 2,
        "first_media_url": "/media/ab/cd/abcd1234.jpg",
        "first_media_type": "image"
      }
    }
  ]
}
```

**Cluster Mode Response** (when zoom < 12 and cluster=true):

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [37.62, 55.75]
      },
      "properties": {
        "cluster": true,
        "point_count": 23,
        "latest_date": "2025-12-18T15:00:00Z"
      }
    }
  ]
}
```

**Examples:**

```bash
# Basic bounding box query
curl "http://localhost:8000/api/map/messages?south=48.0&west=35.0&north=50.0&east=40.0&limit=500"

# With server-side clustering (low zoom)
curl "http://localhost:8000/api/map/messages?south=40.0&west=20.0&north=60.0&east=50.0&zoom=8&cluster=true"

# Timeline filter (last 7 days)
curl "http://localhost:8000/api/map/messages?south=48.0&west=35.0&north=50.0&east=40.0&days=7"

# Date range filter
curl "http://localhost:8000/api/map/messages?south=48.0&west=35.0&north=50.0&east=40.0&start_date=2025-12-01T00:00:00Z&end_date=2025-12-13T23:59:59Z"

# Polygon filter (Ukraine outline)
curl "http://localhost:8000/api/map/messages?south=44.0&west=22.0&north=53.0&east=41.0&polygon=[[22,44],[41,44],[41,53],[22,53]]"
```

**Error Responses:**

```json
// 400 Bad Request
{
  "detail": "south must be less than north"
}

// 400 Bad Request (invalid polygon)
{
  "detail": "Invalid polygon: maximum 100 vertices allowed"
}

// 429 Too Many Requests
{
  "detail": "Rate limit exceeded: 30 requests per minute"
}
```

---

### GET /api/map/clusters

Get Telegram event clusters within a bounding box as GeoJSON.

**URL:** `/api/map/clusters`

**Method:** `GET`

**Query Parameters:**

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| south | float | Yes | -90 to 90 | South boundary latitude |
| west | float | Yes | -180 to 180 | West boundary longitude |
| north | float | Yes | -90 to 90 | North boundary latitude |
| east | float | Yes | -180 to 180 | East boundary longitude |
| limit | int | No | 1 to 1000 | Maximum clusters to return (default: 200) |
| tier | string | No | - | Filter by tier: rumor, unconfirmed, confirmed, verified |
| status | string | No | - | Filter by status: detected, validated, archived |
| start_date | datetime | No | ISO 8601 | Start date for timeline filter |
| end_date | datetime | No | ISO 8601 | End date for timeline filter |
| polygon | string | No | JSON array | Polygon filter as JSON array of [lng, lat] pairs |

**Response:** `200 OK`

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [37.8403, 48.0159]
      },
      "properties": {
        "cluster_id": 456,
        "tier": "confirmed",
        "status": "validated",
        "claim_type": "artillery_strike",
        "channel_count": 5,
        "message_count": 12,
        "detected_at": "2025-12-18T10:00:00Z",
        "last_activity_at": "2025-12-18T14:30:00Z",
        "summary": "Multiple sources report artillery fire near..."
      }
    }
  ]
}
```

**Tier Progression (automatic):**

| Tier | Criteria | Color |
|------|----------|-------|
| rumor | 1 channel | Red |
| unconfirmed | 2-3 channels, same affiliation | Yellow |
| confirmed | 3+ channels, cross-affiliation | Orange |
| verified | Human verified | Green |

**Examples:**

```bash
# All clusters in bounding box
curl "http://localhost:8000/api/map/clusters?south=48.0&west=35.0&north=50.0&east=40.0"

# Only confirmed/verified events
curl "http://localhost:8000/api/map/clusters?south=48.0&west=35.0&north=50.0&east=40.0&tier=confirmed"

# Last 24 hours
curl "http://localhost:8000/api/map/clusters?south=48.0&west=35.0&north=50.0&east=40.0&start_date=2025-12-18T00:00:00Z"
```

---

### GET /api/map/clusters/{cluster_id}/messages

Get all geocoded messages for a specific cluster (used for cluster expansion).

**URL:** `/api/map/clusters/{cluster_id}/messages`

**Method:** `GET`

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| cluster_id | int | Yes | Cluster ID |

**Response:** `200 OK`

```json
{
  "cluster_id": 456,
  "tier": "confirmed",
  "messages": [
    {
      "message_id": 12345,
      "latitude": 48.0159,
      "longitude": 37.8403,
      "content": "Artillery strike reported...",
      "channel_name": "Intel Slava Z",
      "telegram_date": "2025-12-18T10:15:00Z"
    },
    {
      "message_id": 12346,
      "latitude": 48.0165,
      "longitude": 37.8398,
      "content": "Confirmed explosion in...",
      "channel_name": "Ukraine Now",
      "telegram_date": "2025-12-18T10:20:00Z"
    }
  ]
}
```

**Use Case:** Frontend displays cluster as a single marker, then expands to show individual message locations in a spider/circle pattern when clicked.

**Examples:**

```bash
# Get messages for cluster 456
curl "http://localhost:8000/api/map/clusters/456/messages"
```

**Error Responses:**

```json
// 404 Not Found
{
  "detail": "Cluster 456 not found"
}
```

---

### GET /api/map/events

Get curated events within a bounding box as GeoJSON.

**URL:** `/api/map/events`

**Method:** `GET`

**Query Parameters:**

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| south | float | Yes | -90 to 90 | South boundary latitude |
| west | float | Yes | -180 to 180 | West boundary longitude |
| north | float | Yes | -90 to 90 | North boundary latitude |
| east | float | Yes | -180 to 180 | East boundary longitude |
| limit | int | No | 1 to 1000 | Maximum events to return (default: 200) |
| tier | string | No | - | Filter by tier: rumor, unconfirmed, confirmed, verified |
| start_date | datetime | No | ISO 8601 | Start date for timeline filter |
| end_date | datetime | No | ISO 8601 | End date for timeline filter |
| polygon | string | No | JSON array | Polygon filter as JSON array of [lng, lat] pairs |

**Response:** `200 OK`

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [37.8403, 48.0159]
      },
      "properties": {
        "event_id": 789,
        "title": "Artillery Strike Near Donetsk",
        "event_type": "military_action",
        "event_date": "2025-12-18T10:00:00Z",
        "tier": "verified",
        "message_count": 15,
        "channel_count": 8
      }
    }
  ]
}
```

**Difference from Clusters:**
- **Events:** Manually curated by analysts, higher quality
- **Clusters:** Automatically detected by velocity-based algorithm

**Examples:**

```bash
# All events in bounding box
curl "http://localhost:8000/api/map/events?south=48.0&west=35.0&north=50.0&east=40.0"

# Only verified events
curl "http://localhost:8000/api/map/events?south=48.0&west=35.0&north=50.0&east=40.0&tier=verified"
```

---

### GET /api/map/trajectories

Get movement trajectories as GeoJSON LineStrings.

**URL:** `/api/map/trajectories`

**Method:** `GET`

**Query Parameters:**

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| south | float | Yes | -90 to 90 | South boundary latitude |
| west | float | Yes | -180 to 180 | West boundary longitude |
| north | float | Yes | -90 to 90 | North boundary latitude |
| east | float | Yes | -180 to 180 | East boundary longitude |
| limit | int | No | 1 to 1000 | Maximum trajectories to return (default: 200) |
| min_confidence | float | No | 0 to 1 | Minimum location confidence (default: 0.5) |
| start_date | datetime | No | ISO 8601 | Start date for timeline filter |
| end_date | datetime | No | ISO 8601 | End date for timeline filter |

**Response:** `200 OK`

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "LineString",
        "coordinates": [
          [37.6173, 55.7558],
          [37.8403, 48.0159]
        ]
      },
      "properties": {
        "message_id": 12345,
        "origin": "Moscow",
        "destination": "Donetsk",
        "location_count": 2,
        "content": "Convoy departed Moscow heading south...",
        "telegram_date": "2025-12-18T08:00:00Z",
        "channel_name": "Intel Slava Z",
        "channel_folder": "Archive-RU",
        "channel_affiliation": "ru"
      }
    }
  ]
}
```

**Use Case:** Visualize reported troop movements, convoy routes, or equipment transfers.

**Examples:**

```bash
# Movement trajectories in region
curl "http://localhost:8000/api/map/trajectories?south=48.0&west=35.0&north=50.0&east=40.0"
```

---

### GET /api/map/heatmap

Get aggregated heatmap data for message density visualization.

**URL:** `/api/map/heatmap`

**Method:** `GET`

**Query Parameters:**

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| south | float | Yes | -90 to 90 | South boundary latitude |
| west | float | Yes | -180 to 180 | West boundary longitude |
| north | float | Yes | -90 to 90 | North boundary latitude |
| east | float | Yes | -180 to 180 | East boundary longitude |
| grid_size | float | No | 0.01 to 1.0 | Grid cell size in degrees (default: 0.1) |
| min_confidence | float | No | 0 to 1 | Minimum location confidence (default: 0.5) |
| start_date | datetime | No | ISO 8601 | Start date for timeline filter |
| end_date | datetime | No | ISO 8601 | End date for timeline filter |
| polygon | string | No | JSON array | Polygon filter as JSON array of [lng, lat] pairs |

**Response:** `200 OK`

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [37.85, 48.05]
      },
      "properties": {
        "weight": 156
      }
    }
  ]
}
```

**Grid Size Guide:**

| grid_size | Cell Size | Use Case |
|-----------|-----------|----------|
| 0.01 | ~1.1 km | Detailed city-level patterns |
| 0.05 | ~5.5 km | Regional patterns |
| 0.1 | ~11 km | Country-level overview |
| 0.5 | ~55 km | Continental view |

**Examples:**

```bash
# Heatmap with default grid (0.1°)
curl "http://localhost:8000/api/map/heatmap?south=48.0&west=35.0&north=50.0&east=40.0"

# Fine-grained heatmap (0.05°)
curl "http://localhost:8000/api/map/heatmap?south=48.0&west=35.0&north=50.0&east=40.0&grid_size=0.05"

# Last 7 days only
curl "http://localhost:8000/api/map/heatmap?south=48.0&west=35.0&north=50.0&east=40.0&start_date=2025-12-12T00:00:00Z"
```

---

### GET /api/map/locations/suggest

Location autocomplete endpoint for frontend search.

**URL:** `/api/map/locations/suggest`

**Method:** `GET`

**Query Parameters:**

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| q | string | Yes | min_length=2 | Location name prefix (e.g., "Bakh") |
| limit | int | No | 1 to 50 | Maximum suggestions (default: 10) |
| country | string | No | - | Filter by country code (UA, RU) |

**Response:** `200 OK`

```json
{
  "suggestions": [
    {
      "name": "Bakhmut",
      "name_local": "Бахмут",
      "country_code": "UA",
      "latitude": 48.5953,
      "longitude": 38.0003,
      "population": 72310
    },
    {
      "name": "Bakhchysarai",
      "name_local": "Бахчисарай",
      "country_code": "UA",
      "latitude": 44.7547,
      "longitude": 33.8589,
      "population": 27448
    }
  ]
}
```

**Matching Logic:**
- Searches `name_primary`, `name_ascii`, `name_local`, and `aliases`
- Results ordered by population (larger cities first), then alphabetically
- Gazetteer data is static (GeoNames import), so long cache TTL (600s)

**Examples:**

```bash
# Basic autocomplete
curl "http://localhost:8000/api/map/locations/suggest?q=Bakh&limit=10"

# Filter by country
curl "http://localhost:8000/api/map/locations/suggest?q=Kyiv&country=UA"
```

---

### GET /api/map/locations/reverse

Reverse geocoding endpoint - find nearest location to coordinates.

**URL:** `/api/map/locations/reverse`

**Method:** `GET`

**Query Parameters:**

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| lat | float | Yes | -90 to 90 | Latitude |
| lng | float | Yes | -180 to 180 | Longitude |

**Response:** `200 OK`

```json
{
  "name": "Bakhmut",
  "name_local": "Бахмут",
  "country_code": "UA",
  "latitude": 48.5953,
  "longitude": 38.0003,
  "distance_km": 1.2
}
```

**Performance:**
- Uses PostGIS `ST_Distance` with geography type
- Coordinates rounded to 3 decimal places (~100m) for cache efficiency
- Returns nearest populated place from gazetteer

**Examples:**

```bash
# Reverse geocode coordinates
curl "http://localhost:8000/api/map/locations/reverse?lat=48.59&lng=37.99"
```

**Error Responses:**

```json
// 404 Not Found
{
  "detail": "No locations found in gazetteer"
}
```

---

### GET /api/map/hot-locations

Get hottest locations by message count (for sidebar feed).

**URL:** `/api/map/hot-locations`

**Method:** `GET`

**Query Parameters:**

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| hours | int | No | 1 to 168 | Look back period in hours (default: 24) |
| limit | int | No | 1 to 20 | Maximum locations (default: 5) |

**Response:** `200 OK`

```json
{
  "locations": [
    {
      "location_name": "Bakhmut",
      "message_count": 156,
      "latitude": 48.5953,
      "longitude": 38.0003
    },
    {
      "location_name": "Donetsk",
      "message_count": 142,
      "latitude": 48.0159,
      "longitude": 37.8403
    }
  ]
}
```

**Examples:**

```bash
# Top 5 locations in last 24 hours
curl "http://localhost:8000/api/map/hot-locations?hours=24&limit=5"

# Last 7 days
curl "http://localhost:8000/api/map/hot-locations?hours=168&limit=10"
```

---

### GET /api/map/recent-messages

Get most recent geolocated messages (for sidebar feed).

**URL:** `/api/map/recent-messages`

**Method:** `GET`

**Query Parameters:**

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| limit | int | No | 1 to 50 | Maximum messages (default: 10) |

**Response:** `200 OK`

```json
{
  "messages": [
    {
      "message_id": 12345,
      "channel_name": "Intel Slava Z",
      "channel_affiliation": "ru",
      "content": "Russian forces report...",
      "content_translated": "Russian forces report...",
      "location_name": "Bakhmut",
      "telegram_date": "2025-12-18T15:30:00Z",
      "latitude": 48.5953,
      "longitude": 38.0003
    }
  ]
}
```

**Examples:**

```bash
# Last 10 geolocated messages
curl "http://localhost:8000/api/map/recent-messages?limit=10"
```

---

## WebSocket API

### WS /api/map/ws/map/live

WebSocket endpoint for real-time map updates.

**URL:** `ws://localhost:8000/api/map/ws/map/live`

**Protocol:** WebSocket

**Query Parameters:**

| Parameter | Type | Required | Validation | Description |
|-----------|------|----------|------------|-------------|
| south | float | Yes | -90 to 90 | South boundary latitude |
| west | float | Yes | -180 to 180 | West boundary longitude |
| north | float | Yes | -90 to 90 | North boundary latitude |
| east | float | Yes | -180 to 180 | East boundary longitude |

**Security Features:**

1. **Origin Validation:** Prevents Cross-Site WebSocket Hijacking (CSWSH)
   - Checks `Origin` header against `ALLOWED_ORIGINS` env var
   - Falls back to `API_CORS_ORIGINS` or localhost defaults
   - Also checks `FRONTEND_URL` for production deployments

2. **Connection Limiting:** Max connections per IP (default: 10)
   - Prevents DoS through connection exhaustion
   - Configurable via `WEBSOCKET_MAX_CONNECTIONS_PER_IP`

3. **Rate Limiting:** Max 10 messages per second per connection
   - Configurable via `WEBSOCKET_RATE_LIMIT`

**Message Types:**

**Outbound (Server → Client):**

```json
// New feature
{
  "type": "feature",
  "data": {
    "type": "Feature",
    "geometry": {
      "type": "Point",
      "coordinates": [37.8403, 48.0159]
    },
    "properties": {
      "message_id": 12345,
      "location_name": "Donetsk",
      "channel_name": "Intel Slava Z",
      "content": "Artillery strike reported...",
      "confidence": 0.95,
      "extraction_method": "gazetteer",
      "telegram_date": "2025-12-18T15:45:00Z"
    }
  }
}

// Heartbeat (every 30 seconds)
{
  "type": "heartbeat",
  "timestamp": 1702992345
}
```

**Filtering:**
- Only messages within the specified bounding box are sent
- Rate limiting prevents overwhelming the client

**Connection Flow:**

```javascript
// JavaScript example
const ws = new WebSocket(
  'ws://localhost:8000/api/map/ws/map/live?south=48.0&west=35.0&north=50.0&east=40.0'
);

ws.onopen = () => {
  console.log('WebSocket connected');
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);

  if (data.type === 'feature') {
    // Add feature to map
    map.addFeature(data.data);
  } else if (data.type === 'heartbeat') {
    console.log('Heartbeat received');
  }
};

ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};

ws.onclose = (event) => {
  console.log('WebSocket closed:', event.code, event.reason);
};
```

**Close Codes:**

| Code | Reason | Description |
|------|--------|-------------|
| 1000 | Normal closure | Clean disconnect |
| 1008 | Policy violation | Invalid bounding box (south >= north) |
| 4003 | Origin not allowed | Failed origin validation |
| 4008 | Too many connections | Exceeded connection limit per IP |

**Environment Variables:**

```bash
# Allowed origins (comma-separated)
ALLOWED_ORIGINS=http://localhost:3000,https://osintukraine.com

# Or use CORS origins
API_CORS_ORIGINS=http://localhost:3000,https://osintukraine.com

# Frontend URL (production)
FRONTEND_URL=https://v2.osintukraine.com

# Allow connections without Origin header (dev only)
WEBSOCKET_ALLOW_NO_ORIGIN=false

# Max connections per IP
WEBSOCKET_MAX_CONNECTIONS_PER_IP=10

# Rate limit (messages per second)
WEBSOCKET_RATE_LIMIT=10
```

**Reconnection Strategy (recommended):**

```javascript
function connectWebSocket() {
  const ws = new WebSocket(wsUrl);

  ws.onclose = (event) => {
    console.log('WebSocket closed, reconnecting in 5s...');
    setTimeout(connectWebSocket, 5000);
  };

  ws.onerror = (error) => {
    console.error('WebSocket error:', error);
    ws.close();
  };

  return ws;
}
```

---

## GeoJSON Response Format

All Map API endpoints return GeoJSON FeatureCollections compatible with MapLibre GL.

**Coordinate Order:** `[longitude, latitude]` (GeoJSON standard)

> **Note:** Database stores as `(latitude, longitude)`, but responses use GeoJSON order.

### Message Features

```json
{
  "message_id": 12345,
  "content": "...",
  "content_translated": "...",
  "telegram_date": "2025-12-18T15:30:00Z",
  "channel_name": "...",
  "channel_username": "...",
  "channel_affiliation": "ru" | "ua" | "unknown",
  "location_name": "Bakhmut",
  "location_hierarchy": "UA-14",
  "confidence": 0.95,
  "extraction_method": "gazetteer" | "llm" | "nominatim" | "unresolved",
  "precision_level": "high" | "medium" | "low",
  "population": 72310,
  "media_count": 2,
  "first_media_url": "/media/ab/cd/abcd1234.jpg",
  "first_media_type": "image" | "video" | "document"
}
```

**Precision Levels:**

| Level | Criteria | Examples |
|-------|----------|----------|
| high | Neighborhood or small town (<10k) | PPLX, PPLQ feature codes |
| medium | City level (10k-500k) | Default for most cities |
| low | Major city (>500k) or region | ADM* feature codes, large metros |

### Cluster Features

```json
{
  "cluster": true,
  "point_count": 23,
  "latest_date": "2025-12-18T15:00:00Z"
}
```

### Event Cluster Features

```json
{
  "cluster_id": 456,
  "tier": "confirmed",
  "status": "validated",
  "claim_type": "artillery_strike",
  "channel_count": 5,
  "message_count": 12,
  "detected_at": "2025-12-18T10:00:00Z",
  "last_activity_at": "2025-12-18T14:30:00Z",
  "summary": "..."
}
```

### Heatmap Features

```json
{
  "weight": 156
}
```

---

## Filtering Options

### Bounding Box Filter

All endpoints support bounding box filtering via `south`, `west`, `north`, `east` query parameters.

**Validation:**
- `south < north` (required)
- `-90 <= latitude <= 90`
- `-180 <= longitude <= 180`

**Performance:**
- Uses spatial index: `idx_message_locations_bbox` on `(latitude, longitude)`
- Queries are fast even with millions of locations

### Time Range Filter

**Supported Parameters:**

1. **Relative:** `days=7` (last N days)
2. **Absolute:** `start_date` and `end_date` (ISO 8601 format)

**Examples:**

```bash
# Last 24 hours
?days=1

# Last week
?days=7

# Specific date range
?start_date=2025-12-01T00:00:00Z&end_date=2025-12-13T23:59:59Z

# From date onwards
?start_date=2025-12-01T00:00:00Z

# Up to date
?end_date=2025-12-13T23:59:59Z
```

### Polygon Filter

**Format:** JSON array of `[longitude, latitude]` pairs

**Constraints:**
- Maximum 100 vertices (DoS prevention)
- Coordinates validated (-180 to 180, -90 to 90)
- Auto-closes polygon if first != last

**Example:**

```bash
# Ukraine outline (simplified)
?polygon=[[22.14,48.22],[40.23,48.22],[40.23,52.38],[22.14,52.38]]
```

**Query Implementation (PostGIS):**

```sql
WHERE ST_Contains(
  ST_GeomFromText('POLYGON((22.14 48.22, 40.23 48.22, 40.23 52.38, 22.14 52.38, 22.14 48.22))', 4326),
  ST_MakePoint(ml.longitude, ml.latitude)
)
```

### Tier Filter

**Endpoints:** `/clusters`, `/events`

**Values:**
- `rumor` - Single channel report
- `unconfirmed` - 2-3 channels, same affiliation
- `confirmed` - 3+ channels, cross-affiliation
- `verified` - Human verified

**Example:**

```bash
# Only confirmed and verified
?tier=confirmed
?tier=verified
```

### Channel Filter

**Endpoints:** `/messages`

**Parameter:** `channel_id=123`

**Example:**

```bash
# Messages from specific channel
?channel_id=456
```

### Confidence Filter

**Endpoints:** `/messages`, `/trajectories`, `/heatmap`

**Parameter:** `min_confidence=0.8` (0.0 to 1.0)

**Default:** 0.5 (medium confidence)

**Example:**

```bash
# High confidence only
?min_confidence=0.8
```

---

## Common Use Cases

### Use Case 1: Real-Time Map Updates

**Goal:** Show new geolocated messages as they arrive

**Implementation:**

```javascript
// 1. Load initial data
const response = await fetch(
  '/api/map/messages?south=48.0&west=35.0&north=50.0&east=40.0&limit=500'
);
const initialData = await response.json();
map.addGeoJSON(initialData);

// 2. Subscribe to real-time updates
const ws = new WebSocket(
  'ws://localhost:8000/api/map/ws/map/live?south=48.0&west=35.0&north=50.0&east=40.0'
);

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);

  if (data.type === 'feature') {
    map.addFeature(data.data);
  }
};
```

### Use Case 2: Timeline Playback

**Goal:** Show events over time with timeline slider

**Implementation:**

```javascript
// Timeline slider: 2025-12-01 to 2025-12-18
const startDate = '2025-12-01T00:00:00Z';
const endDate = selectedDate; // From slider

const response = await fetch(
  `/api/map/messages?south=48.0&west=35.0&north=50.0&east=40.0` +
  `&start_date=${startDate}&end_date=${endDate}`
);
const data = await response.json();
map.setData(data);
```

### Use Case 3: Cluster Expansion

**Goal:** Show cluster as single marker, expand to individual messages on click

**Implementation:**

```javascript
// 1. Load clusters
const clusters = await fetch('/api/map/clusters?south=48.0&west=35.0&north=50.0&east=40.0');
const clusterData = await clusters.json();
map.addGeoJSON(clusterData);

// 2. On cluster click, expand
map.on('click', 'clusters-layer', async (e) => {
  const clusterId = e.features[0].properties.cluster_id;

  const response = await fetch(`/api/map/clusters/${clusterId}/messages`);
  const messages = await response.json();

  // Display messages in spider/circle pattern
  map.showClusterExpansion(messages.messages);
});
```

### Use Case 4: Heatmap Visualization

**Goal:** Show message density over time

**Implementation:**

```javascript
// Load heatmap data
const response = await fetch(
  '/api/map/heatmap?south=40.0&west=20.0&north=60.0&east=50.0&grid_size=0.1'
);
const heatmapData = await response.json();

// Add to MapLibre as heatmap layer
map.addLayer({
  id: 'message-heatmap',
  type: 'heatmap',
  source: {
    type: 'geojson',
    data: heatmapData
  },
  paint: {
    'heatmap-weight': ['get', 'weight'],
    'heatmap-intensity': 1,
    'heatmap-color': [
      'interpolate',
      ['linear'],
      ['heatmap-density'],
      0, 'rgba(0, 0, 255, 0)',
      0.5, 'rgb(0, 255, 0)',
      1, 'rgb(255, 0, 0)'
    ],
    'heatmap-radius': 20
  }
});
```

### Use Case 5: Location Search & Zoom

**Goal:** Autocomplete location search with map navigation

**Implementation:**

```javascript
// Autocomplete search
const searchInput = document.getElementById('location-search');

searchInput.addEventListener('input', async (e) => {
  const query = e.target.value;

  if (query.length < 2) return;

  const response = await fetch(
    `/api/map/locations/suggest?q=${encodeURIComponent(query)}&limit=10`
  );
  const suggestions = await response.json();

  // Display suggestions
  displaySuggestions(suggestions.suggestions);
});

// On suggestion click, zoom to location
function onSuggestionClick(suggestion) {
  map.flyTo({
    center: [suggestion.longitude, suggestion.latitude],
    zoom: 12
  });

  // Load messages near location
  loadMessagesNear(suggestion.latitude, suggestion.longitude);
}
```

---

## Error Handling

### HTTP Status Codes

| Code | Description | Common Causes |
|------|-------------|---------------|
| 200 | Success | Request completed successfully |
| 400 | Bad Request | Invalid parameters (south >= north, invalid polygon) |
| 404 | Not Found | Cluster/entity not found |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Database connection error, unexpected failure |

### Error Response Format

```json
{
  "detail": "Error message description"
}
```

**Examples:**

```json
// 400 Bad Request
{
  "detail": "south must be less than north"
}

// 400 Bad Request (polygon)
{
  "detail": "Invalid polygon: maximum 100 vertices allowed"
}

// 404 Not Found
{
  "detail": "Cluster 456 not found"
}

// 429 Too Many Requests
{
  "detail": "Rate limit exceeded: 30 requests per minute"
}
```

### Rate Limit Headers

**On Success (200 OK):**
```
X-RateLimit-Limit: 30
X-RateLimit-Remaining: 28
X-RateLimit-Reset: 1702992000
```

**On Rate Limit (429):**
```
X-RateLimit-Limit: 30
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1702992060
Retry-After: 60
```

---

## Performance Tips

### 1. Use Server-Side Clustering

**When:** Zoom levels 0-11, large bounding boxes

```bash
# Without clustering: 10,000+ features
curl "/api/map/messages?south=40&west=20&north=60&east=50&limit=2000"

# With clustering: 100-200 clusters
curl "/api/map/messages?south=40&west=20&north=60&east=50&zoom=8&cluster=true"
```

**Benefit:** 50-100x payload reduction

### 2. Leverage Caching

**Strategy:** Identical requests within TTL window return cached results

```bash
# First request: Cache miss (query database)
# Response time: 150ms

# Second request (within 60s): Cache hit
# Response time: 5ms
```

**Tip:** Use consistent parameter order for better cache hit rate.

### 3. Limit Result Size

**Problem:** Large bounding boxes + high limit = slow queries

**Solution:**

```bash
# Bad: Requesting 2000 messages from continental view
?south=40&west=20&north=60&east=50&limit=2000

# Good: Use clustering or reduce limit
?south=40&west=20&north=60&east=50&zoom=6&cluster=true&limit=500
```

### 4. Use Polygon Filters Sparingly

**Cost:** PostGIS `ST_Contains` is slower than bbox queries

**Recommendation:** Use bbox for initial filter, then polygon for refinement

```bash
# Fast: Bbox only
?south=48&west=35&north=50&east=40

# Slower: Bbox + complex polygon
?south=48&west=35&north=50&east=40&polygon=[...100 vertices...]
```

### 5. Paginate Large Datasets

**Strategy:** Use time-based pagination instead of large limits

```bash
# Instead of: ?limit=5000
# Use: Multiple requests with date ranges

# Page 1: Last 24 hours
?start_date=2025-12-18T00:00:00Z

# Page 2: Previous 24 hours
?start_date=2025-12-17T00:00:00Z&end_date=2025-12-18T00:00:00Z
```

---

## Related Documentation

- **Architecture:** [Architecture Overview](./architecture.md) - System architecture including geolocation pipeline
- **User Guide:** [Map Interface](../user-guide/map-interface.md) - End-user map features
- **API Service:** [API Documentation](services/api.md) - Complete API documentation
- **Database:** [Database Schema](database-schema.md) - Schema for `message_locations`, `telegram_event_clusters`
- **Frontend:** [Frontend API Patterns](frontend-api-patterns.md) - How frontend consumes Map API

---

## Changelog

### 2025-12-19
- Initial comprehensive documentation
- All 12 endpoints documented with examples
- WebSocket security features documented
- Performance optimization guide added
- Common use cases with code examples

---

## Support

**Issues:** Report bugs or request features in GitHub issues

**Contact:** For integration support or questions, see project README

**Monitoring:** Check Prometheus metrics at `/metrics` endpoint

**Health Check:** `GET /health` returns service status

---

**File Locations:**

- **Source Code:** `~/code/osintukraine/osint-intelligence-platform/services/api/src/routers/map.py` (1951 lines)
- **Documentation:** `~/code/osintukraine/osint-platform-docs/docs/developer-guide/map-api.md` (this file)
