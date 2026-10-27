# RSS-Ingestor Service

> External RSS feed ingestion service - correlates news articles with Telegram content

## Overview

The RSS-Ingestor Service monitors external RSS feeds (news outlets, think tanks, OSINT sources) and enriches them with semantic embeddings for correlation with Telegram messages. This enables cross-source intelligence analysis and fact-checking.

**Pipeline Position**: `RSS Feeds → FeedPoller → ArticleProcessor → PostgreSQL → Enrichment (Correlation)`

**Key Technologies**:

- **feedparser**: RSS/Atom feed parsing
- **sentence-transformers**: 384-dim embeddings (all-MiniLM-L6-v2, CPU-only)
- **httpx** + **asyncio-throttle**: Async HTTP client with rate limiting
- **pgvector**: Semantic similarity search
- **asyncio**: Async I/O (feeds are polled sequentially)

**Core Responsibilities**:

1. Poll RSS feeds at configurable intervals (default: 5 minutes)
2. Parse and validate RSS/Atom feed entries
3. Deduplicate articles by URL hash (SHA-256)
4. Generate semantic embeddings for article content
5. Store enriched articles in PostgreSQL
6. Track feed health and statistics

!!! info "Correlation Handled by Enrichment Service"

    The RSS-Ingestor focuses on **article ingestion only**. Cross-correlation with Telegram messages is handled by the enrichment service (`RSSCorrelationTask`) for proper separation of concerns and bidirectional matching.

## Architecture

### Component Diagram

```mermaid
graph TB
    subgraph "External Sources"
        FEEDS[RSS Feeds<br/>Defense.gov, ISW, Pravda, etc.]
    end

    subgraph "RSS-Ingestor Service"
        MAIN[main.py<br/>Orchestrator]
        POLLER[FeedPoller<br/>HTTP Fetch + Parse]
        PROCESSOR[ArticleProcessor<br/>Enrichment + Storage]

        MAIN --> POLLER
        MAIN --> PROCESSOR
    end

    subgraph "Storage"
        PG[(PostgreSQL)]
        REDIS[(Redis Cache)]
    end

    subgraph "Downstream"
        ENR[Enrichment Service<br/>RSSCorrelationTask]
    end

    FEEDS -->|XML/Atom| POLLER
    POLLER -->|Article Dicts| PROCESSOR
    PROCESSOR -->|Store| PG
    PG -.->|Read for Correlation| ENR

    style POLLER fill:#e1f5ff
    style PROCESSOR fill:#fff3e0
    style PG fill:#c8e6c9
```

### Key Files

| File | Purpose | Lines |
|------|---------|-------|
| `src/main.py` | Service orchestrator, polling cycle, configuration | 393 |
| `src/feed_poller.py` | RSS fetch with rate limiting and parsing | 119 |
| `src/article_processor.py` | Deduplication, embedding generation, storage | 105 |
| `src/correlation_engine.py` | Legacy correlation logic; not imported by `main.py` (moved to enrichment `RSSCorrelationTask`) | 341 |
| `requirements.txt` | Dependencies (feedparser, sentence-transformers) | |
| `Dockerfile` | Uses `osint-platform-pytorch-cpu` base image | |

### Data Flow

```mermaid
sequenceDiagram
    participant Main as RSSIngestorService
    participant Poller as FeedPoller
    participant Processor as ArticleProcessor
    participant DB as PostgreSQL
    participant Enrichment as RSSCorrelationTask

    Main->>DB: Load configuration (platform_config)
    Main->>DB: Fetch active feeds (rss_feeds table)

    loop For each feed
        Main->>Poller: fetch(feed_url)
        Poller->>Poller: HTTP GET with rate limiting
        Poller->>Poller: Parse RSS/Atom with feedparser
        Poller-->>Main: List[article_data]

        loop For each article
            Main->>Processor: process(db, feed, article_data)
            Processor->>DB: Check url_hash for duplicate
            alt Article is new
                Processor->>Processor: Generate embedding (all-MiniLM-L6-v2)
                Processor->>DB: INSERT INTO external_news
                Processor-->>Main: True (new article)
            else Article exists
                Processor-->>Main: False (duplicate)
            end
        end

        Main->>DB: Update feed statistics
    end

    Note over Main: Sleep for interval_minutes

    Note over Enrichment: Separately, RSSCorrelationTask runs
    Enrichment->>DB: Query new articles with embeddings
    Enrichment->>DB: Find similar messages via pgvector
    Enrichment->>DB: Create message_news_correlations
```

## Feed Management

### Feed Configuration

RSS feeds are managed via the `rss_feeds` table in PostgreSQL, editable through NocoDB admin panel.

**Database Schema** (`infrastructure/postgres/init.sql`):

```sql
CREATE TABLE IF NOT EXISTS rss_feeds (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL UNIQUE,
    website_url TEXT,
    category VARCHAR(50) NOT NULL,           -- ukraine, russia, neutral, analysis
    trust_level INTEGER NOT NULL CHECK (trust_level >= 1 AND trust_level <= 5),
    language VARCHAR(10) DEFAULT 'en',
    country VARCHAR(10),
    description TEXT,
    active BOOLEAN DEFAULT TRUE,
    last_polled_at TIMESTAMP WITH TIME ZONE,
    last_successful_poll_at TIMESTAMP WITH TIME ZONE,
    poll_failures_count INTEGER DEFAULT 0,
    articles_fetched_total INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**Key Fields**:

| Field | Purpose | Values |
|-------|---------|--------|
| `active` | Enable/disable polling | `true`/`false` |
| `category` | Source perspective | `ukraine`, `russia`, `neutral`, `analysis`, `osint_community` |
| `trust_level` | Source reliability | 1 (low) to 5 (high) |
| `poll_failures_count` | Consecutive failures | Auto-incremented on errors |

### Feed Categories

| Category | Description | Example Sources |
|----------|-------------|-----------------|
| `ukraine` | Ukraine-aligned media | Ukrainska Pravda, Kyiv Independent |
| `russia` | Russia-aligned media | TASS, RIA Novosti (archived) |
| `neutral` | International neutral | BBC, Reuters, AP |
| `analysis` | Think tanks and research | ISW, CSIS, RUSI |
| `osint_community` | OSINT researchers | Bellingcat, Oryx |

### Adding New Feeds

**Via NocoDB Admin Panel** (Recommended):

1. Navigate to `http://localhost:8080` (NocoDB)
2. Open `rss_feeds` table
3. Add new row with required fields:
   - `name`: Display name
   - `url`: RSS/Atom feed URL
   - `category`: Perspective category
   - `trust_level`: 1-5 rating
   - `active`: `true`

**Via SQL**:

```sql
INSERT INTO rss_feeds (name, url, website_url, category, trust_level, language, country, description, active)
VALUES (
    'US DoD Press Releases',
    'https://www.defense.gov/News/Releases/rss.xml',
    'https://www.defense.gov',
    'neutral',
    5,
    'en',
    'US',
    'Official US Department of Defense news releases',
    true
);
```

**Via API** (Future):

```bash
curl -X POST http://localhost:8000/api/feeds \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Institute for the Study of War",
    "url": "https://www.understandingwar.org/rss.xml",
    "category": "analysis",
    "trust_level": 5,
    "language": "en",
    "active": true
  }'
```

## Processing Pipeline

### 1. Feed Polling (`FeedPoller`)

**File**: `services/rss-ingestor/src/feed_poller.py`

**Algorithm**:

1. **Rate Limiting**: 10 requests per 60 seconds (asyncio-throttle)
2. **HTTP Fetch**: GET feed URL with 30-second timeout
3. **Parse**: feedparser library handles RSS 2.0 and Atom 1.0
4. **Extract**: Up to 50 articles per feed (configurable)
5. **Normalize**: Convert to standard article dictionary

**Article Dictionary Structure**:

```python
{
    "title": "Example Article Title",
    "url": "https://example.com/article",
    "url_hash": "a3f5b2c...",  # SHA-256 of URL
    "summary": "Article summary or description",
    "content": "Full article HTML content",
    "author": "John Doe",
    "published_at": datetime(2025, 1, 11, 10, 30, 0),
    "raw_entry": {...}  # Original feedparser entry for debugging
}
```

**Date Parsing Fallback**:

1. Try `published_parsed` field (RSS 2.0)
2. Fall back to `updated_parsed` field (Atom)
3. Default to `datetime.utcnow()` if missing

**URL Extraction**:

1. Try `link` field (standard)
2. Fall back to `id` field (Atom GUID)
3. Skip entry if neither exists

### 2. Article Processing (`ArticleProcessor`)

**File**: `services/rss-ingestor/src/article_processor.py`

**Algorithm**:

1. **Deduplication Check**:
   - Query `external_news` table by `url_hash`
   - Skip if article already exists
   - Return `False` for duplicate

2. **Create Article Object**:
   - Map fields from article_data
   - Set `feed_id`, `source_category`, `source_trust_level` from feed

3. **Generate Semantic Embedding**:
   - Concatenate: `title + summary`
   - Encode with `sentence-transformers` (all-MiniLM-L6-v2, CPU-only)
   - Produces 384-dimensional vector
   - Store in `embedding` field (pgvector)

4. **Store in Database**:
   - INSERT into `external_news` table
   - Commit transaction
   - Return `True` for new article

**Database Schema** (`external_news` table):

```sql
CREATE TABLE IF NOT EXISTS external_news (
    id BIGSERIAL PRIMARY KEY,
    feed_id INTEGER REFERENCES rss_feeds(id) ON DELETE SET NULL,
    source_type VARCHAR(20) DEFAULT 'rss',
    title TEXT NOT NULL,
    content TEXT,
    summary TEXT,
    url TEXT NOT NULL UNIQUE,
    url_hash VARCHAR(64) NOT NULL,
    published_at TIMESTAMP WITH TIME ZONE NOT NULL,
    author VARCHAR(255),
    importance_level VARCHAR,
    tags TEXT[],
    entities JSONB,                          -- Extracted entities (future)
    embedding vector(384),                    -- Semantic embedding
    source_category VARCHAR(50),
    source_trust_level INTEGER,
    source_id INTEGER REFERENCES news_sources(id),
    source_domain VARCHAR(255),
    processed_at TIMESTAMP WITH TIME ZONE,
    correlation_count INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    search_vector tsvector                    -- Full-text search
);

-- Indexes
CREATE INDEX idx_external_news_url_hash ON external_news(url_hash);
CREATE INDEX idx_external_news_embedding ON external_news
  USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_external_news_category ON external_news(source_category, published_at);
```

### 3. Correlation with Telegram (Enrichment Service)

!!! warning "Separation of Concerns"

    The RSS-Ingestor **does not** perform correlations. This is handled by the **Enrichment Service** (`services/enrichment/src/tasks/rss_correlation_task.py`) for:

    - **Proper separation**: Ingestion vs. analysis
    - **Bidirectional matching**: Messages can arrive after articles
    - **Batch processing**: Enrichment runs on schedule, not blocking ingestion

**Correlation Algorithm** (in Enrichment Service):

1. **Semantic Similarity**:
   - Use pgvector `<=>` operator (cosine distance)
   - Query: `1 - (message.content_embedding <=> article.embedding) >= threshold`
   - Default threshold: 0.40 (`correlation_similarity_threshold` in `platform_config`; low on purpose for cross-lingual RU/UK to EN matching)

2. **Time Window Filtering**:
   - Match articles within ±24 hours of the message date (`correlation_time_window_hours`; news often lags Telegram by 12-24 h)

3. **Entity Overlap Scoring**:
   - Currently stored as `0` by `RSSCorrelationTask` (column reserved; not calculated)

4. **Perspective Detection**:
   - Detect opposing viewpoints (Ukraine vs Russia sources)
   - Flag correlations with `perspective_difference = true`

5. **Create Correlation Record**:
   - Store in `message_news_correlations` table
   - Include similarity score, entity overlap, matched entities
   - Set correlation type: `same_event` (>= 0.85), `related_topic` (>= 0.75) or `possibly_related` (below that)
   - Validation columns (`validation_type`, `relevance_explanation`, `confidence`) are filled later by the `rss_validation` enrichment task

**Correlation Schema**:

```sql
CREATE TABLE IF NOT EXISTS message_news_correlations (
    id BIGSERIAL PRIMARY KEY,
    message_id BIGINT NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    news_id BIGINT NOT NULL REFERENCES external_news(id) ON DELETE CASCADE,

    similarity_score NUMERIC NOT NULL,          -- 0-1 (pgvector cosine)
    entity_overlap_score INTEGER,               -- 0-100%
    time_proximity_hours NUMERIC,
    correlation_type VARCHAR(50),               -- same_event, related_topic, possibly_related
    telegram_source_category VARCHAR(50),
    rss_source_category VARCHAR(50),
    perspective_difference BOOLEAN,             -- Opposing viewpoints
    matched_entities JSONB,                     -- {locations: [...], units: [...]}

    validation_type VARCHAR(50),                -- confirms, contradicts, context, alternative
    relevance_explanation TEXT,
    confidence FLOAT,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT uq_message_news_correlation UNIQUE (message_id, news_id)
);
```

## Configuration

### Environment Variables

!!! warning "The `RSS_*` variables in `.env.example` are not read by any service"
    The ingestor reads no `RSS_*` environment variables. Behaviour is controlled by the `platform_config` keys in the next section (editable on the admin page or in NocoDB). `RSS_INGESTION_ENABLED`, `RSS_CORRELATION_*`, `RSS_FACT_CHECK_*`, `RSS_RETENTION_DAYS` etc. in `.env.example` have no effect, and `RSS_CORRELATION_TIME_WINDOW_HOURS=6` there contradicts the real default of 24 h. `RSS_MAX_ARTICLES_PER_POLL` is likewise unused: `FeedPoller()` is built with its default of 50 articles per feed.

What the container does need (from `docker-compose.yml`, no profile, always started):

```bash
# Database (DATABASE_URL is derived from these by shared/python/config/settings.py)
POSTGRES_HOST=postgres  POSTGRES_PORT=5432  POSTGRES_DB  POSTGRES_USER  POSTGRES_PASSWORD
# Required by shared settings validation
REDIS_HOST=redis  MINIO_ENDPOINT  MINIO_ACCESS_KEY  MINIO_SECRET_KEY  JWT_SECRET_KEY
LOG_LEVEL=INFO
# Embedding model comes from the local cache only
HF_HOME=/app/.cache/huggingface
HF_HUB_OFFLINE=1
```

`HF_HUB_OFFLINE=1` means `all-MiniLM-L6-v2` must already be in `./data/huggingface` (shared cache) or pinned under `./data/models` (see `scripts/fetch-models.sh`); the container never downloads it.

### Database Configuration

**Key-Value Configuration** (`platform_config` table):

| Key | Default | Description |
|-----|---------|-------------|
| `ingestion_enabled` | `true` | Master switch for RSS ingestion |
| `ingestion_interval_minutes` | `5` | Polling cycle interval |
| `max_feeds` | `50` | Feeds per cycle |
| `correlation_enabled` | `true` | Enable correlation |
| `correlation_similarity_threshold` | `0.40` | Min similarity (0-1) |
| `correlation_time_window_hours` | `24` | Time window (±hours); read by enrichment |
| `correlation_max_per_message` | `10` | Most similar articles kept per message |
| `fact_check_enabled` | `true` | Queue correlated messages for LLM validation |
| `fact_check_min_importance` | `low` | Minimum message importance to validate |
| `validation_min_similarity` | `0.5` | Similarity floor for validation |
| `retention_days` | `90` | RSS retention (uncorrelated articles only) |

These are admin settings (category `rss`, migration 035) and are listed with defaults in [`runtime_keys.py`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/shared/python/config/runtime_keys.py). The ingestor itself only reads `ingestion_enabled`, `ingestion_interval_minutes` and `max_feeds` (every cycle, so no restart needed); the others are consumed by the enrichment service and API. A NULL value means no override and the current value is kept.

**Loading Configuration** (from `main.py`):

```python
async def load_config(self, db: AsyncSession):
    result = await db.execute(
        text("SELECT key, value FROM platform_config WHERE category = 'rss'")
    )
    config_rows = result.fetchall()
    config_dict = {row[0]: row[1] for row in config_rows if row[1] is not None}

    if 'ingestion_enabled' in config_dict:
        self.enabled = config_dict['ingestion_enabled'].lower() in ('true', '1', 'yes', 't')

    if 'ingestion_interval_minutes' in config_dict:
        self.interval_minutes = int(config_dict['ingestion_interval_minutes'])
```

## Metrics

### Prometheus Metrics

**Endpoint**: none. The service does not expose a metrics endpoint (`prometheus-client` is in `requirements.txt` but unused); the metrics below are proposed only.

**Proposed Metrics**:

```prometheus
# Feeds fetched successfully
rss_feeds_fetched_total{feed="defense_gov", status="success"} 1234

# Articles processed from each feed
rss_articles_processed_total{feed="defense_gov", new="true"} 567
rss_articles_processed_total{feed="defense_gov", new="false"} 890

# Duplicates skipped
rss_duplicates_skipped_total{feed="defense_gov"} 890

# HTTP errors by feed
rss_fetch_errors_total{feed="defense_gov", error="timeout"} 5
rss_fetch_errors_total{feed="defense_gov", error="http_404"} 2

# Embedding generation time
rss_embedding_duration_seconds{quantile="0.95"} 0.8

# Feed polling duration
rss_poll_duration_seconds{feed="defense_gov", quantile="0.95"} 2.5

# Articles stored per cycle
rss_cycle_articles_total 45

# Feed health status
rss_feed_health{feed="defense_gov", status="up"} 1
rss_feed_health{feed="pravda", status="down"} 0
```

### Logging

**Structured Logging** (via `observability` shared module):

```python
from observability import setup_logging, get_logger

setup_logging(service_name="rss-ingestor")
logger = get_logger(__name__)

logger.info(f"Polling feed: {feed.name} ({feed.url})")
logger.debug(f"Fetched {len(articles)} articles from {feed.name}")
logger.info(f"✓ {feed.name}: {new_count} new article(s) ({len(articles)} total in feed)")
logger.error(f"✗ Error polling {feed.name}: {str(e)}")
```

**Log Levels**:

- `INFO`: Cycle start/end, feed polling results, article counts
- `DEBUG`: Individual article processing, embedding generation
- `WARNING`: Configuration load failures, missing embeddings
- `ERROR`: HTTP errors, parsing errors, database errors

**Grafana Dashboard** (Loki integration):

- Feed polling rate (articles/hour)
- Error rate by feed
- Duplicate rate (efficiency metric)
- Embedding generation time (performance)

## Running Locally

### Docker Compose (Recommended)

**Start RSS-Ingestor**:

```bash
# Start all core services (includes rss-ingestor)
docker-compose up -d

# View logs
docker-compose logs -f rss-ingestor

# Restart service
docker-compose restart rss-ingestor

# Stop service
docker-compose stop rss-ingestor
```

**Check Health**:

```bash
# Service status
docker-compose ps rss-ingestor

# Service logs (last 100 lines)
docker-compose logs --tail=100 rss-ingestor

# Follow logs in real-time
docker-compose logs -f rss-ingestor | grep "new article"
```

### Standalone Development

**Prerequisites**:

- Python 3.11+
- PostgreSQL 16+ with pgvector extension
- Redis 7+

**Setup**:

```bash
cd services/rss-ingestor

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export POSTGRES_HOST=localhost POSTGRES_PORT=5432 POSTGRES_DB=osint_platform
export POSTGRES_USER=osint_user POSTGRES_PASSWORD=password
export PYTHONPATH="$PWD/../..:$PWD/../../shared/python"  # imports use both shared.python.* and observability

# Run service
python -m src.main
```

**Expected Output**:

```
INFO - Initializing RSS Ingestion Service...
INFO - Loading sentence-transformers model: all-MiniLM-L6-v2 (CPU-only)
INFO - Sentence transformer model loaded successfully (CPU mode)
INFO - RSS Ingestion Service initialized successfully
INFO - RSS Ingestion Service starting main loop...
INFO - [Cycle 1] Starting RSS polling cycle
INFO - RSS Ingestion: ENABLED | Poll interval: 5 minutes | Max feeds: 50
INFO - Starting polling cycle for 3 active feed(s)
INFO - Polling feed: US DoD Press Releases (https://www.defense.gov/News/Releases/rss.xml)
INFO - Fetched 15 articles from US DoD Press Releases
INFO - Stored new article: Pentagon Announces Military Aid Package... from US DoD Press Releases
INFO - ✓ US DoD Press Releases: 3 new article(s) (15 total in feed)
INFO - [Cycle 1] Complete in 5.2s: 3 new article(s)
INFO - Next cycle in 5 minute(s)
```

### Running Tests

**Unit Tests**:

```bash
cd services/rss-ingestor

# Run all tests
pytest tests/

# Run specific test file
pytest tests/test_feed_poller.py

# Run with coverage
pytest --cov=src tests/

# Run with verbose output
pytest -v tests/
```

**Test Coverage**:

- `test_feed_poller.py`: feed parsing, deduplication, date handling
- `test_correlation_engine.py`: entity overlap, perspective detection (covers the legacy engine, not what runs in production)

## Troubleshooting

### Feed Fetch Failures

**Symptom**: Logs show "Error polling {feed.name}"

**Check Feed URL**:

```bash
# Test HTTP accessibility
curl -I https://www.defense.gov/News/Releases/rss.xml

# Verify XML structure
curl https://www.defense.gov/News/Releases/rss.xml | xmllint --format - | head -50
```

**Common Causes**:

| Error | Cause | Solution |
|-------|-------|----------|
| `403 Forbidden` | Missing/blocked User-Agent (the poller sends httpx's default) | Set a `User-Agent` header on the `httpx.AsyncClient` in `feed_poller.py` |
| `SSL: CERTIFICATE_VERIFY_FAILED` | Invalid SSL cert | Update CA certificates or add exception |
| `Timeout` | Slow feed server | Increase `httpx.AsyncClient(timeout=60.0)` |
| `404 Not Found` | Feed URL changed | Update `url` in `rss_feeds` table |
| `Connection refused` | Service down | Mark feed as `active=false`, retry later |

**Check Service Logs**:

```bash
# Search for specific feed errors
docker-compose logs rss-ingestor | grep "defense.gov"

# Count error types
docker-compose logs rss-ingestor | grep "ERROR" | cut -d':' -f3 | sort | uniq -c
```

### No New Articles

**Symptom**: Feed polling succeeds but `0 new article(s)`

**Possible Causes**:

1. **All articles are duplicates**:
   - Check `external_news.url_hash` for matches
   - Normal behavior for slow-updating feeds

2. **Feed has no new content**:
   - Check feed's `<pubDate>` or `<updated>` fields
   - Some feeds update infrequently (e.g., monthly reports)

3. **Articles filtered out**:
   - Check if `_parse_entry()` returns `None`
   - Missing required fields (title, link, published_at)

**Debug Query**:

```sql
-- Check recent articles from feed
SELECT id, title, published_at, url_hash
FROM external_news
WHERE feed_id = (SELECT id FROM rss_feeds WHERE name = 'US DoD Press Releases')
ORDER BY published_at DESC
LIMIT 10;

-- Check duplicate count
SELECT url_hash, COUNT(*) as count
FROM external_news
GROUP BY url_hash
HAVING COUNT(*) > 1;
```

### Embedding Generation Failures

**Symptom**: Logs show "Failed to generate embedding" or "Embeddings not available"

**Check Model Loading**:

```bash
# Check if sentence-transformers loaded successfully
docker-compose logs rss-ingestor | grep "sentence-transformers model"

# Expected:
# INFO - Sentence transformer model loaded successfully (CPU mode)

# If failed:
# WARNING - Failed to load sentence transformer: ...
```

**Common Causes**:

1. **PyTorch not in base image**:
   - Rebuild base image: `docker build -f infrastructure/docker/base-pytorch-cpu.Dockerfile -t osint-platform-pytorch-cpu:latest .`
   - Rebuild rss-ingestor: `docker-compose build rss-ingestor`

2. **Insufficient memory**:
   - Model requires ~400MB RAM
   - Check Docker memory limits: `docker stats rss-ingestor`

3. **Model not in the local cache**:
   - The container runs with `HF_HUB_OFFLINE=1` and never downloads the model
   - Check `./data/huggingface` (mounted at `/app/.cache/huggingface`) contains `all-MiniLM-L6-v2`, or run `scripts/fetch-models.sh`

**Fallback Behavior**:

If embeddings fail, articles are still stored but without correlation capability:

```python
if self.embedder is not None:
    embedding = self.embedder.encode(text_for_embedding)
    article.embedding = embedding.tolist()
else:
    logger.debug("Embeddings not available - storing article without semantic vector")
    # Article stored without embedding field
```

### Database Connection Errors

**Symptom**: Logs show "database connection failed" or "asyncpg" errors

**Check PostgreSQL**:

```bash
# Verify PostgreSQL is running
docker-compose ps postgres

# Check connectivity from rss-ingestor
docker-compose exec rss-ingestor pg_isready -h postgres -p 5432

# Test database connection
docker-compose exec postgres psql -U osint_user -d osint_platform -c "SELECT COUNT(*) FROM rss_feeds;"
```

**Check Configuration**:

```bash
# DATABASE_URL is built from the POSTGRES_* variables
docker-compose exec rss-ingestor env | grep POSTGRES_
```

**Check pgvector Extension**:

```sql
-- Verify pgvector is installed
SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';

-- Expected: vector | 0.5.0 (or higher)

-- If missing:
CREATE EXTENSION vector;
```

### High Memory Usage

**Symptom**: RSS-Ingestor container uses excessive memory (>2GB)

**Check Memory Usage**:

```bash
# Monitor memory in real-time
docker stats rss-ingestor

# Expected: 400-600MB (sentence-transformers + Python overhead)
# High: >1GB indicates memory leak
```

**Common Causes**:

1. **Too many feeds polled concurrently**:
   - Reduce `max_feeds` (`platform_config`, category `rss`)
   - Sequential polling (current implementation) prevents this

2. **Large article content**:
   - Some feeds include full HTML in `<content>`
   - Not cached, re-fetched each cycle

3. **Embedding batch processing**:
   - sentence-transformers processes one article at a time
   - No batch accumulation

**Memory Optimization**:

```bash
# Limit Docker container memory
# Set mem_limit on the rss-ingestor service in docker-compose, then:
docker-compose up -d rss-ingestor

# Monitor memory after limit
docker stats rss-ingestor
```

### Correlation Count Always Zero

**Symptom**: `external_news.correlation_count` is always 0

**This is Expected** - Correlations are created by the **Enrichment Service**, not RSS-Ingestor.

**Check Enrichment Service**:

```bash
# Verify enrichment service is running
docker-compose ps | grep enrichment

# RSSCorrelationTask runs in the fast-pool worker (enrichment-fast-pool)
docker-compose logs enrichment-fast-pool | grep "RSSCorrelation"

# Check correlation records
docker-compose exec postgres psql -U osint_user -d osint_platform -c \
  "SELECT COUNT(*) FROM message_news_correlations;"
```

**Manual Backfill** (if correlations missing):

```bash
# Run the backfill script from the repo root on the host
python scripts/backfill_rss_correlations.py
```

## Performance Tuning

### High-Volume Feeds

For feeds with 100+ articles per poll (e.g., news aggregators):

**Increase Fetch Limit**:

```python
# In feed_poller.py
poller = FeedPoller(max_articles_per_feed=100)  # Default: 50
```

**Enable Concurrent Processing** (future enhancement):

```python
# Replace sequential loop with asyncio.gather
results = await asyncio.gather(
    *[self.poll_feed(db, feed) for feed in feeds],
    return_exceptions=True
)
```

### Low-Latency Monitoring

For time-sensitive feeds (breaking news, military briefings):

**Reduce Polling Interval**:

```bash
# platform_config (admin page, category rss): ingestion_interval_minutes = 1
```

**Disable Embeddings** (faster ingestion):

```python
# In article_processor.py
self.embedder = None  # Skip embedding generation
# Articles stored immediately, embeddings added by enrichment later
```

### Resource-Constrained Environments

For VPS with limited CPU/memory (e.g., 2GB RAM):

**Reduce Concurrent Feeds**:

```bash
# platform_config (admin page, category rss): max_feeds = 10
```

**Sequential Polling** (already implemented):

```python
# In main.py run_cycle()
for feed in feeds:
    async with self.async_session() as feed_db:
        result = await self.poll_feed(feed_db, feed)
        results.append(result)
# No asyncio.gather - prevents memory spike
```

**Smaller Database Pool**:

```python
# In main.py __init__
self.engine = create_async_engine(
    database_url,
    pool_size=5,        # Reduce from 10
    max_overflow=10,    # Reduce from 20
)
```

## Development

### Code Structure

```
services/rss-ingestor/
├── src/
│   ├── __init__.py
│   ├── main.py                  # Service orchestrator
│   ├── feed_poller.py           # RSS fetching and parsing
│   ├── article_processor.py     # Enrichment and storage
│   └── correlation_engine.py    # Legacy correlation (moved to enrichment)
├── tests/
│   ├── __init__.py
│   ├── test_feed_poller.py
│   └── test_correlation_engine.py
├── requirements.txt             # Dependencies
├── Dockerfile                   # Uses pytorch-cpu base
└── README.md                    # Service overview
```

### Adding New Enrichment

**Example: Extract Entities from Article Content**

```python
# In article_processor.py, process() method

# After creating article object
article = ExternalNews(...)

# Add entity extraction
from shared.python.ai.entity_extractor import EntityExtractor
extractor = EntityExtractor()

if article.content or article.summary:
    text = article.content or article.summary
    entities = extractor.extract(text)
    article.entities = entities  # JSONB field
    logger.debug(f"Extracted entities: {entities}")

# Continue with embedding generation...
```

### Adding New Feed Source Type

**Example: Support Atom 1.0 Feeds**

```python
# In feed_poller.py, _parse_entry() method

# Current: RSS 2.0 only
published_at = entry.get("published_parsed")  # RSS 2.0

# Add: Atom 1.0 support
if not published_at:
    published_at = entry.get("updated_parsed")  # Atom 1.0

# Current: RSS link
url = entry.get("link")

# Add: Atom alternate link
if not url:
    links = entry.get("links", [])
    for link in links:
        if link.get("rel") == "alternate":
            url = link.get("href")
            break
```

### Testing with Mock Feed

**Create Test Feed** (`tests/fixtures/mock_feed.xml`):

```xml
<?xml version="1.0"?>
<rss version="2.0">
  <channel>
    <title>Test Feed</title>
    <link>https://example.com</link>
    <item>
      <title>Test Article</title>
      <link>https://example.com/article1</link>
      <pubDate>Mon, 11 Jan 2025 10:30:00 GMT</pubDate>
      <description>Test article summary</description>
    </item>
  </channel>
</rss>
```

**Test with Mock Server**:

```python
# In tests/test_feed_poller.py
from aioresponses import aioresponses

@pytest.mark.asyncio
async def test_fetch_with_mock_server():
    poller = FeedPoller()

    with aioresponses() as mock:
        # Mock HTTP response
        with open('tests/fixtures/mock_feed.xml') as f:
            mock_xml = f.read()

        mock.get(
            'https://example.com/feed.xml',
            status=200,
            body=mock_xml
        )

        # Fetch and verify
        articles = await poller.fetch('https://example.com/feed.xml')
        assert len(articles) == 1
        assert articles[0]['title'] == 'Test Article'
```

## Related Documentation

- [Architecture Overview](../architecture.md) - System design and data flow
- [Enrichment Service](enrichment.md) - RSSCorrelationTask implementation
- [API Service](api.md) - RSS correlation endpoints
- [Database Tables](../../reference/database-tables.md) - `rss_feeds`, `external_news`, `message_news_correlations`

**External References**:

- [RSS 2.0 Specification](https://www.rssboard.org/rss-specification)
- [Atom 1.0 Specification](https://datatracker.ietf.org/doc/html/rfc4287)
- [feedparser Documentation](https://feedparser.readthedocs.io/)
- [sentence-transformers Documentation](https://www.sbert.net/)
- [pgvector Documentation](https://github.com/pgvector/pgvector)

## Future Enhancements

### Planned Features

1. **Prometheus Metrics** - Track feed health, article rates, errors
2. **Concurrent Feed Polling** - asyncio.gather for faster cycles
3. **API Management** - REST endpoints for feed CRUD operations
4. **Feed Health Monitoring** - Auto-disable failing feeds after N failures
5. **Content Extraction** - Full article scraping for summary-only feeds
6. **Media Download** - Archive images/videos from feed enclosures
7. **Dynamic Polling Intervals** - Per-feed intervals based on update frequency
8. **Feed Discovery** - Auto-discover RSS feeds from website URLs

### Known Limitations

1. **No Media Archival** - Only text content stored (no images/videos from enclosures)
2. **No Full Article Scraping** - Relies on RSS summary/content fields
3. **Sequential Polling** - One feed at a time (slow for 50+ feeds)
4. **No Feed Validation** - Accepts any XML (no schema validation)
5. **No Retry Logic** - Single attempt per feed (failures logged only)
6. **No Rate Limit Handling** - No exponential backoff for HTTP 429
7. **No Prometheus Metrics** - Monitoring via logs only

### Contribution Guidelines

When modifying RSS-Ingestor:

1. **Maintain Separation of Concerns** - Ingestion only, no correlation logic
2. **Preserve CPU-Only Operation** - No GPU dependencies
3. **Use Async I/O** - All database/HTTP operations must be async
4. **Add Tests** - Maintain >80% code coverage
5. **Update Documentation** - Keep this file synchronized with code
6. **Follow PyTorch Rule** - Never add `torch` to `requirements.txt` (use base image)

**Code Review Checklist**:

- [ ] No Telegram client creation (only enrichment may receive client)
- [ ] Uses `osint-platform-pytorch-cpu:latest` base image
- [ ] All database operations use `AsyncSession`
- [ ] Includes unit tests with >80% coverage
- [ ] Logs at appropriate levels (INFO for key events, DEBUG for details)
- [ ] Configuration loaded from `platform_config` table
- [ ] Documentation updated in this file
