# Testing Guide

Guide for writing and running tests in the OSINT Intelligence Platform.

---

## Test Structure

Tests are located alongside each service:

```
services/
├── api/
│   └── tests/
│       ├── conftest.py       # Shared fixtures
│       ├── test_messages.py
│       ├── test_channels.py
│       └── test_entities.py
├── processor/
│   └── tests/
│       ├── conftest.py
│       ├── test_spam_filter.py
│       └── test_llm_classifier.py
└── enrichment/
    └── tests/
        ├── conftest.py
        └── test_embedding.py
```

---

## Running Tests

### Run All Tests for a Service

```bash
# API tests
docker-compose exec api pytest

# Processor tests
docker-compose exec processor pytest

# With coverage
docker-compose exec api pytest --cov=src --cov-report=term-missing
```

### Run Specific Test File

```bash
docker-compose exec api pytest tests/test_messages.py -v
```

### Run Specific Test

```bash
docker-compose exec api pytest tests/test_messages.py::test_get_messages -v
```

### Run Tests Locally (Without Docker)

```bash
# Set up environment
cd services/api
pip install -r requirements.txt
pip install pytest pytest-asyncio pytest-cov

# Set environment variables
export POSTGRES_HOST=localhost
export REDIS_HOST=localhost

# Run tests
pytest tests/ -v
```

---

## Writing Unit Tests

### Basic Test Structure

```python
# services/api/tests/test_example.py
import pytest
from httpx import AsyncClient

from main import app


@pytest.mark.asyncio
async def test_health_endpoint():
    """Health endpoint should return 200."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/health")

    assert response.status_code == 200
    assert response.json()["status"] == "healthy"
```

### Using Fixtures

```python
# services/api/tests/conftest.py
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

from main import app
from database import get_db


@pytest.fixture
async def client():
    """HTTP client for API tests."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client


@pytest.fixture
async def db_session():
    """Database session for tests."""
    engine = create_async_engine(
        "postgresql+asyncpg://osint_user:password@localhost:5432/osint_test"
    )
    async with AsyncSession(engine) as session:
        yield session
        await session.rollback()


@pytest.fixture
async def sample_message(db_session):
    """Create sample message for tests."""
    from models import Message

    message = Message(
        telegram_id=123456,
        content="Test message content",
        channel_id=1,
        telegram_date=datetime.utcnow()
    )
    db_session.add(message)
    await db_session.commit()
    await db_session.refresh(message)
    return message
```

### Testing with Database

```python
@pytest.mark.asyncio
async def test_get_message_by_id(client, sample_message):
    """Should return message by ID."""
    response = await client.get(f"/api/messages/{sample_message.id}")

    assert response.status_code == 200
    data = response.json()
    assert data["id"] == sample_message.id
    assert data["content"] == "Test message content"


@pytest.mark.asyncio
async def test_message_not_found(client):
    """Should return 404 for non-existent message."""
    response = await client.get("/api/messages/999999")

    assert response.status_code == 404
    assert "not found" in response.json()["detail"].lower()
```

---

## Mocking Guidelines

### Mock External Services

```python
from unittest.mock import AsyncMock, patch


@pytest.mark.asyncio
async def test_ollama_classification():
    """Test LLM classification with mocked Ollama."""
    mock_response = {
        "should_archive": True,
        "importance": "high",
        "reasoning": "Military content"
    }

    with patch("llm_classifier.httpx.AsyncClient") as mock_client:
        mock_client.return_value.__aenter__.return_value.post = AsyncMock(
            return_value=AsyncMock(
                json=lambda: {"response": json.dumps(mock_response)}
            )
        )

        classifier = LLMClassifier()
        result = await classifier.classify("Tank spotted near Bakhmut")

        assert result["should_archive"] == True
        assert result["importance"] == "high"
```

### Mock Database Queries

```python
@pytest.mark.asyncio
async def test_get_channels(client):
    """Test channel listing with mocked DB."""
    mock_channels = [
        {"id": 1, "name": "Channel 1", "active": True},
        {"id": 2, "name": "Channel 2", "active": True}
    ]

    with patch("routers.channels.get_channels") as mock_get:
        mock_get.return_value = mock_channels

        response = await client.get("/api/channels")

        assert response.status_code == 200
        assert len(response.json()) == 2
```

### When NOT to Mock

- **Integration tests**: Use real database with test data
- **End-to-end tests**: Run full stack
- **LLM tests**: At least some tests should hit actual Ollama

---

## LLM Classification Tests

The processor has 16+ tests for LLM classification:

```python
# services/processor/tests/test_llm_classifier.py

@pytest.mark.asyncio
async def test_military_content_high_importance(classifier):
    """Military content should be high importance."""
    result = await classifier.classify(
        content="Russian BMP-2 destroyed near Avdiivka",
        channel_name="military_intel",
        folder_tier="lenient"
    )
    assert result["should_archive"] == True
    assert result["importance"] == "high"


@pytest.mark.asyncio
async def test_spam_rejected(classifier):
    """Spam should not be archived."""
    result = await classifier.classify(
        content="🎰 FREE BITCOIN! Click link!",
        channel_name="random",
        folder_tier="lenient"
    )
    assert result["should_archive"] == False


@pytest.mark.asyncio
async def test_strict_tier_selective(classifier):
    """Strict tier should be more selective."""
    borderline = "Weather forecast for Kyiv region"

    lenient = await classifier.classify(borderline, "news", "lenient")
    strict = await classifier.classify(borderline, "news", "strict")

    # Strict should be less likely to archive borderline content
    assert strict["importance"] in ["low", "medium"]


@pytest.mark.asyncio
async def test_ukrainian_language(classifier):
    """Should handle Ukrainian content."""
    result = await classifier.classify(
        content="Російські війська відступають з Херсону",
        channel_name="ua_news",
        folder_tier="lenient"
    )
    assert result["should_archive"] == True


@pytest.mark.asyncio
async def test_russian_language(classifier):
    """Should handle Russian content."""
    result = await classifier.classify(
        content="Взрывы слышны в Белгороде",
        channel_name="ru_news",
        folder_tier="lenient"
    )
    assert result["should_archive"] == True
```

---

## Integration Tests

### Testing with Real Database

```python
# services/api/tests/test_integration.py
import pytest
from sqlalchemy import text


@pytest.mark.integration
@pytest.mark.asyncio
async def test_message_search_integration(db_session):
    """Integration test with real database."""
    # Insert test data
    await db_session.execute(text("""
        INSERT INTO messages (telegram_id, content, channel_id, telegram_date)
        VALUES (1, 'Test content about tanks', 1, NOW())
    """))
    await db_session.commit()

    # Query
    result = await db_session.execute(text("""
        SELECT * FROM messages WHERE content ILIKE '%tanks%'
    """))
    rows = result.fetchall()

    assert len(rows) == 1
    assert "tanks" in rows[0].content
```

### Running Integration Tests

```bash
# Mark integration tests
pytest -m integration

# Skip integration tests
pytest -m "not integration"
```

---

## Test Coverage

### Generate Coverage Report

```bash
# Terminal report
docker-compose exec api pytest --cov=src --cov-report=term-missing

# HTML report
docker-compose exec api pytest --cov=src --cov-report=html
```

### Coverage Targets

| Component | Target | Critical Paths |
|-----------|--------|----------------|
| API | 80% | Endpoints, auth |
| Processor | 70% | Spam filter, LLM |
| Enrichment | 60% | Task base class |

---

## Test Data

### Sample Data Fixtures

```python
# services/api/tests/fixtures.py
SAMPLE_MESSAGES = [
    {
        "content": "Russian convoy spotted near Bakhmut",
        "channel_name": "military_intel",
        "osint_score": 85,
        "importance": "high"
    },
    {
        "content": "Weather update for Kyiv",
        "channel_name": "ua_news",
        "osint_score": 30,
        "importance": "low"
    }
]

SPAM_SAMPLES = [
    "🎰 FREE CRYPTO GIVEAWAY",
    "Join our pump group",
    "💰 EARN $1000/DAY"
]
```

### Database Seeding

```bash
# Seed test database
docker-compose exec postgres psql -U osint_user -d osint_test \
  -f tests/fixtures/seed.sql
```

---

## CI Integration

Tests run automatically on PR via GitHub Actions:

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start services
        run: docker-compose up -d postgres redis

      - name: Run API tests
        run: docker-compose run api pytest --cov=src

      - name: Run processor tests
        run: docker-compose run processor pytest --cov=src
```

---

## Debugging Tests

### Verbose Output

```bash
pytest -v -s  # Show print statements
pytest --tb=long  # Long tracebacks
```

### Debug Single Test

```bash
pytest tests/test_messages.py::test_search -v --pdb
```

### Check Test Database

```bash
docker-compose exec postgres psql -U osint_user -d osint_test
```

---

## Related Documentation

- [Adding Features](adding-features.md) - Test requirements for new features
- [LLM Integration](llm-integration.md) - LLM-specific tests
- [Contributing](contributing.md) - PR test requirements
