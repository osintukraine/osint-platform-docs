# Shared Libraries

Documentation of shared Python libraries used across services.

## Overview

**TODO: Content to be generated from codebase analysis**

The platform uses shared libraries to maintain consistency and reduce code duplication.

**Code**: `/shared/python/`

## Core Modules

### Models (`shared/python/models/`)

**TODO: Document SQLAlchemy models:**

- `message.py` - Message model
- `channel.py` - Channel model
- `entity.py` - Entity models
- `rss_feed.py` - RSS feed model
- `llm_prompt.py` - LLM prompt versioning
- `enrichment_task.py` - Enrichment task tracking

#### Example Model

```python
# TODO: Add example model code from codebase
```

### Database (`shared/python/database/`)

**TODO: Document database utilities:**

- Session management
- Connection pooling
- Transaction helpers
- Query builders

#### Usage Example

```python
# TODO: Add example database usage
```

### AI (`shared/python/ai/`)

**TODO: Document AI utilities:**

- LLM client wrappers
- Prompt management
- Response parsing
- Embedding generation

#### Usage Example

```python
# TODO: Add example AI utility usage
```

## Common Patterns

### Database Sessions

**TODO: Document session management patterns:**

- Context managers
- Transaction handling
- Rollback strategies
- Connection pooling

### Error Handling

**TODO: Document error handling patterns:**

- Exception hierarchies
- Retry logic
- Logging standards
- Error reporting

### Configuration

**TODO: Document configuration patterns:**

- Environment variables
- Configuration classes
- Validation
- Defaults

## Testing Shared Code

**TODO: Document testing approach for shared libraries:**

- Unit tests
- Integration tests
- Test fixtures
- Mock strategies

## Adding New Shared Code

**TODO: Document process for adding shared code:**

1. Identify common pattern
2. Design interface
3. Implement in shared/
4. Write tests
5. Update services to use shared code
6. Document usage

## Best Practices

**TODO: Document best practices:**

- When to share code vs. duplicate
- Interface design
- Backwards compatibility
- Versioning strategies

---

!!! note "Documentation Status"
    This page is a placeholder. Content will be generated from shared/python/ code analysis.
