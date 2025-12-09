# Database Migrations Guide

Guide for modifying the database schema in the OSINT Intelligence Platform.

---

## Philosophy: No Alembic

The platform uses `init.sql` as the single source of truth. **No Alembic migrations.**

### Why?

1. **Simplicity**: One file defines complete schema
2. **Self-hosted**: Full control over deployment timing
3. **Clean testing**: Always start from known state
4. **Predictable**: No migration ordering issues
5. **Readable**: Entire schema visible in one place

### When This Works Well

- Self-hosted deployments with controlled update windows
- Teams comfortable with SQL
- Projects where data can be re-imported if needed

### When This Is Challenging

- Zero-downtime requirements (requires manual migration scripts)
- Frequent schema changes with production data
- Multiple deployment environments with different schema versions

---

## Schema Location

**Source of truth**: `infrastructure/postgres/init.sql`

This file:
- Creates all tables, indexes, constraints
- Defines all functions and triggers
- Inserts seed data (prompts, slang, etc.)
- ~3000+ lines (complete schema)

---

## Making Schema Changes

### Step 1: Edit init.sql

```sql
-- Add new column
ALTER TABLE messages ADD COLUMN new_field TEXT;

-- Or in CREATE TABLE (preferred for new tables)
CREATE TABLE IF NOT EXISTS my_new_table (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Add index
CREATE INDEX IF NOT EXISTS idx_messages_new_field
ON messages(new_field);
```

### Step 2: Test with Clean Rebuild

```bash
# Stop services
docker-compose down

# Remove database volume (DESTRUCTIVE!)
docker volume rm osint-intelligence-platform_postgres_data

# Rebuild and start
docker-compose up -d

# Verify schema
docker-compose exec postgres psql -U osint_user -d osint_platform \
  -c "\d messages"
```

### Step 3: Update ORM Models

Edit `shared/python/models/` to match:

```python
# shared/python/models/message.py
class Message(Base):
    __tablename__ = "messages"

    id = Column(Integer, primary_key=True)
    # ... existing columns ...
    new_field = Column(String, nullable=True)  # Add new column
```

### Step 4: Verify ORM Compatibility

```bash
# Run a simple test to verify ORM works
docker-compose exec api python -c "
from shared.python.models import Message
print('ORM loaded successfully')
"
```

---

## Data Preservation Strategies

When you need to preserve existing data:

### Strategy 1: Export/Import

For additive changes (new columns, new tables):

```bash
# 1. Export data
docker-compose exec postgres pg_dump -U osint_user \
  --data-only \
  osint_platform > backup-$(date +%Y%m%d).sql

# 2. Recreate database with new schema
docker-compose down
docker volume rm osint-intelligence-platform_postgres_data
docker-compose up -d postgres
sleep 10  # Wait for init

# 3. Import data (may need column adjustments)
docker-compose exec -T postgres psql -U osint_user -d osint_platform \
  < backup-$(date +%Y%m%d).sql
```

### Strategy 2: Manual ALTER

For simple changes, skip the volume wipe:

```bash
# Connect to database
docker-compose exec postgres psql -U osint_user -d osint_platform

# Add column with default
ALTER TABLE messages ADD COLUMN new_field TEXT DEFAULT '';

# Add index (CONCURRENTLY = no lock)
CREATE INDEX CONCURRENTLY idx_messages_new_field
ON messages(new_field);

# Exit
\q
```

**Then update init.sql to match** for future deployments.

### Strategy 3: Migration Script

For complex changes, write a Python migration:

```python
# scripts/migrate_add_new_field.py
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy import text

DATABASE_URL = "postgresql+asyncpg://osint_user:password@localhost:5432/osint_platform"

async def migrate():
    engine = create_async_engine(DATABASE_URL)

    async with AsyncSession(engine) as session:
        # Add new column
        await session.execute(text("""
            ALTER TABLE messages
            ADD COLUMN IF NOT EXISTS new_field TEXT
        """))

        # Backfill data
        await session.execute(text("""
            UPDATE messages
            SET new_field = 'default'
            WHERE new_field IS NULL
        """))

        await session.commit()
        print("Migration complete")

if __name__ == "__main__":
    asyncio.run(migrate())
```

Run:
```bash
docker-compose exec api python scripts/migrate_add_new_field.py
```

---

## Common Patterns

### Adding a Column

```sql
-- In init.sql (for new deployments)
CREATE TABLE messages (
    id SERIAL PRIMARY KEY,
    -- existing columns...
    new_field TEXT  -- Add here
);

-- Manual migration (for existing data)
ALTER TABLE messages ADD COLUMN new_field TEXT;
```

### Adding an Index

```sql
-- In init.sql
CREATE INDEX IF NOT EXISTS idx_messages_new_field
ON messages(new_field);

-- Manual (no downtime)
CREATE INDEX CONCURRENTLY idx_messages_new_field
ON messages(new_field);
```

### Adding a Table

```sql
-- In init.sql
CREATE TABLE IF NOT EXISTS my_table (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    message_id INTEGER REFERENCES messages(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_my_table_message_id
ON my_table(message_id);
```

### Removing a Column

```sql
-- Manual (existing deployments)
ALTER TABLE messages DROP COLUMN old_field;

-- Update init.sql: remove column from CREATE TABLE
```

### Renaming a Column

```sql
-- Manual migration
ALTER TABLE messages RENAME COLUMN old_name TO new_name;

-- Update init.sql to use new name
```

---

## Testing Schema Changes

### Local Testing Workflow

```bash
# 1. Make changes to init.sql

# 2. Rebuild database
docker-compose down
docker volume rm osint-intelligence-platform_postgres_data
docker-compose up -d postgres

# 3. Verify schema
docker-compose exec postgres psql -U osint_user -d osint_platform -c "\d messages"

# 4. Run application tests
docker-compose exec api pytest

# 5. Test manually
docker-compose up -d
curl http://localhost:8000/api/messages?limit=5
```

---

## Rollback Procedures

### If Schema Change Breaks Production

```bash
# 1. Stop application
docker-compose stop api processor enrichment

# 2. Restore from backup
docker-compose exec -T postgres psql -U osint_user -d osint_platform \
  < backup-YYYYMMDD.sql

# 3. Revert code changes
git checkout HEAD~1 -- infrastructure/postgres/init.sql
git checkout HEAD~1 -- shared/python/models/

# 4. Restart
docker-compose up -d
```

### If Volume Wipe Was Premature

```bash
# Restore from most recent backup
docker-compose down
docker volume rm osint-intelligence-platform_postgres_data
docker-compose up -d postgres
docker-compose exec -T postgres psql -U osint_user -d osint_platform \
  < backup-YYYYMMDD.sql
docker-compose up -d
```

---

## Best Practices

### DO

- ✅ Test schema changes locally before production
- ✅ Backup database before any change
- ✅ Use `IF NOT EXISTS` for idempotent init.sql
- ✅ Add `ON DELETE CASCADE` for foreign keys
- ✅ Create indexes for commonly queried columns
- ✅ Update ORM models to match schema
- ✅ Update init.sql after manual ALTER commands

### DON'T

- ❌ Edit production database without backup
- ❌ Remove columns without deprecation period
- ❌ Create indexes without `CONCURRENTLY` in production
- ❌ Forget to update init.sql after manual changes
- ❌ Mix ORM and raw SQL for the same operation

---

## Backup Schedule

For production:

```bash
# Daily backup script
#!/bin/bash
BACKUP_DIR="/backups/postgres"
DATE=$(date +%Y%m%d_%H%M)

docker-compose exec -T postgres pg_dump -U osint_user osint_platform \
  | gzip > "${BACKUP_DIR}/osint_platform_${DATE}.sql.gz"

# Keep 7 days of backups
find "${BACKUP_DIR}" -name "*.sql.gz" -mtime +7 -delete
```

---

## Related Documentation

- [Database Tables Reference](../reference/database-tables.md) - Schema documentation
- [Upgrades Guide](../operator-guide/upgrades.md) - Production upgrade procedures
- [Backup & Restore](../operator-guide/backup-restore.md) - Backup procedures
