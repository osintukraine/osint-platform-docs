# Developer Guide

Comprehensive guide for developers contributing to the OSINT Intelligence Platform.

## Overview

This guide covers the platform's architecture, development patterns, and contribution guidelines.

## What You'll Learn

### Core Documentation
- [Architecture Overview](architecture.md) - System design, data flow, key decisions
- [Services Deep Dive](services/index.md) - Detailed documentation of each service
- [Shared Libraries](shared-libraries.md) - Common code and utilities
- [Database Schema](database-schema.md) - Complete schema reference

### Development Patterns
- [Adding Features](adding-features.md) - How to extend the platform
- [Frontend API Patterns](frontend-api-patterns.md) - Client-side API integration
- [LLM Integration](llm-integration.md) - Working with Ollama and prompts
- [Testing Guide](testing-guide.md) - Writing and running tests
- [Database Migrations](database-migrations.md) - Schema change workflow

### Process
- [Contributing](contributing.md) - Contribution guidelines and workflow

## Who This Is For

- **Backend developers** - Python services, API design
- **Frontend developers** - React/Next.js components
- **DevOps engineers** - Docker, deployment, monitoring
- **Open source contributors** - First-time contributors
- **Integration partners** - API consumers, webhooks

## Technology Stack

| Category | Technology | Purpose |
|----------|-----------|---------|
| **Backend** | Python 3.11+ | Service runtime |
| **API** | FastAPI | REST endpoints, OpenAPI docs |
| **Database** | PostgreSQL 16 + pgvector | Data + semantic search |
| **Queue** | Redis Streams | Message queue, task distribution |
| **Frontend** | Next.js 14 | Server-side rendering |
| **LLM** | Ollama | Self-hosted inference |
| **Storage** | MinIO | S3-compatible media storage |
| **Deploy** | Docker Compose | Container orchestration |

## Quick Navigation

<div class="grid cards" markdown>

-   :material-architecture:{ .lg .middle } __Architecture__

    ---

    System design and component interactions

    [:octicons-arrow-right-24: Architecture Guide](architecture.md)

-   :material-apps:{ .lg .middle } __Services__

    ---

    Deep dive into each microservice

    [:octicons-arrow-right-24: Services Guide](services/index.md)

-   :material-plus-circle:{ .lg .middle } __Adding Features__

    ---

    Enrichment tasks, API endpoints, frontend

    [:octicons-arrow-right-24: Feature Guide](adding-features.md)

-   :material-api:{ .lg .middle } __Frontend API Patterns__

    ---

    Client-side API integration patterns

    [:octicons-arrow-right-24: API Patterns](frontend-api-patterns.md)

-   :material-brain:{ .lg .middle } __LLM Integration__

    ---

    Working with Ollama and prompts

    [:octicons-arrow-right-24: LLM Guide](llm-integration.md)

-   :material-test-tube:{ .lg .middle } __Testing__

    ---

    Writing and running tests

    [:octicons-arrow-right-24: Testing Guide](testing-guide.md)

-   :material-database-edit:{ .lg .middle } __Database Migrations__

    ---

    Schema changes without Alembic

    [:octicons-arrow-right-24: Migrations Guide](database-migrations.md)

-   :material-source-pull:{ .lg .middle } __Contributing__

    ---

    Contribution guidelines and PR workflow

    [:octicons-arrow-right-24: Contributing Guide](contributing.md)

</div>

## Development Workflow

1. **Fork and clone** repository
2. **Create feature branch** from `develop`
   ```bash
   git checkout develop
   git checkout -b feature/my-feature
   ```
3. **Implement feature** following patterns in this guide
4. **Run tests** locally
   ```bash
   docker-compose exec api pytest
   ```
5. **Update documentation** (required for all changes)
6. **Submit PR** to `develop`
7. **Code review** and iterate
8. **Merge to master** for production release

## Code Standards

| Language | Style Guide | Tooling |
|----------|-------------|---------|
| Python | PEP 8 | ruff, mypy |
| TypeScript | ESLint | eslint, prettier |
| SQL | Lowercase keywords | pgformatter |
| Git | [Conventional Commits](https://www.conventionalcommits.org/) | - |

### Python Requirements
- Type hints for all function signatures
- Docstrings for public functions/classes
- `async def` for I/O-bound operations
- `pytest` for tests (80% coverage target)

### TypeScript Requirements
- Strict mode enabled
- Type imports from `@/lib/types`
- `NEXT_PUBLIC_API_URL` for all API calls

## Local Development

### Quick Start

```bash
# Clone and setup
git clone https://github.com/osintukraine/osint-intelligence-platform.git
cd osint-intelligence-platform
cp .env.example .env

# Start services
docker-compose up -d

# Verify health
docker-compose ps
curl http://localhost:8000/health
```

### IDE Setup (VS Code)

Recommended extensions:
- Python (Microsoft)
- Pylance
- Docker
- ESLint
- Prettier
- GitLens

### Hot Reload

```bash
# Backend (API service)
docker-compose up -d --build api
# Changes to src/ auto-reload

# Frontend
cd services/frontend-nextjs
npm run dev
# Changes auto-reload at http://localhost:3000
```

---

## Related Documentation

- [Operator Guide](../operator-guide/index.md) - Deployment and operations
- [API Reference](../reference/api-endpoints.md) - All endpoints
- [Environment Variables](../reference/environment-vars.md) - Configuration
