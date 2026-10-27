# Rate Limiting

The OSINT Platform implements Redis-backed rate limiting on expensive API endpoints to prevent abuse and ensure fair resource allocation.

## Overview

**Algorithm**: Sliding window using Redis sorted sets
**Scope**: Per client key: `ip:<address>` for anonymous callers, `user:<id>` for sessions, `apikey:<id>` for API keys (forwarded-IP headers are trusted only from internal networks)
**Behavior**: Fail-open on Redis errors (logs warning, allows request)

```
Request → Get Client IP → Check Redis Sorted Set → Allow/Deny
                                    │
                           ┌────────┴────────┐
                           │   ZREMRANGEBYSCORE    │  (Remove expired)
                           │   ZCARD               │  (Count current)
                           │   ZADD                │  (Add request)
                           │   EXPIRE              │  (Cleanup TTL)
                           └──────────────────────┘
```

## Configuration

Rate limits are configured via environment variables (requests per minute):

| Environment Variable | Default | Endpoint | Use Case |
|---------------------|---------|----------|----------|
| `MAP_MESSAGES_RATE_LIMIT` | 120 | `/api/map/messages` | Map pan/zoom triggers rapid requests |
| `MAP_CLUSTERS_RATE_LIMIT` | 120 | `/api/map/clusters` | Event cluster display |
| `MAP_EVENTS_RATE_LIMIT` | 60 | `/api/map/events` | Confirmed events |
| `MAP_HEATMAP_RATE_LIMIT` | 60 | `/api/map/heatmap` | Density visualization |
| `MAP_SUGGEST_RATE_LIMIT` | 60 | `/api/map/locations/suggest` | Location autocomplete |
| `MAP_REVERSE_RATE_LIMIT` | 60 | `/api/map/locations/reverse` | Reverse geocoding |
| `MAP_CLUSTER_MESSAGES_RATE_LIMIT` | 60 | `/api/map/clusters/{id}/messages` | Cluster expansion |
| `MEDIA_REDIRECT_RATE_LIMIT` | 300 | `/api/media/internal/media-redirect/...` | Cold-path media (anti-leeching; hot-path files come from the Caddy buffer and bypass the API) |

`/api/map/trajectories` shares `MAP_MESSAGES_RATE_LIMIT`. `/api/map/hot-locations` and `/api/map/recent-messages` have no limiter. The map WebSocket has its own limits (`WEBSOCKET_RATE_LIMIT`, `WEBSOCKET_MAX_CONNECTIONS_PER_IP`; see [Map API](map-api.md#websocket-api)).

### Tiers

`rate_limit_dependency(category=...)` (no fixed number) picks a limit from the caller's tier, in requests per minute:

| Tier | default | media | export | map |
|------|---------|-------|--------|-----|
| anonymous | 60 | 30 | 10 | 120 |
| authenticated | 120 | 60 | 30 | 240 |
| premium | 300 | 150 | 100 | 600 |
| admin | 1000 | 500 | 500 | 1000 |

An API key's tier comes from `api_keys.rate_limit_tier`; sessions are `authenticated`, or `admin` for admins. The map and media endpoints above pass a fixed limit, which overrides the tier.

**Note**: Higher defaults (120/min) for messages/clusters endpoints account for frequent map interactions during pan/zoom.

## Response Headers

All rate-limited responses include headers:

```http
X-RateLimit-Limit: 120
X-RateLimit-Remaining: 117
X-RateLimit-Reset: 1702992060
```

On rate limit exceeded (HTTP 429):

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 120
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1702992060
Retry-After: 60

{"detail": "Rate limit exceeded. Please try again later."}
```

`Retry-After` is the full window (60 s), not the time left in it.

## Implementation Details

### Sliding Window Algorithm

Uses Redis sorted sets where:
- Key: `rate_limit:map:{endpoint}:{client_key}` (client key as above, e.g. `ip:192.168.1.1`)
- Score: Request timestamp (Unix epoch)
- Member: Unique request identifier

```python
# Pipeline operations (atomic)
pipe.zremrangebyscore(key, "-inf", window_start)  # Remove expired
pipe.zcard(key)                                    # Count current
pipe.zadd(key, {now: now})                         # Add request
pipe.expire(key, window_seconds + 1)               # Cleanup TTL
```

### Client IP Detection

`get_client_ip()` supports reverse proxies (Caddy) without allowing spoofing: `X-Forwarded-For` (first entry) and `X-Real-IP` are honoured **only** when the direct connection comes from an internal network (Docker `172.16.0.0/12`, `10/8`, `192.168/16`, loopback). Otherwise the socket address is used. Ports are stripped so one IP shares one bucket.

### Fail-Open Behavior

On Redis connection errors, the limiter:
1. Logs warning with error details
2. Allows the request (fail-open)
3. Reports full limit as remaining

**Rationale**: Brief Redis outages shouldn't block legitimate users.

## Usage in Code

### As FastAPI Dependency

```python
from ..utils.rate_limit import rate_limit_dependency

# Fixed limit
@router.get("/expensive")
async def expensive_endpoint(
    request: Request,
    _rate_limit: None = Depends(rate_limit_dependency(30, "expensive")),
):
    return {"data": "..."}

# Tier-based limit (recommended for new endpoints)
@router.get("/files")
async def files(
    request: Request,
    _rate_limit: None = Depends(rate_limit_dependency(category="media")),
):
    ...
```

### Direct Usage

```python
from ..utils.rate_limit import RateLimiter

limiter = RateLimiter()
allowed, info = await limiter.check_rate_limit(
    client_id="192.168.1.1",
    endpoint="map_messages",
    limit=120
)

if not allowed:
    raise HTTPException(status_code=429, detail="Rate limit exceeded")
```

### Adding Headers to Responses

```python
from ..utils.rate_limit import add_rate_limit_headers

@router.get("/data")
async def get_data(request: Request, response: Response):
    # Needs the rate_limit_dependency on the route (it stores the info on request.state)
    await add_rate_limit_headers(request, response)
    return {"data": "..."}
```

## Redis Key Structure

```
rate_limit:map:map_messages:ip:192.168.1.1
rate_limit:map:map_clusters:ip:192.168.1.1
rate_limit:map:map_heatmap:user:3f2a...
```

Keys auto-expire after window + 1 second for cleanup.

## Monitoring

Rate limit events are logged:

```json
{
  "level": "warning",
  "message": "Rate limit exceeded",
  "client_id": "ip:192.168.1.1",
  "endpoint": "map_messages",
  "limit": 120,
  "current_count": 121
}
```

Redis errors are also logged:

```json
{
  "level": "warning",
  "message": "Rate limit Redis error (failing open)",
  "error": "Connection refused",
  "client_id": "ip:192.168.1.1",
  "endpoint": "map_messages"
}
```

## Best Practices

### Frontend Integration

```typescript
// Handle 429 responses gracefully
async function fetchWithRetry(url: string, retries = 3): Promise<Response> {
  const response = await fetch(url);

  if (response.status === 429 && retries > 0) {
    const retryAfter = parseInt(response.headers.get('Retry-After') || '60');
    await new Promise(r => setTimeout(r, retryAfter * 1000));
    return fetchWithRetry(url, retries - 1);
  }

  return response;
}
```

### Debouncing Map Interactions

```typescript
// Debounce map move events to reduce requests
const debouncedFetch = useMemo(
  () => debounce((bbox: BBox) => fetchMessages(bbox), 300),
  []
);

map.on('moveend', () => {
  debouncedFetch(map.getBounds());
});
```

## Not Yet Implemented

1. **Global rate limits**: no cross-endpoint cap to stop distributed abuse
2. **Tiers on the map endpoints**: they use fixed limits, so authenticated users and API keys get the same map limits as anonymous callers

## Related Documentation

- [Map API Reference](map-api.md) - Endpoints that use rate limiting
- [API Endpoints](../reference/api-endpoints.md) - Full API reference
- [WebSocket Security](map-api.md#websocket-api) - WebSocket connection limits

---

**File Location**: `services/api/src/utils/rate_limit.py` in the platform repository
