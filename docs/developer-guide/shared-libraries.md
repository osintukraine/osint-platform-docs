# Shared Libraries

Documentation of shared Python libraries used across services.

---

## Overview

The `shared/python/` package contains **80+ Python files** of common code used by all services:

- **Models**: ~40 SQLAlchemy model modules (`models/`)
- **Configuration**: Pydantic settings plus admin-editable runtime settings (`config/runtime.py`)
- **Observability**: Logging, metrics, tracing
- **AI Utilities**: Ollama client, OpenAI-compatible gateway client, classification council
- **Storage**: MinIO media archiver, multi-box selection
- **Translation**: DeepL integration with allowance handling
- **Telegram**: client factory and session file names
- **Hardware, hashing, utils**: hardware tiers, perceptual hashes, circuit breaker, graceful shutdown

**Location**: `/shared/python/`

**Installation**: Not a pip package. Each service `Dockerfile` installs `shared/python/requirements.txt`, copies the directory to `/shared/python` and puts it on the path, so modules import by top-level name (`from models import Message`, `from config.settings import settings`):
```dockerfile
COPY shared/python/requirements.txt /shared/requirements.txt
RUN pip install -r /shared/requirements.txt
COPY shared/python /shared/python
ENV PYTHONPATH="/shared/python:${PYTHONPATH}"
```
Some services also add the directory with `sys.path.insert` for local runs.

---

## Package Structure

```
shared/python/
├── __init__.py              # Package metadata (v0.1.0)
├── requirements.txt         # Shared dependencies
├── models/                  # SQLAlchemy models (~40 files)
│   ├── __init__.py         # Exports all models
│   ├── base.py             # AsyncSessionLocal, Base, engine
│   ├── message.py          # Core message model
│   ├── channel.py          # Channel/folder management
│   └── ...                 # one module per table group
├── config/                  # Configuration management
│   ├── settings.py         # Pydantic Settings class (tier 1: environment)
│   ├── runtime.py          # runtime_config(): DB override > env > default (tier 2)
│   ├── runtime_keys.py     # Registry of admin-editable keys
│   └── constants.py
├── observability/           # Logging and metrics
│   ├── __init__.py         # setup_logging, record_* helpers
│   ├── logging.py          # Structured JSON logging
│   └── metrics.py          # Prometheus metrics
├── ai/                      # LLM utilities
│   ├── ollama_client.py    # Async Ollama wrapper, create_ollama_client()
│   ├── openai_compat.py    # OpenAICompatClient / GatewayGenerateClient (LLM gateway)
│   └── council.py          # Classification council (reprocessing votes)
├── services/                # model_selector.py, model_validator.py
├── database/                # Database utilities
├── notifications/           # NotificationClient
├── storage/                 # BoxSelector, MinioClientPool
├── telegram/                # client_factory.py, session_names.py
├── translation/             # translator.py, deepl_limits.py
├── hardware/                # Hardware detection and tiers
├── hashing/                 # message_hasher.py, perceptual.py
├── utils/                   # circuit_breaker.py, graceful_shutdown.py
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
    echo=settings.DEBUG,
    pool_size=settings.POSTGRES_POOL_SIZE,      # Default: 20
    max_overflow=settings.POSTGRES_MAX_OVERFLOW, # Default: 10
    pool_timeout=settings.POSTGRES_POOL_TIMEOUT,
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
    Message, Channel, ChannelSubmission, User,

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

    # Validation and audit
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
    content_embedding: Mapped[Optional[Vector]] = mapped_column(Vector(384), nullable=True)

    # Relationships
    channel = relationship("Channel", back_populates="messages")
    tags = relationship("MessageTag", back_populates="message")
```

### Key Models

| Model | Table | Purpose |
|-------|-------|---------|
| `Message` | `messages` | Core message storage with enrichment |
| `Channel` | `channels` | Telegram channels with folder/rule |
| `CuratedEntity` | `curated_entities` | Curated entity reference data |
| `OpenSanctionsEntity` | `opensanctions_entities` | Sanctions data |
| `LLMPrompt` | `llm_prompts` | Versioned LLM prompts |
| `Event` | `events` | Timeline events from RSS and Telegram |
| `RSSFeed` | `rss_feeds` | RSS feed configuration |
| `ExternalNews` | `external_news` | Ingested news articles |
| `MessageValidation` | `message_validations` | Cross-validation results |
| `DecisionLog` | `decision_log` | Archive decision audit trail |
| `TelegramEventCluster` | `telegram_event_clusters` | Velocity-detected Telegram clusters |
| `StorageBox` | `storage_boxes` | Media storage boxes (`models/storage.py`) |

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
    MINIO_BUCKET_NAME: str = Field(default="osint-media")

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

### Runtime configuration (admin-editable)

Settings an operator changes in the admin UI without a restart go through `config/runtime.py`. A key must be registered in `config/runtime_keys.py`; its value is the `platform_config.value` override, else the environment variable, else the default. Overrides are cached for `ttl` seconds (default 30), and a database outage falls back to the last loaded values.

```python
from config.runtime import runtime_config

rc = runtime_config("processor")
if await rc.get("translation_enabled"):
    ...
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
    worker_id="processor-1",
    channel_id=123,
    duration_seconds=0.15,
)

# Record spam detection
record_spam_detection(channel_id=123, is_spam=True, spam_reason="financial", confidence=0.95)

# Record LLM request
record_llm_request(model="chat", status="success", duration_seconds=1.2, tokens=350)
```

---

## AI Utilities (`ai/`)

### LLM clients

`create_ollama_client()` is the factory for single-prompt `generate()` calls. With `LLM_PROVIDER=openai` and `LLM_GATEWAY_URL` set it returns a `GatewayGenerateClient` (alias from `LLM_GATEWAY_MODEL`); otherwise an async `OllamaClient`:

```python
from ai import create_ollama_client

async with create_ollama_client(timeout=180.0) as client:
    response = await client.generate(
        model="chat",          # Ollama model name, or ignored in favour of the gateway alias
        prompt="Analyze this message...",
        temperature=0.3,
        max_tokens=500,
    )
    result = response["response"]
```

The processor's chat-style calls use `OpenAICompatClient` (`ai/openai_compat.py`), which mirrors the part of `ollama.AsyncClient` the platform uses and raises `LanePaused` when a gateway lane is paused after a 401/403. See [LLM Integration](llm-integration.md).

### Circuit breaker

`utils/circuit_breaker.py` provides `CircuitBreaker` (CLOSED, OPEN, HALF_OPEN) for LLM tasks: consecutive failures or a per-message timeout open it, and it retries after `recovery_time`.

---

## Media Archiver

Content-addressed storage with SHA-256 deduplication:

```python
# shared/python/media_archiver.py (the processor has its own copy: services/processor/src/media_archiver.py)

from media_archiver import MediaArchiver

archiver = MediaArchiver(redis_client=redis, storage_box_id="default")

# Download from Telegram, hash, deduplicate, write to the local buffer,
# queue the sync job; returns the media_files id
media_file_id = await archiver.archive_media(
    telegram_client=client,
    telegram_message=telethon_message,
    message_db_id=message.id,
    session=session,
)
```

Files land in the local buffer first and `media-sync` moves them to the storage box; see [Media Storage](media-storage.md). `BoxSelector` and `MinioClientPool` live in `storage/`.

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
from telegram.session_names import session_name   # '<role>-<account>' session file names
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
5. **Write tests** next to the code (`shared/python/<package>/tests/`)
6. **Update services** to use shared code
7. **Document** usage patterns

---

## Related Documentation

- [Database Schema](database-schema.md) - Table definitions
- [Database Migrations](database-migrations.md) - Schema changes
- [Adding Features](adding-features.md) - Using shared code in features
