# Developer Guide

Comprehensive guide for developers contributing to the OSINT Intelligence Platform.

---

## Overview

This guide covers the platform's architecture, development patterns, and contribution guidelines.

## What You'll Learn

- [Architecture Overview](architecture.md) - System design and component interactions
- [Services Deep Dive](services/index.md) - Detailed documentation of each service
- [Shared Libraries](shared-libraries.md) - Common code and utilities (58 Python files)
- [Database Schema](database-schema.md) - PostgreSQL schema (45 tables, 4658 lines)
- [Adding Features](adding-features.md) - How to extend the platform
- [LLM Integration](llm-integration.md) - Working with Ollama and prompts
- [Frontend API Patterns](frontend-api-patterns.md) - Next.js API client usage
- [Testing Guide](testing-guide.md) - Test patterns and fixtures
- [Database Migrations](database-migrations.md) - Schema change workflow
- [Contributing](contributing.md) - Contribution guidelines

## Who This Is For

- Backend developers
- Frontend developers
- DevOps engineers
- Open source contributors
- Integration partners

---

## Technology Stack

| Layer | Technology | Version |
|-------|------------|---------|
| **Backend** | Python | 3.11+ |
| **API Framework** | FastAPI | 0.109.0 |
| **Database** | PostgreSQL + pgvector | 16+ |
| **Message Queue** | Redis Streams | 7+ |
| **Frontend** | Next.js | 14.2 |
| **LLM Inference** | Ollama | Latest |
| **Object Storage** | MinIO | Latest |
| **Deployment** | Docker Compose | v2 |

**Key Libraries:**
- SQLAlchemy 2.0 (async ORM)
- Telethon (Telegram MTProto)
- Pydantic v2 (validation)
- TanStack Query (frontend data fetching)
- pgvector (vector similarity search)

---

## Quick Navigation

<div class="grid cards" markdown>

-   :material-architecture:{ .lg .middle } __Architecture__

    ---

    Understand system design and component interactions

    [:octicons-arrow-right-24: Architecture Guide](architecture.md)

-   :material-apps:{ .lg .middle } __Services__

    ---

    Deep dive into each microservice (15+ services)

    [:octicons-arrow-right-24: Services Guide](services/index.md)

-   :material-library:{ .lg .middle } __Shared Libraries__

    ---

    Common code: models, config, observability

    [:octicons-arrow-right-24: Libraries Guide](shared-libraries.md)

-   :material-database:{ .lg .middle } __Database Schema__

    ---

    45 tables with pgvector embeddings

    [:octicons-arrow-right-24: Schema Guide](database-schema.md)

-   :material-plus-circle:{ .lg .middle } __Adding Features__

    ---

    Enrichment tasks, API endpoints, components

    [:octicons-arrow-right-24: Feature Guide](adding-features.md)

-   :material-brain:{ .lg .middle } __LLM Integration__

    ---

    Ollama prompts and classification

    [:octicons-arrow-right-24: LLM Guide](llm-integration.md)

</div>

---

## Development Workflow

### Branch Strategy

```
master (production)
  ↑
develop (integration)
  ↑
feature/your-feature (development)
```

1. **Create feature branch** from `develop`
2. **Implement feature** with tests
3. **Run test suite**: `pytest`
4. **Submit PR** to `develop`
5. **Code review** and CI checks
6. **Merge to develop** after approval
7. **Merge to master** for production release

### Local Development

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f api processor

# Run tests
docker-compose exec api pytest

# Rebuild after changes
docker-compose build --no-cache api
docker-compose up -d api
```

---

## Code Standards

### Python

- **Style**: PEP 8, enforced by `ruff`
- **Type hints**: Required for public functions
- **Docstrings**: Google style
- **Imports**: Sorted by `isort`

```python
async def process_message(
    message_id: int,
    session: AsyncSession,
) -> ProcessingResult:
    """Process a single message through the pipeline.

    Args:
        message_id: Database message ID
        session: Database session

    Returns:
        ProcessingResult with status and metadata
    """
```

### TypeScript

- **Style**: ESLint + Prettier
- **Types**: Strict mode enabled
- **Components**: Functional with hooks

### Git Commits

Use conventional commit format:

```
feat(processor): add entity extraction stage
fix(api): handle missing embeddings gracefully
docs(readme): update installation steps
refactor(enrichment): simplify task registration
```

---

## Development Environment

### Recommended Setup

**IDE**: VS Code with extensions:
- Python (ms-python.python)
- Pylance (type checking)
- ESLint + Prettier (frontend)
- Docker (ms-azuretools.vscode-docker)

### Environment Variables

Copy `.env.example` to `.env` and configure:

```bash
# Required
POSTGRES_PASSWORD=your_secure_password
REDIS_PASSWORD=your_redis_password
TELEGRAM_API_ID=12345678
TELEGRAM_API_HASH=abcdef123456

# Optional (for full functionality)
DEEPL_API_KEY=your_key  # Translation
YENTE_API_KEY=your_key  # OpenSanctions
```

### Hot Reload

- **API**: Uvicorn auto-reload in development
- **Frontend**: Next.js fast refresh
- **Processor**: Restart container after changes

---

## Quick Reference

| Task | Guide |
|------|-------|
| Add enrichment task | [Adding Features](adding-features.md) |
| Add API endpoint | [Adding Features](adding-features.md#adding-an-api-endpoint) |
| Change database schema | [Database Migrations](database-migrations.md) |
| Modify LLM prompts | [LLM Integration](llm-integration.md) |
| Frontend API calls | [Frontend API Patterns](frontend-api-patterns.md) |
| Write tests | [Testing Guide](testing-guide.md) |

---

## Related Documentation

- [Operator Guide](../operator-guide/index.md) - Deployment and operations
- [Reference](../reference/index.md) - API endpoints, env vars, database tables
