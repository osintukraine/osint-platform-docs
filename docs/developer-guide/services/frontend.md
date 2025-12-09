# Frontend Service

> Next.js 14 web application providing OSINT analyst interface for searching, browsing, and analyzing archived intelligence

## Overview

The Frontend Service is a modern React-based web application built with Next.js 14, serving as the primary user interface for the OSINT Intelligence Platform. It provides a rich, responsive experience for analysts to search, filter, visualize, and interact with archived Telegram data.

**Key Capabilities:**
- Full-text and semantic search across 254+ channels
- AI-enriched message cards with 3 density modes (compact, detailed, immersive)
- Interactive graph visualizations (Cytoscape, ReactFlow)
- Real-time data fetching with React Query
- Dark/light theme support with CSS variable theming
- RSS/Atom/JSON feed subscription with signed URLs

**Technology Stack:**
- **Framework**: Next.js 14.2 (App Router architecture)
- **Runtime**: Bun 1.0 (faster than Node.js)
- **Language**: TypeScript 5
- **Styling**: TailwindCSS 3.4 with custom CSS variables
- **State Management**: TanStack Query (React Query) v5
- **Graph Visualization**: Cytoscape.js 3.33, ReactFlow 11.11
- **Charts**: Recharts 2.13
- **Authentication**: Ory Kratos integration (placeholder)
- **Icons**: Lucide React

## Architecture

### Directory Structure

```
services/frontend-nextjs/
├── app/                        # Next.js App Router pages
│   ├── layout.tsx              # Root layout with providers
│   ├── page.tsx                # Home (message browser)
│   ├── search/                 # Unified search
│   ├── messages/               # Message detail pages
│   ├── channels/               # Channel browser
│   ├── entities/               # Entity profiles
│   ├── events/                 # Event timeline
│   ├── admin/                  # Admin panel
│   │   ├── page.tsx            # Dashboard
│   │   ├── channels/           # Channel management
│   │   ├── stats/              # Platform statistics
│   │   ├── spam/               # Spam review
│   │   ├── prompts/            # LLM prompt editor
│   │   └── ...
│   ├── about/                  # Architecture visualization
│   ├── settings/               # User settings
│   └── unified/                # Unified search (semantic)
├── components/                 # Reusable components
│   ├── PostCard.tsx            # Message display (3 density modes)
│   ├── MediaLightbox.tsx       # Fullscreen media viewer
│   ├── SearchFilters.tsx       # 15+ filter controls
│   ├── HeaderNav.tsx           # Navigation bar
│   ├── admin/                  # Admin components
│   ├── social-graph/           # Social network viz
│   └── about/                  # Architecture diagram
├── lib/                        # Utilities and API client
│   ├── api.ts                  # API client (60+ functions)
│   ├── types.ts                # TypeScript types
│   ├── query-provider.tsx      # React Query setup
│   └── utils.ts                # Helpers
├── contexts/                   # React contexts
│   └── AuthContext.tsx         # Auth (placeholder)
├── public/                     # Static assets
├── next.config.js              # Next.js configuration
├── tailwind.config.ts          # TailwindCSS config
├── Dockerfile                  # Multi-stage build with Bun
└── package.json                # Dependencies
```

### Key Patterns

**1. Server-Side Rendering (SSR) with Client-Side Hydration**

Next.js App Router uses React Server Components by default. Data fetching happens on the server for initial page load, then React Query manages client-side updates.

```typescript
// app/page.tsx - Server Component
export default async function HomePage({ searchParams }: HomePageProps) {
  // Fetch on server for initial render
  const result = await searchMessages(params);

  return (
    <BrowseMessages
      messages={result.items}
      currentPage={result.page}
      // ... passes server-fetched data to client components
    />
  );
}
```

**2. API URL Pattern (Critical for Docker)**

The frontend uses different API URLs depending on execution context:

```typescript
// lib/api.ts
function getApiUrl(): string {
  // Server-side (Next.js in Docker container)
  if (typeof window === 'undefined') {
    return process.env.API_URL || 'http://api:8000';
  }

  // Client-side (browser)
  return process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';
}
```

> **Warning:** Never use relative paths (`/api/...`) for API calls. Next.js rewrites don't work reliably in Docker. Always use absolute URLs from `NEXT_PUBLIC_API_URL`.

**3. React Query for Client-Side Data Fetching**

```typescript
// Example: Fetching entity data client-side
import { useQuery } from '@tanstack/react-query';
import { getEntity } from '@/lib/api';

function EntityProfile({ source, id }: Props) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['entity', source, id],
    queryFn: () => getEntity(source, id),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });

  // ... render entity data
}
```

**4. Three-Density Message Display**

PostCard component adapts to three viewing modes:

- **Compact** (120px): Timeline scrolling, 2-line preview
- **Detailed** (auto height): Full content + AI enrichment
- **Immersive** (fullscreen): Deep analysis modal

```typescript
<PostCard
  message={message}
  channel={channel}
  density="detailed"
  onDensityChange={(newDensity) => setDensity(newDensity)}
/>
```

## App Routes

### Public Routes

| Route | Description | Data Source |
|-------|-------------|-------------|
| `/` | **Home / Message Browser** | `GET /api/messages` with filters |
| `/search` | **Unified Search** | `GET /api/search` (text + semantic) |
| `/messages/[id]` | **Message Detail** | `GET /api/messages/{id}` |
| `/channels` | **Channel List** | `GET /api/channels` |
| `/channels/[username]` | **Channel Profile** | `GET /api/channels?username=...` |
| `/entities/[source]/[id]` | **Entity Profile** | `GET /api/entities/{source}/{id}` |
| `/events` | **Event Timeline** | `GET /api/events` |
| `/events/[id]` | **Event Detail** | `GET /api/events/{id}` |
| `/about` | **Architecture Visualization** | Static + `GET /api/platform/stats` |

### Admin Routes

| Route | Description | Auth Required |
|-------|-------------|---------------|
| `/admin` | **Dashboard** | Future (Ory Kratos) |
| `/admin/channels` | **Channel Management** | Future |
| `/admin/stats` | **Platform Statistics** | Future |
| `/admin/spam` | **Spam Review Queue** | Future |
| `/admin/prompts` | **LLM Prompt Editor** | Future |
| `/admin/entities` | **Entity Curation** | Future |
| `/admin/config` | **System Configuration** | Future |
| `/admin/export` | **Data Export** | Future |
| `/admin/audit` | **Audit Logs** | Future |
| `/settings/feed-tokens` | **Feed Token Management** | Future |

> **Note:** Authentication is currently a placeholder. Ory Kratos is deployed but not integrated. All routes are publicly accessible.

## Key Components

### PostCard

**Purpose:** Display message content with adaptive density modes.

**Features:**
- 3 view modes: compact (120px), detailed (auto), immersive (fullscreen)
- AI enrichment display (sentiment, urgency, key phrases, summary)
- Entity chips (OpenSanctions + curated knowledge graph)
- Media handling (image, video, audio, document)
- Social graph indicators (forwards, replies, comments)
- Translation toggle (original ↔ translated)
- RSS validation panel integration

**Usage:**
```typescript
import { PostCard } from '@/components/PostCard';

<PostCard
  message={message}
  channel={channel}
  density="detailed"
  onDensityChange={(mode) => setDensity(mode)}
  onClick={() => router.push(`/messages/${message.id}`)}
/>
```

**Props:**
- `message: Message` - Message object from API
- `channel?: Channel` - Channel metadata
- `density?: 'compact' | 'detailed' | 'immersive'` - View mode
- `onDensityChange?: (mode) => void` - Density change handler
- `onClick?: () => void` - Card click handler

### MediaLightbox

**Purpose:** Full-screen media viewer with keyboard navigation.

**Features:**
- Supports image, video, document
- Left/Right arrow navigation
- Touch swipe on mobile
- Thumbnail strip for albums
- Escape key to close
- Preloads adjacent images

**Usage:**
```typescript
import { MediaLightbox } from '@/components/MediaLightbox';

<MediaLightbox
  mediaUrls={message.media_urls}
  initialIndex={0}
  isOpen={lightboxOpen}
  onClose={() => setLightboxOpen(false)}
  getMediaType={(url) => detectMediaType(url)}
/>
```

### SearchFilters

**Purpose:** 15+ filter controls for message search.

**Features:**
- Text search (original + translated)
- Semantic search toggle (AI embeddings)
- Country filter (Ukraine 🇺🇦, Russia 🇷🇺)
- Channel dropdown (254+ channels)
- Media type filters
- Importance level (high/medium/low)
- Topic classification (12 categories)
- Language detection
- Date range (last N days OR from/to)
- Engagement thresholds (min views/forwards)
- Spam filter controls
- Human review status
- Comments filter
- Sort by/order
- RSS feed subscription (signed URLs)

**Usage:**
```typescript
import { SearchFilters } from '@/components/SearchFilters';

<SearchFilters initialParams={searchParams} />
// Automatically navigates on filter change
```

### EntityRelationshipGraph

**Purpose:** Interactive Cytoscape.js graph for entity relationships.

**Features:**
- Displays corporate, political, and personal relationships
- Color-coded relationship types
- Node sizing by connection count
- Edge labels with relationship names
- Layout algorithms (cose-bilkent, fcose)
- Export as PNG
- Zoom/pan controls

**Usage:**
```typescript
import { EntityRelationshipGraph } from '@/components/EntityRelationshipGraph';

<EntityRelationshipGraph
  source={entitySource}
  entityId={entityId}
  relationships={relationshipsData}
/>
```

### HeaderNav

**Purpose:** Main navigation bar with theme toggle.

**Features:**
- Logo + title
- Navigation links (Home, Search, Channels, Entities, Events, Admin, About)
- Active route highlighting
- Theme toggle (dark/light)
- Responsive mobile menu

## API Integration

### API Client (`lib/api.ts`)

The API client provides 60+ typed functions for interacting with the backend.

**Core Pattern:**

```typescript
/**
 * Fetch helper with error handling and authentication
 */
async function fetchApi<T>(path: string, options?: RequestInit): Promise<T> {
  const url = `${getApiUrl()}${path}`;

  const response = await fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
    cache: 'no-store', // Disable Next.js caching - data is dynamic
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({ detail: 'Unknown error' }));
    throw new Error(error.detail || `API error: ${response.status}`);
  }

  return response.json();
}
```

**Why Relative Paths Don't Work in Docker:**

Next.js runs in a container where the internal hostname is `frontend`, but the API is at `http://api:8000`. Browser-side code runs in the user's browser, which can't resolve the `api` Docker network hostname.

**Solution:** Use `NEXT_PUBLIC_API_URL` for all client-side calls:

```typescript
// ✅ CORRECT - Client-side component
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';
const res = await fetch(`${API_URL}/api/messages`);

// ❌ WRONG - Doesn't work in Docker
const res = await fetch('/api/messages'); // Next.js rewrite fails
```

### Key API Functions

**Messages:**
```typescript
searchMessages(params: SearchParams): Promise<SearchResult>
getMessage(id: number): Promise<Message>
getSimilarMessages(id: number): Promise<Message[]>
getMessageTimeline(id: number, params): Promise<TimelineResult>
```

**Channels:**
```typescript
getChannels(): Promise<Channel[]>
getChannel(id: number): Promise<Channel>
getChannelByUsername(username: string): Promise<Channel>
```

**Entities:**
```typescript
getEntity(source: 'curated' | 'opensanctions', id: string): Promise<EntityDetail>
getEntityMessages(source, id, params): Promise<LinkedMessage[]>
searchEntities(params): Promise<{ items: Entity[], total: number }>
getEntityRelationships(source, id): Promise<RelationshipsResponse>
```

**Events:**
```typescript
getEvents(params): Promise<EventListResponse>
getEvent(id: number, params): Promise<Event>
getEventsForMessage(id: number): Promise<{ events: Event[] }>
getEventTimeline(id: number): Promise<TimelineResult>
```

**Unified Search:**
```typescript
unifiedSearch(params: {
  q: string;
  mode?: 'text' | 'semantic';
  types?: string; // 'messages,entities,channels,events'
}): Promise<UnifiedSearchResponse>
```

**Social Graph:**
```typescript
getMessageSocialGraph(id: number, params): Promise<SocialGraphData>
getEngagementTimeline(id: number, params): Promise<EngagementData>
getMessageComments(id: number, params): Promise<CommentThread>
```

## Styling

### TailwindCSS Configuration

**CSS Variables for Theming:**

```typescript
// tailwind.config.ts
theme: {
  extend: {
    colors: {
      // Background colors
      'bg-base': 'var(--bg-base)',
      'bg-elevated': 'var(--bg-elevated)',
      'bg-secondary': 'var(--bg-secondary)',

      // Text colors
      'text-primary': 'var(--text-primary)',
      'text-secondary': 'var(--text-secondary)',

      // Accent colors
      'accent-primary': '#4a9eff',
      'accent-danger': '#ef4444',
      // ...
    },
  },
},
```

**Safelist for Dynamic Classes:**

Topic badges use dynamic classes that need to be preserved:

```typescript
safelist: [
  'topic-combat',
  'topic-civilian',
  'topic-diplomatic',
  'topic-equipment',
  'topic-general',
  'topic-rule_based',
],
```

### Dark/Light Mode

Theme switching via `next-themes`:

```typescript
import { useTheme } from 'next-themes';

const { theme, setTheme } = useTheme();

<button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>
  Toggle Theme
</button>
```

CSS variables are defined in `app/globals.css`:

```css
:root {
  --bg-base: #ffffff;
  --text-primary: #1a1a1a;
  /* ... */
}

.dark {
  --bg-base: #0a0a0a;
  --text-primary: #f5f5f5;
  /* ... */
}
```

### Responsive Design

TailwindCSS breakpoints:

```typescript
// Responsive example
<div className="
  grid grid-cols-1        // Mobile: 1 column
  md:grid-cols-2          // Tablet: 2 columns
  lg:grid-cols-3          // Desktop: 3 columns
  gap-4
">
```

## Configuration

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `NEXT_PUBLIC_API_URL` | API base URL (client-side) | `http://localhost:8000` | Yes |
| `NEXT_PUBLIC_MINIO_URL` | MinIO media URL (client-side) | `http://localhost:9000/telegram-archive` | Yes |
| `NEXT_PUBLIC_BASE_URL` | Frontend base URL (OpenGraph) | `http://localhost:3000` | No |
| `API_URL` | API URL (server-side) | `http://api:8000` | No |
| `MINIO_URL` | MinIO URL (server-side) | `http://minio:9000/telegram-archive` | No |
| `NEXT_PUBLIC_BASE_PATH` | Base path for subpath deployments | `` | No |

> **Critical:** `NEXT_PUBLIC_*` variables are baked into the bundle at **build time**. If you change them, you must rebuild the Docker image.

### Docker Build Arguments

```dockerfile
# Dockerfile
ARG NEXT_PUBLIC_API_URL=http://localhost:8000
ARG NEXT_PUBLIC_MINIO_URL=http://localhost:9000/telegram-archive
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
ENV NEXT_PUBLIC_MINIO_URL=${NEXT_PUBLIC_MINIO_URL}
```

**Build command:**
```bash
docker build \
  --build-arg NEXT_PUBLIC_API_URL=https://api.example.com \
  --build-arg NEXT_PUBLIC_MINIO_URL=https://media.example.com/telegram-archive \
  -t osint-frontend:latest \
  .
```

### Next.js Configuration

```javascript
// next.config.js
const nextConfig = {
  output: 'standalone', // For Docker deployment
  basePath: process.env.NEXT_PUBLIC_BASE_PATH || '',

  images: {
    remotePatterns: [
      {
        protocol: 'http',
        hostname: 'minio',
        port: '9000',
        pathname: '/telegram-archive/**',
      },
    ],
  },

  poweredByHeader: false, // Security: remove X-Powered-By header
};
```

## Development

### Local Development

**Prerequisites:**
- Bun 1.0+ (or Node.js 18+)
- Running API service at `http://localhost:8000`
- Running MinIO at `http://localhost:9000`

**Setup:**

```bash
cd services/frontend-nextjs

# Install dependencies
bun install

# Create .env.local
cp .env.example .env.local
# Edit .env.local with your API URLs

# Run development server
bun run dev
# Open http://localhost:3000
```

**Available Scripts:**

```bash
bun run dev          # Development server with hot reload
bun run build        # Production build
bun run start        # Start production server
bun run lint         # ESLint
bun run type-check   # TypeScript type checking
bun run test         # Vitest unit tests
```

### Hot Reload

Next.js dev server watches for file changes and automatically:
- Recompiles components
- Refreshes browser (Fast Refresh)
- Preserves component state (where possible)

**React Query Devtools:**

Enabled in development mode:

```typescript
// lib/query-provider.tsx
{process.env.NODE_ENV === 'development' && (
  <ReactQueryDevtools initialIsOpen={false} position="bottom" />
)}
```

Access at bottom of page to inspect query cache, refetch queries, and debug.

### Building for Production

```bash
# Build with default env vars
bun run build

# Build with custom API URL (for Docker)
NEXT_PUBLIC_API_URL=https://api.example.com bun run build
```

**Output:**

```
.next/
├── standalone/          # Self-contained server bundle
│   ├── server.js
│   └── node_modules/
├── static/              # Static assets (JS, CSS)
└── ...
```

## Troubleshooting

### API Connection Errors

**Problem:** `Failed to fetch from /api/messages`

**Cause:** Using relative URLs instead of `NEXT_PUBLIC_API_URL`.

**Solution:**

```typescript
// ❌ Wrong
const res = await fetch('/api/messages');

// ✅ Correct
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';
const res = await fetch(`${API_URL}/api/messages`);
```

### Hydration Errors

**Problem:** `Text content does not match server-rendered HTML`

**Cause:** Client-rendered content differs from server-rendered HTML (e.g., using `localStorage` without checking `typeof window`).

**Solution:**

```typescript
// ❌ Wrong - SSR will render null, client will render value
const [value, setValue] = useState(localStorage.getItem('key'));

// ✅ Correct - Use useEffect for client-only code
const [value, setValue] = useState(null);
useEffect(() => {
  if (typeof window !== 'undefined') {
    setValue(localStorage.getItem('key'));
  }
}, []);
```

### Media Not Loading

**Problem:** Images/videos show broken icons.

**Cause:** MinIO URL not accessible from browser.

**Solution:**

1. Check `NEXT_PUBLIC_MINIO_URL` is set correctly
2. Ensure MinIO is accessible at that URL from browser
3. Check MinIO bucket policy allows public read

```bash
# Test MinIO access
curl http://localhost:9000/telegram-archive/media/test.jpg
```

### Build Fails with "Module not found"

**Problem:** `Cannot find module '@/components/...'`

**Cause:** TypeScript path alias not configured.

**Solution:**

Check `tsconfig.json`:

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./*"]
    }
  }
}
```

### Dark Mode Not Working

**Problem:** Theme toggle doesn't change colors.

**Cause:** CSS variables not defined or `next-themes` not configured.

**Solution:**

1. Ensure `ThemeProvider` wraps app in `layout.tsx`
2. Check `globals.css` has `:root` and `.dark` CSS variables
3. Use TailwindCSS classes that reference CSS variables

```typescript
// ✅ Uses CSS variable
className="bg-bg-base text-text-primary"

// ❌ Hard-coded color (won't theme switch)
className="bg-white text-black"
```

### React Query Not Refetching

**Problem:** Data doesn't update when navigating back to page.

**Cause:** Query is cached and `staleTime` hasn't expired.

**Solution:**

```typescript
// Option 1: Reduce staleTime
const { data } = useQuery({
  queryKey: ['messages'],
  queryFn: fetchMessages,
  staleTime: 0, // Always refetch
});

// Option 2: Manual refetch
const { data, refetch } = useQuery(...);
<button onClick={() => refetch()}>Refresh</button>

// Option 3: Invalidate cache
const queryClient = useQueryClient();
queryClient.invalidateQueries({ queryKey: ['messages'] });
```

## Testing

**Unit Tests (Vitest):**

```bash
bun run test              # Run tests
bun run test:run          # Run once (CI)
bun run test:coverage     # Coverage report
```

**Example test:**

```typescript
// components/__tests__/PostCard.test.tsx
import { render, screen } from '@testing-library/react';
import { PostCard } from '@/components/PostCard';

test('renders message content', () => {
  const message = { id: 1, content: 'Test message', /* ... */ };
  render(<PostCard message={message} density="compact" />);

  expect(screen.getByText('Test message')).toBeInTheDocument();
});
```

**Integration Testing:**

Currently no integration tests. Future additions:
- Playwright for E2E testing
- MSW for API mocking
- Storybook for component development

## Related Documentation

- [API Service Documentation](./api.md) - Backend REST API
- [User Guide: Search & Filter](../../user-guide/search-and-filter.md) - End-user search guide
- [Architecture Overview](../../architecture/overview.md) - System architecture
- [Deployment Guide](../../deployment/docker-compose.md) - Docker deployment

## Common Tasks

### Adding a New Route

1. Create file in `app/` directory:

```typescript
// app/new-route/page.tsx
export default function NewRoutePage() {
  return (
    <div>
      <h1>New Route</h1>
    </div>
  );
}
```

2. Add navigation link:

```typescript
// components/HeaderNav.tsx
<Link href="/new-route">New Route</Link>
```

3. Add metadata:

```typescript
export const metadata = {
  title: 'New Route | OSINT Platform',
  description: 'Description for SEO',
};
```

### Adding a New Filter

1. Add state to `SearchFilters.tsx`:

```typescript
const [newFilter, setNewFilter] = useState('');
```

2. Add to `buildFilterUrl`:

```typescript
if (newFilter) params.set('new_filter', newFilter);
```

3. Add UI control:

```typescript
<select
  value={newFilter}
  onChange={(e) => {
    setNewFilter(e.target.value);
    router.push(buildFilterUrl({ newFilter: e.target.value }));
  }}
>
  <option value="">All</option>
  <option value="value1">Value 1</option>
</select>
```

4. Update API client to pass filter to backend.

### Adding a New Component

1. Create component file:

```typescript
// components/NewComponent.tsx
'use client'; // If needs client-side interactivity

interface NewComponentProps {
  data: string;
}

export function NewComponent({ data }: NewComponentProps) {
  return <div>{data}</div>;
}
```

2. Export from index (optional):

```typescript
// components/index.ts
export { NewComponent } from './NewComponent';
```

3. Use in page:

```typescript
import { NewComponent } from '@/components/NewComponent';

<NewComponent data="test" />
```

---

**Last Updated:** 2025-12-09
**Version:** 1.0
**Maintainer:** OSINT Platform Team
