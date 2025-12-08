# Adding Features

Guide for extending the OSINT Intelligence Platform with new features.

## Overview

**TODO: Content to be generated from codebase analysis**

This guide covers common feature addition patterns and best practices.

## Feature Types

### Adding a New Enrichment Task

**TODO: Document how to create new enrichment tasks:**

1. Create task class inheriting from `BaseEnrichmentTask`
2. Implement `process()` method
3. Register task in enrichment coordinator
4. Add configuration
5. Test task execution

#### Example Task Structure

```python
# TODO: Add example enrichment task code
from shared.enrichment import BaseEnrichmentTask

class MyEnrichmentTask(BaseEnrichmentTask):
    def process(self, message):
        # Task logic here
        pass
```

### Adding a New Intelligence Rule

**TODO: Document how to create custom intelligence rules:**

1. Define rule in configuration
2. Implement rule evaluation logic
3. Test rule matching
4. Deploy configuration

### Adding a New API Endpoint

**TODO: Document how to add API endpoints:**

1. Define route in FastAPI
2. Implement handler
3. Add authentication/authorization
4. Document endpoint
5. Add tests

#### Example Endpoint

```python
# TODO: Add example API endpoint code
from fastapi import APIRouter

router = APIRouter()

@router.get("/my-endpoint")
async def my_endpoint():
    return {"status": "ok"}
```

### Adding a New Frontend Feature

**TODO: Document how to add frontend features:**

1. Create new components
2. Add API integration
3. Update navigation
4. Add tests
5. Update documentation

### Adding a New Entity Source

**TODO: Document how to integrate new entity sources:**

1. Create entity parser
2. Implement enrichment task
3. Map to standard entity schema
4. Test entity extraction
5. Deploy

## Development Workflow

**TODO: Document step-by-step development workflow:**

1. **Branching Strategy**
   - Always work on `develop` branch
   - Create feature branch from `develop`
   - Merge to `master` only for production

2. **Local Development**
   - Use Docker Compose for local testing
   - Hot reload configuration
   - Test with real services

3. **Testing**
   - Write unit tests
   - Write integration tests
   - Test in Docker environment
   - Verify no regressions

4. **Code Review**
   - Submit PR to `develop`
   - Address review comments
   - Ensure tests pass

5. **Deployment**
   - Merge to `develop`
   - Test in staging
   - Merge to `master` for production

## Critical Rules

### Telegram Session Management

**TODO: Emphasize critical rules from CLAUDE.md:**

```python
# ✅ CORRECT - Pass client from main.py
class MyTask:
    def __init__(self, telegram_client: Optional[TelegramClient]):
        self.client = telegram_client

# ❌ WRONG - Never create standalone client
client = TelegramClient(...)  # DON'T DO THIS
```

### Database Schema Changes

**TODO: Explain schema change workflow:**

1. Modify `init.sql`
2. Test with clean rebuild
3. Document changes
4. Update models

### LLM Prompt Changes

**TODO: Explain LLM prompt modification workflow:**

1. Read `docs/architecture/LLM_PROMPTS.md`
2. Understand prompt evolution
3. Create new version
4. Deactivate old version
5. Test with actual model
6. Update documentation

## Testing New Features

**TODO: Document testing requirements:**

- Unit test coverage
- Integration tests
- End-to-end tests
- Performance tests
- Security tests

## Documentation Requirements

**TODO: Document what documentation to update:**

- Code comments and docstrings
- README updates
- API documentation
- User guide updates
- Changelog entries

## Common Pitfalls

**TODO: Document common mistakes from PITFALLS_FROM_PRODUCTION.md:**

- Session management errors
- Transaction handling issues
- Rate limiting violations
- Memory leaks
- Inefficient queries

---

!!! tip "Before You Start"
    Review the relevant service documentation and existing code patterns before implementing new features.

!!! note "Documentation Status"
    This page is a placeholder. Content will be generated from development patterns and CLAUDE.md guidelines.
