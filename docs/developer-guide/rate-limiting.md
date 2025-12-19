# Rate Limiting

The OSINT Platform implements Redis-backed rate limiting on expensive API endpoints to prevent abuse and ensure fair resource allocation.

## Overview

**Algorithm**: Sliding window using Redis sorted sets
**Scope**: Per-IP rate limiting (supports X-Forwarded-For for proxied requests)
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
Retry-After: 45

{
  "detail": "Rate limit exceeded. Please try again later.",
  "retry_after": 45
}
```

## Implementation Details

### Sliding Window Algorithm

Uses Redis sorted sets where:
- Key: `rate_limit:map:{endpoint}:{client_ip}`
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

Supports reverse proxy scenarios:

```python
# Priority order:
1. X-Forwarded-For header (first entry)
2. X-Real-IP header (nginx)
3. Direct client.host
```

### Fail-Open Behavior

On Redis connection errors, the limiter:
1. Logs warning with error details
2. Allows the request (fail-open)
3. Reports full limit as remaining

**Rationale**: Brief Redis outages shouldn't block legitimate users.

## Usage in Code

### As FastAPI Dependency

```python
from src.utils.rate_limit import rate_limit_dependency

@router.get("/expensive")
async def expensive_endpoint(
    request: Request,
    rate_limit: None = Depends(rate_limit_dependency(
        requests_per_minute=30,
        endpoint_name="expensive"
    ))
):
    # Rate limiting handled by dependency
    return {"data": "..."}
```

### Direct Usage

```python
from src.utils.rate_limit import RateLimiter

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
from src.utils.rate_limit import add_rate_limit_headers

@router.get("/data")
async def get_data(request: Request, response: Response):
    await add_rate_limit_headers(request, response)
    return {"data": "..."}
```

## Redis Key Structure

```
rate_limit:map:messages:192.168.1.1
rate_limit:map:clusters:192.168.1.1
rate_limit:map:heatmap:10.0.0.1
```

Keys auto-expire after window + 1 second for cleanup.

## Monitoring

Rate limit events are logged:

```json
{
  "level": "warning",
  "message": "Rate limit exceeded",
  "client_id": "192.168.1.1",
  "endpoint": "messages",
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
  "client_id": "192.168.1.1",
  "endpoint": "messages"
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

## Future Enhancements

Potential improvements (not yet implemented):

1. **API Key Authentication**: Higher limits for authenticated integrations
2. **Per-User Limits**: User-based rather than IP-based for authenticated users
3. **Tiered Limits**: Different limits for different user roles
4. **Global Rate Limits**: Cross-endpoint limits to prevent distributed abuse

## Related Documentation

- [Map API Reference](map-api.md) - Endpoints that use rate limiting
- [API Endpoints](../reference/api-endpoints.md) - Full API reference
- [WebSocket Security](map-api.md#websocket-api) - WebSocket connection limits

---

**File Location**: `/home/rick/code/osintukraine/osint-intelligence-platform/services/api/src/utils/rate_limit.py`

**Last Updated**: 2025-12-19
