# Backlog Re-classification

Messages archived while the LLM was unavailable (or in the CPU era) have `osint_topic = 'pending'`: they are in the archive but have no topic or importance. The **backlog worker** works through them with the LLM, newest first, on the gateway's *batch* lane. It also re-checks messages that were flagged as spam.

Developer details: [LLM_GATEWAY.md](https://github.com/osintukraine/osint-intelligence-platform/blob/master/docs/architecture/LLM_GATEWAY.md).

## Requirements

- An LLM gateway with a batch lane: `LLM_PROVIDER=openai`, `LLM_GATEWAY_URL`, and `LLM_GATEWAY_BATCH_API_KEY` / `LLM_GATEWAY_BATCH_MODEL` (see [LLM Gateway](external-llm.md)).
- The compose profile `llm-gateway`, which starts the `backlog-worker` service.
- Migration 038 applied (`./scripts/migrate.sh`): it creates the `message_reclassifications` table.

## Turning it on

The worker does nothing until it is enabled. Use **Admin, Config, Platform Settings**; changes apply within about 30 seconds, no restart.

| Admin setting | Default | Meaning |
|---|---|---|
| `backlog_reclassify_enabled` | off | Master switch (`.env` fallback: `BACKLOG_RECLASSIFY_ENABLED`) |
| `backlog_spam_first` | on | Run the spam re-check pass before the pending pass |
| `backlog_concurrency` | 8 (range 1 to 16) | LLM calls in flight; match the gateway's batch-lane cap (`.env` fallback: `BACKLOG_CONCURRENCY`) |

## What it does

Two passes, each newest first:

1. **Spam re-check** (when `backlog_spam_first` is on). Each message flagged as spam is judged again and the verdict is applied:
    - still spam: stays spam;
    - not spam and Ukraine-relevant: `is_spam` is cleared and it returns to the feed;
    - not spam and not Ukraine-relevant: `is_spam` is cleared and the message is hidden with reason `off_topic`.
2. **Pending pass.** Messages with `osint_topic = 'pending'` get a topic and importance.

Every verdict (spam, relevance, archive decision, analysis text, model, prompt) is stored in the `message_reclassifications` table, so you can review what the LLM decided. Apart from the cases above, already archived posts are not hidden or un-hidden by this worker.

If the LLM is unavailable the worker writes nothing; the chunk is retried and skipped after three tries.

## Watching progress

```bash
docker compose logs -f backlog-worker

docker compose exec postgres psql -U osint_user -d osint_platform -c \
  "SELECT count(*) AS still_pending FROM messages WHERE osint_topic = 'pending';"

docker compose exec postgres psql -U osint_user -d osint_platform -c \
  "SELECT count(*) AS verdicts, count(*) FILTER (WHERE is_spam) AS spam FROM message_reclassifications;"
```

## Stopping it

Switch `backlog_reclassify_enabled` off in the admin page. The worker idles; nothing is rolled back. To stop the container, run `docker compose stop backlog-worker` (or remove `llm-gateway` from `COMPOSE_PROFILES`).

!!! note
    Backlog traffic shares the GPU with live classification. If realtime latency suffers, lower `backlog_concurrency`.
