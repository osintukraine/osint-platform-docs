# Database Migrations Guide

Guide for modifying the database schema in the OSINT Intelligence Platform.

---

## Philosophy: Hybrid Migration Strategy

The platform uses a **hybrid approach** combining the best of both worlds:

- **`init.sql`** - Single source of truth for fresh deployments
- **SQL migrations** - Incremental changes for production systems

### Why Hybrid?

| Approach | Fresh Install | Production Update |
|----------|---------------|-------------------|
| init.sql only | Perfect | Requires data wipe |
| Migrations only | Complex ordering | Perfect |
| **Hybrid** | **Perfect** | **Perfect** |

This mirrors industry standards used by GitLab, Mastodon, and other mature platforms.

### Benefits

1. **Fresh installs**: Clean, fast bootstrap from `init.sql`
2. **Production updates**: Safe incremental migrations without data loss
3. **Schema tracking**: `schema_migrations` table tracks applied changes (keyed by migration file name)
4. **Predictable**: No ORM magic, pure SQL you can review
5. **Rollback**: Each migration includes DOWN instructions

---

## Schema Locations

### Source of Truth

**`infrastructure/postgres/init.sql`**

- Creates all tables, indexes, constraints
- Defines all functions and triggers
- Inserts seed data (prompts, slang, etc.)
- ~7,200 lines (complete schema)
- Used for fresh deployments

### Migration Files

**`infrastructure/postgres/migrations/`**

- `000_template.sql` - Template for new migrations
- `001_*.sql` ... `058_*.sql` (and counting) - Incremental migrations. Numeric prefixes are not unique (there are several `002_*` and `012_*` files), so the *file name* is the identity
- `README.md` - Migration workflow documentation

### Migration Tracking

**`schema_migrations` table**

```sql
CREATE TABLE schema_migrations (
    version VARCHAR(50) PRIMARY KEY,      -- file name without .sql, e.g. '058_messages_pending_enrichment_indexes'
    description TEXT NOT NULL,            -- Human-readable description
    applied_at TIMESTAMPTZ DEFAULT NOW(), -- When applied
    checksum VARCHAR(64),                 -- Optional integrity check
    applied_by VARCHAR(100)               -- Who ran it
);
```

`scripts/migrate.sh` records each migration it applies, using the file name stem as `version`. Legacy rows hold only the numeric prefix (`'010'`) and still count as applied. Recent migrations therefore do not insert into `schema_migrations` themselves.

---

## Choosing Your Workflow

### Fresh Deployment (New Install)

Use `init.sql` directly - no migrations needed:

```bash
docker-compose up -d postgres
# init.sql runs automatically, schema_migrations seeded with version '000'

# init.sql already contains every migration: mark the files as applied,
# otherwise migrate.sh would try to replay them
./scripts/migrate.sh --baseline
```

### Production Update (Existing Data)

Use the migration system:

```bash
# Check what's pending
./scripts/migrate.sh --dry-run

# Apply migrations
./scripts/migrate.sh

# Verify
./scripts/migrate.sh --status
```

---

## Creating a Migration

### Step 1: Copy the Template

```bash
cd infrastructure/postgres/migrations
# Take the next free number: ls | tail -3  (currently 058, so use 059)
cp 000_template.sql 059_add_user_preferences.sql
```

### Step 2: Edit the Migration

```sql
-- ============================================================================
-- Migration: 059 - Add user preferences table
-- ============================================================================
-- Date: 2026-10-28
-- Author: your name
--
-- Description:
--   Adds user_preferences table for storing UI preferences per user.
--
-- Prerequisites:
--   - None
--
-- Rollback:
--   See DOWN section at bottom of file
-- ============================================================================

-- ============================================================================
-- UP MIGRATION
-- ============================================================================

BEGIN;

CREATE TABLE IF NOT EXISTS user_preferences (
    id SERIAL PRIMARY KEY,
    user_id TEXT NOT NULL UNIQUE,
    theme VARCHAR(20) DEFAULT 'system',
    notifications_enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_user_preferences_user_id
ON user_preferences(user_id);

-- No INSERT INTO schema_migrations needed: scripts/migrate.sh records the file name

COMMIT;

-- ============================================================================
-- DOWN MIGRATION (Rollback)
-- ============================================================================
-- Run these statements manually to rollback this migration.
-- WARNING: Data loss may occur. Test in staging first.
--
-- BEGIN;
--
-- DROP TABLE IF EXISTS user_preferences;
--
-- DELETE FROM schema_migrations WHERE version = '059_add_user_preferences';
--
-- COMMIT;
```

### Step 3: Update init.sql

**Important**: After creating a migration, also add the changes to `init.sql`:

```sql
-- In infrastructure/postgres/init.sql
CREATE TABLE IF NOT EXISTS user_preferences (
    id SERIAL PRIMARY KEY,
    user_id TEXT NOT NULL UNIQUE,
    theme VARCHAR(20) DEFAULT 'system',
    notifications_enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_user_preferences_user_id
ON user_preferences(user_id);
```

This ensures fresh deployments get the new schema automatically.

### Step 4: Update ORM Models

Edit `shared/python/models/` to match:

```python
# shared/python/models/user_preferences.py
from datetime import datetime

from sqlalchemy import Boolean, DateTime, String, func
from sqlalchemy.orm import Mapped, mapped_column

from .base import Base


class UserPreferences(Base):
    __tablename__ = "user_preferences"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[str] = mapped_column(String, unique=True)
    theme: Mapped[str] = mapped_column(String(20), default="system")
    notifications_enabled: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

Then export the class from `shared/python/models/__init__.py`.

### Step 5: Test the Migration

```bash
# Test on fresh install (rebuild)
docker-compose down
docker volume rm osint-intelligence-platform_postgres_data
docker-compose up -d postgres
./scripts/migrate.sh --status  # Should show only '000'
./scripts/migrate.sh --baseline  # Mark existing files as applied

# Test migration path (simulate production)
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "DELETE FROM schema_migrations WHERE version = '059_add_user_preferences'"
./scripts/migrate.sh --dry-run  # Should list 059
./scripts/migrate.sh            # Apply it
./scripts/migrate.sh --status   # Should list 059 again
```

---

## Using migrate.sh

The helper script manages migrations:

```bash
# Show help
./scripts/migrate.sh --help

# Show current status
./scripts/migrate.sh --status

# Preview pending migrations (no changes)
./scripts/migrate.sh --dry-run

# Apply all pending migrations (stops at the first error: psql ON_ERROR_STOP)
./scripts/migrate.sh

# Mark every migration file as applied without running it
# (fresh init.sql installs; one-time fix for untracked production databases)
./scripts/migrate.sh --baseline
```

`scripts/deploy.sh` runs `migrate.sh` as part of a deploy. Each file is piped to `psql` as a whole, so a file may contain its own `BEGIN`/`COMMIT` or none.

### Environment Variables

```bash
DB_SERVICE=postgres            # docker-compose service name
POSTGRES_USER=osint_user       # Database user
POSTGRES_DB=osint_platform     # Database name
```

---

## Common Migration Patterns

### Adding a Column

```sql
-- UP
BEGIN;

ALTER TABLE messages ADD COLUMN priority INTEGER DEFAULT 0;
CREATE INDEX IF NOT EXISTS idx_messages_priority ON messages(priority);


COMMIT;

-- DOWN (commented)
-- ALTER TABLE messages DROP COLUMN priority;
```

### Adding an Index (Production-Safe)

```sql
-- UP
BEGIN;

-- CONCURRENTLY cannot run inside a transaction: the file has no BEGIN/COMMIT
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_messages_channel_date
ON messages(channel_id, telegram_date);
```

`messages` is large and written continuously, so migrations that touch it avoid blocking writes: `CONCURRENTLY` for indexes and a `SET lock_timeout` for `ALTER TABLE`. A failed `CREATE INDEX CONCURRENTLY` leaves an *invalid* index behind, and `IF NOT EXISTS` then keeps it: `DROP INDEX CONCURRENTLY IF EXISTS` first (see `058_messages_pending_enrichment_indexes.sql`).

### Renaming a Column

```sql
-- UP
BEGIN;

ALTER TABLE messages RENAME COLUMN old_name TO new_name;


COMMIT;

-- DOWN
-- ALTER TABLE messages RENAME COLUMN new_name TO old_name;
```

### Adding a Table with Foreign Key

```sql
-- UP
BEGIN;

CREATE TABLE IF NOT EXISTS message_reactions (
    id SERIAL PRIMARY KEY,
    message_id INTEGER NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    reaction TEXT NOT NULL,
    count INTEGER DEFAULT 0,
    UNIQUE(message_id, reaction)
);

CREATE INDEX IF NOT EXISTS idx_message_reactions_message_id
ON message_reactions(message_id);


COMMIT;

-- DOWN
-- DROP TABLE IF EXISTS message_reactions;
```

### Data Backfill

```sql
-- UP
BEGIN;

-- Add column
ALTER TABLE channels ADD COLUMN member_count INTEGER;

-- Backfill with default
UPDATE channels SET member_count = 0 WHERE member_count IS NULL;

-- Add NOT NULL constraint after backfill
ALTER TABLE channels ALTER COLUMN member_count SET DEFAULT 0;
ALTER TABLE channels ALTER COLUMN member_count SET NOT NULL;


COMMIT;
```

---

## Production Deployment Workflow

### Pre-Deployment

1. **Backup database**:
   ```bash
   docker-compose exec postgres pg_dump -U osint_user osint_platform \
     | gzip > backup-$(date +%Y%m%d_%H%M).sql.gz
   ```

2. **Test migration in staging**:
   ```bash
   # On staging server
   ./scripts/migrate.sh --dry-run
   ./scripts/migrate.sh
   ```

3. **Review migration SQL**:
   ```bash
   cat infrastructure/postgres/migrations/00X_*.sql
   ```

### Deployment

```bash
# 1. Pull latest code
git pull origin master

# 2. Check pending migrations
./scripts/migrate.sh --dry-run

# 3. Apply migrations (services can stay running for additive changes)
./scripts/migrate.sh

# 4. Restart services to pick up ORM changes
docker-compose restart api processor-worker

# 5. Verify
./scripts/migrate.sh --status
curl http://localhost:8000/health
```

### Rollback

If something goes wrong:

```bash
# 1. Stop affected services
docker-compose stop api processor-worker

# 2. Run DOWN migration manually
docker-compose exec postgres psql -U osint_user -d osint_platform <<'EOF'
BEGIN;
-- Paste DOWN migration statements here
DROP TABLE IF EXISTS new_table;
DELETE FROM schema_migrations WHERE version = '<migration file name>';
COMMIT;
EOF

# 3. Revert code
git checkout HEAD~1

# 4. Restart services
docker-compose up -d
```

---

## Migration Numbering

Use the next three-digit number (check `ls infrastructure/postgres/migrations | tail`). Early migrations reused prefixes; do not.

```
057_analytics_daily_summary_unique.sql
058_messages_pending_enrichment_indexes.sql
059_your_change.sql
```

### Naming Conventions

- `add_*` - Adding new tables/columns/indexes
- `remove_*` - Removing schema elements
- `rename_*` - Renaming tables/columns
- `update_*` - Modifying existing structures
- `backfill_*` - Data migrations

---

## Testing Schema Changes

### Local Development Workflow

```bash
# 1. Make changes to init.sql AND create migration

# 2. Test fresh install
docker-compose down
docker volume rm osint-intelligence-platform_postgres_data
docker-compose up -d postgres
./scripts/migrate.sh --status

# 3. Test migration path
# Reset to simulate existing deployment
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "DELETE FROM schema_migrations WHERE version <> '000'"
./scripts/migrate.sh

# 4. Verify schema
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "\d your_table"

# 5. Run application tests (from the repo root)
pytest
```

---

## Backup Schedule

For production:

```bash
#!/bin/bash
# /etc/cron.d/osint-backup

BACKUP_DIR="/backups/postgres"
DATE=$(date +%Y%m%d_%H%M)

# Daily backup
docker-compose exec -T postgres pg_dump -U osint_user osint_platform \
  | gzip > "${BACKUP_DIR}/osint_platform_${DATE}.sql.gz"

# Keep 7 days of backups
find "${BACKUP_DIR}" -name "*.sql.gz" -mtime +7 -delete
```

---

## Best Practices

### DO

- Always create both a migration file AND update init.sql
- Use `IF NOT EXISTS` / `IF EXISTS` for idempotency
- Include DOWN migration instructions (commented)
- Test on staging before production
- Backup before applying migrations
- Use `CONCURRENTLY` for index creation in production
- Keep migrations small and focused
- Number migrations sequentially

### DON'T

- Skip updating init.sql after creating a migration
- Create migrations without testing locally
- Apply migrations without backup
- Use Alembic/ORM-generated migrations
- Create non-reversible migrations without documentation
- Combine unrelated changes in one migration

---

## Historical Note: Alembic

The platform previously used Alembic migrations during early development. These have been archived to `migrations/alembic-archive/` for historical reference. The hybrid approach (init.sql + manual SQL migrations) provides better control for self-hosted deployments.

---

## Related Documentation

- [Database Tables Reference](../reference/database-tables.md) - Schema documentation
- [Upgrades Guide](../operator-guide/upgrades.md) - Production upgrade procedures
- [Backup & Restore](../operator-guide/backup-restore.md) - Backup procedures
- [Migrations README](https://github.com/osintukraine/osint-intelligence-platform/blob/master/infrastructure/postgres/migrations/README.md) - In-repo migration docs
