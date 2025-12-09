# Frontend API Patterns

Guide for making API calls from the Next.js frontend.

---

## Overview

The frontend has two patterns for API calls:

1. **Centralized API client** (`lib/api.ts`) - Preferred for standard pages
2. **Direct fetch with NEXT_PUBLIC_API_URL** - Used in admin pages

The centralized client handles server vs client context automatically.

---

## Centralized API Client

### Location

`services/frontend-nextjs/lib/api.ts`

### How It Works

```typescript
// lib/api.ts (line 27)
function getApiUrl(): string {
  // Server-side (Next.js server components)
  if (typeof window === 'undefined') {
    return process.env.API_URL || 'http://api:8000';
  }

  // Client-side (browser) - use relative URLs or NEXT_PUBLIC_API_URL
  return process.env.NEXT_PUBLIC_API_URL || '';
}
```

**Key points:**
- **Server-side** (SSR): Uses `API_URL` → `http://api:8000` (Docker internal network)
- **Client-side** (browser): Uses `NEXT_PUBLIC_API_URL` or empty string (relative URLs)

### Available Functions

```typescript
import {
  searchMessages,
  getMessage,
  getSimilarMessages,
  getChannels,
  getChannel,
  getEntity,
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

export default async function MessagePage({ params }: { params: { id: string } }) {
  const message = await getMessage(parseInt(params.id));
  const similar = await getSimilarMessages(parseInt(params.id), 5);

  return (
    <div>
      <h1>{message.content}</h1>
      {/* ... */}
    </div>
  );
}
```

---

## Direct Fetch Pattern (Admin Pages)

Admin pages use direct fetch with `NEXT_PUBLIC_API_URL`:

```typescript
// app/admin/page.tsx (line 33)
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';

// Usage
const response = await fetch(`${API_URL}/api/admin/stats`);
```

### Why Admin Pages Are Different

Admin pages often:
- Call endpoints not in the centralized client
- Need more control over fetch options
- Are client-side only (use `'use client'`)

### Admin Page Pattern

```typescript
// app/admin/channels/page.tsx
'use client';

const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';

export default function AdminChannelsPage() {
  const [channels, setChannels] = useState([]);

  useEffect(() => {
    fetch(`${API_URL}/api/admin/channels`)
      .then(res => res.json())
      .then(data => setChannels(data));
  }, []);

  return <div>{/* ... */}</div>;
}
```

---

## Environment Variables

| Variable | Context | Default | Purpose |
|----------|---------|---------|---------|
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
// lib/api.ts (line 101)
async function fetchApi<T>(path: string, options?: RequestInit): Promise<T> {
  const url = `${getApiUrl()}${path}`;

  // Get auth token if available
  const token = getAuthToken();
  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
    ...(options?.headers as Record<string, string>),
  };

  if (token) {
    headers['Authorization'] = `Bearer ${token}`;
  }

  const response = await fetch(url, {
    ...options,
    headers,
    cache: 'no-store',  // Dynamic data, no caching
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({ detail: 'Unknown error' }));
    throw new Error(error.detail || `API error: ${response.status}`);
  }

  return response.json();
}
```

### Features

- **Automatic auth** - Injects Bearer token if available
- **Error handling** - Parses error.detail from API responses
- **No caching** - Uses `cache: 'no-store'` for dynamic data
- **Expected error filtering** - Doesn't log expected errors (missing embeddings, etc.)

---

## Media URLs

For archived media (images, videos), use `getMediaUrl()`:

```typescript
import { getMediaUrl } from '@/lib/api';

// Convert S3 key to full URL
const imageUrl = getMediaUrl(message.s3_key);
// Returns: http://localhost:9000/telegram-archive/media/2f/a1/abc123.jpg

// For OpenGraph tags (must be publicly accessible)
const ogImageUrl = getMediaUrl(message.s3_key, true);  // forceExternal=true
```

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
  telegram_date: string;
  is_spam: boolean;
  osint_topic: string | null;
  importance_level: string | null;
  media_type: string | null;
  s3_key: string | null;
  // ... more
}

interface SearchParams {
  q?: string;
  channel_id?: number;
  topic?: string;
  days?: number;
  limit?: number;
  offset?: number;
  sort?: 'desc' | 'asc';
}

interface SearchResult {
  items: Message[];
  total: number;
  offset: number;
  limit: number;
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
// lib/api.ts (line 285)
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
| Admin pages | Direct fetch with `NEXT_PUBLIC_API_URL` |
| New endpoints not in client | Add to `lib/api.ts` or use direct fetch |
| Server components (SSR) | `lib/api.ts` works automatically |
| Client components | `lib/api.ts` or direct fetch |

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
- [Environment Variables](../reference/environment-vars.md) - Configuration
