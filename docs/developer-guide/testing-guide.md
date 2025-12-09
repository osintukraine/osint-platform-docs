# Testing Guide

Guide for writing and running tests in the OSINT Intelligence Platform.

---

## Test Structure

Tests are organized by service:

```
osint-intelligence-platform/
├── tests/                          # Root-level shared tests
│   ├── conftest.py                 # Shared fixtures (db_session, sample_message, etc.)
│   ├── test_rss_schema.py
│   ├── test_entity_relationship_model.py
│   └── test_message_entity_model.py
├── services/
│   ├── api/tests/                  # API service tests
│   │   ├── test_api_messages.py
│   │   ├── test_api_channels.py
│   │   └── test_rss_feeds.py
│   ├── processor/tests/            # Processor service tests
│   │   ├── test_llm_classifier.py
│   │   ├── test_spam_filter.py
│   │   ├── test_rule_engine.py
│   │   ├── test_message_router.py
│   │   ├── test_entity_extractor.py
│   │   ├── test_entity_matcher.py
│   │   └── test_integration_processor_pipeline.py
│   ├── notifier/tests/             # Notifier service tests
│   │   ├── test_batcher.py
│   │   ├── test_formatter.py
│   │   ├── test_publisher.py
│   │   └── test_router.py
│   ├── rss-ingestor/tests/         # RSS ingestor tests
│   │   ├── test_correlation_engine.py
│   │   └── test_feed_poller.py
│   └── opensanctions/tests/        # OpenSanctions tests
│       ├── test_entity_embedding_generator.py
│       └── test_opensanctions_client.py
```

**Note:** The enrichment service does not currently have a tests directory.

---

## Running Tests

### Run All Tests

```bash
# From project root
pytest

# With verbose output
pytest -v

# With coverage report
pytest --cov=services --cov-report=term-missing
```

### Run Tests for Specific Service

```bash
# API tests
pytest services/api/tests/ -v

# Processor tests
pytest services/processor/tests/ -v

# Single test file
pytest services/processor/tests/test_llm_classifier.py -v
```

### Run Tests by Marker

```bash
# Unit tests only (fast)
pytest -m unit

# Integration tests only
pytest -m integration

# Skip slow tests
pytest -m "not slow"
```

### Run in Docker

```bash
# API tests in container
docker-compose exec api pytest services/api/tests/ -v

# Processor tests in container
docker-compose exec processor pytest services/processor/tests/ -v
```

---

## pytest.ini Configuration

The project uses `pytest.ini` at the root:

```ini
[pytest]
# Test paths
testpaths = tests services/*/tests

# Asyncio mode
asyncio_mode = auto

# Coverage
addopts =
    --verbose
    --strict-markers
    --tb=short
    --cov=services
    --cov=shared
    --cov-report=term-missing
    --cov-report=html:htmlcov
    --cov-fail-under=80

# Markers
markers =
    unit: Unit tests (fast, isolated)
    integration: Integration tests (slower, require services)
    api: API endpoint tests
    slow: Slow-running tests
    requires_db: Tests that require database
    requires_redis: Tests that require Redis
    requires_ollama: Tests that require Ollama
```

---

## Shared Fixtures

Root-level fixtures in `tests/conftest.py`:

### Database Fixtures

```python
# tests/conftest.py

@pytest_asyncio.fixture
async def db_engine():
    """Create in-memory SQLite engine for testing."""
    engine = create_async_engine(
        "sqlite+aiosqlite:///:memory:",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()


@pytest_asyncio.fixture
async def db_session(db_engine) -> AsyncGenerator[AsyncSession, None]:
    """Create database session for testing."""
    async_session = sessionmaker(
        db_engine, class_=AsyncSession, expire_on_commit=False
    )
    async with async_session() as session:
        yield session
```

### Sample Data Fixtures

```python
# tests/conftest.py

@pytest.fixture
def sample_channel():
    """Create sample channel for testing."""
    return Channel(
        telegram_id=1234567890,
        username="test_channel",
        title="Test Channel",
        folder="Archive-Test",
        rule="archive_all",
        active=True,
    )


@pytest.fixture
def sample_message():
    """Create sample message for testing."""
    return Message(
        id=1,
        message_id=1,
        channel_id=1234567890,
        content="Russian forces attacked Bakhmut with artillery.",
        telegram_date=datetime(2024, 1, 15, 10, 30, 0),
        is_spam=False,
        osint_topic="combat",
    )


@pytest.fixture
def spam_message():
    """Create spam message for testing."""
    return Message(
        id=2,
        message_id=2,
        channel_id=1234567890,
        content="🔥 CRYPTO PUMP 🚀 Buy now! 1000x gains guaranteed!",
        is_spam=True,
        spam_confidence=0.95,
    )


@pytest.fixture
def combat_message():
    """Create high-value combat message."""
    return Message(
        id=3,
        message_id=3,
        channel_id=1234567890,
        content="47th Mechanized Brigade destroyed Russian T-90 tank with Javelin ATGM near Avdiivka.",
        osint_topic="combat",
        media_type="video",
    )
```

---

## Writing Tests

### Unit Test Pattern

```python
# services/processor/tests/test_llm_classifier.py

import pytest
from unittest.mock import patch

class TestParseResponse:
    """Tests for _parse_llm_response method."""

    @pytest.fixture
    def classifier(self):
        """Create classifier instance for testing (mocked)."""
        with patch('src.llm_classifier.AsyncClient'):
            classifier = LLMClassifier.__new__(LLMClassifier)
            classifier._prompt_cache = {}
            classifier.model_name = "test-model"
            return classifier

    def test_parse_chain_of_thought_response(self, classifier):
        """Test parsing response with <analysis> tags."""
        response = """<analysis>
ENTITIES: 55th brigade
GEOGRAPHY: Pokrovsk direction
</analysis>
{"is_spam": false, "topic": "combat", "importance": "high", "should_archive": true}"""

        result = classifier._parse_llm_response(response)

        assert result.is_spam == False
        assert result.topic == "combat"
        assert result.importance == "high"
        assert result.should_archive == True
```

### Testing JSON Parsing Edge Cases

```python
def test_parse_malformed_confidence(self, classifier):
    """Test that invalid confidence values are normalized to 0.8."""
    response = '{"is_spam": false, "topic": "combat", "confidence": 1.5}'

    result = classifier._parse_llm_response(response)

    assert result.confidence == 0.8  # Normalized from invalid 1.5


def test_parse_spam_sets_archive_false(self, classifier):
    """Test that spam messages are never archived."""
    response = '{"is_spam": true, "spam_type": "financial", "should_archive": true}'

    result = classifier._parse_llm_response(response)

    assert result.is_spam == True
    assert result.should_archive == False  # Overridden by is_spam
```

### Integration Test Pattern

```python
# services/processor/tests/test_integration_processor_pipeline.py

import pytest

@pytest.mark.integration
@pytest.mark.requires_db
class TestProcessorPipeline:
    """Integration tests for full processor pipeline."""

    @pytest.mark.asyncio
    async def test_message_flows_through_pipeline(self, db_session):
        """Test that a message flows through all processor stages."""
        # Setup
        message = create_test_message()

        # Execute
        result = await process_message(message, db_session)

        # Verify
        assert result.processed == True
        assert result.topic is not None
```

---

## Mocking External Services

### Mocking Ollama

```python
from unittest.mock import AsyncMock, patch

@pytest.fixture
def mock_ollama():
    """Mock Ollama client for testing."""
    with patch('src.llm_classifier.AsyncClient') as mock:
        client = AsyncMock()
        client.chat.return_value = {
            "message": {
                "content": '{"is_spam": false, "topic": "combat"}'
            }
        }
        mock.return_value = client
        yield client


async def test_classification_with_mock(mock_ollama):
    classifier = LLMClassifier()
    result = await classifier.classify("Test content")
    assert result.topic == "combat"
```

### Mocking Database

```python
@pytest.mark.asyncio
async def test_with_db_fixture(db_session):
    """Test using the shared db_session fixture."""
    # Session is already created with in-memory SQLite
    channel = Channel(telegram_id=123, username="test")
    db_session.add(channel)
    await db_session.commit()

    # Query it back
    result = await db_session.get(Channel, channel.id)
    assert result.username == "test"
```

---

## Test Coverage

### Coverage Configuration

From `pytest.ini`:

```ini
--cov=services
--cov=shared
--cov-report=term-missing
--cov-report=html:htmlcov
--cov-fail-under=80
```

### Generate Coverage Report

```bash
# Terminal report
pytest --cov=services --cov-report=term-missing

# HTML report
pytest --cov=services --cov-report=html
# Open htmlcov/index.html in browser
```

### Current Test Coverage

| Service | Test Files | Notes |
|---------|------------|-------|
| **api** | 3 files | Messages, channels, RSS |
| **processor** | 7 files | LLM classifier, spam, routing, entities |
| **notifier** | 4 files | Batching, formatting, publishing |
| **rss-ingestor** | 2 files | Correlation, polling |
| **opensanctions** | 2 files | Client, embeddings |
| **enrichment** | 0 files | No tests yet |

---

## Common Test Patterns

### Testing Async Functions

```python
import pytest

@pytest.mark.asyncio
async def test_async_function():
    result = await some_async_function()
    assert result is not None
```

### Testing with Markers

```python
@pytest.mark.unit
def test_fast_unit_test():
    """Fast unit test."""
    assert True


@pytest.mark.integration
@pytest.mark.requires_db
async def test_slow_integration():
    """Slow integration test requiring database."""
    pass


@pytest.mark.slow
def test_very_slow_operation():
    """Mark for skipping in quick runs."""
    pass
```

### Parameterized Tests

```python
@pytest.mark.parametrize("topic,expected", [
    ("combat", True),
    ("spam", False),
    ("COMBAT", True),  # Case insensitive
    ("invalid", False),
])
def test_is_valid_topic(topic, expected):
    assert is_valid_topic(topic) == expected
```

---

## Debugging Tests

### Run Single Test with Output

```bash
pytest services/processor/tests/test_llm_classifier.py::TestParseResponse::test_parse_chain_of_thought_response -v -s
```

### Drop into Debugger on Failure

```bash
pytest --pdb
```

### Show Full Tracebacks

```bash
pytest --tb=long
```

### Run Tests Matching Pattern

```bash
pytest -k "spam"  # Run tests with "spam" in name
```

---

## Adding Tests to a New Service

1. **Create tests directory:**
   ```bash
   mkdir services/my-service/tests
   ```

2. **Create test file:**
   ```python
   # services/my-service/tests/test_my_feature.py
   import pytest

   def test_basic_functionality():
       assert True
   ```

3. **Use shared fixtures:**
   ```python
   # Import from root conftest
   def test_with_sample_message(sample_message):
       assert sample_message.content is not None
   ```

4. **Run tests:**
   ```bash
   pytest services/my-service/tests/ -v
   ```

---

## Related Documentation

- [Adding Features](adding-features.md) - Test requirements for new features
- [LLM Integration](llm-integration.md) - LLM classifier tests
- [Contributing](contributing.md) - PR test requirements
