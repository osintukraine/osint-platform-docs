# Embedding Models

The embedding spam filter uses a sentence-transformer model. Since October 2026 such models are **provisioned on the host and mounted read-only**; services never download them at runtime.

Developer rationale: [LLM_GATEWAY.md](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docs/architecture/LLM_GATEWAY.md).

## How it works

- [`infrastructure/models/models.txt`](https://github.com/osintukraine/osint-intelligence-platform/blob/master/infrastructure/models/models.txt) lists each model with its Hugging Face repository and a pinned revision.
- `./scripts/fetch-models.sh` downloads them into `./data/models/<name>`. It runs on the host (it uses a throwaway Docker container), is idempotent, and skips models whose pinned revision is already present.
- Compose mounts `./data/models` at `/models` (read-only) and sets `HF_HUB_OFFLINE=1`, so a missing model fails fast instead of reaching the Hugging Face Hub.
- Services refer to a model by **path**: `EMBEDDING_SPAM_MODEL=/models/paraphrase-multilingual-MiniLM-L12-v2`. A bare repository name is not enough: sentence-transformers would still try to contact the Hub.

## Install and upgrade step

Run it on every fresh install and after changing `models.txt`, before starting the stack:

```bash
./scripts/fetch-models.sh                      # all models in models.txt
./scripts/fetch-models.sh all-MiniLM-L6-v2     # one model
ls data/models
```

After fetching a new model, recreate the services that load it (`docker compose up -d --force-recreate processor-worker processor-worker-backfill backlog-worker`, plus `backlog-enrich` / `backlog-reprocess` if you run them). Without the model files the processor cannot load the spam filter.

## Spam filter thresholds

Thresholds depend on the model, so change them together. Values calibrated on 2026-10-23 for the multilingual model:

| Setting | Value | Meaning |
|---|---|---|
| `EMBEDDING_SPAM_MODEL` | `/models/paraphrase-multilingual-MiniLM-L12-v2` | Separates spam on Russian and Ukrainian posts |
| `EMBEDDING_SPAM_THRESHOLD` (admin: `embedding_spam_threshold`) | 0.75 | Similarity at or above which a message counts as spam |
| `llm_prefilter_spam_similarity` (admin setting) | 0.80 | With an LLM: spam at or above this skips the LLM call |
| `EMBEDDING_OFF_TOPIC_THRESHOLD` | 0.70 | Off-topic (quarantine) similarity |

The English-only `all-MiniLM-L6-v2` had 7% precision against the LLM's verdicts on Russian and Ukrainian posts, so it is not suitable for the spam filter. The built-in default for `llm_prefilter_spam_similarity` is 0.90; the calibrated 0.80 is set in the admin page (**Admin, Config, Platform Settings**).

## What is not affected

Stored message embeddings and search-query embeddings (384 dimensions, local Ollama `all-minilm`) are unchanged and live in pgvector. Changing those models would mean re-embedding existing data.
