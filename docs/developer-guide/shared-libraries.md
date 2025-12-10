# Shared Libraries

Documentation of shared Python libraries used across services.

---

## Overview

The `shared/python/` package contains **58 Python files** of common code used by all services:

- **Models**: 29 SQLAlchemy models (~4000 lines)
- **Configuration**: Pydantic settings with validation
- **Observability**: Logging, metrics, tracing
- **AI Utilities**: Ollama client wrapper
- **Storage**: MinIO media archiver
- **Translation**: DeepL integration

**Location**: `/shared/python/`

**Installation**: Installed as editable package in each service's Docker container:
```dockerfile
COPY shared/python /app/shared/python
RUN pip install -e /app/shared/python
```

---

## Package Structure

```
shared/python/
├── __init__.py              # Package metadata (v0.1.0)
├── requirements.txt         # Shared dependencies
├── models/                  # SQLAlchemy models (29 files)
│   ├── __init__.py         # Exports all models
│   ├── base.py             # AsyncSessionLocal, Base, engine
│   ├── message.py          # Core message model
│   ├── channel.py          # Channel/folder management
│   └── ...                 # 26 more model files
├── config/                  # Configuration management
│   ├── __init__.py
│   └── settings.py         # Pydantic Settings class
├── observability/           # Logging and metrics
│   ├── __init__.py         # setup_logging, record_* helpers
│   ├── logging.py          # Structured JSON logging
│   └── metrics.py          # Prometheus metrics
├── ai/                      # LLM utilities
│   ├── __init__.py
│   └── ollama_client.py    # Async Ollama wrapper
├── database/                # Database utilities
├── notifications/           # NotificationClient
├── storage/                 # Storage abstractions
├── telegram/                # Telegram utilities
├── translation/             # DeepL integration
├── utils/                   # General utilities
├── media_archiver.py        # Content-addressed media storage
└── translation.py           # Translation service
```

---

## Models (`models/`)

### Database Session Management

The `base.py` file provides async SQLAlchemy setup:

```python
# shared/python/models/base.py

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

# Create async engine with connection pooling
engine = create_async_engine(
    settings.DATABASE_URL.replace("postgresql://", "postgresql+asyncpg://"),
    pool_size=settings.POSTGRES_POOL_SIZE,      # Default: 20
    max_overflow=settings.POSTGRES_MAX_OVERFLOW, # Default: 10
    pool_pre_ping=True,                          # Verify connections
    pool_recycle=settings.POSTGRES_POOL_RECYCLE, # Default: 3600s
)

# Session factory
AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autoflush=False,
)

# FastAPI dependency
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### Model Exports

All models are exported from `models/__init__.py`:

```python
from models import (
    # Base
    Base, engine, AsyncSessionLocal, get_db,

    # Core
    Message, Channel, User,

    # Media
    MediaFile, MessageMedia,

    # Entities
    CuratedEntity, OpenSanctionsEntity,
    MessageEntity, OpenSanctionsMessageEntity,
    EntityRelationship,

    # RSS Intelligence
    RSSFeed, NewsSource, ExternalNews,
    MessageNewsCorrelation,

    # AI/LLM
    LLMPrompt, ModelConfiguration,
    MessageTag, TagStats,
    MilitarySlang, build_slang_glossary,

    # Events
    Event, EventMessage,

    # Validation
    MessageValidation, DecisionLog,

    # Translation
    TranslationConfig, TranslationUsage,

    # Other
    MessageQuarantine, MessageComment,
    FeedToken, ExportJob, ViralPost,
)
```

### Model Pattern

All models follow this pattern:

```python
# shared/python/models/message.py

from sqlalchemy import BigInteger, Boolean, DateTime, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship
from pgvector.sqlalchemy import Vector

from .base import Base


class Message(Base):
    """Telegram messages with AI enrichment."""

    __tablename__ = "messages"
    __table_args__ = (
        UniqueConstraint('channel_id', 'message_id', name='uq_messages_channel_message'),
    )

    # Primary key
    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)

    # Telegram identifiers
    message_id: Mapped[int] = mapped_column(BigInteger, nullable=False, index=True)
    channel_id: Mapped[int] = mapped_column(
        Integer, ForeignKey("channels.id", ondelete="CASCADE"), nullable=False, index=True
    )

    # Content
    content: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    content_translated: Mapped[Optional[str]] = mapped_column(Text, nullable=True)

    # Vector embedding (pgvector)
    embedding: Mapped[Optional[list]] = mapped_column(Vector(384), nullable=True)

    # Relationships
    channel = relationship("Channel", back_populates="messages")
    tags = relationship("MessageTag", back_populates="message")
```

### Key Models (29 total)

| Model | Table | Purpose |
|-------|-------|---------|
| `Message` | `messages` | Core message storage with enrichment |
| `Channel` | `channels` | Telegram channels with folder/rule |
| `CuratedEntity` | `curated_entities` | Custom entity knowledge graph |
| `OpenSanctionsEntity` | `opensanctions_entities` | Sanctions data |
| `LLMPrompt` | `llm_prompts` | Versioned LLM prompts |
| `Event` | `events` | Detected OSINT events |
| `RSSFeed` | `rss_feeds` | RSS feed configuration |
| `ExternalNews` | `external_news` | Ingested news articles |
| `MessageValidation` | `message_validations` | Cross-validation results |
| `DecisionLog` | `decision_logs` | Archive decision audit trail |

---

## Configuration (`config/`)

### Settings Class

Centralized configuration using Pydantic Settings:

```python
# shared/python/config/settings.py

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Application settings loaded from environment variables."""

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=True,
        extra="ignore",
    )

    # Environment
    ENVIRONMENT: str = Field(default="development")
    DEBUG: bool = Field(default=False)
    LOG_LEVEL: str = Field(default="INFO")

    # Database
    POSTGRES_HOST: str = Field(default="postgres")
    POSTGRES_PORT: int = Field(default=5432)
    POSTGRES_DB: str = Field(default="osint_platform")
    POSTGRES_USER: str = Field(default="osint_user")
    POSTGRES_PASSWORD: str = Field(...)  # Required
    POSTGRES_POOL_SIZE: int = Field(default=20)

    # Redis
    REDIS_HOST: str = Field(default="redis")
    REDIS_PORT: int = Field(default=6379)
    REDIS_PASSWORD: str = Field(...)  # Required

    # MinIO
    MINIO_ENDPOINT: str = Field(...)  # Required
    MINIO_ACCESS_KEY: str = Field(...)
    MINIO_SECRET_KEY: str = Field(...)
    MINIO_BUCKET_NAME: str = Field(default="telegram-archive")

    # Telegram
    TELEGRAM_API_ID: Optional[int] = Field(None)
    TELEGRAM_API_HASH: Optional[str] = Field(None)
    TELEGRAM_SESSION_PATH: Path = Field(default=Path("/data/sessions"))

    # Computed properties
    @property
    def DATABASE_URL(self) -> str:
        return f"postgresql://{self.POSTGRES_USER}:{self.POSTGRES_PASSWORD}@{self.POSTGRES_HOST}:{self.POSTGRES_PORT}/{self.POSTGRES_DB}"

    @property
    def REDIS_URL(self) -> str:
        return f"redis://:{self.REDIS_PASSWORD}@{self.REDIS_HOST}:{self.REDIS_PORT}/{self.REDIS_DB}"


# Global instance
settings = Settings()
```

### Usage

```python
from config.settings import settings

# Access configuration
db_url = settings.DATABASE_URL
pool_size = settings.POSTGRES_POOL_SIZE
```

---

## Observability (`observability/`)

### Structured Logging

JSON-formatted logging for Loki aggregation:

```python
from observability import setup_logging, get_logger

# Initialize at service startup
setup_logging(service_name="processor")
logger = get_logger(__name__)

# Use structured fields
logger.info("Message processed", extra={
    "message_id": 12345,
    "channel_id": 67890,
    "topic": "combat",
    "latency_ms": 150,
})
```

### Trace Context

Cross-service request correlation:

```python
from observability import set_trace_id, get_trace_id, LogContext

# Set trace ID (from Redis message)
set_trace_id(message["trace_id"])

# Get current trace ID
trace_id = get_trace_id()

# Context manager for structured logging
with LogContext(message_id=123, channel_id=456):
    logger.info("Processing message")  # Includes message_id, channel_id
```

### Prometheus Metrics

Pre-defined metrics helpers:

```python
from observability import (
    record_message_processed,
    record_spam_detection,
    record_llm_request,
    record_media_archived,
    record_api_request,
)

# Record message processing
record_message_processed(
    channel_id=123,
    topic="combat",
    importance="high",
    latency_seconds=0.15,
)

# Record spam detection
record_spam_detection(spam_type="financial", confidence=0.95)

# Record LLM request
record_llm_request(model="qwen2.5:3b", success=True, latency_seconds=1.2)
```

---

## AI Utilities (`ai/`)

### Ollama Client

Async wrapper for Ollama API:

```python
from ai import create_ollama_client

async with create_ollama_client(
    host="http://ollama:11434",
    timeout=180.0,
) as client:
    response = await client.generate(
        model="qwen2.5:3b",
        prompt="Analyze this message...",
        temperature=0.3,
        max_tokens=500,
    )
    result = response["response"]
```

---

## Media Archiver

Content-addressed storage with SHA-256 deduplication:

```python
# shared/python/media_archiver.py

from media_archiver import MediaArchiver

archiver = MediaArchiver(minio_client)

# Archive media (returns existing key if duplicate)
s3_key = await archiver.archive_media(
    data=media_bytes,
    content_type="image/jpeg",
    original_filename="photo.jpg",
)
# Returns: "media/2f/a1/2fa1b3c4d5e6f7...abc.jpg"

# Check if media exists
exists = await archiver.media_exists(sha256_hash)

# Get media URL
url = archiver.get_media_url(s3_key)
```

**Path Format**: `media/{hash[:2]}/{hash[2:4]}/{hash}.{ext}`

---

## Usage in Services

### Import Pattern

```python
# In any service
from models import Message, Channel, AsyncSessionLocal, get_db
from config.settings import settings
from observability import setup_logging, get_logger, record_message_processed
from ai import create_ollama_client
```

### FastAPI Dependency

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from models import get_db, Message

@app.get("/messages/{id}")
async def get_message(id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Message).where(Message.id == id))
    return result.scalar_one_or_none()
```

### Enrichment Task Pattern

```python
from models import AsyncSessionLocal, Message

async def process_batch(messages, session):
    for msg in messages:
        # Process message
        await session.execute(
            text("UPDATE messages SET field = :value WHERE id = :id"),
            {"id": msg.id, "value": result}
        )
    # Base class handles commit
```

---

## Adding Shared Code

### When to Share

Share code when:
- Used by 2+ services
- Core business logic
- Database models
- Configuration patterns

Keep in service when:
- Service-specific logic
- One-off utilities
- Experimental features

### Process

1. **Identify pattern** used across services
2. **Design interface** with clear API
3. **Implement** in `shared/python/`
4. **Add exports** to `__init__.py`
5. **Write tests** in `tests/`
6. **Update services** to use shared code
7. **Document** usage patterns

---

## Related Documentation

- [Database Schema](database-schema.md) - Table definitions
- [Database Migrations](database-migrations.md) - Schema changes
- [Adding Features](adding-features.md) - Using shared code in features
