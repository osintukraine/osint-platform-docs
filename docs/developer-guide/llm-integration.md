# LLM Integration Guide

Guide for working with the Ollama-based LLM classification system in the processor service.

---

## Overview

The processor service uses a unified LLM classifier (`llm_classifier.py`) that handles all message classification in a single call:

- **Spam detection** - Identifies financial, promotional, off-topic, and forwarding spam
- **Topic classification** - Categorizes into 13 topics
- **Importance level** - Assigns high/medium/low
- **Archive decision** - Determines if message should be archived

Prompts are stored in the database (`llm_prompts` table) and can be edited via NocoDB without service restarts.

---

## Classification Output

### ClassificationResult Fields

The `classify()` method returns a `ClassificationResult` dataclass:

```python
@dataclass
class ClassificationResult:
    is_spam: bool
    spam_type: Optional[str]       # financial, promotional, off_topic, forwarding
    spam_reason: Optional[str]
    topic: str                     # One of 13 topics
    importance: str                # high, medium, low
    should_archive: bool
    reasoning: str
    archive_reason: Optional[str]
    confidence: float              # 0.0-1.0, default 0.8
    latency_ms: int
    is_ukraine_relevant: bool
    relevance_reason: Optional[str]
    channel_tier: str              # archive, monitor, discover
    analysis_text: str             # Chain-of-thought from <analysis> tags
    raw_response: str              # Full LLM response for debugging
```

### Topic Categories (13)

| Topic | Description |
|-------|-------------|
| `combat` | Battles, strikes, attacks, artillery, explosions |
| `equipment` | Tanks, drones, missiles, weapons deliveries |
| `casualties` | KIA/WIA counts, losses, POW information |
| `movements` | Troop movements, convoys, redeployments |
| `infrastructure` | Energy grid, bridges, civilian infrastructure |
| `humanitarian` | Evacuations, civilian impact, aid deliveries |
| `diplomatic` | Sanctions, negotiations, statements |
| `intelligence` | OSINT analysis, reconnaissance, geolocation |
| `propaganda` | Disinformation, information warfare |
| `units` | Brigade/battalion mentions, commander updates |
| `locations` | Frontline updates, city status, positions |
| `general` | Analysis, commentary, mixed content |
| `uncertain` | Ambiguous content flagged for review |

### Importance Levels

| Level | Criteria |
|-------|----------|
| `high` | Breaking news, significant events, confirmed intelligence |
| `medium` | Routine updates, unconfirmed reports |
| `low` | Commentary, repetitive updates, minor news |

### Spam Types

| Type | Examples |
|------|----------|
| `financial` | Crypto, investments, trading bots, casino |
| `promotional` | Channel ads, merchandise, donation requests |
| `off_topic` | Personal posts, memes, dating, horoscopes |
| `forwarding` | Mass-forwarded chain messages |

---

## Channel Tiers

The LLM's archive decision is influenced by the channel's tier, derived from folder patterns:

| Folder Pattern | Tier | Archive Behavior |
|----------------|------|------------------|
| `Archive-*` | `archive` | Lenient - archive unless clearly low-value |
| `Monitor-*` | `monitor` | Strict - only high OSINT value |
| `Discover-*` | `discover` | Very strict - only exceptional content |

```python
# services/processor/src/llm_classifier.py (line 355)
@staticmethod
def derive_tier(channel_rule: str, folder: Optional[str] = None) -> str:
    if folder:
        folder_lower = folder.lower()
        if folder_lower.startswith("archive"):
            return "archive"
        elif folder_lower.startswith("monitor"):
            return "monitor"
        elif folder_lower.startswith("discover"):
            return "discover"
    # Fall back to rule-based mapping...
```

---

## Prompt Architecture

### Chain-of-Thought Format

The classifier uses chain-of-thought prompting with `<analysis>` tags. The assistant response is prefilled with `<analysis>` to force reasoning before JSON output:

```python
# services/processor/src/llm_classifier.py (line 673)
response = await self.client.chat(
    model=self.model_name,
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_prompt},
        {"role": "assistant", "content": "<analysis>"},  # Prefill forces analysis first
    ],
    # ...
)
```

**Expected LLM output format:**

```
<analysis>
This message contains "прилетіло" (strike landed) which is Ukrainian military slang
indicating a combat event. The mention of Новоросійск suggests a strike on
Russian territory. This is significant combat intelligence.
</analysis>
{
  "is_spam": false,
  "topic": "combat",
  "importance": "high",
  "should_archive": true,
  "is_ukraine_relevant": true,
  "reasoning": "Strike on Novorossiysk port - significant combat event"
}
```

### User Prompt Structure

Messages are wrapped in XML tags for clear separation:

```xml
<tier>ARCHIVE</tier>
<channel>military_intel</channel>

<message>
В Новоросійску міцно прилетіло
</message>

<translation>
It hit hard in Novorossiysk
</translation>
```

---

## Database Schema

### llm_prompts Table

Prompts are stored in `infrastructure/postgres/init.sql`:

```sql
CREATE TABLE IF NOT EXISTS llm_prompts (
    id SERIAL PRIMARY KEY,
    task VARCHAR(50) NOT NULL,           -- e.g., 'message_classification'
    name VARCHAR(100) NOT NULL,          -- Human-readable identifier
    prompt_type VARCHAR(20) NOT NULL,    -- 'system' or 'user_template'
    content TEXT NOT NULL,               -- The actual prompt text

    -- Versioning
    version INTEGER NOT NULL DEFAULT 1,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Model selection (per-prompt override)
    model_name VARCHAR(100),             -- e.g., 'qwen2.5:3b'
    model_parameters JSONB DEFAULT '{}', -- {"temperature": 0.3}
    task_category VARCHAR(50),           -- 'processor', 'enrichment'

    -- Metadata
    description TEXT,
    variables TEXT[],                    -- Template variables
    expected_output_format TEXT,

    -- Performance tracking
    usage_count INTEGER DEFAULT 0,
    avg_latency_ms INTEGER,
    error_count INTEGER DEFAULT 0,
    last_error TEXT,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE,

    CONSTRAINT uq_prompt_task_version UNIQUE (task, version)
);
```

### Query Pattern

```python
# services/processor/src/llm_classifier.py (line 504)
result = await session.execute(
    select(LLMPrompt)
    .where(LLMPrompt.task == task)
    .where(LLMPrompt.is_active == True)
    .where(LLMPrompt.prompt_type == "system")
    .order_by(LLMPrompt.version.desc())
    .limit(1)
)
prompt = result.scalar_one_or_none()
```

---

## Prompt Management

### Creating a New Prompt Version

1. **Insert new version in database:**

```sql
INSERT INTO llm_prompts (task, name, prompt_type, content, version, is_active)
VALUES (
    'message_classification',
    'Message Classification v4',
    'system',
    'Your new prompt content here...',
    4,
    FALSE  -- Start inactive
);
```

2. **Test with specific messages** (via processor logs or test script)

3. **Activate new version:**

```sql
-- Deactivate old
UPDATE llm_prompts
SET is_active = FALSE
WHERE task = 'message_classification' AND is_active = TRUE;

-- Activate new
UPDATE llm_prompts
SET is_active = TRUE
WHERE task = 'message_classification' AND version = 4;
```

4. **Monitor performance** via `usage_count`, `avg_latency_ms`, `error_count`

### Cache Behavior

Prompts are cached for 5 minutes to avoid database queries on every message:

```python
# services/processor/src/llm_classifier.py (line 310)
PROMPT_CACHE_TTL = timedelta(minutes=5)
```

Changes take effect within 5 minutes without restart.

### Military Slang Injection

Prompts can include `{{MILITARY_SLANG}}` placeholder, which gets replaced with the glossary from `military_slang` table:

```python
# services/processor/src/llm_classifier.py (line 522)
if "{{MILITARY_SLANG}}" in prompt_content:
    slang_glossary = await self._get_slang_glossary()
    prompt_content = prompt_content.replace("{{MILITARY_SLANG}}", slang_glossary)
```

---

## Model Configuration

### Default Models

| Role | Model | Notes |
|------|-------|-------|
| **Primary** | `qwen2.5:3b` | Superior RU/UK support, 32k context |
| **Fallback** | `llama3.2:3b` | Used if primary unavailable |

### Model Selection

Model is read from database at runtime (line 392-413):

```python
async def _get_model(self):
    if self.model_override:
        return get_model(self.model_override)

    if self.async_session_maker:
        async with self.async_session_maker() as session:
            selector = ModelSelector(session)
            model = await selector.get_model_for_task(ModelTask.CLASSIFICATION)
            if model:
                return model

    # Fallback
    return get_model("qwen2.5:3b") or get_model("llama3.2:3b")
```

### Per-Prompt Model Override

The `model_name` column in `llm_prompts` allows per-prompt model selection.

---

## Fallback Handling

When the LLM fails (timeout, parse error), a conservative fallback is used:

```python
# services/processor/src/llm_classifier.py (line 1038)
def _fallback_classification(self, content: str, tier: str, latency_ms: int):
    # Basic keyword spam detection
    spam_keywords = ["crypto", "usdt", "invest", "casino", "forex", "subscribe to"]
    is_spam = any(kw in content.lower() for kw in spam_keywords)

    # Conservative: archive only for 'archive' tier
    should_archive = not is_spam and tier == "archive"

    return ClassificationResult(
        is_spam=is_spam,
        topic="general",
        importance="medium",
        should_archive=should_archive,
        confidence=0.5,
        # ...
    )
```

### JSON Repair

Truncated JSON responses (from `num_predict` limits) are repaired via regex extraction:

```python
# services/processor/src/llm_classifier.py (line 771)
def _parse_json_with_repair(self, json_text: str, analysis_text: str = "") -> dict:
    # Try normal parse
    # Try closing brackets/quotes
    # Extract fields via regex
    # Parse analysis text for hints
    # Return defaults
```

---

## Usage Example

```python
from llm_classifier import LLMClassifier

classifier = LLMClassifier()

result = await classifier.classify(
    content="В Новоросійску міцно прилетіло",
    channel_name="military_intel",
    channel_rule="archive_all",
    channel_folder="Archive-UA",
    content_translated="It hit hard in Novorossiysk"
)

if result.is_spam:
    # Skip spam
    pass
elif result.should_archive:
    # Archive with metadata
    save_message(
        topic=result.topic,
        importance=result.importance,
        reasoning=result.reasoning
    )
```

---

## Statistics

The classifier tracks statistics accessible via `get_stats()`:

```python
stats = classifier.get_stats()
# Returns:
{
    "total_classified": 1234,
    "total_spam": 56,
    "total_archived": 1100,
    "spam_rate": 0.045,
    "archive_rate": 0.89,
    "avg_time_seconds": 1.2,
    "errors": 3,
    "model_switches": 0,
    "prompt_reloads": 5,
    "topics_distribution": {"combat": 400, "equipment": 200, ...},
    "current_model": "qwen2.5:3b"
}
```

---

## Debugging

### Check Prompt Loading

```bash
docker-compose logs processor-worker | grep -i "prompt\|version"
```

### Check Classification

```bash
docker-compose logs processor-worker | grep -i "Classified:"
```

### Check Analysis Text

The `analysis_text` field captures chain-of-thought reasoning:

```python
result = await classifier.classify(content)
print(result.analysis_text)  # LLM's reasoning
print(result.raw_response)   # Full response for debugging
```

---

## Related Documentation

- [Adding Features](adding-features.md) - Creating enrichment tasks
- [Services: Processor](services/processor.md) - Processor architecture
- [Environment Variables](../reference/environment-variables.md) - Ollama configuration
