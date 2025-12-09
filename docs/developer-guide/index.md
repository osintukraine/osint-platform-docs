# Developer Guide

Comprehensive guide for developers contributing to the OSINT Intelligence Platform.

## Overview

This guide covers the platform's architecture, development patterns, and contribution guidelines.

## What You'll Learn

- [Architecture Overview](architecture.md) - System design and component interactions
- [Services Deep Dive](services/index.md) - Detailed documentation of each service
- [Shared Libraries](shared-libraries.md) - Common code and utilities
- [Database Schema](database-schema.md) - Complete schema reference
- [Adding Features](adding-features.md) - How to extend the platform
- [Contributing](contributing.md) - Contribution guidelines and workflow

## Who This Is For

- Backend developers
- Frontend developers
- DevOps engineers
- Open source contributors
- Integration partners

## Technology Stack

**TODO: Document complete technology stack:**

- Python 3.11+ (backend)
- FastAPI (API framework)
- PostgreSQL 16 + pgvector (database)
- Redis Streams (message queue)
- Next.js 14 (frontend)
- Ollama (LLM inference)
- MinIO (object storage)
- Docker & Docker Compose (deployment)

## Quick Navigation

<div class="grid cards" markdown>

-   :material-architecture:{ .lg .middle } __Architecture__

    ---

    Understand system design and component interactions

    [:octicons-arrow-right-24: Architecture Guide](architecture.md)

-   :material-apps:{ .lg .middle } __Services__

    ---

    Deep dive into each microservice

    [:octicons-arrow-right-24: Services Guide](services/index.md)

-   :material-library:{ .lg .middle } __Shared Libraries__

    ---

    Common code and utilities documentation

    [:octicons-arrow-right-24: Libraries Guide](shared-libraries.md)

-   :material-database:{ .lg .middle } __Database Schema__

    ---

    Complete database schema reference

    [:octicons-arrow-right-24: Schema Guide](database-schema.md)

-   :material-plus-circle:{ .lg .middle } __Adding Features__

    ---

    How to extend the platform

    [:octicons-arrow-right-24: Feature Guide](adding-features.md)

-   :material-source-pull:{ .lg .middle } __Contributing__

    ---

    Contribution guidelines and workflow

    [:octicons-arrow-right-24: Contributing Guide](contributing.md)

</div>

## Development Workflow

**TODO: Document development workflow:**

1. Fork and clone repository
2. Create feature branch on `develop`
3. Implement feature with tests
4. Run test suite
5. Submit pull request
6. Code review
7. Merge to `develop`
8. Deploy to `master` for production

## Code Standards

**TODO: Document coding standards:**

- Python: PEP 8, type hints, docstrings
- TypeScript: ESLint, Prettier
- Git: Conventional commits
- Testing: pytest, minimum coverage requirements

## Development Environment

**TODO: Document local development setup:**

- IDE recommendations (VS Code, PyCharm)
- Required extensions
- Debug configurations
- Hot reload setup

---

**TODO: Content to be generated from codebase analysis and ARCHITECTURE.md**
