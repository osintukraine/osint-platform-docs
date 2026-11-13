# Contributing

Guidelines for contributing to the OSINT Intelligence Platform.

---

## Overview

We welcome contributions from the community! This guide covers the workflow, standards, and processes for contributing.

**Repository**: [github.com/osintukraine/osint-intelligence-platform](https://github.com/osintukraine/osint-intelligence-platform)

---

## Getting Started

### Fork and Clone

```bash
# Fork repository on GitHub (click "Fork" button)

# Clone your fork
git clone https://github.com/YOUR_USERNAME/osint-intelligence-platform.git
cd osint-intelligence-platform

# Add upstream remote
git remote add upstream https://github.com/osintukraine/osint-intelligence-platform.git
```

### Set Up Development Environment

```bash
# Check out develop branch
git checkout develop

# Copy environment file
cp .env.example .env
# Edit .env with your credentials (POSTGRES_PASSWORD, REDIS_PASSWORD, etc.)

# Start development environment
docker-compose up -d

# Verify services are running
docker-compose ps
```

---

## Contribution Workflow

### 1. Create Feature Branch

Always branch from `develop`:

```bash
# Sync with upstream
git checkout develop
git pull upstream develop

# Create feature branch
git checkout -b feature/my-new-feature
```

**Branch naming conventions:**
- `feature/description` - New features
- `fix/description` - Bug fixes
- `docs/description` - Documentation
- `refactor/description` - Code refactoring

### 2. Make Changes

- Follow [code style guidelines](#code-style-guidelines)
- Write tests for new features
- Update documentation as needed
- Commit frequently with clear messages

### 3. Test Your Changes

```bash
# Run unit tests (from the repo root)
pytest

# Run specific service tests
pytest services/processor/tests/ -v

# Frontend
cd services/frontend-nextjs && npm run lint && npm run type-check && npm run test:run

# Test in Docker
docker-compose up -d
docker-compose logs -f api processor

# Verify API is responding
curl http://localhost:8000/health
```

### 4. Commit Changes

Use conventional commit format:

```bash
git commit -m "feat(processor): add entity extraction stage"
git commit -m "fix(api): handle missing embeddings gracefully"
git commit -m "docs(readme): update installation steps"
git commit -m "refactor(enrichment): simplify task registration"
```

#### Commit Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

| Type | Description |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no code change |
| `refactor` | Code change without feature/fix |
| `test` | Adding tests |
| `chore` | Build process, dependencies |

### 5. Submit Pull Request

```bash
# Push to your fork
git push origin feature/my-new-feature
```

1. Go to GitHub and click "New Pull Request"
2. Base: `develop` ← Compare: `feature/my-new-feature`
3. Fill out PR template with:
   - Description of changes
   - Related issues
   - Testing performed
   - Screenshots (if UI changes)
4. Request review from maintainers
5. Address feedback
6. Wait for approval and merge

---

## Code Style Guidelines

### Python

- **Style**: PEP 8, enforced by `ruff` (run `ruff check .`); `mypy` for types
- **Type hints**: Required for all public functions
- **Docstrings**: Google style
- **Line length**: Maximum 100 characters
- **Imports**: Sorted by ruff's `I` rules (config in `pyproject.toml`)

```python
async def process_message(
    message_id: int,
    session: AsyncSession,
    *,
    skip_spam_check: bool = False,
) -> ProcessingResult:
    """Process a single message through the pipeline.

    Args:
        message_id: Database message ID
        session: Database session
        skip_spam_check: Skip spam filtering (for reprocessing)

    Returns:
        ProcessingResult with status and metadata

    Raises:
        MessageNotFoundError: If message doesn't exist
    """
    ...
```

### TypeScript

- **Style**: ESLint (`npm run lint`); type check with `npm run type-check`
- **Types**: Strict mode, no `any`
- **Components**: Functional with hooks
- **Exports**: Named exports preferred

```typescript
interface MessageProps {
  id: number;
  content: string;
  channelName: string;
}

export function MessageCard({ id, content, channelName }: MessageProps) {
  return (
    <div className="rounded-lg border p-4">
      <h3>{channelName}</h3>
      <p>{content}</p>
    </div>
  );
}
```

### SQL

- **Keywords**: UPPERCASE
- **Identifiers**: lowercase_snake_case
- **Indentation**: 4 spaces
- **Constraints**: Named with convention (`uq_table_column`, `idx_table_column`)

```sql
CREATE TABLE IF NOT EXISTS messages (
    id BIGSERIAL PRIMARY KEY,
    channel_id INTEGER NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    content TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    CONSTRAINT uq_messages_channel_message UNIQUE (channel_id, message_id)
);

CREATE INDEX idx_messages_channel_date
ON messages(channel_id, telegram_date DESC);
```

---

## Testing Requirements

### Minimum Standards

- **Coverage**: `pytest.ini` enforces `--cov-fail-under=80`; pass `--no-cov` when running a single file
- **Unit tests**: All new functions
- **Integration tests**: API endpoints

### Test Structure

```
tests/                          # Root-level shared tests
├── conftest.py                 # Shared fixtures
└── test_*.py

services/*/tests/               # Service-specific tests
├── test_unit_*.py
└── test_integration_*.py
```

### Running Tests

```bash
# All tests
pytest

# With coverage
pytest --cov=services --cov-report=term-missing

# Specific markers
pytest -m unit           # Fast unit tests
pytest -m integration    # Integration tests
pytest -m "not slow"     # Skip slow tests
# Markers: unit, integration, api, slow, requires_db, requires_redis, requires_ollama
```

---

## Documentation Requirements

Update documentation when:

1. **New features**: Add to relevant guide
2. **API changes**: Update API reference
3. **Environment variables**: Update env vars reference
4. **Schema changes**: Update database schema docs

Documentation lives in the `osint-platform-docs` repository; code-adjacent notes live in the platform repo's `docs/` directory.

---

## Review Process

### Automated Checks

Run these locally before opening a PR (the repository has no CI workflow checked in):

- Linting (`ruff`, ESLint)
- Type checking (`mypy`, `tsc --noEmit`)
- Unit tests (`pytest`, `vitest`)
- Build verification (`docker-compose build <service>`)

### Manual Review

Maintainers check:

1. Code quality and style
2. Test coverage
3. Documentation updates
4. Security implications
5. Performance impact

### Merge Criteria

- Lint, type checks and tests pass
- At least 1 approving review
- No unresolved comments
- Branch is up to date with `develop`

---

## Critical Rules

### NEVER Do These

!!! danger "Critical Don'ts"
    - **NEVER** create standalone Telegram clients (use passed client)
    - **NEVER** use Alembic (archived). Schema changes go in `init.sql` **and** a numbered SQL migration in `infrastructure/postgres/migrations/`
    - **NEVER** commit secrets or credentials
    - **NEVER** push directly to `master` or `develop`
    - **NEVER** skip tests for "quick fixes"

### Always Do These

!!! success "Critical Do's"
    - **ALWAYS** branch from `develop`
    - **ALWAYS** test locally before PR
    - **ALWAYS** update docs with code changes
    - **ALWAYS** use conventional commits
    - **ALWAYS** request review before merge

---

## Getting Help

- **Bugs**: [GitHub Issues](https://github.com/osintukraine/osint-intelligence-platform/issues)
- **Questions**: GitHub Issues
- **Security**: Email maintainers directly

---

!!! tip "First Time Contributors"
    Look for issues labeled `good first issue` to get started!

---

## Related Documentation

- [Development Environment](index.md#development-environment) - Setup guide
- [Adding Features](adding-features.md) - Feature development guide
- [Testing Guide](testing-guide.md) - Test patterns and fixtures
