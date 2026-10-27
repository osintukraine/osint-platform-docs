# Upgrades Guide

Guide to upgrading the OSINT Intelligence Platform safely.

---

## Pre-Upgrade Checklist

Before any upgrade:

- [ ] **Backup database**: `./scripts/backup.sh` or manual pg_dump
- [ ] **Backup Telegram sessions**: `cp -r telegram_sessions telegram_sessions.backup` (`backup.sh` also archives them)
- [ ] **Backup .env**: `cp .env .env.backup`
- [ ] **Note current versions**: `docker compose ps` and `git log -1 --oneline`
- [ ] **Read the commit log / release notes** for breaking changes (`git log --oneline HEAD..origin/master`)
- [ ] **Schedule maintenance window** (10-30 minutes typical)
- [ ] **Notify users** if applicable

---

## Upgrade Types

### Routine Updates

Production updates use the deploy script, which pulls the code, applies pending SQL migrations, restarts services and runs a health check.

```bash
./scripts/deploy.sh --dry-run    # preview
./scripts/deploy.sh              # pull, migrate, restart
./scripts/deploy.sh --build      # also rebuild all images first
./scripts/deploy.sh --build-pytorch                    # rebuild only the PyTorch-based services
./scripts/deploy.sh --build-service media-worker       # rebuild only named services (repeatable)
./scripts/deploy.sh --skip-health                      # skip the final health check
./scripts/deploy.sh --force-recreate                   # recreate containers even if nothing changed
./scripts/deploy.sh --quick      # pull and restart, skip migrations
```

After an update that changes `infrastructure/models/models.txt`, run `./scripts/fetch-models.sh` (see [Embedding Models](model-provisioning.md)); `deploy.sh` does not do it.

**Downtime**: a few minutes (container restart).

### Schema Changes

Schema changes ship as numbered SQL migrations in `infrastructure/postgres/migrations/`. They are applied in place and keep your data; **do not delete the database volume to upgrade**.

```bash
./scripts/migrate.sh --status    # applied and pending migrations
./scripts/migrate.sh --dry-run   # preview pending migrations
./scripts/migrate.sh             # apply them
```

Take a backup first (`./scripts/backup.sh`). Some migrations create indexes `CONCURRENTLY` and can take a while on a large `messages` table; they do not block writes. Recent migration files document their rollback SQL in the header.

If a database was created from `init.sql` and has never had migrations tracked, `./scripts/migrate.sh --baseline` marks all existing migration files as applied without running them.

### Configuration Updates

Changes to `.env` or Docker Compose without code changes.

```bash
# 1. Edit configuration
nano .env

# 2. Restart affected services
docker compose up -d --force-recreate
```

**Downtime**: <1 minute (graceful restart)

---

## Database Schema Workflow

`infrastructure/postgres/init.sql` creates the schema on a fresh install; migrations in `infrastructure/postgres/migrations/` carry existing installs forward. See [Database Migrations](../developer-guide/database-migrations.md) for how they are written.

Check what is applied:

```bash
./scripts/migrate.sh --status
```

Migrations are numbered sequentially (currently up to the 058 series). `./scripts/migrate.sh --status` is the source of truth for what your database has; read the header of any pending migration before applying it, as large index builds are called out there. Migrations that insert a new classifier prompt version leave it inactive until you activate it (see [LLM Gateway](external-llm.md#prompt-versions)).

---

## Data Migration Strategies

When schema changes require data preservation:

### Strategy 1: Export/Transform/Import

For additive changes (new columns, tables):

```bash
# 1. Export relevant tables
docker compose exec postgres pg_dump -U osint_user \
  --data-only --table=messages --table=channels \
  osint_platform > data-export.sql

# 2. Recreate database with new schema
docker compose down
docker volume ls | grep postgres_data      # note the exact volume name
docker volume rm <project>_postgres_data
docker compose up -d postgres

# 3. Wait for schema creation (init.sql)
sleep 10

# 4. Import data (may need transformation)
docker compose exec -T postgres psql -U osint_user -d osint_platform < data-export.sql
```

### Strategy 2: Parallel Database

For complex migrations:

```bash
# 1. Start second database
docker run -d --name postgres-new \
  -e POSTGRES_USER=osint_user \
  -e POSTGRES_PASSWORD=yourpassword \
  -e POSTGRES_DB=osint_platform \
  -v new_postgres_data:/var/lib/postgresql/data \
  postgres:16

# 2. Apply new schema to new database

# 3. Write migration script (Python recommended)
# - Read from old database
# - Transform as needed
# - Write to new database

# 4. Verify migration
# - Row counts match
# - Spot check data integrity

# 5. Switch over
docker compose down
# Point to new database in docker-compose.yml
docker compose up -d
```

### Strategy 3: Manual SQL Migration

For simple changes:

```sql
-- Add new column with default
ALTER TABLE messages ADD COLUMN new_field TEXT DEFAULT '';

-- Add new index
CREATE INDEX CONCURRENTLY idx_messages_new ON messages(new_field);

-- Backfill data
UPDATE messages SET new_field = computed_value WHERE new_field = '';
```

---

## Rolling Updates

For zero-downtime updates of stateless services:

### Processor Workers

```bash
# Check the queue depth (stream lag is what matters)
docker compose exec redis redis-cli XLEN telegram:messages:realtime
docker compose exec redis redis-cli XLEN telegram:messages:backfill

# processor-worker is one compose service with N replicas
# (PROCESSOR_WORKER_REPLICAS). Rebuild and recreate it; messages stay in the
# Redis stream while the workers restart and are consumed afterwards.
docker compose up -d --no-deps --build processor-worker
```

### API

```bash
# API is stateless, quick restart
docker compose up -d --no-deps --build api
```

### Listener

```bash
# Listener maintains Telegram session - restart carefully
docker compose stop listener
docker compose up -d --build listener

# Verify reconnected
docker compose logs --tail=50 listener | grep -i "connected"
```

---

## Rollback Procedures

### Quick Rollback (Code Only)

```bash
# 1. Identify previous commit
git log --oneline -10

# 2. Revert to previous version
git checkout <commit-hash>

# 3. Rebuild and restart
docker compose build
docker compose up -d
```

### Full Rollback (With Data)

```bash
# 1. Stop services
docker compose down

# 2. Restore database from backup
docker volume rm <project>_postgres_data
docker compose up -d postgres
# Restore the dump made by backup.sh (see Backup & Restore)
./scripts/restore.sh backups/<timestamp>

# 3. Revert code
git checkout <previous-commit>

# 4. Restart
docker compose build
docker compose up -d
```

### Emergency Procedures

If upgrade causes critical failure:

```bash
# 1. Immediate: Stop broken services
docker compose stop

# 2. Restore from backup
git checkout <last-known-good>
docker compose build
docker compose up -d

# 3. Investigate logs
docker compose logs --tail=200 > incident-$(date +%Y%m%d).log

# 4. Post-mortem: Document what went wrong
```

---

## Update Order of Operations

When updating multiple components:

1. **Infrastructure first** (postgres, redis, minio)
2. **Data services** (listener)
3. **Processing** (processor, enrichment)
4. **API** (api)
5. **Frontend** (frontend)
6. **Monitoring** (prometheus, grafana)

---

## Version Compatibility

### Docker Images

Pin versions in production:

The compose file already pins most third-party images (Redis, Prometheus, Grafana, Loki, exporters, Yente, MinIO). `postgres` uses the locally built `osint-platform-postgres:latest` (PostgreSQL 16 with pgvector), and `ollama` and `nocodb` use `:latest`. If you need reproducible rebuilds, pin those two in `docker-compose.yml` as well.

```yaml
services:
  ollama:
    image: ollama/ollama:<pinned-version>
```

### Python Dependencies

Check `requirements.txt` compatibility:

```bash
# Before upgrade
pip freeze > requirements.lock

# After upgrade, compare
diff requirements.lock <(pip freeze)
```

---

## Post-Upgrade Verification

After any upgrade:

```bash
# 1. All services running
docker compose ps

# 2. API responding
curl http://localhost:8000/health

# 3. Messages processing
docker compose exec redis redis-cli XLEN telegram:messages:realtime
# Should not keep growing

# 4. No errors in logs
docker compose logs --tail=100 | grep -i error

# 5. Grafana dashboards loading
# Open http://localhost:3001 and verify data

# 6. Recent messages in database
docker compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(*) FROM messages WHERE created_at > NOW() - INTERVAL '1 hour';"
```

---

## Scheduled Maintenance

### Weekly

```bash
# Optimize database
docker compose exec postgres psql -U osint_user -d osint_platform \
  -c "VACUUM ANALYZE;"
```

### Monthly

```bash
# Check for image updates
docker compose pull

# Review logs for warnings
docker compose logs --since 720h | grep -i "warning\|deprecated"
```

### Quarterly

```bash
# Full backup, then a restore test on a scratch host
./scripts/backup.sh
./scripts/restore.sh backups/<timestamp> --dry-run

# Review dependencies for security updates
pip-audit  # If installed
```

---

## Related Documentation

- [Backup & Restore](backup-restore.md) - Detailed backup procedures
- [Troubleshooting](troubleshooting.md) - Common upgrade issues
- [Configuration](configuration.md) - Environment variables
- [Database Tables Reference](../reference/database-tables.md) - Schema documentation

---

**Golden Rule**: Always backup before upgrading. Test in non-production first when possible.
