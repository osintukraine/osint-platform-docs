# OSINT Intelligence Platform Documentation

Comprehensive documentation for the **OSINT Intelligence Platform** - a self-hosted system for archiving, enriching, and analyzing Telegram channels with AI-powered classification, semantic search, and configurable intelligence rules.

## Key Features

- **Telegram Archiving** - Real-time monitoring with folder-based channel management
- **AI Classification** - Self-hosted LLM (Ollama) for importance scoring and topic classification
- **Semantic Search** - pgvector embeddings for "find similar content" queries
- **Entity Matching** - Automatic linking to curated military/political entities
- **RSS Distribution** - "Subscribe to any search" with authenticated feeds
- **Social Graph** - Forward chain tracking and influence analysis
- **Fully Self-Hosted** - No cloud dependencies, complete data sovereignty

## Documentation Sections

| Section | Audience | Description |
|---------|----------|-------------|
| [**Getting Started**](docs/getting-started/) | Everyone | Platform overview, quick start, core concepts |
| [**User Guide**](docs/user-guide/) | Analysts | Searching, RSS feeds, entities, notifications |
| [**Operator Guide**](docs/operator-guide/) | Admins | Installation, configuration, monitoring, backups |
| [**Developer Guide**](docs/developer-guide/) | Developers | Architecture, services, contributing |
| [**Security Guide**](docs/security-guide/) | Security | Authentication (Ory Kratos), CrowdSec, hardening |
| [**Tutorials**](docs/tutorials/) | Everyone | Step-by-step guides (5-30 minutes each) |
| [**Reference**](docs/reference/) | Everyone | API endpoints, env vars, database schema, Docker |

## Quick Links

- [Quick Start](docs/getting-started/quick-start.md) - Get running in 10 minutes
- [Architecture](docs/developer-guide/architecture.md) - System design overview
- [Installation](docs/operator-guide/installation.md) - Full deployment guide
- [API Reference](docs/reference/api-endpoints.md) - REST API documentation

## Source Code

Platform repository: [osintukraine/osint-intelligence-platform](https://github.com/osintukraine/osint-intelligence-platform)

## Serving Docs Locally

```bash
pip install -r requirements.txt
mkdocs serve
```

## License

Same as the main OSINT Intelligence Platform repository.
