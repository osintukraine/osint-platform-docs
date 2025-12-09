# Adding Features

Guide for extending the OSINT Intelligence Platform with new features.

---

## Adding an Enrichment Task

Enrichment tasks process messages in the background. The platform has 20+ tasks (embedding, translation, AI tagging, etc.).

### Step 1: Create Task Class

Inherit from `BaseEnrichmentTask` and implement required methods:

```python
# services/enrichment/src/tasks/my_task.py
from typing import Any, List
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from tasks.base import BaseEnrichmentTask


class MyEnrichmentTask(BaseEnrichmentTask):
    """Example enrichment task."""

    def get_task_name(self) -> str:
        """Unique task identifier (used in metrics, logs, progress tracking)."""
        return "my_task"

    def requires_llm(self) -> bool:
        """Return True if task uses Ollama (serializes LLM calls)."""
        return False  # Set True for LLM tasks

    def get_priority(self) -> int:
        """Higher = more urgent. 100=critical, 50=normal, 25=low."""
        return 50

    async def get_work_query(self, batch_size: int, last_processed_id: int = 0) -> text:
        """SQL query to fetch messages needing enrichment."""
        return text("""
            SELECT id, content, channel_id
            FROM messages
            WHERE id > :last_id
              AND my_field IS NULL  -- Not yet processed
            ORDER BY id ASC
            LIMIT :batch_size
        """)

    async def process_batch(self, messages: List[Any], session: AsyncSession) -> int:
        """Process messages. Return count of successfully processed."""
        processed = 0
        for msg in messages:
            try:
                # Your enrichment logic here
                result = self.do_enrichment(msg.content)

                # Update database
                await session.execute(
                    text("UPDATE messages SET my_field = :result WHERE id = :id"),
                    {"result": result, "id": msg.id}
                )
                processed += 1
            except Exception as e:
                logger.warning(f"Failed to process message {msg.id}: {e}")

        return processed

    def do_enrichment(self, content: str) -> str:
        """Your custom enrichment logic."""
        return f"enriched: {content[:50]}"
```

### Step 2: Register Task in Coordinator

Edit `services/enrichment/src/coordinator.py`:

```python
from tasks.my_task import MyEnrichmentTask

# In the coordinator's task registration
TASKS = {
    # ... existing tasks ...
    "my_task": MyEnrichmentTask(),
}
```

### Step 3: Add to Worker Pool

Edit `docker-compose.yml` to assign task to a worker:

```yaml
enrichment-fast-pool:
  environment:
    TASKS: "embedding,translation,entity_matching,my_task"  # Add here
```

Or create a dedicated worker:

```yaml
enrichment-my-task:
  <<: *enrichment-base
  environment:
    TASKS: "my_task"
    REDIS_QUEUE: "enrich:my_task"
  profiles:
    - enrichment
```

### Step 4: Add Configuration (Optional)

Add environment variables in `.env.example`:

```bash
# My Task Configuration
MY_TASK_BATCH_SIZE=100
MY_TASK_ENABLED=true
```

### Task Types by Worker

| Worker | Tasks | Characteristics |
|--------|-------|-----------------|
| `enrichment-fast-pool` | embedding, translation, entity_matching | CPU-bound, no LLM |
| `enrichment-ai-tagging` | ai_tagging | LLM-intensive |
| `enrichment-telegram` | comment_*, engagement_polling | Telegram API calls |
| `enrichment-rss-validation` | rss_validation, rss_correlation | RSS processing |
| `enrichment-maintenance` | cleanup, discovery_evaluator, wikidata | Periodic tasks |

---

## Adding an API Endpoint

### Step 1: Create Router

```python
# services/api/src/routers/my_feature.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel

from database import get_db
from schemas import MessageResponse

router = APIRouter(prefix="/my-feature", tags=["My Feature"])


class MyRequest(BaseModel):
    """Request schema."""
    query: str
    limit: int = 10


class MyResponse(BaseModel):
    """Response schema."""
    results: list
    total: int


@router.get("/", response_model=MyResponse)
async def get_my_feature(
    query: str,
    limit: int = 10,
    db: AsyncSession = Depends(get_db)
):
    """
    Get my feature data.

    - **query**: Search query
    - **limit**: Maximum results (1-100)
    """
    if limit > 100:
        raise HTTPException(status_code=400, detail="Limit must be ≤100")

    # Your logic here
    results = await fetch_results(db, query, limit)

    return MyResponse(results=results, total=len(results))


@router.post("/process")
async def process_something(
    request: MyRequest,
    db: AsyncSession = Depends(get_db)
):
    """Process something."""
    # Your logic here
    return {"status": "processed", "query": request.query}
```

### Step 2: Register Router

Edit `services/api/src/main.py`:

```python
from routers.my_feature import router as my_feature_router

# In app setup
app.include_router(my_feature_router)
```

### Step 3: Add Schemas (If Complex)

Edit `services/api/src/schemas.py`:

```python
class MyFeatureSchema(BaseModel):
    id: int
    name: str
    created_at: datetime

    class Config:
        from_attributes = True  # For ORM compatibility
```

### Step 4: Add Tests

```python
# services/api/tests/test_my_feature.py
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_get_my_feature(client: AsyncClient):
    response = await client.get("/my-feature/?query=test")
    assert response.status_code == 200
    assert "results" in response.json()


@pytest.mark.asyncio
async def test_limit_validation(client: AsyncClient):
    response = await client.get("/my-feature/?query=test&limit=999")
    assert response.status_code == 400
```

### Step 5: Document Endpoint

Update `docs/reference/api-endpoints.md` with the new endpoint.

---

## Adding a Frontend Feature

### Step 1: Create Component

```tsx
// services/frontend-nextjs/components/MyFeature.tsx
'use client';

import { useState, useEffect } from 'react';

const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';

interface MyData {
  id: number;
  name: string;
}

export function MyFeature() {
  const [data, setData] = useState<MyData[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function fetchData() {
      try {
        const res = await fetch(`${API_URL}/api/my-feature?limit=10`);
        if (!res.ok) throw new Error('Failed to fetch');
        const json = await res.json();
        setData(json.results);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error');
      } finally {
        setLoading(false);
      }
    }
    fetchData();
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      {data.map(item => (
        <div key={item.id}>{item.name}</div>
      ))}
    </div>
  );
}
```

### Step 2: Add Page (If Needed)

```tsx
// services/frontend-nextjs/app/my-feature/page.tsx
import { MyFeature } from '@/components/MyFeature';

export default function MyFeaturePage() {
  return (
    <main className="container mx-auto p-4">
      <h1 className="text-2xl font-bold mb-4">My Feature</h1>
      <MyFeature />
    </main>
  );
}
```

### Step 3: Add Types

```typescript
// services/frontend-nextjs/lib/types.ts
export interface MyFeatureData {
  id: number;
  name: string;
  created_at: string;
}
```

### Frontend API Pattern

**Always use `NEXT_PUBLIC_API_URL`** - never relative paths:

```typescript
// ✅ CORRECT
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';
const res = await fetch(`${API_URL}/api/messages`);

// ❌ WRONG - Won't work in Docker
const res = await fetch('/api/messages');
```

See [Frontend API Patterns](frontend-api-patterns.md) for details.

---

## Documentation Requirements

All features must include documentation updates:

| Change Type | Update Required |
|-------------|-----------------|
| New API endpoint | `docs/reference/api-endpoints.md` |
| New env variable | `docs/reference/environment-vars.md` |
| New service | `docs/reference/docker-services.md` |
| Schema change | `docs/reference/database-tables.md` |
| User workflow | `docs/user-guide/` or `docs/tutorials/` |
| Developer pattern | `docs/developer-guide/` |

---

## Critical Rules

### 1. Telegram Session Management

**NEVER create standalone Telegram clients.** Pass client from main.py:

```python
# ✅ CORRECT
class MyTask:
    def __init__(self, telegram_client: Optional[TelegramClient]):
        self.client = telegram_client

# ❌ WRONG - Never do this
client = TelegramClient(...)  # Creates conflicting session
```

### 2. Database Changes

Use `init.sql` as source of truth. No Alembic migrations:

```bash
# To test schema changes:
docker-compose down
docker volume rm osint-intelligence-platform_postgres_data
docker-compose up -d
```

See [Database Migrations](database-migrations.md) for details.

### 3. LLM Prompt Changes

Before modifying LLM prompts:

1. Read `docs/architecture/LLM_PROMPTS.md` in the platform repo
2. Create new version (don't edit active version)
3. Test with actual qwen2.5:3b model
4. Update documentation

See [LLM Integration](llm-integration.md) for details.

---

## Development Workflow

1. **Create feature branch** from `develop`
   ```bash
   git checkout develop
   git checkout -b feature/my-feature
   ```

2. **Implement feature** following patterns above

3. **Test locally**
   ```bash
   docker-compose up -d --build
   pytest services/api/tests/
   ```

4. **Update documentation** (this is required, not optional)

5. **Create PR** to `develop`
   ```bash
   git push -u origin feature/my-feature
   gh pr create --base develop
   ```

6. **Merge to master** for production (after review)

---

## Related Documentation

- [Frontend API Patterns](frontend-api-patterns.md) - Client-side API integration
- [LLM Integration](llm-integration.md) - Working with Ollama
- [Testing Guide](testing-guide.md) - Writing tests
- [Database Migrations](database-migrations.md) - Schema changes
- [Contributing](contributing.md) - Code style and PR guidelines
