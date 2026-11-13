# Translation Backfill Service

## Overview

The Translation Backfill Service is a **standalone batch processor** that translates historical messages in bulk. It was designed to backfill translations for messages archived before translation support was added to the platform, or to retry failed translations.

!!! info "Service Status: Legacy"
    This service has been **superseded by the Enrichment Service's TranslationTask** (`services/enrichment/src/tasks/translation.py`, run by the fast-pool worker). The standalone code remains in the repository for one-time bulk operations, but **it is not defined in `docker-compose.yml`** (only a `replicas: 0` stub in `docker-compose.development.yml` mentions it), so `docker-compose run translation-backfill` does not work as written. Build and run it from its own directory (see Running Locally).

### Purpose

- Bulk translate historical messages missing translations
- Retry failed translations from real-time processing
- One-time migration tasks (e.g., translating entire archives)
- Development/testing of translation pipeline

### Key Features

| Feature | Description |
|---------|-------------|
| **DeepL Integration** | High-quality translations using DeepL Pro API (superior to Google Translate) |
| **Google Fallback** | Automatic fallback to Google Translate if DeepL unavailable |
| **Batch Processing** | Configurable batch sizes for efficient API usage |
| **Progress Tracking** | None; each run re-selects `content_translated IS NULL`, so reruns resume naturally |
| **Rate Limiting** | None built in; a failed DeepL call falls back to Google, and failures are counted as errors |
| **Language Detection** | Optional `--skip-english` skips messages detected as English; otherwise everything is sent to the translator |

---

## Architecture

### Service Position

```
┌─────────────┐
│ PostgreSQL  │
│  messages   │  ← Selects WHERE content_translated IS NULL
└──────┬──────┘
       │
       ▼
┌──────────────────────────┐
│ Translation Backfill     │
│                          │
│ ┌────────────────────┐   │
│ │ Language Detection │   │
│ └────────┬───────────┘   │
│          │               │
│          ▼               │
│ ┌────────────────────┐   │
│ │ DeepL API Client   │───┼───► DeepL API
│ └────────┬───────────┘   │       (api-free.deepl.com)
│          │ (fallback)    │
│          ▼               │
│ ┌────────────────────┐   │
│ │ Google Translate   │───┼───► Google Translate
│ └────────┬───────────┘   │       (Free)
└──────────┼──────────────┘
           │
           ▼
    ┌──────────────┐
    │ UPDATE       │
    │ messages SET │
    │ content_     │
    │ translated   │
    └──────────────┘
```

### Key Files

| File | Purpose | Lines |
|------|---------|-------|
| `translate_backfilled_messages_standalone.py` | Main translation script (no `src/` dir, no shared-library imports) | 339 |
| `requirements.txt` | Dependencies (deepl, deep-translator, langdetect, asyncpg, psycopg2, sqlalchemy) | |
| `Dockerfile` | `python:3.11-slim`; loops the script every `TRANSLATION_INTERVAL` seconds (default 3600) | |
| `README.md` | Service documentation | |

---

## Translation Pipeline

### 1. Language Detection

Uses `langdetect` library to identify source language:

```python
from langdetect import detect

def detect_language(text: str) -> Optional[str]:
    # Use first 1000 chars for faster, more accurate detection
    sample = text[:1000] if len(text) > 1000 else text
    lang = detect(sample)  # Returns 'ru', 'uk', 'en', etc.
    return lang
```

**Supported Languages:**
- Russian (`ru`) and Ukrainian (`uk`) are the target use case
- English (`en`) - skipped only with `--skip-english`
- Any other language the provider supports (`GoogleTranslator` uses `source="auto"`)

### 2. DeepL Translation (Primary)

**API Configuration:**

```python
import deepl

# Initialize client
translator = deepl.Translator(DEEPL_API_KEY)

# Translate text
result = translator.translate_text(
    text,
    target_lang="EN-US",  # Always translate to US English
)

# Extract results
translation = result.text
source_lang = result.detected_source_lang.lower()  # 'ru', 'uk', etc.
```

**DeepL API Tiers:**

| Tier | Quota | Cost | API URL |
|------|-------|------|---------|
| **Free** | 500,000 chars/month | €0 | `https://api-free.deepl.com/v2/translate` |
| **Pro** | Pay-per-use | €20/million chars | `https://api.deepl.com/v2/translate` |

!!! tip "Cost Savings"
    DeepL Free tier provides 500,000 characters/month (€0/month). Typical message is ~200 characters, so **~2,500 messages/month free**. For large-scale operations, Pro tier costs €0.004/message.

### 3. Google Translate (Fallback)

If DeepL unavailable or API key not configured:

```python
from deep_translator import GoogleTranslator

translator = GoogleTranslator(source="auto", target="en")
translated = translator.translate(text)
```

**When Fallback Triggers:**
- No `DEEPL_API_KEY` configured
- Any DeepL exception (the client catches all errors, including 429/456 quota errors)
- User passes `--google-only` flag

### 4. Database Update

Updates message record with translation data:

```sql
UPDATE messages
SET content_translated = 'Translated text...',
    language_detected = 'ru',                -- Source language
    translation_target = 'en',               -- Target language
    translation_provider = 'deepl',          -- 'deepl' or 'google'
    translation_timestamp = NOW(),           -- When translated
    translation_cost_usd = 0.0               -- Always 0.0 (hard-coded for both providers)
WHERE id = 12345;
```

**Database Schema:**

```sql
-- From infrastructure/postgres/init.sql
ALTER TABLE messages ADD COLUMN
    language_detected VARCHAR(10),           -- Auto-detected language
    content_translated TEXT,                 -- English translation
    translation_target VARCHAR(10),          -- Always 'en'
    translation_provider VARCHAR(50),        -- 'deepl', 'google', etc.
    translation_timestamp TIMESTAMP WITH TIME ZONE,
    translation_cost_usd NUMERIC(10, 6);

-- Index for finding untranslated messages
CREATE INDEX idx_messages_pending_translation
ON messages(id)
WHERE content_translated IS NULL AND is_spam = false;
```

---

## Batch Processing

### Message Selection Query

```python
# get_untranslated_messages(): the whole result set is loaded, then processed in --batch-size chunks
query = """
    SELECT id, content
    FROM messages
    WHERE is_backfilled = true          -- Only backfilled messages
      AND content_translated IS NULL    -- No translation yet
      AND content IS NOT NULL
      AND content != ''
    -- No is_spam filter, no ORDER BY; LIMIT only when --limit is given
"""
```

### Batch Workflow

```python
async def process_batch(messages, batch_size=10):
    for i in range(0, len(messages), batch_size):
        batch = messages[i:i + batch_size]

        for msg_id, content in batch:
            # 1. Detect language (only with --skip-english)
            if skip_english and detect_language(content) == 'en':
                continue

            # 2. Translate
            translation = await translate(content)

            # 3. Update database
            await update_translation(msg_id, translation)

        # 4. Commit batch
        await session.commit()
        logger.info(f"Batch {i//batch_size + 1} complete")
```

### Progress Tracking

Service maintains internal state but does **not** use a dedicated progress table (unlike the enrichment service):

```python
# Progress logged to stdout
logger.info(f"✅ Message 12345: ru → en (247 chars via deepl)")
logger.info(f"💾 Batch committed: 10 translations")
```

**Check Progress via SQL:**

```sql
-- Overall progress
SELECT
    COUNT(*) FILTER (WHERE content_translated IS NULL) as pending,
    COUNT(*) FILTER (WHERE content_translated IS NOT NULL) as completed,
    ROUND(100.0 * COUNT(*) FILTER (WHERE content_translated IS NOT NULL) / COUNT(*), 2) as pct_complete
FROM messages
WHERE created_at >= '2022-02-24'
  AND is_spam = false;

-- Progress by channel
SELECT
    c.name,
    COUNT(*) FILTER (WHERE m.content_translated IS NULL) as pending,
    COUNT(*) FILTER (WHERE m.content_translated IS NOT NULL) as completed
FROM messages m
JOIN channels c ON m.channel_id = c.id
WHERE m.is_spam = false
GROUP BY c.id, c.name
ORDER BY pending DESC;
```

---

## Configuration

### Environment Variables

```bash
# PostgreSQL connection (read by the script; no port variable, default 5432)
POSTGRES_HOST=localhost              # default localhost
POSTGRES_DB=osint_platform          # default osint_platform
POSTGRES_USER=osint_user            # default osint_user
POSTGRES_PASSWORD=your_password     # default empty

# DeepL API (optional)
DEEPL_API_KEY=your_deepl_key_here  # Leave empty to use Google only. The script always uses deepl.Translator's default endpoint selection; DEEPL_API_URL from shared settings is not read

# Read only by the Dockerfile CMD loop (not by the Python script)
TRANSLATION_BATCH_SIZE=20           # passed as --batch-size (script default 10)
TRANSLATION_SKIP_ENGLISH=true       # any non-empty value adds --skip-english
TRANSLATION_INTERVAL=3600           # seconds between runs
```

### Command-Line Arguments

```bash
# Limit number of messages (testing)
python3 translate_backfilled_messages_standalone.py --limit 100

# Use Google Translate only (no DeepL)
python3 translate_backfilled_messages_standalone.py --google-only

# Custom batch size
python3 translate_backfilled_messages_standalone.py --batch-size 50

# Skip English messages
python3 translate_backfilled_messages_standalone.py --skip-english
```

---

## Running Locally

### One-Time Backfill

Process all untranslated messages once:

```bash
# Via Python directly (recommended; the service is not in docker-compose.yml)
cd services/translation-backfill
pip install -r requirements.txt
POSTGRES_HOST=localhost POSTGRES_PASSWORD=... DEEPL_API_KEY=your_key \
  python3 translate_backfilled_messages_standalone.py

# Or build the image from the service directory (its Dockerfile copies requirements.txt from the context root)
docker build -t osint-translation-backfill services/translation-backfill
docker run --rm --network <compose_network> -e POSTGRES_HOST=postgres -e POSTGRES_PASSWORD=... osint-translation-backfill
```

### Testing with Limited Messages

```bash
# Translate only 10 messages (for testing)
python3 translate_backfilled_messages_standalone.py --limit 10

# Test Google Translate fallback
python3 translate_backfilled_messages_standalone.py --google-only --limit 5
```

### Continuous Mode (Daemon)

Run as background service that checks periodically:

!!! warning "No compose service"
    The image's CMD loops forever (script, then sleep `TRANSLATION_INTERVAL`), but there is no `translation-backfill` entry in `docker-compose.yml`; `docker-compose.development.yml` only sets `replicas: 0` for that name. To use daemon mode you would have to add a service definition yourself (build context `services/translation-backfill`, `POSTGRES_*` and `DEEPL_API_KEY` env). Continuous translation is handled by the enrichment `translation` task.

### Scheduled Backfill (Cron)

For production, run as scheduled job:

```bash
# Add to crontab
crontab -e

# Run backfill daily at 2 AM
0 2 * * * cd /path/to/platform/services/translation-backfill && python3 translate_backfilled_messages_standalone.py >> /var/log/translation-backfill.log 2>&1
```

---

## Troubleshooting

### API Key Errors (403 Forbidden)

**Symptom:** DeepL API returns 403 Forbidden

**Check API key validity:**
```bash
curl -X POST https://api-free.deepl.com/v2/usage \
  -H "Authorization: DeepL-Auth-Key $DEEPL_API_KEY"
```

**Common Causes:**
- Expired API key (regenerate at [deepl.com/pro-checkout](https://www.deepl.com/pro-checkout))
- Wrong API URL (free tier uses `api-free.deepl.com`, pro uses `api.deepl.com`)
- API key not set in environment

**Solution:**
```bash
# Get new API key from DeepL
# Update .env file
DEEPL_API_KEY=your_new_key_here

# Re-run the script / restart your container
```

### Quota Exceeded (456)

**Symptom:** DeepL returns 456 error after processing some messages

**Check usage:**
```bash
curl -X POST https://api-free.deepl.com/v2/usage \
  -H "Authorization: DeepL-Auth-Key $DEEPL_API_KEY"
```

**Response:**
```json
{
    "character_count": 480000,
    "character_limit": 500000
}
```

**Solutions:**

1. **Wait for monthly reset** (free tier resets on signup anniversary)
2. **Upgrade to Pro tier** (€20/million chars, no hard limit)
3. **Use Google Translate fallback**:
   ```bash
   python3 translate_backfilled_messages_standalone.py --google-only
   ```

### Slow Processing

**Symptom:** Service processes messages very slowly

**Diagnose:**
```bash
# Check the script output for provider failures
grep -i "failed\|fall" translation-backfill.log

# Check database connection
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(*) FROM messages WHERE content_translated IS NULL;"
```

**Solutions:**

1. **Increase batch size** (uses more memory):
   ```bash
   python3 translate_backfilled_messages_standalone.py --batch-size 50
   ```

2. **Check network latency** to DeepL API:
   ```bash
   curl -w "@curl-format.txt" -o /dev/null -s https://api-free.deepl.com/v2/usage
   ```

3. **Use Pro tier** for higher throughput (no rate limits)

### Database Connection Errors

**Symptom:** Cannot connect to PostgreSQL

**Check connectivity:**
```bash
psql "postgresql://osint_user:password@<host>:5432/osint_platform" -c "SELECT 1;"
```

**Common Causes:**
- PostgreSQL service not running: `docker-compose ps postgres`
- Wrong credentials in `.env` file
- Network issues between containers

**Solution:**
```bash
# Verify PostgreSQL is running
docker-compose ps postgres

# Check environment variables
docker-compose config | grep POSTGRES

# Restart PostgreSQL if needed
docker-compose restart postgres
```

### Translation Failures

**Symptom:** Some messages fail to translate

**Check logs:**
```bash
grep "❌" translation-backfill.log
```

**Common Causes:**
- Message too long (>10,000 characters)
- Invalid UTF-8 encoding
- Network timeout to translation API
- Unsupported language

**Debug specific message:**
```sql
-- Get message that failed
SELECT id, LEFT(content, 100), LENGTH(content)
FROM messages
WHERE id = 12345;

-- Check for encoding issues
SELECT id, content::bytea
FROM messages
WHERE id = 12345;
```

---

## Migration to Enrichment Service

!!! warning "Deprecation Notice"
    The standalone translation-backfill service has been **integrated into the Enrichment Service**. New deployments should use the enrichment worker instead.

### Why Migration Happened

The standalone service was replaced to:

1. **Unified architecture** - All background tasks in one service
2. **Better resource management** - Shared worker pool
3. **Improved monitoring** - Centralized metrics
4. **Simplified deployment** - One service instead of many

### Enrichment Service Equivalent

**Old (translation-backfill):** run the standalone script as above.

**New (enrichment):** `TranslationTask` is registered in `services/enrichment/src/workers/fast_worker.py` (task key `translation`) and runs inside the `enrichment-fast-pool` container; there is no per-task CLI. It uses DeepL with Google fallback and pauses on DeepL quota or rate-limit signals (`translation.deepl_limits`) instead of failing the batch.

**Code Location:**
- **Old:** `/services/translation-backfill/translate_backfilled_messages_standalone.py`
- **New:** `/services/enrichment/src/tasks/translation.py`

### When to Use Standalone Service

The standalone service is still useful for:

- **One-time bulk operations** - Translating entire historical archive
- **Development/testing** - Isolated testing of translation logic
- **Migration tasks** - Moving data between platforms
- **Emergency backfill** - When enrichment service unavailable

### Comparison Table

| Feature | Translation-Backfill (Standalone) | Enrichment Service |
|---------|-----------------------------------|-------------------|
| **Architecture** | Single-purpose script | Multi-task worker |
| **Deployment** | Separate container | Shared worker pool |
| **Progress tracking** | Logs only | Cursor-based (`id > :last_id`) |
| **Monitoring** | None (no Prometheus metrics) | Worker metrics |
| **Task scheduling** | Cron or manual | Worker loop |
| **Resource usage** | Dedicated resources | Shared with other tasks |
| **Use case** | Bulk one-time backfill | Continuous enrichment |

---

## Performance Optimization

### Estimating Completion Time

```sql
-- Calculate characters remaining
SELECT SUM(LENGTH(content)) as chars_remaining
FROM messages
WHERE content_translated IS NULL
  AND is_spam = false;

-- Example: 5,000,000 characters remaining
-- DeepL Free tier: 500,000 chars/month
-- Time needed: 10 months (or upgrade to Pro)

-- Pro tier at 20 requests/min × 1000 chars/request = 20,000 chars/min
-- Time = 5,000,000 / 20,000 = 250 minutes = 4.2 hours
```

### Batch Size Tuning

| Batch Size | Memory Usage | Speed | API Usage | Recommended For |
|------------|--------------|-------|-----------|-----------------|
| 10 | Low (256MB) | Slow | Conservative | Testing, low memory |
| 20 | Medium (512MB) | Medium | Balanced | Production default |
| 50 | High (1GB) | Fast | Aggressive | Bulk operations |
| 100 | Very High (2GB) | Fastest | Heavy | One-time backfill |

### Database Indexes

Ensure these indexes exist for optimal performance:

```sql
-- Index for finding untranslated messages (already in init.sql)
CREATE INDEX IF NOT EXISTS idx_messages_pending_translation
ON messages(id)
WHERE content_translated IS NULL AND is_spam = false;

-- Index for language filtering
CREATE INDEX IF NOT EXISTS idx_messages_language
ON messages(language_detected);

-- Index for backfilled messages
CREATE INDEX IF NOT EXISTS idx_messages_backfilled
ON messages(is_backfilled)
WHERE is_backfilled = true;
```

---

## Cost Analysis

### Free Tier Usage

**DeepL Free:**
- **Quota:** 500,000 characters/month
- **Typical message:** 200 characters
- **Messages/month:** 2,500
- **Cost:** €0

**Google Translate:**
- **Quota:** Unlimited (with rate limiting)
- **Cost:** €0

### Pro Tier Pricing

**DeepL Pro:**
- **Price:** €20 per million characters
- **Cost per message (200 chars):** €0.004
- **Example:** 100,000 messages = €400

### Cost Comparison

| Volume | Free Tier (DeepL) | Pro Tier (DeepL) | Google Translate |
|--------|-------------------|------------------|------------------|
| **1,000 messages** | €0 | €0.80 | €0 |
| **10,000 messages** | €0 (20 months) | €8.00 | €0 |
| **100,000 messages** | €0 (200 months) | €80.00 | €0 |
| **1,000,000 messages** | Not feasible | €800.00 | €0 |

!!! tip "Recommendation"
    - **< 500K chars/month:** Use DeepL Free (better quality than Google)
    - **> 500K chars/month:** Use Pro tier for bulk, then free tier for ongoing
    - **Quality not critical:** Use Google Translate (free, unlimited)

---

## Metrics

None. The script exports no Prometheus metrics; it logs per-message results and a final summary (translated, skipped, errors). The Dockerfile healthcheck only opens a psycopg2 connection.

---

## Related Documentation

### Internal Documentation

- [Enrichment Service](enrichment.md) - Successor to translation-backfill
- [Processor Service](processor.md) - Real-time translation during ingestion
- [Architecture Overview](../architecture.md) - System design
- [Database Tables](../../reference/database-tables.md) - Translation fields

### External Resources

- [DeepL API Documentation](https://www.deepl.com/docs-api) - Official API reference
- [DeepL Pricing](https://www.deepl.com/pro-checkout) - Free vs Pro tiers
- [langdetect Library](https://github.com/Mimino666/langdetect) - Language detection
- [deep-translator](https://github.com/nidhaloff/deep-translator) - Google Translate wrapper

---

## Summary

The Translation Backfill Service is a **standalone batch processor** designed for one-time bulk translation operations. While it has been superseded by the Enrichment Service for ongoing translation, it remains valuable for:

- Historical backfills of large message archives
- Testing translation pipeline in isolation
- Emergency translation when enrichment service unavailable
- Development and debugging of translation logic

**Key Takeaways:**

- Uses **DeepL Pro API** (high quality) with **Google Translate fallback**
- Processes messages in **configurable batches** (script default 10; Dockerfile loop default 20)
- Supports **language detection** to skip English messages
- Tracks progress via **database state** (`content_translated IS NULL`); only `is_backfilled = true` messages are selected
- **Free tier provides 500,000 chars/month** (~2,500 messages)
- **Pro tier costs €20/million chars** (€0.004/message)

For new deployments, use the **Enrichment Service's TranslationTask** for continuous translation. Use this standalone service only for bulk operations or special cases.
