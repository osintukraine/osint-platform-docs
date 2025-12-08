# What is the OSINT Intelligence Platform?

The OSINT Intelligence Platform is a self-hosted system for **archiving, enriching, and analyzing Telegram channels** with multi-model AI, semantic search, and real-time intelligence distribution.

## What It Does

The platform solves a critical problem for OSINT analysts: **Telegram content disappears**. Channels can be deleted, media expires after weeks, and critical intelligence is lost forever. This platform ensures permanent archival while adding intelligent enrichment.

###Core Capabilities

**Archive Everything**
: Monitor 254+ Telegram channels simultaneously with automatic spam filtering, deduplication, and content-addressed media storage. Messages and media are permanently archived before they disappear.

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

Study information operations, propaganda patterns, and social networks with 3+ years of archived data and graph analysis tools.

## What Makes It Different

### Revolutionary Channel Management

**No admin panel required.** Manage channels directly in your Telegram app using folders:

- Drag channels to `Archive-UA` folder → Archives everything after spam filter
- Drag to `Monitor-RU` folder → Archives only high-importance messages
- Platform detects changes within 5 minutes automatically

This folder-based approach takes **30 seconds** instead of 10 minutes of SSH/config editing.

### Self-Hosted AI (Zero LLM Costs)

Six local LLM models (Qwen, Llama, Gemma, Phi, Granite) run on your hardware via Ollama. **No API costs, no rate limits, no privacy concerns.** Switch models at runtime via database configuration - no code deployments needed.

### Cost-Effective Architecture

Target cost: **€30-90/month** for VPS hosting (vs €300/month for legacy systems). Aggressive spam filtering (95%+ accuracy) and content-addressed deduplication save 75-80% on storage costs.

### Battle-Tested Since 2022

Built from 3+ years of production experience monitoring the Ukraine conflict. Spam patterns, entity extraction rules, and importance classification are refined from real-world intelligence collection.

### Privacy-First Design

Self-hosted PostgreSQL, MinIO, Ollama - no cloud dependencies. Your data never leaves your infrastructure. Perfect for air-gapped deployments or sensitive research.

## Key Features

### Intelligence Collection

- **254+ Channels**: Monitor hundreds of sources simultaneously
- **Real-Time Ingestion**: Sub-second message latency
- **Historical Backfill**: Fetch messages from any date (configurable)
- **Media Archival**: Content-addressed storage with SHA-256 deduplication
- **Automatic Discovery**: Platform finds new channels via forward chain analysis

### AI/ML Enrichment

- **Multi-Model Classification**: 6 LLM models with runtime switching
- **Importance Levels**: High/medium/low classification for filtering
- **Spam Filter**: >95% accuracy, battle-tested rules
- **Entity Extraction**: Military units, equipment, locations, coordinates
- **Semantic Embeddings**: 384-dim vectors for similarity search
- **AI Tagging**: Keywords, topics, emotions, urgency
- **Translation**: DeepL Pro (free) + Google Translate fallback

### Search & Discovery

- **Full-Text Search**: PostgreSQL GIN index, <100ms queries
- **Semantic Search**: pgvector + HNSW index on 1M+ messages
- **Hybrid Search**: Combine text and semantic ranking
- **15+ Filters**: Date range, channel, media type, importance, tags, sentiment
- **Network Graph**: Visualize channel relationships and forward chains

### Intelligence Distribution

- **Dynamic RSS Feeds**: Subscribe to any search query
- **REST API**: OpenAPI/Swagger documentation
- **Real-Time Notifications**: ntfy server with 14 topic categories
- **RSS Correlation**: Cross-reference Telegram with news feeds
- **Knowledge Graph**: 1,425 curated entities (equipment, people, units)

### Operations & Monitoring

- **NocoDB Admin UI**: Airtable-like interface over PostgreSQL
- **Grafana Dashboards**: 4 pre-configured monitoring views
- **Prometheus Metrics**: All services instrumented
- **Dozzle Logs**: Real-time Docker log viewer
- **Health Checks**: Automated service monitoring

## Use Cases

### Conflict Monitoring

Archive and analyze Telegram channels covering the Ukraine conflict. Track military units, equipment movements, and combat reports with importance classification and entity extraction.

### Evidence Preservation

Permanently archive potential war crimes evidence before channels are deleted. Content-addressed storage ensures integrity, and metadata tracks original sources.

### Information Operations Research

Study propaganda patterns, bot networks, and disinformation campaigns. Semantic search finds coordinated messaging across multiple channels.

### Cross-Source Verification

Correlate Telegram messages with RSS news feeds using semantic similarity. Detect fact-checking opportunities and perspective differences (Ukraine vs Russia sources).

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

### Services

**29 Docker containers** organized into layers:

- **Core Infrastructure**: PostgreSQL 16 + pgvector, Redis 7, MinIO, Ollama
- **Application Layer**: Listener, Processor (2 replicas), Enrichment (6 workers), API, Frontend
- **Monitoring Stack**: Prometheus, Grafana, AlertManager, ntfy, Exporters
- **Authentication** (optional): Ory Kratos, Ory Oathkeeper, Mailslurper

### Processing Pipeline

1. **Listener** monitors Telegram folders, pushes messages to Redis Streams
2. **Processor** filters spam, classifies importance, extracts entities, archives media
3. **Enrichment** generates embeddings, AI tags, translations (background tasks)
4. **API** serves search queries, RSS feeds, and semantic similarity
5. **Frontend** provides web UI for browsing and searching

### Data Flow

**Real-Time Path** (Processor): <1s latency for spam filter, routing, importance classification
**Background Path** (Enrichment): Hours OK for embeddings, AI tagging, social graph extraction

This separation ensures live messages are archived immediately while expensive AI operations run asynchronously.

## System Requirements

### Minimum

- **CPU**: 2 cores (4 recommended)
- **RAM**: 8GB (16GB recommended)
- **Storage**: 50GB SSD (plus media storage needs)
- **OS**: Linux with Docker 20.10+

### Recommended (Production)

- **CPU**: 8 cores (Ryzen 9 7940HS or equivalent)
- **RAM**: 16-32GB (for multiple LLM models)
- **Storage**: 100GB SSD + object storage for media
- **Network**: 100Mbps+ for Telegram API

### Media Storage Estimates

- **60TB** for 3 years of unfiltered archival (254 channels)
- **12-15TB** after spam filtering + deduplication (75-80% savings)
- **Scales linearly** with channel count and retention period

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

## Project History

- **Feb 24, 2022**: Project started (day of Russian invasion)
- **2022**: 1,200+ hours building legacy system
- **2023-2024**: Maintenance mode, 60TB archived
- **Nov 2025**: New platform reaches production-ready status

**Current operational data**: 254 channels, 60TB media, 3 years of archives.

## What's Next?

Now that you understand what the platform does, proceed to the [Quick Start Guide](quick-start.md) to get it running on your system.

Or explore:

- [Core Concepts](concepts.md) to understand key terminology
- [Architecture Overview](../architecture/overview.md) for technical deep dive
- [Features Catalog](../reference/features.md) for complete feature list
