# LLM Integration Guide

How the processor and background workers use LLMs. The developer reference lives in the platform repo and is the single source of truth; this page is a map and does not copy it.

- [LLM_GATEWAY.md](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docs/architecture/LLM_GATEWAY.md): providers, the OpenAI-compatible client, lanes and keys, status handling (429, 401/403 lane pause), the real-time classification paths, post-LLM rules enforced in code, prompt v12, background workers, embedding models, queue caps.
- [LLM_PROMPTS.md](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docs/architecture/LLM_PROMPTS.md): prompt evolution, design decisions and constraints. Read it before changing any prompt, classifier or parsing code, and update it afterwards.

Operator-facing setup is in [LLM Gateway and Providers](../operator-guide/external-llm.md).

---

## Overview

The processor's classifier (`services/processor/src/llm_classifier.py`) decides, per message, spam, Ukraine relevance, topic, importance and whether to archive. Three paths exist, chosen from runtime settings:

1. **Embedding filter only** (`realtime_use_embedding`): CPU-era mode; topic and importance stay `pending`.
2. **LLM classification, guarded by the embedding filter**: very obvious spam (similarity at or above `llm_prefilter_spam_similarity`) is decided without the LLM; if the LLM is unavailable the embedding verdict is used.
3. **No LLM**: tier-based fallback.

Some rules are enforced in code whatever the model says: spam and non-relevant posts are never archived, a spam verdict on a conflict topic is overridden, and channel tiers set how strict archiving is.

## Providers

| `LLM_PROVIDER` | Client |
|---|---|
| `openai` | `OpenAICompatClient` (`shared/python/ai/openai_compat.py`), which implements the part of `ollama.AsyncClient` the platform uses, so prompts and parsing are shared with Ollama |
| `ollama` | Ollama client |
| `none` | No LLM |

## Prompts

Prompts are rows in `llm_prompts` (`task`, `version`, `is_active`, `expected_output_format`). Never edit a row in place: insert a new version and flip `is_active`; rollback is flipping it back. Production runs v12 (migration 037), which carries a JSON schema in `expected_output_format` and so makes the classifier request structured output. The activation and rollback SQL is in [LLM Gateway and Providers](../operator-guide/external-llm.md#prompt-versions). `{{MILITARY_SLANG}}` is injected from the `military_slang` table by every caller.

## Background LLM work

| Worker | Lane | Purpose |
|---|---|---|
| `backlog-worker` | batch | Re-classifies `pending` messages and re-checks spam ([operator page](../operator-guide/backlog-reclassification.md)) |
| `enrichment-fact-check` | audit | Audits a sample of recent LLM classifications ([service page](services/fact-check.md)) |
| `enrichment-rss-validation` | batch | Validates RSS correlations ([service page](services/rss-ingestor.md)) |

## Related Documentation

- [Processor Service](services/processor.md)
- [Fact-Check Service](services/fact-check.md)
- [Environment Variables](../reference/environment-variables.md)
- [Embedding Models](../operator-guide/model-provisioning.md)
