# What is the OSINT Intelligence Platform?

The OSINT Intelligence Platform is a self-hosted system for **archiving, enriching, and analyzing Telegram channels** with multi-model AI, semantic search, and real-time intelligence distribution.

## What It Does

The platform solves a critical problem for OSINT analysts: **Telegram content disappears**. Channels can be deleted, media expires after weeks, and critical intelligence is lost forever. This platform ensures permanent archival while adding intelligent enrichment.

### Core Capabilities

**Archive Everything**
: Monitor hundreds of Telegram channels simultaneously with automatic spam filtering, deduplication, and content-addressed media storage. Messages and media are permanently archived before they disappear.

**Enrich with AI**
: Every message is analyzed by AI to classify importance (high/medium/low), extract entities (people, locations, equipment), detect sentiment, and generate semantic embeddings for similarity search.

**Search Intelligently**
: Full-text search, semantic similarity, and 15+ filters let you find exactly what you need across years of archived content. Search by meaning, not just keywords.

**Distribute via RSS**
: Subscribe to any search query combination as an RSS feed. Share intelligence via email, Discord, Mastodon, or any RSS-compatible platform.

## Who It's For

### OSINT Analysts

Monitor conflict zones, track military movements, and preserve evidence of war crimes with permanent archival and AI-powered importance classification.

### Journalists & Researchers

Build searchable archives of Telegram channels for investigative reporting. Cross-reference claims with RSS news feeds and entity knowledge graphs.

### Intelligence Professionals

Deploy self-hosted infrastructure with no cloud dependencies. Semantic search finds related content across languages. Entity extraction identifies key actors automatically.

### Academic Researchers

Study information operations, propaganda patterns, and social networks with comprehensive archival and graph analysis tools.

## What Makes It Different

### Revolutionary Channel Management

**No admin panel required.** Manage channels directly in your Telegram app using folders:

- Drag channels to `Archive-UA` folder → LLM archives most content (lenient mode)
- Drag to `Monitor-RU` folder → LLM archives only medium- and high-importance content (strict mode)
- Platform detects changes within 5 minutes automatically

This folder-based approach takes **30 seconds** instead of 10 minutes of SSH/config editing.

!!! note "12-Character Folder Limit"
    Telegram limits folder names to 12 characters. Use `-UA` and `-RU` suffixes, not full country names.

### Self-Hosted AI

The platform talks to an LLM you run yourself: a local Ollama model on modest hardware, or an OpenAI-compatible gateway in front of your own GPU host. No data goes to third-party AI services. See [LLM Gateway and Providers](../operator-guide/external-llm.md).

### Cost-Effective Architecture

Self-hosted design minimizes ongoing costs. Layered spam filtering (embedding filter plus LLM) and content-addressed deduplication keep storage far below naive archival.

### Battle-Tested Design

Built from years of production experience monitoring conflict zones. Spam patterns, entity extraction rules, and importance classification are refined from real-world intelligence collection.

### Privacy-First Design

Self-hosted PostgreSQL, MinIO and LLM (Ollama or your own GPU gateway) - no cloud AI dependencies. Your data never leaves your infrastructure. Perfect for air-gapped deployments or sensitive research.

## Key Features

### Intelligence Collection

- **Scalable Monitoring**: Monitor hundreds of channels simultaneously
- **Real-Time Ingestion**: Messages are queued as they arrive and processed within seconds
- **Historical Backfill**: Fetch messages from any date (configurable)
- **Media Archival**: Content-addressed storage with SHA-256 deduplication
- **Automatic Discovery**: Platform finds new channels via forward chain analysis
- **Multi-Account Support**: Optional separate Telegram accounts per source region (`multi-account` profile)

### AI/ML Enrichment

- **LLM Classification**: One model call decides spam, relevance, topic, importance and whether to archive; the model comes from your own LLM provider
- **Importance Levels**: High/medium/low classification for filtering
- **Spam Filter**: Embedding pre-filter plus LLM verdict
- **Entity Extraction**: Military units, equipment, locations, coordinates
- **Semantic Embeddings**: 384-dim vectors for similarity search
- **AI Tagging**: Keywords, topics, emotions, urgency
- **Translation**: DeepL (free API tier) with Google Translate as fallback for most errors

### Entity Knowledge Graph

- **OpenSanctions Integration**: Link messages to sanctioned entities via [Yente API](https://www.opensanctions.org/docs/yente/)
- **Wikidata Enrichment**: Automatic property fetching for matched entities
- **Curated Entity Lists**: Import custom CSVs for domain-specific entities (military units, equipment, people)
- **Entity Linking**: Automatic mention detection and linking to knowledge base
- **Relationship Mapping**: Track connections between entities

### Search & Discovery

- **Full-Text Search**: PostgreSQL GIN index, <100ms queries
- **Semantic Search**: pgvector + HNSW index for meaning-based search
- **Unified Search**: One query across messages, events, RSS articles and entities, in keyword or semantic mode
- **15+ Filters**: Date range, channel, media type, importance, tags, sentiment
- **Social Graph**: Per-message graph of author, forwards, replies, comments and reactions
- **Map**: Geolocated messages and event clusters

### Intelligence Distribution

- **Dynamic RSS Feeds**: Subscribe to any search query
- **REST API**: OpenAPI/Swagger documentation
- **Real-Time Notifications**: ntfy server with configurable topic categories
- **RSS Correlation**: Cross-reference Telegram with news feeds

### Operations & Monitoring

- **NocoDB Admin UI**: Airtable-like interface over PostgreSQL
- **Grafana Dashboards**: Pre-configured monitoring views
- **Prometheus Metrics**: All services instrumented
- **Health Checks**: Automated service monitoring

## Use Cases

### Conflict Monitoring

Archive and analyze Telegram channels covering active conflicts. Track military units, equipment movements, and combat reports with importance classification and entity extraction.

### Evidence Preservation

Permanently archive potential war crimes evidence before channels are deleted. Content-addressed storage ensures integrity, and metadata tracks original sources.

### Information Operations Research

Study propaganda patterns, bot networks, and disinformation campaigns. Semantic search finds coordinated messaging across multiple channels.

### Cross-Source Verification

Correlate Telegram messages with RSS news feeds using semantic similarity. Detect fact-checking opportunities and perspective differences across sources.

### Intelligence Briefings

Generate dynamic RSS feeds for specific search queries. Distribute high-importance content to analysts via email, Discord, or custom platforms using N8N workflows.

## Architecture Overview

```
Telegram → Listener → Redis → Processor → PostgreSQL/MinIO
                                  ↓
                             Enrichment
                                  ↓
                                 API → Frontend
```

### Service Layers

The platform is organized into containerized service layers:

- **Core Infrastructure**: PostgreSQL 16 + pgvector, Redis 7, MinIO, Ollama (search embeddings)
- **Application Layer**: Listener, Processor, Enrichment workers, API, Frontend
- **Monitoring Stack** (optional profile): Prometheus, Grafana, Loki, AlertManager, ntfy, exporters
- **Authentication** (optional): Ory Kratos, Ory Oathkeeper
- **Entity Services** (optional): Yente (OpenSanctions matching), entity ingestion, NocoDB admin UI

### Processing Pipeline

1. **Listener** monitors Telegram folders, pushes messages to Redis Streams
2. **Processor** filters spam, classifies importance via LLM, extracts entities, archives media
3. **Enrichment** generates embeddings, AI tags, translations (background tasks)
4. **API** serves search queries, RSS feeds, and semantic similarity
5. **Frontend** provides web UI for browsing and searching

### Data Flow

**Real-Time Path** (Processor): <1s latency for spam filter, routing, LLM classification
**Background Path** (Enrichment): Hours OK for embeddings, AI tagging, social graph extraction

This separation ensures live messages are archived immediately while expensive AI operations run asynchronously.

## System Requirements

### Minimum

- **CPU**: 2 cores (4 recommended)
- **RAM**: 8GB (16GB recommended for AI models)
- **Storage**: 50GB SSD (plus media storage needs)
- **OS**: Linux with Docker 20.10+

### Recommended (Production)

- **CPU**: 8+ cores
- **RAM**: 16-32GB (for multiple LLM models)
- **Storage**: 100GB SSD + object storage for media
- **Network**: 100Mbps+ for Telegram API

### Media Storage Planning

Storage needs depend on:

- **Number of channels** being monitored
- **Channel activity** (messages per day)
- **Media types** (text-only vs image/video heavy)
- **Spam filtering rate** (typically removes 80-90% of content)
- **Deduplication savings** (largest for forwarded content)

Use the spam filter and content-addressed storage to significantly reduce storage requirements.

## Technology Stack

### Backend

- **Python 3.11+**: FastAPI, Telethon, sentence-transformers
- **PostgreSQL 16**: pgvector extension for semantic search
- **Redis 7**: Streams for message queuing
- **MinIO**: S3-compatible object storage
- **LLM**: self-hosted gateway (vLLM behind LiteLLM) or local Ollama; local Ollama also serves search embeddings

### Frontend

- **Next.js 15**: React framework with App Router
- **Tailwind CSS**: Utility-first styling
- **shadcn/ui**: Accessible component library

### Monitoring

- **Prometheus**: Metrics collection and alerting
- **Grafana 11**: Visualization and dashboards
- **ntfy**: Self-hosted notification delivery

### Entity Data Sources

- **[OpenSanctions/Yente](https://www.opensanctions.org/)**: Sanctions and PEP data with fuzzy matching API
- **[Wikidata](https://www.wikidata.org/)**: Structured knowledge base for entity enrichment
- **Custom CSVs**: Import your own entity lists (military units, equipment, people)

## What's Next?

Now that you understand what the platform does, proceed to the [Quick Start Guide](quick-start.md) to get it running on your system.

Or explore:

- [Core Concepts](concepts.md) to understand key terminology
- [Architecture Overview](../developer-guide/architecture.md) for technical deep dive
