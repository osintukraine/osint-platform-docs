# Upgrades Guide

Guide to upgrading the OSINT Intelligence Platform safely.

---

## Pre-Upgrade Checklist

Before any upgrade:

- [ ] **Backup database**: `./scripts/backup.sh` or manual pg_dump
- [ ] **Backup Telegram session**: `cp -r data/sessions data/sessions.backup`
- [ ] **Backup .env**: `cp .env .env.backup`
- [ ] **Note current versions**: `docker-compose ps` and `git log -1 --oneline`
- [ ] **Read CHANGELOG.md** for breaking changes
- [ ] **Schedule maintenance window** (10-30 minutes typical)
- [ ] **Notify users** if applicable

---

## Upgrade Types

### Minor Updates (Routine)

Code changes without schema modifications.

```bash
# 1. Pull latest code
git fetch origin
git pull origin master

# 2. Rebuild and restart
docker-compose build
docker-compose up -d

# 3. Verify health
docker-compose ps
curl http://localhost:8000/health
```

**Downtime**: ~2-5 minutes (container restart)

### Major Updates (Schema Changes)

Updates that modify `init.sql` require database recreation.

```bash
# 1. Backup database FIRST
docker-compose exec postgres pg_dump -U osint_user osint_platform > backup-$(date +%Y%m%d).sql

# 2. Pull latest code
git fetch origin
git pull origin master

# 3. Stop all services
docker-compose down

# 4. Remove database volume (DESTRUCTIVE)
docker volume rm osint-intelligence-platform_postgres_data

# 5. Rebuild and start
docker-compose build
docker-compose up -d

# 6. Verify database recreated
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "\dt" | head -20
```

**Downtime**: 5-15 minutes
**Data**: Lost unless migrated (see Data Migration section)

### Configuration Updates

Changes to `.env` or Docker Compose without code changes.

```bash
# 1. Edit configuration
nano .env

# 2. Restart affected services
docker-compose up -d --force-recreate
```

**Downtime**: <1 minute (graceful restart)

---

## Database Schema Workflow

The platform uses `init.sql` as the source of truth. **No Alembic migrations.**

### Why No Migrations?

1. **Simplicity**: Single file defines complete schema
2. **Self-hosted**: Full control over deployment
3. **Clean state**: Testing always starts fresh
4. **Predictable**: No migration ordering issues

### Testing Schema Changes

Before applying schema changes to production:

```bash
# 1. Create test environment
docker-compose -f docker-compose.yml -f docker-compose.test.yml up -d postgres-test

# 2. Apply new schema
docker-compose exec postgres-test psql -U osint_user -d osint_test \
  -f /infrastructure/postgres/init.sql

# 3. Verify tables created
docker-compose exec postgres-test psql -U osint_user -d osint_test \
  -c "\dt"

# 4. Test ORM compatibility (optional)
pytest tests/integration/test_database.py

# 5. Cleanup
docker-compose -f docker-compose.yml -f docker-compose.test.yml down postgres-test
```

### Schema Version Tracking

Check current schema by looking at table structure:

```sql
-- Check if new columns exist
\d+ messages

-- Check for new tables
\dt

-- Check for new indexes
\di
```

---

## Data Migration Strategies

When schema changes require data preservation:

### Strategy 1: Export/Transform/Import

For additive changes (new columns, tables):

```bash
# 1. Export relevant tables
docker-compose exec postgres pg_dump -U osint_user \
  --data-only --table=messages --table=channels \
  osint_platform > data-export.sql

# 2. Recreate database with new schema
docker-compose down
docker volume rm osint-intelligence-platform_postgres_data
docker-compose up -d postgres

# 3. Wait for schema creation
sleep 10

# 4. Import data (may need transformation)
docker-compose exec -T postgres psql -U osint_user -d osint_platform < data-export.sql
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
docker-compose down
# Point to new database in docker-compose.yml
docker-compose up -d
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
# Scale down gracefully (drain queue)
docker-compose exec redis redis-cli XLEN telegram_messages
# Wait until queue is small (<100)

# Update one worker at a time
docker-compose up -d --no-deps --build processor-worker-1
sleep 30  # Verify healthy
docker-compose up -d --no-deps --build processor-worker-2
```

### API

```bash
# API is stateless, quick restart
docker-compose up -d --no-deps --build api
```

### Listener

```bash
# Listener maintains Telegram session - restart carefully
docker-compose stop listener
docker-compose up -d --build listener

# Verify reconnected
docker-compose logs --tail=50 listener | grep "connected"
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
docker-compose build
docker-compose up -d
```

### Full Rollback (With Data)

```bash
# 1. Stop services
docker-compose down

# 2. Restore database from backup
docker volume rm osint-intelligence-platform_postgres_data
docker-compose up -d postgres
docker-compose exec -T postgres psql -U osint_user -d osint_platform < backup.sql

# 3. Revert code
git checkout <previous-commit>

# 4. Restart
docker-compose build
docker-compose up -d
```

### Emergency Procedures

If upgrade causes critical failure:

```bash
# 1. Immediate: Stop broken services
docker-compose stop

# 2. Restore from backup
git checkout <last-known-good>
docker-compose build
docker-compose up -d

# 3. Investigate logs
docker-compose logs --tail=200 > incident-$(date +%Y%m%d).log

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

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16.1  # Pin major.minor
  redis:
    image: redis:7.2-alpine
  ollama:
    image: ollama/ollama:0.1.32
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
docker-compose ps

# 2. API responding
curl http://localhost:8000/health

# 3. Messages processing
docker-compose exec redis redis-cli XLEN telegram_messages
# Should not be growing

# 4. No errors in logs
docker-compose logs --tail=100 | grep -i error

# 5. Grafana dashboards loading
# Open http://localhost:3001 and verify data

# 6. Recent messages in database
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "SELECT COUNT(*) FROM messages WHERE created_at > NOW() - INTERVAL '1 hour';"
```

---

## Scheduled Maintenance

### Weekly

```bash
# Optimize database
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "VACUUM ANALYZE;"
```

### Monthly

```bash
# Check for image updates
docker-compose pull

# Review logs for warnings
docker-compose logs --since 720h | grep -i "warning\|deprecated"
```

### Quarterly

```bash
# Full backup and restore test
./scripts/backup.sh
./scripts/restore-test.sh

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
