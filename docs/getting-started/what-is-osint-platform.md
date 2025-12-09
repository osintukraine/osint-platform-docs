# What is the OSINT Intelligence Platform?

The OSINT Intelligence Platform is a self-hosted system for **archiving, enriching, and analyzing Telegram channels** with multi-model AI, semantic search, and real-time intelligence distribution.

## What It Does

The platform solves a critical problem for OSINT analysts: **Telegram content disappears**. Channels can be deleted, media expires after weeks, and critical intelligence is lost forever. This platform ensures permanent archival while adding intelligent enrichment.

### Core Capabilities

**Archive Everything**
: Monitor hundreds of Telegram channels simultaneously with automatic spam filtering, deduplication, and content-addressed media storage. Messages and media are permanently archived before they disappear.

**Enrich with AI**
: Every message is analyzed by multiple AI models to classify importance (high/medium/low), extract entities (people, locations, equipment), detect sentiment, and generate semantic embeddings for similarity search.

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
- Drag to `Monitor-RU` folder → LLM archives only high-value OSINT (strict mode)
- Platform detects changes within 5 minutes automatically

This folder-based approach takes **30 seconds** instead of 10 minutes of SSH/config editing.

!!! note "12-Character Folder Limit"
    Telegram limits folder names to 12 characters. Use `-UA` and `-RU` suffixes, not full country names.

### Self-Hosted AI (Zero LLM Costs)

Six local LLM models (Qwen, Llama, Gemma, Phi, Granite) run on your hardware via Ollama. **No API costs, no rate limits, no privacy concerns.** Switch models at runtime via database configuration - no code deployments needed.

### Cost-Effective Architecture

Self-hosted design minimizes ongoing costs. Aggressive spam filtering (95%+ accuracy) and content-addressed deduplication save 75-80% on storage costs compared to naive archival.

### Battle-Tested Design

Built from years of production experience monitoring conflict zones. Spam patterns, entity extraction rules, and importance classification are refined from real-world intelligence collection.

### Privacy-First Design

Self-hosted PostgreSQL, MinIO, Ollama - no cloud dependencies. Your data never leaves your infrastructure. Perfect for air-gapped deployments or sensitive research.

## Key Features

### Intelligence Collection

- **Scalable Monitoring**: Monitor hundreds of channels simultaneously
- **Real-Time Ingestion**: Sub-second message latency
- **Historical Backfill**: Fetch messages from any date (configurable)
- **Media Archival**: Content-addressed storage with SHA-256 deduplication
- **Automatic Discovery**: Platform finds new channels via forward chain analysis
- **Multi-Account Support**: Separate accounts for different source regions

### AI/ML Enrichment

- **Multi-Model Classification**: 6 LLM models with runtime switching
- **Importance Levels**: High/medium/low classification for filtering
- **Spam Filter**: >95% accuracy, battle-tested rules
- **Entity Extraction**: Military units, equipment, locations, coordinates
- **Semantic Embeddings**: 384-dim vectors for similarity search
- **AI Tagging**: Keywords, topics, emotions, urgency
- **Translation**: DeepL Pro (free) + Google Translate fallback

### Entity Knowledge Graph

- **OpenSanctions Integration**: Link messages to sanctioned entities via [Yente API](https://www.opensanctions.org/docs/yente/)
- **Wikidata Enrichment**: Automatic property fetching for matched entities
- **Curated Entity Lists**: Import custom CSVs for domain-specific entities (military units, equipment, people)
- **Entity Linking**: Automatic mention detection and linking to knowledge base
- **Relationship Mapping**: Track connections between entities

### Search & Discovery

- **Full-Text Search**: PostgreSQL GIN index, <100ms queries
- **Semantic Search**: pgvector + HNSW index for meaning-based search
- **Hybrid Search**: Combine text and semantic ranking
- **15+ Filters**: Date range, channel, media type, importance, tags, sentiment
- **Network Graph**: Visualize channel relationships and forward chains

### Intelligence Distribution

- **Dynamic RSS Feeds**: Subscribe to any search query
- **REST API**: OpenAPI/Swagger documentation
- **Real-Time Notifications**: ntfy server with configurable topic categories
- **RSS Correlation**: Cross-reference Telegram with news feeds

### Operations & Monitoring

- **NocoDB Admin UI**: Airtable-like interface over PostgreSQL
- **Grafana Dashboards**: Pre-configured monitoring views
- **Prometheus Metrics**: All services instrumented
- **Dozzle Logs**: Real-time Docker log viewer
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

- **Core Infrastructure**: PostgreSQL 16 + pgvector, Redis 7, MinIO, Ollama
- **Application Layer**: Listener, Processor, Enrichment workers, API, Frontend
- **Monitoring Stack**: Prometheus, Grafana, AlertManager, ntfy, Exporters
- **Authentication** (optional): Ory Kratos, Ory Oathkeeper

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
- **Deduplication savings** (typically 50-70% for forwarded content)

Use the spam filter and content-addressed storage to significantly reduce storage requirements.

## Technology Stack

### Backend

- **Python 3.11+**: FastAPI, Telethon, sentence-transformers
- **PostgreSQL 16**: pgvector extension for semantic search
- **Redis 7**: Streams for message queuing
- **MinIO**: S3-compatible object storage
- **Ollama**: Local LLM inference (Qwen, Llama, Gemma, Phi, Granite)

### Frontend

- **Next.js 14**: React framework with App Router
- **Bun**: Fast JavaScript runtime
- **Tailwind CSS**: Utility-first styling
- **shadcn/ui**: Accessible component library

### Monitoring

- **Prometheus**: Metrics collection and alerting
- **Grafana 11**: Visualization and dashboards
- **ntfy**: Self-hosted notification delivery
- **Dozzle**: Real-time log viewer

### Entity Data Sources

- **[OpenSanctions/Yente](https://www.opensanctions.org/)**: Sanctions and PEP data with fuzzy matching API
- **[Wikidata](https://www.wikidata.org/)**: Structured knowledge base for entity enrichment
- **Custom CSVs**: Import your own entity lists (military units, equipment, people)

## What's Next?

Now that you understand what the platform does, proceed to the [Quick Start Guide](quick-start.md) to get it running on your system.

Or explore:

- [Core Concepts](concepts.md) to understand key terminology
- [Architecture Overview](../developer-guide/architecture.md) for technical deep dive
