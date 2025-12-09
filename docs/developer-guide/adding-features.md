# Adding Features

Guide for extending the OSINT Intelligence Platform with new features.

---

## Overview

The platform supports several extension points:

1. **Enrichment Tasks** - Background processing for message enrichment
2. **API Endpoints** - New REST endpoints in the FastAPI backend
3. **Frontend Components** - New UI features in Next.js

This guide focuses on the most common pattern: adding enrichment tasks.

---

## Adding an Enrichment Task

Enrichment tasks run in the background to enhance message data. Examples include:
- AI tagging (keyword/topic extraction)
- Entity matching (link messages to entities)
- Translation (DeepL integration)
- Embeddings (vector generation)

### Step 1: Create Task Class

Create a new file in `services/enrichment/src/tasks/`:

```python
# services/enrichment/src/tasks/my_new_task.py
"""
My New Enrichment Task.

Description of what this task does.
"""

import logging
from typing import Any, List

from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from .base import BaseEnrichmentTask

logger = logging.getLogger(__name__)


class MyNewTask(BaseEnrichmentTask):
    """
    My task description.

    Implements the three required methods:
    - get_task_name(): Return unique task identifier
    - get_work_query(): Return SQL to fetch messages needing enrichment
    - process_batch(): Process a batch of messages
    """

    def __init__(self, my_config_option: str = "default"):
        """
        Initialize task with configuration.

        Args:
            my_config_option: Description of config option
        """
        self.my_config = my_config_option
        self.messages_processed = 0
        self.errors = 0

    def requires_llm(self) -> bool:
        """
        Return True if task uses Ollama LLM.

        LLM tasks run SEQUENTIALLY to prevent Ollama contention.
        Non-LLM tasks run in PARALLEL with configurable concurrency.

        Returns:
            True if task calls Ollama, False otherwise
        """
        return False  # Change to True if using Ollama

    def get_priority(self) -> int:
        """
        Return task priority (0-100, higher = runs first).

        Priority Guidelines:
        - 100: Critical for UX (search, display)
        - 75: Important background enrichment
        - 50: Nice-to-have enrichment (default)
        - 25: Low priority maintenance
        """
        return 50

    def get_task_name(self) -> str:
        """Return unique task identifier."""
        return "my_new_task"

    async def get_work_query(self, batch_size: int, last_processed_id: int = 0) -> text:
        """
        Return SQL query to fetch messages needing enrichment.

        Query requirements:
        - Order by id ASC for resumability
        - Filter out already-enriched messages
        - Use :batch_size and :last_id parameters

        Args:
            batch_size: Number of messages to fetch
            last_processed_id: Resume from this message ID

        Returns:
            SQLAlchemy text() query
        """
        return text("""
            SELECT m.id, m.content, m.content_translated
            FROM messages m
            WHERE m.id > :last_id
              AND m.is_spam = false
              AND m.content IS NOT NULL
              -- Add your "not yet processed" condition:
              AND m.my_new_field IS NULL
            ORDER BY m.id ASC
            LIMIT :batch_size
        """)

    async def process_batch(self, messages: List[Any], session: AsyncSession) -> int:
        """
        Process a batch of messages.

        The base class handles:
        - Progress tracking
        - Metrics recording
        - Transaction commit (after this method returns)
        - Error handling

        Args:
            messages: List of message rows from get_work_query()
            session: Database session for updates

        Returns:
            Number of messages successfully processed
        """
        processed = 0

        for msg in messages:
            try:
                # Your enrichment logic here
                result = await self._enrich_message(msg)

                # Update database
                await session.execute(
                    text("""
                        UPDATE messages
                        SET my_new_field = :value
                        WHERE id = :id
                    """),
                    {"id": msg.id, "value": result}
                )

                processed += 1
                self.messages_processed += 1

            except Exception as e:
                logger.error(f"Error processing message {msg.id}: {e}")
                self.errors += 1
                continue

        return processed

    async def _enrich_message(self, msg) -> str:
        """Your enrichment logic here."""
        # Use msg.content or msg.content_translated
        content = msg.content_translated or msg.content
        # Process and return result
        return "enriched_value"
```

### Step 2: Register Task in main.py

Add your task to `services/enrichment/src/main.py`:

```python
# services/enrichment/src/main.py

# Add import at top
from .tasks.my_new_task import MyNewTask

# In main() function, after creating coordinator:
if 'my_new_task' in config.enabled_tasks:
    task = MyNewTask(
        my_config_option=config.my_task_config,
    )
    coordinator.register_task(task)
    logger.info("✓ My new task registered")
```

### Step 3: Add Configuration

Add environment variable to `services/enrichment/src/config.py`:

```python
# In EnrichmentConfig class
my_task_config: str = os.getenv("MY_TASK_CONFIG", "default")
```

Add to `docker-compose.yml`:

```yaml
enrichment:
  environment:
    - ENRICHMENT_TASKS=translation,entity_matching,my_new_task
    - MY_TASK_CONFIG=production_value
```

### Step 4: Database Schema (if needed)

If your task writes to new columns, add them to `infrastructure/postgres/init.sql`:

```sql
-- In messages table definition
ALTER TABLE messages ADD COLUMN my_new_field TEXT;

-- Or create new table
CREATE TABLE IF NOT EXISTS my_enrichment_results (
    id SERIAL PRIMARY KEY,
    message_id INTEGER REFERENCES messages(id) ON DELETE CASCADE,
    result_data JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_my_enrichment_message_id
ON my_enrichment_results(message_id);
```

Then rebuild database (see [Database Migrations](database-migrations.md)).

### Step 5: Test

```bash
# Run enrichment with just your task
ENRICHMENT_TASKS=my_new_task docker-compose up -d enrichment

# Check logs
docker-compose logs -f enrichment | grep -i "my_new_task"

# Verify database
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(*) FROM messages WHERE my_new_field IS NOT NULL"
```

---

## LLM Task Example

For tasks that use Ollama:

```python
class MyLLMTask(BaseEnrichmentTask):
    """Task using Ollama LLM."""

    def requires_llm(self) -> bool:
        """Mark as LLM task - runs sequentially."""
        return True

    def get_priority(self) -> int:
        """High priority - runs first among LLM tasks."""
        return 100

    async def process_batch(self, messages: List[Any], session: AsyncSession) -> int:
        # Import shared Ollama client
        from ai import create_ollama_client

        async with create_ollama_client(
            host=self.ollama_host,
            timeout=180.0
        ) as client:
            for msg in messages:
                response = await client.generate(
                    model="qwen2.5:3b",
                    prompt=f"Analyze: {msg.content}",
                    temperature=0.3,
                    max_tokens=500,
                )
                # Process response...
```

---

## Adding an API Endpoint

### Step 1: Create Router or Add to Existing

```python
# services/api/src/routers/my_feature.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from ..database import get_db
from ..schemas import MyFeatureResponse

router = APIRouter(prefix="/api/my-feature", tags=["my-feature"])


@router.get("/{item_id}", response_model=MyFeatureResponse)
async def get_my_feature(
    item_id: int,
    db: AsyncSession = Depends(get_db)
):
    """Get my feature data."""
    result = await db.execute(
        select(MyModel).where(MyModel.id == item_id)
    )
    item = result.scalar_one_or_none()

    if not item:
        raise HTTPException(status_code=404, detail="Not found")

    return MyFeatureResponse(
        id=item.id,
        data=item.data,
    )
```

### Step 2: Register Router

Add to `services/api/src/main.py`:

```python
from .routers import my_feature

app.include_router(my_feature.router)
```

### Step 3: Add Schema

Add Pydantic models to `services/api/src/schemas.py`:

```python
class MyFeatureResponse(BaseModel):
    id: int
    data: str

    class Config:
        from_attributes = True
```

### Step 4: Add Frontend API Function

Add to `services/frontend-nextjs/lib/api.ts`:

```typescript
export async function getMyFeature(id: number): Promise<MyFeatureType> {
  return fetchApi<MyFeatureType>(`/api/my-feature/${id}`);
}
```

---

## Adding Frontend Components

### Server Component (SSR)

```typescript
// services/frontend-nextjs/app/my-feature/[id]/page.tsx
import { getMyFeature } from '@/lib/api';

export default async function MyFeaturePage({
  params
}: {
  params: { id: string }
}) {
  const data = await getMyFeature(parseInt(params.id));

  return (
    <div>
      <h1>{data.title}</h1>
      {/* ... */}
    </div>
  );
}
```

### Client Component

```typescript
// services/frontend-nextjs/components/MyComponent.tsx
'use client';

import { useState, useEffect } from 'react';
import { getMyFeature } from '@/lib/api';

export default function MyComponent({ id }: { id: number }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    getMyFeature(id)
      .then(setData)
      .finally(() => setLoading(false));
  }, [id]);

  if (loading) return <div>Loading...</div>;
  return <div>{/* ... */}</div>;
}
```

---

## Critical Rules

### Telegram Client Handling

**NEVER create standalone Telegram clients in tasks.** Pass from main.py:

```python
# ✅ CORRECT
class MyTask:
    def __init__(self, telegram_client: Optional[TelegramClient]):
        self.client = telegram_client

# ❌ WRONG - Never do this
client = TelegramClient(...)  # in task class
```

### Transaction Handling

The `BaseEnrichmentTask.run_cycle()` handles commits automatically:

```python
# ✅ CORRECT - Base class commits after process_batch returns
async def process_batch(self, messages, session):
    await session.execute(text("UPDATE ..."))
    return count  # Commit happens automatically

# ❌ WRONG - Don't commit manually in enrichment tasks
async def process_batch(self, messages, session):
    await session.execute(text("UPDATE ..."))
    await session.commit()  # Don't do this!
```

### Database Schema Changes

Always update `init.sql` after manual ALTER commands:

```bash
# 1. Apply manually to running database
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "ALTER TABLE messages ADD COLUMN new_field TEXT"

# 2. Update init.sql for future deployments
# Edit infrastructure/postgres/init.sql

# 3. Update ORM model
# Edit shared/python/models/message.py
```

---

## Testing New Features

### Unit Tests

Create test file in service's `tests/` directory:

```python
# services/enrichment/tests/test_my_new_task.py
import pytest
from unittest.mock import AsyncMock, patch

from src.tasks.my_new_task import MyNewTask


class TestMyNewTask:
    @pytest.fixture
    def task(self):
        return MyNewTask(my_config="test")

    def test_get_task_name(self, task):
        assert task.get_task_name() == "my_new_task"

    def test_requires_llm_false(self, task):
        assert task.requires_llm() == False

    @pytest.mark.asyncio
    async def test_process_batch(self, task, db_session, sample_message):
        # Test batch processing
        messages = [sample_message]
        result = await task.process_batch(messages, db_session)
        assert result == 1
```

### Run Tests

```bash
# Run specific test file
pytest services/enrichment/tests/test_my_new_task.py -v

# Run with coverage
pytest services/enrichment/tests/ --cov=services/enrichment/src
```

---

## Development Workflow

1. **Create feature branch** from `develop`
2. **Implement feature** following patterns above
3. **Write tests** with >80% coverage
4. **Test locally** with Docker Compose
5. **Submit PR** to `develop`
6. **Address review** comments
7. **Merge to develop** after approval
8. **Merge to master** for production

---

## Related Documentation

- [LLM Integration](llm-integration.md) - Working with Ollama prompts
- [Frontend API Patterns](frontend-api-patterns.md) - API client usage
- [Database Migrations](database-migrations.md) - Schema changes
- [Testing Guide](testing-guide.md) - Test patterns and fixtures
