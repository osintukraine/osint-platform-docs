# Frontend API Patterns

Guide for making API calls from the Next.js frontend.

---

## Overview

The frontend has two patterns for API calls:

1. **Centralized API client** (`lib/api.ts`) - Preferred for standard pages
2. **Admin client** (`lib/admin-api.ts`: `adminFetch`, `adminGet`, `adminPost`, `adminApi`, ...) - Used by admin pages; always sends session cookies

The centralized client handles server vs client context automatically. For data that several components share or refresh, use TanStack Query hooks in `hooks/` (for example `useNetworkGraph.ts`) on top of these clients.

---

## Centralized API Client

### Location

`services/frontend-nextjs/lib/api.ts`

### How It Works

```typescript
// lib/api.ts (simplified; the real function also validates the URL and throws on a bad one)
function getApiUrl(): string {
  // Server-side (Next.js server components)
  if (typeof window === 'undefined') {
    return process.env.API_URL || 'http://api:8000';
  }

  // Client-side (browser): relative URLs when unset, so /api/* goes through Caddy
  return process.env.NEXT_PUBLIC_API_URL || '';
}
```

**Key points:**
- **Server-side** (SSR): Uses `API_URL` → `http://api:8000` (Docker internal network)
- **Client-side** (browser): Uses `NEXT_PUBLIC_API_URL` or empty string (relative URLs, routed Caddy → Oathkeeper → API)
- `authFetch(url, options)` is a `fetch` wrapper with `credentials: 'include'`; use it for calls that need the Ory Kratos session cookie

### Available Functions

```typescript
import {
  searchMessages,
  getMessage,
  getSimilarMessages,
  getChannels,
  getChannel,
  getEntity,
  searchEntities,
  getMessageNetwork,
  getChannelNetwork,
  getMessageSocialGraph,
  getEntityMessages,
  getEntityRelationships,
  getMessageValidation,
  getEvents,
  getEvent,
  unifiedSearch,
  // ... more
} from '@/lib/api';
```

### Usage Example

```typescript
// app/messages/[id]/page.tsx
import { getMessage, getSimilarMessages } from '@/lib/api';

export default async function MessagePage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;  // Next.js 15: params is async
  const message = await getMessage(parseInt(id));
  const similar = await getSimilarMessages(parseInt(id), 5);

  return (
    <div>
      <h1>{message.content}</h1>
      {/* ... */}
    </div>
  );
}
```

---

## Admin Pages

Admin pages call `lib/admin-api.ts`, which includes credentials (session cookies) with every request and throws an `Error` carrying the API's `detail`:

```typescript
// app/admin/channels/page.tsx
'use client';

import { adminGet, adminPost } from '@/lib/admin-api';

const channels = await adminGet<Channel[]>('/api/admin/channels');
await adminPost('/api/admin/channels/42/restore', {});
```

`adminGet`, `adminPost`, `adminPut`, `adminPatch`, `adminDelete` wrap `adminFetch`; the same functions are grouped in the `adminApi` object. Do not use raw `fetch()` in admin pages: without `credentials: 'include'` the browser does not send the Kratos cookie and the API answers 401.

---

## Environment Variables

| Variable | Context | Default | Purpose |
|----------|---------|---------|---------|
| `API_URL` | Server-side (runtime) | `http://api:8000` | Internal Docker network |
| `NEXT_PUBLIC_API_URL` | Client-side (**build time**) | empty = relative URLs | Browser access to the API |
| `NEXT_PUBLIC_MEDIA_URL` | Client-side (build time) | empty = relative `/media/...` | Override media base URL (CDN) |
| `NEXT_PUBLIC_BASE_URL` | Client-side (build time) | `http://localhost:3000` | Absolute URLs for OpenGraph tags |
| `INTERNAL_API_URL` | `next.config.js` rewrites | `http://api:8000` | Dev fallback: proxies `/media/*` to the API |
| `MINIO_URL` | Server-side | `http://minio:9000/osint-media` | Internal object storage |

!!! warning "NEXT_PUBLIC_* are baked into the bundle"
    Next.js inlines `NEXT_PUBLIC_*` at build time. The frontend `Dockerfile` takes them as build `ARG`s (set under `build.args` in `docker-compose.yml`); changing them in the runtime environment has no effect on browser code. Rebuild: `docker-compose build frontend`.

### Docker Compose Configuration

```yaml
# docker-compose.yml
frontend:
  build:
    args:
      NEXT_PUBLIC_API_URL: ${NEXT_PUBLIC_API_URL:-http://localhost:8000}
      NEXT_PUBLIC_BASE_URL: ${NEXT_PUBLIC_BASE_URL:-http://localhost:3000}
  environment:
    - API_URL=http://api:8000
    - MINIO_URL=http://minio:9000/osint-media
```

----------|---------|---------|---------|
| `API_URL` | Server-side | `http://api:8000` | Internal Docker network |
| `NEXT_PUBLIC_API_URL` | Client-side | `http://localhost:8000` | Browser access to API |
| `MINIO_URL` | Server-side | `http://minio:9000/telegram-archive` | Internal media storage |
| `NEXT_PUBLIC_MINIO_URL` | Client-side | `http://localhost:9000/telegram-archive` | Browser media access |

### Docker Compose Configuration

```yaml
# docker-compose.yml
frontend:
  environment:
    - API_URL=http://api:8000
    - NEXT_PUBLIC_API_URL=http://localhost:8000
    - MINIO_URL=http://minio:9000/telegram-archive
    - NEXT_PUBLIC_MINIO_URL=http://localhost:9000/telegram-archive
```

---

## fetchApi Helper

The centralized client uses a `fetchApi` helper with built-in features:

```typescript
// lib/api.ts (simplified)
async function fetchApi<T>(path: string, options?: RequestInit): Promise<T> {
  // Browser GETs with the same path that are in flight share one request
  // (not on the server: the map would be shared between users)
  ...
  const response = await fetch(`${getApiUrl()}${path}`, {
    ...options,
    headers,             // Content-Type JSON + Bearer token when window.__AUTH_TOKEN__ is set
    cache: 'no-store',   // Dynamic data, no caching
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({ detail: 'Unknown error' }));
    throw new Error(error.detail || `API error: ${response.status}`);
  }

  return response.json();
}
```

### Features

- **Automatic auth** - Injects a Bearer token if `window.__AUTH_TOKEN__` is set (session-cookie calls go through `authFetch`)
- **Request sharing** - Identical in-flight browser GETs are de-duplicated
- **Error handling** - Parses error.detail from API responses
- **No caching** - Uses `cache: 'no-store'` for dynamic data
- **Expected error filtering** - Doesn't log expected errors (missing embeddings, etc.)

---

## Media URLs

For archived media (images, videos), use `getMediaUrl()`. It always returns a browser-accessible URL, even during SSR:

```typescript
import { getMediaUrl } from '@/lib/api';

// Convert S3 key to a URL (relative: Caddy routes /media/* to the local buffer or storage box)
const imageUrl = getMediaUrl(item.url);
// Returns: /media/2f/a1/abc123.jpg  (or NEXT_PUBLIC_MEDIA_URL + path)

// For OpenGraph tags (must be absolute and publicly accessible)
const ogImageUrl = getMediaUrl(item.url, true);  // forceExternal=true
// Returns: ${NEXT_PUBLIC_BASE_URL}/media/2f/a1/abc123.jpg
```

Messages carry `media_items: MediaItem[]` (`url` is the S3 key, plus `mime_type` and a semantic `media_type`); `media_urls` is deprecated. See [Media Storage](media-storage.md) for how files reach `/media/*`.

---

## Type Safety

Types are defined in `lib/types.ts`:

```typescript
// lib/types.ts
import type {
  Message,
  SearchResult,
  SearchParams,
  Channel,
  Event,
  EntityDetail,
  // ... more
} from './types';
```

### Key Types

```typescript
interface Message {
  id: number;
  message_id: number;
  channel_id: number;
  content: string;
  content_translated: string | null;
  telegram_date: string | null;
  is_spam: boolean;
  osint_topic: string | null;
  importance_level: string | null;
  media_type: string | null;
  media_items: MediaItem[];
  // ... more
}

interface SearchParams {
  q?: string;
  channel_id?: number;
  topic?: string;
  importance_level?: string;
  days?: number;
  page?: number;
  page_size?: number;
  cursor?: string;        // keyset cursor from next_cursor; page is ignored when set
  sort_by?: string;
  sort_order?: 'asc' | 'desc';
  // ... more filters
}

interface SearchResult {
  items: Message[];
  total: number;
  page: number;
  page_size: number;
  total_pages: number;
  has_next: boolean;
  has_prev: boolean;
  total_is_estimate?: boolean;  // true when total is a planner estimate
}
```

---

## Error Handling

### In Components

```typescript
'use client';

import { useState, useEffect } from 'react';
import { getMessage } from '@/lib/api';

export default function MessageComponent({ id }: { id: number }) {
  const [message, setMessage] = useState<Message | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    getMessage(id)
      .then(setMessage)
      .catch(err => setError(err.message))
      .finally(() => setLoading(false));
  }, [id]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!message) return <div>Not found</div>;

  return <div>{message.content}</div>;
}
```

### API Error Format

The API returns errors as:

```json
{
  "detail": "Message not found"
}
```

The `fetchApi` helper extracts this and throws as `Error`:

```typescript
catch (err) {
  console.log(err.message);  // "Message not found"
}
```

---

## Validation API Pattern

The validation endpoint uses a special pattern with 202 status:

```typescript
// lib/api.ts
export async function getMessageValidation(
  messageId: number
): Promise<ValidationResponse | ValidationPendingResponse | null> {
  const url = `${getApiUrl()}/api/messages/${messageId}/validation`;

  const response = await fetch(url, { headers });

  // 202 Accepted - validation is pending in background
  if (response.status === 202) {
    return await response.json() as ValidationPendingResponse;
  }

  // 200 OK - validation results available
  if (response.ok) {
    const data = await response.json();
    if (data.total_articles_found === 0) {
      return null;  // No validation to show
    }
    return data as ValidationResponse;
  }

  // Error
  throw new Error(error.detail);
}
```

---

## Which Pattern to Use?

| Scenario | Pattern |
|----------|---------|
| Standard pages (messages, search, entities) | Use `lib/api.ts` functions |
| Admin pages | `lib/admin-api.ts` (`adminGet`, `adminPost`, ...) |
| New endpoints not in client | Add to `lib/api.ts` (or `lib/api/<area>.ts`, e.g. `vessels.ts`) |
| Server components (SSR) | `lib/api.ts` works automatically |
| Client components with shared or refreshing data | TanStack Query hook in `hooks/` calling `lib/api.ts` |

### Adding New API Functions

```typescript
// lib/api.ts

/**
 * Get my new feature data
 */
export async function getMyFeature(id: number): Promise<MyFeatureType> {
  return fetchApi<MyFeatureType>(`/api/my-feature/${id}`);
}

// Add to exports
export const api = {
  // ... existing
  getMyFeature,
};
```

---

## Related Documentation

- [Adding Features](adding-features.md) - Full-stack feature guide
- [API Reference](../reference/api-endpoints.md) - All API endpoints
- [Environment Variables](../reference/environment-variables.md) - Configuration
