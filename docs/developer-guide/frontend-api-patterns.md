# Frontend API Patterns

Guide for integrating the Next.js frontend with the FastAPI backend.

---

## The NEXT_PUBLIC_API_URL Pattern

**This is the most important pattern.** All client-side API calls must use `NEXT_PUBLIC_API_URL`.

### Why Not Relative Paths?

```typescript
// ❌ WRONG - This won't work in Docker
const res = await fetch('/api/messages');
```

**Problem**: Next.js runs in a container. When the browser makes a request to `/api/messages`, it goes to the Next.js server (port 3000), not the FastAPI backend (port 8000). Next.js rewrites don't work reliably in Docker.

### The Correct Pattern

```typescript
// ✅ CORRECT - Always works
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';

const res = await fetch(`${API_URL}/api/messages?limit=10`);
```

**Why this works**:
1. Client-side code runs in the browser
2. Browser can reach `localhost:8000` directly
3. In production, `NEXT_PUBLIC_API_URL` points to `https://api.yourdomain.com`

### Reference Implementation

See `services/frontend-nextjs/app/admin/page.tsx`:

```typescript
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';

export default function AdminPage() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch(`${API_URL}/api/admin/channels`)
      .then(res => res.json())
      .then(setData);
  }, []);

  // ...
}
```

---

## Error Handling

### Standard Error Response Format

The API returns errors in this format:

```json
{
  "detail": "Error message here"
}
```

Or for validation errors:

```json
{
  "detail": [
    {
      "loc": ["query", "limit"],
      "msg": "ensure this value is less than or equal to 100",
      "type": "value_error.number.not_le"
    }
  ]
}
```

### Error Handling Pattern

```typescript
async function fetchData(): Promise<Data | null> {
  const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';

  try {
    const res = await fetch(`${API_URL}/api/messages`);

    if (!res.ok) {
      const error = await res.json();
      throw new Error(error.detail || `HTTP ${res.status}`);
    }

    return await res.json();
  } catch (err) {
    console.error('API error:', err);
    // Show user-friendly error
    return null;
  }
}
```

### Using Error Boundaries

```tsx
// components/ErrorBoundary.tsx
'use client';

import { Component, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback: ReactNode;
}

interface State {
  hasError: boolean;
}

export class ErrorBoundary extends Component<Props, State> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

---

## Loading States

### Skeleton Components

```tsx
// components/MessageSkeleton.tsx
export function MessageSkeleton() {
  return (
    <div className="animate-pulse">
      <div className="h-4 bg-gray-200 rounded w-3/4 mb-2"></div>
      <div className="h-4 bg-gray-200 rounded w-1/2"></div>
    </div>
  );
}

// Usage
function MessageList() {
  const [loading, setLoading] = useState(true);
  const [messages, setMessages] = useState([]);

  if (loading) {
    return (
      <div className="space-y-4">
        {[...Array(5)].map((_, i) => (
          <MessageSkeleton key={i} />
        ))}
      </div>
    );
  }

  return messages.map(msg => <Message key={msg.id} {...msg} />);
}
```

### React Suspense (Next.js 13+)

```tsx
// app/messages/page.tsx
import { Suspense } from 'react';
import { MessageList } from '@/components/MessageList';
import { MessageSkeleton } from '@/components/MessageSkeleton';

export default function MessagesPage() {
  return (
    <Suspense fallback={<MessageSkeleton />}>
      <MessageList />
    </Suspense>
  );
}
```

---

## Type Safety

### Shared Types Location

Types are defined in `services/frontend-nextjs/lib/types.ts`:

```typescript
// lib/types.ts
export interface Message {
  id: number;
  telegram_id: number;
  content: string;
  channel_id: number;
  channel_name: string;
  telegram_date: string;
  osint_score: number;
  importance_level: 'high' | 'medium' | 'low';
  topics: string[];
  is_spam: boolean;
  media: MediaItem[];
}

export interface Channel {
  id: number;
  telegram_id: number;
  name: string;
  username: string | null;
  folder: string;
  rule: 'archive_all' | 'selective_archive' | 'discovery';
  active: boolean;
}

export interface Entity {
  id: string;
  source: 'curated' | 'opensanctions';
  name: string;
  entity_type: string;
  country: string | null;
  description: string | null;
  aliases: string[];
}
```

### Typing API Responses

```typescript
// lib/api.ts
import type { Message, Channel } from './types';

const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';

interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  pages: number;
}

export async function getMessages(params: {
  page?: number;
  limit?: number;
  q?: string;
}): Promise<PaginatedResponse<Message>> {
  const searchParams = new URLSearchParams();
  if (params.page) searchParams.set('page', String(params.page));
  if (params.limit) searchParams.set('limit', String(params.limit));
  if (params.q) searchParams.set('q', params.q);

  const res = await fetch(`${API_URL}/api/messages?${searchParams}`);
  if (!res.ok) throw new Error('Failed to fetch messages');

  return res.json();
}

export async function getChannel(id: number): Promise<Channel> {
  const res = await fetch(`${API_URL}/api/channels/${id}`);
  if (!res.ok) throw new Error('Failed to fetch channel');

  return res.json();
}
```

---

## Data Fetching Patterns

### Client-Side Fetching (useState + useEffect)

```tsx
'use client';

import { useState, useEffect } from 'react';
import type { Message } from '@/lib/types';

export function RecentMessages() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';

    fetch(`${API_URL}/api/messages?limit=10&importance_level=high`)
      .then(res => {
        if (!res.ok) throw new Error('Failed to fetch');
        return res.json();
      })
      .then(data => setMessages(data.items))
      .catch(err => setError(err.message))
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <ul>
      {messages.map(msg => (
        <li key={msg.id}>{msg.content.substring(0, 100)}...</li>
      ))}
    </ul>
  );
}
```

### Server-Side Fetching (Next.js Server Components)

```tsx
// app/messages/page.tsx (Server Component)
import type { Message } from '@/lib/types';

async function getMessages(): Promise<Message[]> {
  // Server-side: use internal Docker network
  const API_URL = process.env.API_URL || 'http://api:8000';

  const res = await fetch(`${API_URL}/api/messages?limit=10`, {
    next: { revalidate: 60 } // Cache for 60 seconds
  });

  if (!res.ok) throw new Error('Failed to fetch messages');

  const data = await res.json();
  return data.items;
}

export default async function MessagesPage() {
  const messages = await getMessages();

  return (
    <ul>
      {messages.map(msg => (
        <li key={msg.id}>{msg.content}</li>
      ))}
    </ul>
  );
}
```

**Note**: Server components can use `http://api:8000` (Docker internal network). Client components must use `NEXT_PUBLIC_API_URL`.

### SWR for Real-Time Data

```tsx
'use client';

import useSWR from 'swr';
import type { Message } from '@/lib/types';

const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';

const fetcher = (url: string) => fetch(url).then(res => res.json());

export function LiveMessages() {
  const { data, error, isLoading } = useSWR<{ items: Message[] }>(
    `${API_URL}/api/messages?limit=10`,
    fetcher,
    { refreshInterval: 30000 } // Refresh every 30 seconds
  );

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error loading messages</div>;

  return (
    <ul>
      {data?.items.map(msg => (
        <li key={msg.id}>{msg.content}</li>
      ))}
    </ul>
  );
}
```

---

## Environment Variables

### Client-Side Variables

Must be prefixed with `NEXT_PUBLIC_`:

```bash
# .env.local
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_RSS_URL=http://localhost:8000/rss
NEXT_PUBLIC_BASE_URL=http://localhost:3000
```

### Server-Side Variables

No prefix needed (not exposed to browser):

```bash
# .env.local
API_URL=http://api:8000  # Docker internal URL
DATABASE_URL=postgresql://...
```

### Production Configuration

```bash
# .env.production
NEXT_PUBLIC_API_URL=https://api.osintukraine.com
NEXT_PUBLIC_RSS_URL=https://api.osintukraine.com/rss
NEXT_PUBLIC_BASE_URL=https://osintukraine.com
```

---

## Common Mistakes

### 1. Using Relative Paths

```typescript
// ❌ WRONG
fetch('/api/messages')

// ✅ CORRECT
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';
fetch(`${API_URL}/api/messages`)
```

### 2. Forgetting NEXT_PUBLIC_ Prefix

```typescript
// ❌ WRONG - Won't be available in browser
const API_URL = process.env.API_URL;

// ✅ CORRECT
const API_URL = process.env.NEXT_PUBLIC_API_URL;
```

### 3. Not Handling Errors

```typescript
// ❌ WRONG - No error handling
const data = await fetch(url).then(r => r.json());

// ✅ CORRECT
const res = await fetch(url);
if (!res.ok) {
  const error = await res.json();
  throw new Error(error.detail || 'Request failed');
}
const data = await res.json();
```

### 4. Missing Loading States

```tsx
// ❌ WRONG - Flash of empty content
return <div>{messages.map(...)}</div>;

// ✅ CORRECT
if (loading) return <Skeleton />;
if (error) return <Error message={error} />;
return <div>{messages.map(...)}</div>;
```

---

## Related Documentation

- [Adding Features](adding-features.md) - Full feature development workflow
- [API Endpoints Reference](../reference/api-endpoints.md) - All available endpoints
- [Services: Frontend](services/frontend.md) - Frontend architecture
