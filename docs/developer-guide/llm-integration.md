# LLM Integration Guide

Guide for working with the Ollama-based LLM classification system.

---

## Overview

The platform uses Ollama for local LLM inference:
- **Realtime Ollama** (`ollama`): Processor classification, API queries
- **Batch Ollama** (`ollama-batch`): Enrichment tasks (AI tagging)

Primary use: Classifying Telegram messages for OSINT relevance and archival decisions.

---

## Prompt Architecture

### Chain-of-Thought Format

All prompts use a structured chain-of-thought format:

```
<thinking>
[LLM reasoning about the message]
</thinking>

<topics>
[List of OSINT topics identified]
</topics>

<answer>
{
  "should_archive": true/false,
  "importance": "high/medium/low",
  "reasoning": "Brief explanation"
}
</answer>
```

### Why This Format?

1. **Thinking section**: Forces LLM to reason before deciding
2. **Topics section**: Explicit topic classification
3. **Answer section**: Structured JSON for parsing
4. **Separation**: Makes parsing reliable even with verbose models

---

## Prompt Versioning

Prompts are stored in the database, not code. This allows:
- Runtime switching without restart
- A/B testing different prompts
- Rollback capability
- Version history

### Database Table: `llm_prompts`

```sql
CREATE TABLE llm_prompts (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,           -- 'osint_classifier'
    version TEXT NOT NULL,        -- 'v7'
    system_prompt TEXT NOT NULL,
    user_prompt_template TEXT NOT NULL,
    is_active BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(name, version)
);
```

### Current Active Version

```sql
SELECT version FROM llm_prompts
WHERE name = 'osint_classifier' AND is_active = TRUE;
-- Returns: v7
```

---

## Creating a New Prompt Version

### Step 1: Read Existing Prompts

```sql
-- Get current active prompt
SELECT version, system_prompt, user_prompt_template
FROM llm_prompts
WHERE name = 'osint_classifier' AND is_active = TRUE;
```

### Step 2: Create New Version

```sql
INSERT INTO llm_prompts (name, version, system_prompt, user_prompt_template, is_active)
VALUES (
    'osint_classifier',
    'v8',
    'You are an OSINT analyst...',  -- New system prompt
    'Analyze this message...',       -- New user template
    FALSE  -- NOT active yet
);
```

### Step 3: Test New Version

```python
# In processor tests or development
from llm_classifier import LLMClassifier

classifier = LLMClassifier(prompt_version='v8')  # Specify version
result = classifier.classify(test_message)
print(result)
```

### Step 4: Activate New Version

```sql
-- Deactivate old version
UPDATE llm_prompts SET is_active = FALSE
WHERE name = 'osint_classifier' AND is_active = TRUE;

-- Activate new version
UPDATE llm_prompts SET is_active = TRUE
WHERE name = 'osint_classifier' AND version = 'v8';
```

### Step 5: Restart Processor

```bash
docker-compose restart processor-worker
```

---

## Fallback Strategy

The classifier has a 4-tier fallback for resilience:

```
Tier 1: Active DB prompt (v7)
    ↓ (if not found)
Tier 2: Previous DB prompt (v6)
    ↓ (if not found)
Tier 3: Older DB prompt (v5)
    ↓ (if not found)
Tier 4: Hardcoded default prompt
```

### Implementation

```python
# services/processor/src/llm_classifier.py
def get_prompt(self) -> tuple[str, str]:
    """Get prompt with fallback strategy."""
    # Try active version
    prompt = self.db.query(LLMPrompt).filter(
        LLMPrompt.name == 'osint_classifier',
        LLMPrompt.is_active == True
    ).first()

    if prompt:
        return prompt.system_prompt, prompt.user_prompt_template

    # Fallback to previous versions
    for version in ['v6', 'v5', 'v4']:
        prompt = self.db.query(LLMPrompt).filter(
            LLMPrompt.name == 'osint_classifier',
            LLMPrompt.version == version
        ).first()
        if prompt:
            logger.warning(f"Using fallback prompt {version}")
            return prompt.system_prompt, prompt.user_prompt_template

    # Last resort: hardcoded
    logger.error("Using hardcoded default prompt")
    return DEFAULT_SYSTEM_PROMPT, DEFAULT_USER_TEMPLATE
```

---

## Folder Tier Strictness

The LLM's `should_archive` decision is influenced by folder tier:

| Folder Pattern | Tier | LLM Behavior |
|----------------|------|--------------|
| `Archive-*` | Lenient | Archive most relevant content |
| `Monitor-*` | Strict | Only high-value OSINT |
| `Discover-*` | Very Strict | Prove channel's worth |

### How It Works

The folder tier is passed to the LLM in the user prompt:

```python
user_prompt = f"""
Analyze this Telegram message for OSINT value.

Channel: {channel_name}
Folder tier: {folder_tier}  # "lenient", "strict", or "very_strict"
Content: {message_content}

For "{folder_tier}" tier, be {"more permissive" if tier == "lenient" else "selective"}
about archival decisions.
"""
```

---

## Response Parsing

### Expected Format

```json
{
  "should_archive": true,
  "importance": "high",
  "reasoning": "Military equipment sighting with coordinates"
}
```

### Parsing Logic

```python
def parse_response(self, response: str) -> dict:
    """Parse LLM response with fallbacks."""
    # Try to extract JSON from <answer> tags
    answer_match = re.search(r'<answer>(.*?)</answer>', response, re.DOTALL)
    if answer_match:
        try:
            return json.loads(answer_match.group(1).strip())
        except json.JSONDecodeError:
            pass

    # Fallback: Try to find JSON anywhere in response
    json_match = re.search(r'\{[^}]+\}', response)
    if json_match:
        try:
            return json.loads(json_match.group())
        except json.JSONDecodeError:
            pass

    # Last resort: Parse keywords
    return {
        "should_archive": "archive" in response.lower(),
        "importance": self._guess_importance(response),
        "reasoning": "Failed to parse structured response"
    }
```

---

## Testing LLM Classification

### Unit Tests

```python
# services/processor/tests/test_llm_classifier.py
import pytest
from llm_classifier import LLMClassifier

@pytest.fixture
def classifier():
    return LLMClassifier(ollama_url="http://ollama:11434")

def test_military_content_classified_high(classifier):
    """Military content should be high importance."""
    result = classifier.classify(
        content="Russian tank column spotted near Bakhmut",
        channel_name="military_intel",
        folder_tier="lenient"
    )
    assert result["should_archive"] == True
    assert result["importance"] == "high"

def test_spam_not_archived(classifier):
    """Spam should not be archived."""
    result = classifier.classify(
        content="🎰 FREE CRYPTO GIVEAWAY! Click here!",
        channel_name="random_channel",
        folder_tier="strict"
    )
    assert result["should_archive"] == False

def test_strict_tier_is_selective(classifier):
    """Strict tier should reject borderline content."""
    borderline = "Weather update: Rain expected in Kyiv"

    lenient_result = classifier.classify(borderline, "news", "lenient")
    strict_result = classifier.classify(borderline, "news", "strict")

    # Lenient might archive, strict should not
    assert strict_result["should_archive"] == False or \
           strict_result["importance"] == "low"
```

### Integration Tests

```bash
# Test with actual Ollama
docker-compose exec processor pytest tests/test_llm_classifier.py -v
```

### Manual Testing

```python
# Quick test script
import httpx

response = httpx.post(
    "http://localhost:11434/api/generate",
    json={
        "model": "qwen2.5:3b",
        "prompt": "Classify: Russian artillery strike on Kharkiv",
        "stream": False
    }
)
print(response.json()["response"])
```

---

## Performance Optimization

### Model Selection

| Model | Speed | Quality | Best For |
|-------|-------|---------|----------|
| `gemma2:2b` | Fast | 75% | Development |
| `qwen2.5:3b` | Medium | 87% | Production (RU/UK) |
| `llama3.2:3b` | Medium | 85% | Fallback |

### CPU Allocation

```yaml
# docker-compose.yml
ollama:
  deploy:
    resources:
      limits:
        cpus: '6.0'   # Realtime needs priority
        memory: 8G
```

### Batching (Enrichment)

For AI tagging (batch enrichment), process multiple messages:

```python
# Batch classification is slower per-message but more efficient
messages = get_batch(50)
for msg in messages:
    result = classifier.classify(msg)
    # Process result
```

---

## Debugging LLM Issues

### Check Ollama Health

```bash
# Is Ollama running?
curl http://localhost:11434/api/tags

# Test generation
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5:3b",
  "prompt": "Hello",
  "stream": false
}'
```

### Check Prompt Loading

```bash
docker-compose logs processor-worker | grep -i "prompt\|version"
```

### Check Classification Logs

```bash
docker-compose logs processor-worker | grep -i "classify\|should_archive"
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Slow responses | CPU contention | Increase Ollama CPU limit |
| "Model not found" | Model not pulled | `docker-compose exec ollama ollama pull qwen2.5:3b` |
| Inconsistent results | Prompt too vague | Add more examples to prompt |
| JSON parse errors | Model formatting | Improve prompt structure |

---

## Prompt Evolution History

Full prompt history is documented in the platform repo:

**Location**: `docs/architecture/LLM_PROMPTS.md`

**Versions**:
- v2: Initial chain-of-thought
- v3: Added folder tier
- v4: Improved JSON formatting
- v5: Added topic classification
- v6: Multilingual support (RU/UK)
- v7: Current production version

**Before making changes**, read this document to understand design decisions.

---

## Related Documentation

- [Adding Features](adding-features.md) - Creating enrichment tasks
- [Services: Processor](services/processor.md) - Processor architecture
- [Services: Enrichment](services/enrichment.md) - AI tagging task
- [Performance Tuning](../operator-guide/performance-tuning.md) - Ollama optimization
