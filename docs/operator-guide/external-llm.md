# LLM Gateway and Providers

The platform's LLM work (message classification, fact-check, RSS validation, backlog re-classification) runs against one of three providers, chosen with `LLM_PROVIDER` in `.env`.

| `LLM_PROVIDER` | Use it for |
|---|---|
| `openai` | An OpenAI-compatible **LLM gateway** in front of a GPU host (LiteLLM in front of vLLM is what production runs). Recommended when you have a GPU. |
| `ollama` | Local or remote Ollama, CPU or small GPU. The default for fresh installs. |
| `none` | No LLM. Messages are decided by the embedding filter and tier rules only. |

Developer internals (client behaviour, lane rules, classification code paths) are in [LLM_GATEWAY.md](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docs/architecture/LLM_GATEWAY.md). Variable defaults are in [`.env.example`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/.env.example).

## Gateway setup

Add to `.env` (values come from whoever runs your gateway; never commit them):

```bash
LLM_PROVIDER=openai
LLM_ENABLED=true
LLM_GATEWAY_URL=<base URL of the gateway>
LLM_GATEWAY_API_KEY=<key for the realtime lane>
LLM_GATEWAY_MODEL=osint-realtime       # alias served by the gateway
LLM_GATEWAY_TIMEOUT=120                # seconds per request
```

If the gateway's hostname does not resolve inside containers (for example a private-network name), pin it:

```bash
LLM_GATEWAY_HOST=<gateway hostname>
LLM_GATEWAY_IP=<its address>
```

Compose turns these into `extra_hosts` for every service that calls the gateway. Keep the real URL, hostname and address in `.env` only.

Then add the `llm-gateway` profile:

```bash
# .env
COMPOSE_PROFILES=enrichment-lite,monitoring,auth,llm-gateway

docker compose up -d
```

The `llm-gateway` profile starts `backlog-worker`, `enrichment-fact-check` and `enrichment-rss-validation`. With it you do not need the local `ollama-batch` service.

## Lanes

The gateway caps parallelism per API key and schedules by model alias, so each kind of work gets its own key and alias (a "lane"):

| Lane | Used by | Alias variable | Key variable |
|---|---|---|---|
| realtime | `processor-worker` (live classification) | `LLM_GATEWAY_MODEL` (`osint-realtime`) | `LLM_GATEWAY_API_KEY` |
| audit | `enrichment-fact-check` | `FACT_CHECK_MODEL` (`osint-audit`) | `LLM_GATEWAY_AUDIT_API_KEY` |
| batch | `backlog-worker`, `enrichment-rss-validation` | `LLM_GATEWAY_BATCH_MODEL`, `RSS_VALIDATION_MODEL` (`osint-batch`) | `LLM_GATEWAY_BATCH_API_KEY` |

The audit and batch keys fall back to `LLM_GATEWAY_API_KEY` when unset. The aliases above are the names production uses; they must exist on your gateway.

Fact-check uses a reasoning ("thinking") alias, so keep `FACT_CHECK_NUM_PREDICT` at 4096 or higher: reasoning tokens count against the limit.

## How the platform reacts to gateway errors

| Response | Meaning | What the platform does |
|---|---|---|
| 429 | The lane is at its parallel or rate cap | Retries with backoff (1, 2, 4 s) |
| 502, 503, 504, connection error | Backend reloading or down | Retries, then falls back (below) |
| 401 | Key blocked | **Pauses that lane for 5 minutes**, sends nothing, then probes once |
| 403 | Alias not allowed for this key | Same pause as 401 |

**Fallback.** When the LLM is unavailable, the processor keeps archiving: the embedding filter's verdict replaces the classifier's, and topic and importance stay `pending` for the [backlog worker](backlog-reclassification.md) to fill in later. The backlog worker writes nothing while the LLM is unavailable.

To save GPU time, the processor also skips the LLM for obvious spam: when a message's spam similarity is at or above the admin setting `llm_prefilter_spam_similarity` (0.80 in production), the embedding filter decides alone. See [Embedding models](model-provisioning.md).

## Prompt versions

The classifier prompt lives in the `llm_prompts` table. Production runs **v12**, which asks the model for structured JSON output. Never edit a prompt in place: add a version, then flip `is_active`.

Activate v12 (migration 037 inserts it inactive) or roll back to v11:

```sql
-- activate v12
UPDATE llm_prompts SET is_active = (version = 12)
 WHERE task = 'message_classification' AND prompt_type = 'system';

-- roll back to v11
UPDATE llm_prompts SET is_active = (version = 11)
 WHERE task = 'message_classification' AND prompt_type = 'system';
```

Run it with `docker compose exec postgres psql -U osint_user -d osint_platform`. Prompt history and design notes: [LLM_PROMPTS.md](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docs/architecture/LLM_PROMPTS.md).

!!! warning
    v12 is written for a large model behind the gateway. Do not activate it with a small Ollama model.

## Using Ollama instead (no GPU)

```bash
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://ollama:11434     # or a remote Ollama
OLLAMA_MODEL=<model>
```

The local `ollama` service also serves **search-query embeddings** (`all-minilm`), so it stays in the stack even with `LLM_PROVIDER=openai`. `ollama-batch` (profiles `enrichment-standard`, `enrichment-full`) is the CPU-era batch endpoint and is not used with the gateway.

For a remote Ollama set `OLLAMA_BASE_URL`, `OLLAMA_API_KEY` and `OLLAMA_EXTERNAL_MODE=true` (adds retries). Fact-check has its own `FACT_CHECK_OLLAMA_HOST`, `FACT_CHECK_OLLAMA_API_KEY` and `FACT_CHECK_OLLAMA_EXTERNAL_MODE`. For model sizing by hardware, run `./scripts/bootstrap-hardware.sh show`.

## Checking it works

```bash
# Provider and lane settings (secrets filtered out)
grep -E '^(LLM_PROVIDER|LLM_GATEWAY_(URL|MODEL|BATCH_MODEL)|FACT_CHECK_MODEL|RSS_VALIDATION_MODEL)=' .env

# Gateway errors and lane pauses
docker compose logs --since 15m processor-worker backlog-worker enrichment-fact-check | grep -iE 'paused|401|403|429|timeout'
```

If a lane is paused or the gateway is unreachable, see [Troubleshooting](troubleshooting.md#llm-gateway-problems).
