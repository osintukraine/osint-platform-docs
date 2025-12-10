# OpenSanctions Service

## Overview

The **OpenSanctions Service** enriches Telegram messages with entity intelligence by matching person and organization names against sanctions lists, PEP (Politically Exposed Persons) databases, and other OpenSanctions datasets. This enables powerful entity-centric intelligence queries and compliance risk assessment.

**Key Capabilities:**

- **Entity Matching**: Fuzzy matching against OpenSanctions API with configurable confidence thresholds
- **Risk Classification**: Automatic detection of sanctioned entities, PEPs, criminals, and corporate entities
- **Semantic Search**: Generates 384-dimensional embeddings for entity similarity search
- **Entity Relationships**: Tracks family, business, and ownership relationships
- **Batch Processing**: Processes messages in configurable batches with API rate limiting
- **Composable Design**: Feature flag enabled - can be disabled without affecting the platform

!!! tip "Independence"
    The OpenSanctions service is completely independent. It can be enabled/disabled via configuration without impacting other platform services.

## What is OpenSanctions?

[OpenSanctions](https://www.opensanctions.org) is a global database of entities subject to sanctions, regulatory enforcement, and investigation. It aggregates data from over 100 sources including:

- **Sanctions**: OFAC, EU, UN, UK sanctions lists
- **PEPs**: Politically Exposed Persons from multiple countries
- **Criminal Entities**: International crime databases
- **Corporate Registries**: Beneficial ownership and corporate structures

The platform provides both a public API and a self-hosted option via [Yente](https://yente.followthemoney.tech/).

## Architecture

### Component Overview

```
Telegram Messages (PostgreSQL)
         ↓
OpenSanctions Service
         ↓
    ┌────┴────────────┬──────────────┬───────────────┐
    ↓                 ↓              ↓               ↓
NER Extraction    API Matching   Embeddings      Storage
(spaCy)          (OpenSanctions) (Transformers)  (PostgreSQL)
```

### Backend Modes

The service supports two backend modes:

#### 1. API Mode (Default)

Uses the public OpenSanctions API at `https://api.opensanctions.org`

**Advantages:**
- Simple setup, no additional infrastructure
- Always up-to-date data
- Free for non-profit/journalism/academia

**Disadvantages:**
- Rate limited (100 requests/minute)
- Requires internet connectivity
- Requires API key

#### 2. Yente Mode (Recommended for Production)

Uses a self-hosted Yente instance with Elasticsearch

**Advantages:**
- No rate limits
- Works offline after initial data sync
- Full control over data updates
- Faster response times (local network)

**Disadvantages:**
- Requires additional infrastructure (Elasticsearch + Yente)
- 8-10GB storage for index
- Periodic data synchronization needed

### Data Flow

```mermaid
graph TD
    A[Messages] --> B[Find Unenriched Messages]
    B --> C[NER Extraction]
    C --> D{Entities Found?}
    D -->|No| A
    D -->|Yes| E[Match vs OpenSanctions API]
    E --> F{Match Found?}
    F -->|No| A
    F -->|Yes| G[Generate Embeddings]
    G --> H[Store Entity]
    H --> I[Link to Message]
    I --> A
```

**Processing Steps:**

1. **Message Selection**: Find non-spam messages without OpenSanctions enrichment
2. **Entity Extraction**: Use spaCy NER to extract person/organization names from Russian/Ukrainian text
3. **API Matching**: Query OpenSanctions API with extracted entities
4. **Risk Classification**: Determine if entity is sanctioned, PEP, or corporate
5. **Embedding Generation**: Create 384-dim semantic embeddings using sentence-transformers
6. **Storage**: Store entity in `opensanctions_entities` and link to message via `opensanctions_message_entities`

## Entity Matching

### Named Entity Recognition (NER)

The service uses spaCy models to extract entity names from message content:

**Models:**
- `ru_core_news_md`: Russian language NER (40MB)
- `xx_ent_wiki_sm`: Multilingual fallback for Ukrainian (10MB)

**Entity Types Extracted:**
- **PER/PERSON**: Individual names (e.g., "Владимир Путин", "Зеленський")
- **ORG**: Organizations (e.g., "Газпром", "Вагнер")
- **LOC/GPE**: Locations (for context, not matched against OpenSanctions)

**Language Detection:**

The NER extractor automatically detects language based on character frequency:
- Ukrainian-specific characters: `іїєґ`
- Cyrillic (Russian or Ukrainian)
- Falls back to regex patterns if spaCy unavailable

### API Matching Process

```python
# Example matching logic
entities = ["Vladimir Putin", "Wagner Group"]
matches = await opensanctions_client.match_entities(
    entities=entities,
    dataset="default",  # or "sanctions", "peps"
    threshold=0.85      # 0.0-1.0 confidence
)
```

**Matching Algorithm:**

1. Query sent to OpenSanctions API with entity names
2. API returns best matches with confidence scores (0.0-1.0)
3. Results above threshold are stored
4. Entity properties extracted (nationality, birth date, positions)
5. Risk classification determined from topics (`sanction`, `role.pep`, etc.)

### Match Confidence Thresholds

| Threshold | Use Case | Expected Results |
|-----------|----------|------------------|
| **0.70** | Broad matching | More results, some false positives |
| **0.85** | Balanced (default) | Good precision/recall balance |
| **0.95** | Strict matching | High precision, may miss variants |

!!! warning "Threshold Tuning"
    Start with 0.85 and adjust based on your needs. Too low = false positives, too high = missed entities.

## Data Sources

### Available Datasets

The OpenSanctions API provides three main datasets:

#### 1. Default Dataset (`default`)

Combined dataset including sanctions, PEPs, and criminal entities.

**Includes:**
- All sanctions lists (OFAC, EU, UN, UK, etc.)
- PEPs from multiple countries
- Criminal watchlists
- Corporate enforcement actions

**Best for:** General entity screening

#### 2. Sanctions Dataset (`sanctions`)

Only sanctioned entities.

**Includes:**
- US OFAC SDN (Specially Designated Nationals)
- EU Sanctions List
- UN Security Council Sanctions
- UK Financial Sanctions
- Country-specific sanctions (Canada, Australia, etc.)

**Best for:** Compliance-focused screening

#### 3. PEPs Dataset (`peps`)

Only Politically Exposed Persons.

**Includes:**
- Current government officials
- Former government officials
- Close associates of PEPs
- Family members

**Best for:** AML/KYC compliance

### Data Update Frequency

- **API Mode**: Real-time (data updated by OpenSanctions team)
- **Yente Mode**: Depends on sync schedule (configure via Yente settings)

## Configuration

### Environment Variables

Set these variables in your `.env` file or Docker environment:

#### Core Settings

```bash
# Enable/disable the service
OPENSANCTIONS_ENABLED=true

# Backend mode: "api" or "yente"
OPENSANCTIONS_BACKEND=yente

# API key (required for api mode, optional for yente)
OPENSANCTIONS_API_KEY=your-api-key-here
```

#### Backend Configuration

```bash
# Base URL (auto-configured based on BACKEND mode)
# - api mode: https://api.opensanctions.org (default)
# - yente mode: http://yente:8000 (default)
OPENSANCTIONS_BASE_URL=

# Dataset to query: default, sanctions, or peps
OPENSANCTIONS_DATASET=default

# Match confidence threshold (0.0-1.0)
OPENSANCTIONS_MATCH_THRESHOLD=0.85
```

#### Processing Configuration

```bash
# Batch size for message processing
OPENSANCTIONS_BATCH_SIZE=10

# Seconds between processing batches
OPENSANCTIONS_POLL_INTERVAL=60

# Generate semantic embeddings for entities
OPENSANCTIONS_GENERATE_EMBEDDINGS=true

# Embedding model (sentence-transformers)
OPENSANCTIONS_EMBEDDING_MODEL=all-MiniLM-L6-v2
```

#### Database

```bash
DATABASE_URL=postgresql+asyncpg://osint_user:password@postgres:5432/osint_platform
```

### Configuration Object

The service uses a centralized configuration object in `shared/python/config/opensanctions_config.py`:

```python
from config.opensanctions_config import opensanctions_config

# Check if enabled
if opensanctions_config.enabled:
    # Check backend mode
    if opensanctions_config.is_yente_mode:
        print(f"Using Yente at {opensanctions_config.base_url}")
    elif opensanctions_config.is_api_mode:
        print(f"Using OpenSanctions API at {opensanctions_config.base_url}")

    # Validate configuration
    is_valid, error = opensanctions_config.validate()
    if not is_valid:
        print(f"Configuration error: {error}")
```

## Running Locally

### Prerequisites

1. **Python 3.11+** with required dependencies
2. **PostgreSQL 16** with pgvector extension
3. **OpenSanctions API key** (for API mode) or **Yente setup** (for Yente mode)

### Option 1: Docker Compose (Recommended)

#### Enable OpenSanctions Service

The OpenSanctions service runs under the `opensanctions` Docker Compose profile:

```bash
# Start all services including OpenSanctions
docker-compose --profile opensanctions up -d

# Or start the entire stack (includes all profiles)
./scripts/stack-manager.sh start
```

This starts three containers:
- `osint-yente-index`: Elasticsearch for entity indexing
- `osint-yente`: Yente API service
- `osint-opensanctions`: OpenSanctions enrichment service

#### View Logs

```bash
# OpenSanctions service logs
docker-compose logs -f opensanctions

# Yente API logs
docker-compose logs -f yente

# All OpenSanctions-related services
docker-compose logs -f opensanctions yente yente-index
```

#### Restart Service

```bash
# Restart only OpenSanctions service
docker-compose restart opensanctions

# Rebuild after code changes
docker-compose up -d --build opensanctions
```

### Option 2: Standalone Development

```bash
cd services/opensanctions

# Install dependencies
pip install -r requirements.txt

# Download spaCy models
python -m spacy download ru_core_news_md
python -m spacy download xx_ent_wiki_sm

# Set environment variables
export OPENSANCTIONS_ENABLED=true
export OPENSANCTIONS_API_KEY=your-key-here
export DATABASE_URL=postgresql+asyncpg://osint_user:password@localhost:5432/osint_platform

# Run service
python src/main.py
```

### Getting an API Key

For API mode, you need an OpenSanctions API key:

1. Visit [https://www.opensanctions.org/api/](https://www.opensanctions.org/api/)
2. Sign up (free for non-profit, journalism, academia)
3. Generate API key
4. Add to `.env`: `OPENSANCTIONS_API_KEY=your-key-here`

!!! info "Free Access"
    OpenSanctions provides free API access for non-profit, journalism, and academic use cases.

### Yente Self-Hosted Setup

For production deployments, use Yente mode:

1. **Start Yente services:**

```bash
docker-compose --profile opensanctions up -d yente-index yente
```

2. **Wait for initial data sync** (5-10 minutes):

```bash
# Check Yente health
curl http://localhost:8000/healthz

# Check indexed entities
curl http://localhost:8000/stats
```

3. **Configure OpenSanctions service:**

```bash
OPENSANCTIONS_ENABLED=true
OPENSANCTIONS_BACKEND=yente
OPENSANCTIONS_BASE_URL=http://yente:8000
```

4. **Start OpenSanctions service:**

```bash
docker-compose up -d opensanctions
```

## Database Schema

### Tables

The service interacts with these PostgreSQL tables:

#### `opensanctions_entities`

Entity registry with embeddings.

**Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL | Primary key |
| `opensanctions_id` | VARCHAR(255) | External OpenSanctions ID (e.g., "ofac-13661") |
| `entity_type` | VARCHAR(50) | Entity type (Person, Company, etc.) |
| `schema_type` | VARCHAR(50) | OpenSanctions schema type |
| `name` | TEXT | Entity name |
| `description` | TEXT | Full bio from OpenSanctions |
| `properties` | JSONB | Entity properties (nationality, birth date, etc.) |
| `aliases` | TEXT[] | Alternative names |
| `datasets` | TEXT[] | Source datasets (e.g., ["us_ofac_sdn"]) |
| `risk_classification` | VARCHAR(50) | Risk level: sanctioned, pep, corporate |
| `entity_embedding` | VECTOR(384) | Semantic embedding for similarity search |
| `embedding_model` | VARCHAR(100) | Model used (all-MiniLM-L6-v2) |
| `first_seen` | TIMESTAMP | First mention |
| `last_seen` | TIMESTAMP | Most recent mention |
| `mention_count` | INTEGER | Total mentions across messages |

**Indexes:**

```sql
-- Unique constraint on OpenSanctions ID
CREATE UNIQUE INDEX idx_opensanctions_entities_external_id
    ON opensanctions_entities(opensanctions_id);

-- Full-text search on name
CREATE INDEX idx_opensanctions_name_trgm
    ON opensanctions_entities USING gin (name gin_trgm_ops);

-- Vector similarity search
CREATE INDEX idx_opensanctions_entities_embedding
    ON opensanctions_entities
    USING ivfflat (entity_embedding vector_cosine_ops);

-- GIN indexes for JSONB
CREATE INDEX idx_opensanctions_entities_datasets_gin
    ON opensanctions_entities USING GIN(datasets);
CREATE INDEX idx_opensanctions_entities_properties_gin
    ON opensanctions_entities USING GIN(properties);
```

#### `opensanctions_message_entities`

Junction table linking messages to OpenSanctions entities.

**Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL | Primary key |
| `message_id` | INTEGER | Foreign key to `messages` |
| `entity_id` | INTEGER | Foreign key to `opensanctions_entities` |
| `match_score` | FLOAT | API match confidence (0.0-1.0) |
| `match_method` | VARCHAR(50) | How matched: async_enrichment, real_time, manual |
| `extraction_method` | VARCHAR(50) | Source: opensanctions_api |
| `context_snippet` | TEXT | Message excerpt mentioning entity |
| `match_features` | JSONB | API response features |
| `matched_at` | TIMESTAMP | When match occurred |

**Constraints:**

```sql
-- Prevent duplicate message-entity pairs
CONSTRAINT uq_opensanctions_message_entity UNIQUE (message_id, entity_id)
```

#### `entity_relationships`

Entity relationship graph (family, business, ownership).

**Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL | Primary key |
| `from_entity_id` | INTEGER | Source entity (opensanctions_entities) |
| `to_entity_id` | INTEGER | Target entity (opensanctions_entities) |
| `relationship_type` | VARCHAR(50) | Type: family, business, ownership, associate |
| `properties` | JSONB | Relationship metadata |
| `source` | VARCHAR(100) | Data source |
| `confidence` | FLOAT | Relationship confidence |
| `discovered_at` | TIMESTAMP | Discovery timestamp |

### Materialized View: `unified_message_entities`

Combines both curated entities and OpenSanctions entities for fast unified queries:

```sql
-- Refresh periodically
REFRESH MATERIALIZED VIEW unified_message_entities;

-- Query example
SELECT * FROM unified_message_entities
WHERE entity_source = 'opensanctions'
  AND risk_classification = 'sanctioned';
```

## API Endpoints

The OpenSanctions service exposes endpoints through the main API service.

### Entity Search

```http
GET /api/entities/search?query=Putin&source=opensanctions
```

**Parameters:**
- `query` (required): Entity name or partial name
- `source` (optional): Filter by source (opensanctions, curated, all)
- `entity_type` (optional): Filter by type (Person, Company, etc.)
- `limit` (optional): Results limit (default: 50)

**Response:**

```json
{
  "entities": [
    {
      "opensanctions_id": "ofac-13661",
      "source": "opensanctions",
      "name": "Vladimir PUTIN",
      "entity_type": "Person",
      "risk_classification": "sanctioned",
      "score": 0.95
    }
  ],
  "count": 1
}
```

### Entity Details

```http
GET /api/entities/opensanctions/ofac-13661
```

**Response:**

```json
{
  "id": 123,
  "opensanctions_id": "ofac-13661",
  "name": "Vladimir Vladimirovich PUTIN",
  "entity_type": "Person",
  "schema_type": "Person",
  "risk_classification": "sanctioned",
  "properties": {
    "name": ["Vladimir PUTIN"],
    "nationality": ["Russia"],
    "birthDate": ["1952-10-07"],
    "topics": ["sanction", "role.pep"],
    "position": ["President of Russia"]
  },
  "datasets": ["us_ofac_sdn", "eu_fsf", "gb_hmt_sanctions"],
  "mention_count": 42,
  "first_seen": "2024-01-15T10:30:00Z",
  "last_seen": "2024-11-28T14:22:00Z"
}
```

### Entity Messages

```http
GET /api/entities/opensanctions/ofac-13661/messages?limit=20&offset=0
```

**Response:**

```json
{
  "messages": [
    {
      "id": 12345,
      "content": "...",
      "match_score": 0.92,
      "context_snippet": "Putin announced...",
      "matched_at": "2024-11-28T14:22:00Z"
    }
  ],
  "total": 42,
  "offset": 0,
  "limit": 20
}
```

## Performance

### Benchmarks

**Processing Speed:**
- Batch processing: 10 messages per batch (configurable)
- NER extraction: ~50ms per message (spaCy)
- API matching: ~200-500ms per batch (depends on entity count)
- Embedding generation: ~100ms per entity (sentence-transformers)

**Throughput:**
- API mode: ~60 messages/minute (rate limit: 100 req/min)
- Yente mode: ~200-300 messages/minute (no rate limit)

### Optimization Strategies

#### 1. Entity Caching

Entities are cached in the database to avoid duplicate API calls:

```python
# Check if entity exists before querying API
existing = await session.execute(
    select(OpenSanctionsEntity).where(
        OpenSanctionsEntity.opensanctions_id == match.opensanctions_id
    )
)
entity = existing.scalar_one_or_none()
```

#### 2. Batch Processing

Messages are processed in batches to reduce API overhead:

```python
# Configure batch size
OPENSANCTIONS_BATCH_SIZE=20  # Larger batches = faster processing
```

#### 3. Disable Embeddings (Optional)

If semantic search isn't needed, disable embedding generation:

```bash
OPENSANCTIONS_GENERATE_EMBEDDINGS=false
```

This reduces processing time by ~50%.

#### 4. Increase Poll Interval

For lower-priority enrichment, increase the poll interval:

```bash
OPENSANCTIONS_POLL_INTERVAL=300  # Check every 5 minutes
```

### Resource Usage

**Memory:**
- Base: ~200MB
- With spaCy models loaded: ~500MB
- With sentence-transformers loaded: ~800MB

**CPU:**
- Light: 1-5% (idle)
- Heavy: 20-40% (active processing)

**Storage:**
- spaCy models: ~50MB
- sentence-transformers model: ~100MB
- Entity embeddings: ~1.5KB per entity

## Monitoring

### Health Checks

#### Service Status

```bash
# Docker container status
docker-compose ps opensanctions

# Expected output: Up (healthy)
```

#### Live Logs

```bash
# OpenSanctions service logs
docker-compose logs -f opensanctions

# Look for:
# - "OpenSanctions service initialized"
# - "Processing batch of X messages"
# - "OpenSanctions matched Y/Z entities"
```

#### Database Checks

```bash
# Count enriched entities
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(*) FROM opensanctions_entities;"

# Recent matches
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT name, risk_classification, mention_count
      FROM opensanctions_entities
      ORDER BY last_seen DESC LIMIT 10;"

# Message-entity links
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(*) FROM opensanctions_message_entities;"
```

### Metrics

The service exposes Prometheus metrics (if observability is configured):

```
# Entity matching metrics
opensanctions_entities_matched_total
opensanctions_api_calls_total
opensanctions_processing_duration_seconds

# Entity counts
opensanctions_entities_total{risk_classification="sanctioned"}
opensanctions_entities_total{risk_classification="pep"}
```

### Alerts

Configure alerts for:

1. **Service Down**: Container unhealthy for >5 minutes
2. **API Errors**: HTTP 4xx/5xx responses from OpenSanctions API
3. **Processing Lag**: No messages processed for >1 hour
4. **Low Match Rate**: <5% entity match rate (may indicate threshold too high)

## Troubleshooting

### Service Not Starting

**Symptom:** Container exits immediately or health check fails

**Check configuration:**

```bash
docker-compose logs opensanctions | grep ERROR
```

**Common issues:**

1. **Missing API key (API mode):**

```
Invalid OpenSanctions configuration: OPENSANCTIONS_API_KEY required
```

**Solution:** Set `OPENSANCTIONS_API_KEY` in `.env`

2. **Invalid dataset:**

```
Invalid OPENSANCTIONS_DATASET 'invalid_dataset'
```

**Solution:** Use one of: `default`, `sanctions`, `peps`

3. **Database connection failure:**

```
Could not connect to database
```

**Solution:** Verify `DATABASE_URL` and ensure PostgreSQL is running

### No Entities Being Matched

**Symptom:** Service running but no entities stored in database

**Diagnostics:**

```bash
# Check if messages have content
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(*) FROM messages WHERE is_spam = false AND content IS NOT NULL;"

# Check for NER extraction logs
docker-compose logs opensanctions | grep "NER extracted"

# Verify API connectivity
docker-compose logs opensanctions | grep "OpenSanctions matched"
```

**Common issues:**

1. **No extractable entities:**

```
NER extracted 0 entities from message 12345
```

**Solution:** Messages may not contain person/organization names. This is normal for many messages.

2. **API key invalid:**

```
OpenSanctions API error: 401 - Unauthorized
```

**Solution:** Verify API key at [https://www.opensanctions.org/api/](https://www.opensanctions.org/api/)

3. **Threshold too strict:**

```
OpenSanctions matched 0/5 entities
```

**Solution:** Lower match threshold:

```bash
OPENSANCTIONS_MATCH_THRESHOLD=0.70  # Try 0.70 instead of 0.85
```

4. **Rate limit exceeded (API mode):**

```
OpenSanctions API rate limit exceeded - waiting before retry
```

**Solution:** Switch to Yente mode or increase `OPENSANCTIONS_POLL_INTERVAL`

### Slow Processing

**Symptom:** Messages enriched slowly, large processing lag

**Performance analysis:**

```bash
# Check batch size
docker-compose logs opensanctions | grep "batch_size"

# Check API response times
docker-compose logs opensanctions | grep "duration"
```

**Optimization options:**

1. **Increase batch size:**

```bash
OPENSANCTIONS_BATCH_SIZE=20  # Process more messages per batch
```

2. **Disable embeddings:**

```bash
OPENSANCTIONS_GENERATE_EMBEDDINGS=false  # 50% faster
```

3. **Switch to Yente mode:**

Yente is 3-5x faster than API mode (no rate limits, local network)

4. **Reduce poll interval:**

```bash
OPENSANCTIONS_POLL_INTERVAL=30  # Check more frequently
```

### Yente Issues

**Symptom:** Yente service not responding or not indexing data

**Check Yente health:**

```bash
# Yente status
curl http://localhost:8000/healthz

# Elasticsearch status
curl http://localhost:9200/_cluster/health

# Indexed entity count
curl http://localhost:8000/stats
```

**Common issues:**

1. **Elasticsearch not started:**

```
Connection refused to elasticsearch
```

**Solution:** Start yente-index first:

```bash
docker-compose up -d yente-index
# Wait 60 seconds for Elasticsearch to initialize
docker-compose up -d yente
```

2. **No data indexed:**

```json
{"entities": 0, "datasets": 0}
```

**Solution:** Trigger initial data sync:

```bash
# Check yente logs for sync status
docker-compose logs yente | grep "sync"

# Data sync can take 5-10 minutes on first start
```

3. **Out of memory:**

```
Elasticsearch heap out of memory
```

**Solution:** Increase Elasticsearch heap size in docker-compose.yml:

```yaml
environment:
  - "ES_JAVA_OPTS=-Xms4g -Xmx4g"  # Increase from 2g to 4g
```

### spaCy Model Errors

**Symptom:** NER extraction fails

```
OSError: [E050] Can't find model 'ru_core_news_md'
```

**Solution:** Rebuild container to download models:

```bash
docker-compose build opensanctions
docker-compose up -d opensanctions
```

Or install manually in container:

```bash
docker-compose exec opensanctions python -m spacy download ru_core_news_md
docker-compose exec opensanctions python -m spacy download xx_ent_wiki_sm
docker-compose restart opensanctions
```

## Integration Examples

### Query Messages by Entity

Find all messages mentioning a specific entity:

```python
from sqlalchemy import select, join
from models import Message, OpenSanctionsMessageEntity, OpenSanctionsEntity

# Find messages mentioning "Vladimir Putin"
query = (
    select(Message)
    .join(OpenSanctionsMessageEntity)
    .join(OpenSanctionsEntity)
    .where(OpenSanctionsEntity.name.ilike("%Putin%"))
    .order_by(Message.created_at.desc())
)

result = await session.execute(query)
messages = result.scalars().all()
```

### Semantic Entity Search

Find entities similar to a query using embeddings:

```python
from sqlalchemy import text

# Generate query embedding
from entity_embedding_generator import EntityEmbeddingGenerator
generator = EntityEmbeddingGenerator()
query_embedding = generator.generate_entity_embedding({
    "name": "Russian military official",
    "description": "High-ranking officer"
})

# Find similar entities
query = text("""
    SELECT
        name,
        risk_classification,
        1 - (entity_embedding <=> :query_embedding) as similarity
    FROM opensanctions_entities
    WHERE entity_embedding IS NOT NULL
    ORDER BY entity_embedding <=> :query_embedding
    LIMIT 10
""")

result = await session.execute(query, {
    "query_embedding": query_embedding.tolist()
})
```

### Risk Classification Filter

Find all sanctioned entities mentioned in last 30 days:

```python
from datetime import datetime, timedelta
from sqlalchemy import select, and_

cutoff = datetime.now() - timedelta(days=30)

query = (
    select(OpenSanctionsEntity)
    .where(and_(
        OpenSanctionsEntity.risk_classification == "sanctioned",
        OpenSanctionsEntity.last_seen >= cutoff
    ))
    .order_by(OpenSanctionsEntity.mention_count.desc())
)

result = await session.execute(query)
sanctioned_entities = result.scalars().all()
```

## Related Documentation

- **[Enrichment Service](enrichment.md)** - Background enrichment coordination
- **[Processor Service](processor.md)** - Real-time message processing
- **[Entity Ingestion Service](entity-ingestion.md)** - Curated entity management
- **[Database Schema](../../reference/database-tables.md)** - Entity tables and relationships
- **[API Reference](../../reference/api-endpoints.md#entities)** - Entity API endpoints

## External Resources

- [OpenSanctions Documentation](https://www.opensanctions.org/docs/)
- [OpenSanctions API Reference](https://www.opensanctions.org/docs/api/matching/)
- [Yente Self-Hosted Setup](https://yente.followthemoney.tech/)
- [spaCy Russian Models](https://spacy.io/models/ru)
- [sentence-transformers](https://www.sbert.net/)

## Future Enhancements

Planned improvements for the OpenSanctions service:

- [ ] **Redis Caching**: Cache frequently matched entities in Redis for faster lookups
- [ ] **Retry Logic**: Exponential backoff for API failures
- [ ] **Relationship Extraction**: Parse entity relationships from OpenSanctions API
- [ ] **Entity Clustering**: Use HDBSCAN to discover entity networks
- [ ] **Real-time Enrichment**: Webhook integration for immediate enrichment
- [ ] **Metrics Exporter**: Prometheus metrics for monitoring
- [ ] **Multi-language NER**: Support for more languages beyond Russian/Ukrainian
- [ ] **Entity Deduplication**: Merge duplicate entities across sources

---

**Questions or issues?** Check the [troubleshooting](#troubleshooting) section or open an issue on GitHub.
