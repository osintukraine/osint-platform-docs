# Fact-Check Service

**Classification validation service for quality assurance of LLM classifications**

## Overview

The Fact-Check Service validates message classifications by re-analyzing previously classified messages using a separate LLM evaluation. It identifies discrepancies between original and fact-check classifications, enabling both automated corrections and human review workflows.

```mermaid
graph LR
    A[Messages DB] --> B[Stratified Sampler]
    B --> C[LLM Re-classification]
    C --> D{Match?}
    D -->|Yes| E[Log Match]
    D -->|No| F{Confidence?}
    F -->|≥85%| G[Auto-hide or reclassify 'uncertain'; else review]
    F -->|60-85%| H[Queue Review]
    F -->|<60%| I[Log Only]
    G --> J[Audit Log]
    H --> J

    style C fill:#ffe1e1
    style G fill:#e1ffe1
    style H fill:#fff4e1
```

### Key Characteristics

| Aspect | Fact-Check Service |
|--------|-------------------|
| **Processing Mode** | Background batch sampling |
| **Speed** | Slow (LLM inference per message) |
| **Priority** | Low (25) - runs when LLM resources available |
| **Worker** | `enrichment-fact-check` compose service, metrics on port 9203 |
| **Purpose** | Classification quality assurance |
| **Actions** | Auto-reclassify, auto-hide, queue for review |

!!! success "Production Stats"
    - **Sampling**: 50 messages per cycle across all topics, after any pending council votes
    - **Accuracy Target**: 95%+ classification match rate
    - **Council**: also casts the auditor vote on `council_pending` decisions (see [Classification Council](#classification-council))
    - **Review Queue**: Surfaces misclassifications for human review
    - **Lane**: runs on the LLM gateway's *audit* lane (reasoning alias); with `LLM_PROVIDER=ollama` it uses the configured Ollama endpoint

## Architecture

### Component Overview

```mermaid
graph TD
    subgraph "Fact-Check Worker"
        W[FactCheckWorker] --> T[FactCheckTask]
    end

    subgraph "Data Flow"
        T -->|Sample| DB[(PostgreSQL)]
        T -->|Classify| LLM[LLM]
        T -->|Store| FC[classification_fact_checks]
        T -->|Audit| AL[admin_audit_log]
    end

    subgraph "Admin UI"
        API[/admin/fact-check API] --> FC
        UI[Frontend] --> API
    end

    style T fill:#ffe1e1
    style LLM fill:#e1f5ff
```

### Files

| Component | Location |
|-----------|----------|
| **Worker** | `/services/enrichment/src/workers/fact_check_worker.py` |
| **Task** | `/services/enrichment/src/tasks/fact_check.py` |
| **API Router** | `/services/api/src/routers/admin/fact_check.py` |
| **Frontend** | `/services/frontend-nextjs/app/admin/fact-check/page.tsx` |

## Classifier Modes

The fact-check service supports two classification modes, configurable via `FACT_CHECK_CLASSIFIER_MODE`:

### Unified Mode (Default)

Re-classifies messages from scratch using the `message_classification` prompt, then compares with original.

```mermaid
sequenceDiagram
    participant FC as Fact-Check
    participant LLM as LLM
    participant DB as PostgreSQL

    FC->>DB: Sample message (with original classification)
    FC->>LLM: Classify message (fresh)
    LLM->>FC: New classification
    FC->>FC: Compare original vs new
    FC->>DB: Store result + action
```

**Advantages**:
- Independent classification (unbiased by original)
- Same prompt as processor (apples-to-apples comparison)
- Catches systematic prompt issues

### Modular Mode

Passes original classification to LLM for validation using `fact_check_validate` prompt.

```mermaid
sequenceDiagram
    participant FC as Fact-Check
    participant LLM as LLM
    participant DB as PostgreSQL

    FC->>DB: Sample message + original classification
    FC->>LLM: Validate original classification
    LLM->>FC: Validation result (match/mismatch)
    FC->>DB: Store result + action
```

**Advantages**:
- Faster (single-purpose prompt)
- Direct validation feedback
- LLM provides reasoning for disagreement

!!! note "Mode Selection"
    Unified mode is recommended for production as it provides unbiased re-classification. The match calculation is performed in code (not by LLM) for reliability.

## Sampling Strategy

### Stratified Topic Sampling

Messages are sampled proportionally across all topics to ensure balanced coverage:

```sql
WITH sampled AS (
    SELECT m.id, m.content, m.osint_topic,
           ROW_NUMBER() OVER (PARTITION BY m.osint_topic ORDER BY RANDOM()) as rn
    FROM messages m
    WHERE m.osint_topic IS NOT NULL
      AND m.id NOT IN (SELECT message_id FROM classification_fact_checks)
)
SELECT * FROM sampled WHERE rn <= :per_topic LIMIT :batch_size
```

**Parameters**:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `FACT_CHECK_SAMPLE_SIZE` | 50 | Total messages per cycle |
| `FACT_CHECK_PER_TOPIC` | 5 in code, `3` in compose | Max messages per topic |

This ensures rare topics (e.g., `disinformation`) get checked alongside common ones (e.g., `combat`).

## Confidence-Based Actions

Based on the fact-check confidence score, the service takes different actions:

```mermaid
graph TD
    A[Discrepancy Found] --> B{Confidence?}
    B -->|≥85%| C[Auto-Action]
    B -->|60-85%| D[Queue Review]
    B -->|<60%| E[Log Only]

    C --> F{Verdict?}
    F -->|off_topic spam| G[Auto-Hide Message]
    F -->|original was 'uncertain'| H[Auto-Reclassify]
    F -->|other topic flip| D

    style C fill:#e1ffe1
    style D fill:#fff4e1
    style E fill:#e1e1e1
```

### Action Thresholds

| Threshold | Action | Description |
|-----------|--------|-------------|
| ≥85% | **Auto-Action** | Auto-hide for confident `off_topic`; auto-reclassify only when the original topic was `uncertain`; any other confident disagreement is queued for review |
| 60-85% | **Queue Review** | Sets `needs_human_review=TRUE` |
| <60% | **Log Only** | Records discrepancy for analysis |

!!! note "Why confident topic flips are not applied"
    The auditor shares the classifier's prompt, so a confident topic flip is usually a boundary call (e.g. movements vs locations). An audit found hundreds of such flips had been applied, so `audit_action()` in `fact_check.py` now only replaces an `uncertain` original automatically.

### Auto-Hide (Off-Topic Detection)

When fact-check detects `spam_type='off_topic'` with high confidence:

```python
# Message is hidden and marked as spam
UPDATE messages SET
    is_hidden = TRUE,
    hidden_reason = 'fact_check: off_topic (confidence 0.92)',
    is_spam = TRUE,
    spam_type = 'off_topic'
WHERE id = :message_id
```

### Auto-Reclassify (Uncertain Originals)

When the original topic was `uncertain` and fact-check returns a different topic with high confidence:

```python
# Topic is updated to fact-check result
UPDATE messages SET osint_topic = :new_topic WHERE id = :message_id
```

### Audit Trail

All automated actions are logged to `admin_audit_log`:

```python
INSERT INTO admin_audit_log (
    kratos_identity_id,  -- SYSTEM_UUID for automated actions
    action,              -- e.g. 'fact_check_auto_reclassify', 'fact_check_auto_hide' (see fact_check.py)
    resource_type,       -- 'message'
    resource_id,         -- message ID
    details              -- JSON with original/new values, confidence, reasoning
)
```

## Classification Council

Besides random sampling, the worker casts the auditor vote of the **classification council** (`shared/python/ai/council.py`). When a post is sent back through the pipeline with `council=true` and the current prompt disagrees with what the platform shows (outcome: spam / off_topic / low_value / archive, plus topic for archived posts), nothing is applied: the new decision is logged in `decision_log` as `council_pending`. Each cycle, `fact_check.py` selects these first, re-classifies the post on the audit lane, and the majority of three decides:

| Auditor says | `decision_log.verification_status` | Result |
|--------------|-----------------------------------|--------|
| Same as the new decision | `council_approved` | Decision reprocessor sends the post back; the processor replays the stored decision (no further LLM call) |
| Same as the current state | `council_rejected` | Post keeps its state |
| All three differ | `council_human` | Human review queue, post unchanged |

Council votes run unified mode only. Concurrency is `FACT_CHECK_COUNCIL_CONCURRENCY` (admin runtime key `fact_check_council_concurrency`, default 2, range 1-8); a post the auditor cannot answer is dropped from the head of the queue after 3 batches. Only disagreements that change whether the post is shown are voted on; the rest are applied directly.

## Configuration

### Environment Variables

Defaults are in `docker-compose.yml` (service `enrichment-fact-check`); the gateway variables are in [`.env.example`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/.env.example). Operator view: [LLM Gateway and Providers](../../operator-guide/external-llm.md).

```bash
# Provider and lane
LLM_PROVIDER=openai                   # or ollama
LLM_GATEWAY_URL=...
LLM_GATEWAY_AUDIT_API_KEY=...         # audit lane key (falls back to LLM_GATEWAY_API_KEY)
FACT_CHECK_MODEL=osint-audit          # gateway alias (a reasoning model); compose default is qwen2.5:3b (Ollama option only)
FACT_CHECK_NUM_PREDICT=4096           # reasoning tokens count: keep >= 4096 (compose default 500 suits Ollama)

# Classifier Mode
FACT_CHECK_CLASSIFIER_MODE=unified    # "unified" or "modular"

# Sampling
FACT_CHECK_LOOKBACK_DAYS=7            # only recent LLM-classified messages (not 'pending')
FACT_CHECK_SAMPLE_SIZE=50             # Messages per cycle
FACT_CHECK_PER_TOPIC=3                # Max per topic (compose default; code default is 5)
FACT_CHECK_COUNCIL_CONCURRENCY=       # empty = runtime default (2); admin page overrides

# Action Thresholds
FACT_CHECK_AUTO_ACTION_THRESHOLD=0.85 # Auto-action confidence
FACT_CHECK_HUMAN_REVIEW_THRESHOLD=0.60 # Queue review confidence

# Pacing (a thinking audit costs about as much as two batch streams)
FACT_CHECK_TIME_BUDGET_SECONDS=600    # Max processing time per cycle
FACT_CHECK_BATCH_SIZE=10              # Messages per batch
FACT_CHECK_CYCLE_INTERVAL_SECONDS=300 # Wait between cycles

# Ollama option (LLM_PROVIDER=ollama)
FACT_CHECK_OLLAMA_HOST=http://ollama:11434
FACT_CHECK_OLLAMA_API_KEY=
FACT_CHECK_OLLAMA_EXTERNAL_MODE=false
```

### Compose profiles

The service starts with the `enrichment-standard`, `enrichment-full` or `llm-gateway` profile. It sends the same user prompt and JSON schema as the processor and injects the same military slang glossary. Gateway status handling (429 backoff, 401/403 lane pause) is described in [LLM_GATEWAY.md](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docs/architecture/LLM_GATEWAY.md).

---

## Database Schema

### classification_fact_checks Table

```sql
CREATE TABLE classification_fact_checks (
    id BIGSERIAL PRIMARY KEY,
    message_id BIGINT NOT NULL REFERENCES messages(id) ON DELETE CASCADE UNIQUE,

    -- Original classification (from processor)
    original_topic VARCHAR(50),
    original_is_spam BOOLEAN,
    original_spam_type VARCHAR(50),
    original_confidence NUMERIC(5,2),
    original_prompt_version INTEGER,

    -- Fact-check classification
    factcheck_topic VARCHAR(50),
    factcheck_is_spam BOOLEAN,
    factcheck_spam_type VARCHAR(50),
    factcheck_confidence NUMERIC(5,2),
    factcheck_prompt_version INTEGER,
    factcheck_model VARCHAR(100),
    factcheck_reasoning TEXT,

    -- Comparison results
    classification_match BOOLEAN NOT NULL,
    topic_match BOOLEAN NOT NULL,
    spam_match BOOLEAN NOT NULL,
    discrepancy_type VARCHAR(50),  -- 'topic_mismatch', 'spam_mismatch', 'both_mismatch'

    -- Review tracking
    human_reviewed BOOLEAN DEFAULT FALSE,
    human_correct_topic VARCHAR(50),
    human_correct_is_spam BOOLEAN,
    human_correct_spam_type VARCHAR(50),
    human_notes TEXT,
    reviewed_by VARCHAR(100),
    reviewed_at TIMESTAMP WITH TIME ZONE,

    -- Action tracking
    action_taken VARCHAR(50),  -- 'logged_only', 'auto_reclassify', 'auto_hide', 'queued_review', 'council_*'
    action_taken_at TIMESTAMP WITH TIME ZONE,

    -- Metrics
    processing_time_ms INTEGER,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Partial index for pending review queries
CREATE INDEX idx_fact_checks_human_review ON classification_fact_checks
    (human_reviewed) WHERE human_reviewed = FALSE AND classification_match = FALSE;
```

## Admin API

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/admin/fact-check/stats` | Dashboard statistics |
| GET | `/api/admin/fact-check/pending` | Pending review queue |
| GET | `/api/admin/fact-check/reviewed` | Already reviewed items |
| GET | `/api/admin/fact-check/all` | All fact-checks (paginated) |
| POST | `/api/admin/fact-check/{fact_check_id}/approve` | Accept the fact-check classification (applies it to the message) |
| POST | `/api/admin/fact-check/{fact_check_id}/reject` | Keep the original classification |
| POST | `/api/admin/fact-check/{message_id}/unhide` | Unhide a message hidden by auto-hide |
| DELETE | `/api/admin/fact-check/{message_id}` | Delete a message |
| POST | `/api/admin/fact-check/bulk-approve` | Bulk approve fact-checks |
| POST | `/api/admin/fact-check/bulk-delete` | Bulk delete |

All endpoints require an admin user.

### Stats Response

```json
{
  "total_checked": 1500,
  "pending_review": 45,
  "discrepancies_total": 78,
  "topic_mismatches": 52,
  "spam_mismatches": 26,
  "accuracy_rate": 94.8,
  "discrepancies_by_type": {
    "topic_mismatch": 52,
    "spam_mismatch": 23,
    "both_mismatch": 3,
    "match": 1422
  }
}
```

### Pending Review Response

```json
{
  "items": [
    {
      "id": 123,
      "message_id": 45678,
      "content_preview": "Breaking: Ukrainian forces...",
      "channel_name": "War Monitor",
      "original_topic": "general",
      "factcheck_topic": "combat",
      "factcheck_confidence": 0.78,
      "factcheck_reasoning": "Message describes active military engagement...",
      "discrepancy_type": "topic_mismatch",
      "created_at": "2025-12-29T10:30:00Z"
    }
  ],
  "total": 45,
  "page": 1,
  "page_size": 50
}
```

## Metrics

### Prometheus Endpoint

The `enrichment-fact-check` worker serves metrics on port **9203** (`METRICS_PORT` in compose; the worker's built-in fallback is 9197):

```bash
curl http://localhost:9203/metrics
```

### Key Metrics

The worker uses the shared enrichment metrics (`services/enrichment/src/metrics.py`); there are no fact-check-specific series. Per-task labels use `task="fact_check"`.

```prometheus
enrichment_messages_processed_total{task="fact_check"}
enrichment_errors_total{task="fact_check", error_type="..."}
enrichment_task_duration_seconds{task="fact_check"}
llm_requests_total{task="fact_check", model="...", status="..."}
```

Discrepancy rates and actions come from SQL on `classification_fact_checks` (`classification_match`, `discrepancy_type`, `action_taken`) or the `/stats` endpoint.

## Running Locally

```bash
# Start the worker (profiles: enrichment-standard, enrichment-full, enrichment or llm-gateway)
docker-compose --profile llm-gateway up -d enrichment-fact-check
docker-compose logs -f enrichment-fact-check

# Outside Docker (from services/enrichment, with the compose env exported)
python -m src.workers.fact_check_worker
```

Verify: `curl -s localhost:9203/metrics | grep 'task="fact_check"'` and `SELECT COUNT(*) FROM classification_fact_checks;`.

## Troubleshooting

### No Work to Do

**Symptom**: Worker logs "No more work to do" immediately

**Cause**: All classified messages have been fact-checked

**Solution**: Wait for new messages to be classified, or reset fact-check data:

```sql
-- Check unprocessed messages
SELECT COUNT(*) FROM messages
WHERE osint_topic IS NOT NULL
  AND id NOT IN (SELECT message_id FROM classification_fact_checks);

-- Reset for fresh fact-checking (careful!)
TRUNCATE classification_fact_checks;
```

### High Discrepancy Rate

**Symptom**: Accuracy rate drops below 90%

**Possible Causes**:

1. **Prompt drift**: Processor and fact-check using different prompt versions
2. **Model mismatch**: Different models have different classification tendencies
3. **Edge cases**: Certain topics/content types classify inconsistently

**Investigation**:

```sql
-- Check discrepancy distribution by topic
SELECT original_topic, factcheck_topic, COUNT(*)
FROM classification_fact_checks
WHERE classification_match = FALSE
GROUP BY original_topic, factcheck_topic
ORDER BY COUNT(*) DESC;

-- Check by confidence range
SELECT
    CASE
        WHEN factcheck_confidence >= 0.85 THEN 'high'
        WHEN factcheck_confidence >= 0.60 THEN 'medium'
        ELSE 'low'
    END as confidence_band,
    COUNT(*)
FROM classification_fact_checks
WHERE classification_match = FALSE
GROUP BY 1;
```

### LLM Timeout or Empty Answers

**Symptom**: `ReadTimeout` errors, or answers cut off with no JSON

**Cause**: the gateway lane is busy or paused, or `FACT_CHECK_NUM_PREDICT` is too low for a reasoning alias (reasoning tokens count toward the limit)

**Solution**: keep `FACT_CHECK_NUM_PREDICT` at 4096 or higher, check the lane (see [Troubleshooting](../../operator-guide/troubleshooting.md#llm-gateway-problems)), and lengthen `FACT_CHECK_CYCLE_INTERVAL_SECONDS` if the audit competes with other lanes. With Ollama on CPU, raise `OLLAMA_TIMEOUT`.

### Pending Review Queue Growing

**Symptom**: `pending_review` count keeps increasing

**Solution**: Review items in admin UI or adjust thresholds:

```bash
# Lower auto-action threshold to auto-resolve more
FACT_CHECK_AUTO_ACTION_THRESHOLD=0.75

# Or increase review threshold to log more instead
FACT_CHECK_HUMAN_REVIEW_THRESHOLD=0.70
```

## Integration with Processor

The fact-check service complements the [Processor Service](processor.md) by validating its classifications:

```mermaid
graph TD
    A[Telegram Message] --> B[Processor]
    B -->|Classify| C[messages.osint_topic]
    C --> D[Fact-Check Sampler]
    D --> E[LLM Re-classify]
    E --> F{Match?}
    F -->|No| G[Action/Review]
    G -->|Update| C

    style B fill:#e1ffe1
    style E fill:#ffe1e1
```

**Key Integration Points**:

1. **Same Prompt**: Unified mode uses identical `message_classification` prompt
2. **Same Schema**: Uses the processor's JSON schema and slang glossary; the model alias differs (`FACT_CHECK_MODEL`, the audit lane)
3. **Feedback Loop**: Auto-reclassifications update the original classification

## See Also

- [Processor Service](processor.md) - Real-time message classification
- [Enrichment Service](enrichment.md) - Background task orchestration
- [LLM Integration](../llm-integration.md) - Prompt design and LLM usage
- [Database Tables](../../reference/database-tables.md) - Schema reference
