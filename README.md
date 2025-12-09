# OSINT Intelligence Platform Documentation

Comprehensive documentation for the **OSINT Intelligence Platform** - a production-ready system for archiving, enriching, and analyzing Telegram channels with multi-model AI enrichment, semantic search, and configurable intelligence rules.

## Platform Overview

| Stat | Value |
|------|-------|
| **Channels Monitored** | 254+ Telegram channels |
| **Curated Entities** | 1,425 (military equipment, individuals, organizations) |
| **Services** | 29 Docker containers |
| **Monthly Cost** | ~€230 (fully self-hosted) |

### Key Features

- **Telegram Archiving** - Real-time monitoring with folder-based channel management
- **AI Classification** - Self-hosted LLM (Ollama) for importance scoring and topic classification
- **Semantic Search** - pgvector embeddings for "find similar content" queries
- **Entity Matching** - Automatic linking to curated military/political entities
- **RSS Distribution** - "Subscribe to any search" with authenticated feeds
- **Social Graph** - Forward chain tracking and influence analysis

## Documentation Sections

| Section | Audience | Description |
|---------|----------|-------------|
| [**Getting Started**](docs/getting-started/) | Everyone | Platform overview, quick start, core concepts |
| [**User Guide**](docs/user-guide/) | OSINT Analysts | Searching, RSS feeds, entities, notifications |
| [**Operator Guide**](docs/operator-guide/) | Admins | Installation, configuration, monitoring, backups |
| [**Developer Guide**](docs/developer-guide/) | Developers | Architecture, all 12 services, contributing |
| [**Security Guide**](docs/security-guide/) | Security | Authentication (Ory Kratos), CrowdSec, hardening |
| [**Tutorials**](docs/tutorials/) | Everyone | Step-by-step guides (5-30 minutes each) |
| [**Reference**](docs/reference/) | Everyone | API endpoints, env vars, database schema, Docker |

## Quick Links

- **Quick Start**: [docs/getting-started/quick-start.md](docs/getting-started/quick-start.md)
- **Architecture**: [docs/developer-guide/architecture.md](docs/developer-guide/architecture.md)
- **API Reference**: [docs/reference/api-endpoints.md](docs/reference/api-endpoints.md)
- **Troubleshooting**: [docs/operator-guide/troubleshooting.md](docs/operator-guide/troubleshooting.md)

## Main Repository

The platform source code is at: [osintukraine/osint-intelligence-platform](https://github.com/osintukraine/osint-intelligence-platform)

## Serving Docs Locally

```bash
pip install -r requirements.txt
mkdocs serve
# Visit http://127.0.0.1:8000
```

## Documentation Stats

- **52 documentation files**
- **33,000+ lines** of content
- Generated from actual codebase analysis (December 2025)

## License

Same as the main OSINT Intelligence Platform repository.
