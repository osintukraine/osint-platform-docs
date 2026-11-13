# Production Gotchas

Hard-won lessons from deploying and operating the OSINT Intelligence Platform. **Check this page before your first production deployment.**

!!! warning "Based on real production issues"
    These gotchas are based on real production issues. If you encounter something not listed here, please contribute!

---

## Quick Reference

| Issue | Symptom | Section |
|-------|---------|---------|
| HTTPS redirects fail | 307 redirects go to `http://` | [HTTPS Redirect Issues](#https-redirect-issues) |
| Frontend calls wrong API | API calls go to localhost in production | [Frontend Build Args](#frontend-build-args) |
| Need to know what's public | Which endpoints require auth? | [Public vs Authenticated](#public-vs-authenticated-endpoints) |
| New stack manager | How to use the Python tooling | [Stack Manager](#stack-manager) |
| Media archival fails silently | No files appearing in MinIO | [Storage Boxes](#storage-boxes-default-entry) |
| Session file not found | Telegram auth fails after upgrade | [Session Files](#session-file-location-and-names) |

---

## HTTPS Redirect Issues

### Problem

When deploying behind a reverse proxy (Caddy) with HTTPS, FastAPI's automatic trailing slash redirects generate `http://` URLs instead of `https://`, breaking the redirect chain.

### Symptoms

- Browser shows "mixed content" warnings
- API calls fail silently or show CORS errors
- `curl -I https://yoursite.com/api/channels` shows:
  ```
  HTTP/2 307
  location: http://yoursite.com/api/channels/
  ```
  Note the `http://` in the Location header.

### Root Cause

FastAPI doesn't automatically respect the `X-Forwarded-Proto` header that Caddy sends. Without this, FastAPI assumes all requests are HTTP and generates HTTP redirect URLs.

### Solution

This is **already fixed** in the codebase. The fix adds `ProxyHeadersMiddleware` from uvicorn:

```python
# services/api/src/main.py
from uvicorn.middleware.proxy_headers import ProxyHeadersMiddleware

app.add_middleware(ProxyHeadersMiddleware, ...)  # reads X-Forwarded-Proto / -For only
```

**If you're on the latest code, you don't need to do anything.**

### Prevention

- Always pull the latest code before deploying
- Don't remove `ProxyHeadersMiddleware` from `main.py`
- Test redirects with `curl -I` before going live

---

## Frontend Build Args

### Problem

Next.js `NEXT_PUBLIC_*` environment variables are baked into the JavaScript bundle at **build time**, not runtime. If you only set them in `environment:`, the frontend will use wrong URLs.

### Symptoms

- Frontend makes API calls to `localhost:8000` instead of your production URL
- Network tab shows requests going to the wrong host
- Console errors about CORS or connection refused

### Root Cause

```yaml
# docker-compose.yml

# ❌ WRONG - These only affect server-side rendering, not client-side
frontend:
  environment:
    NEXT_PUBLIC_API_URL: https://example.com

# ✅ CORRECT - Must be in build.args for client-side JavaScript
frontend:
  build:
    args:
      NEXT_PUBLIC_API_URL: ${NEXT_PUBLIC_API_URL}
      NEXT_PUBLIC_BASE_URL: ${NEXT_PUBLIC_BASE_URL}
      NEXT_PUBLIC_MINIO_URL: ${NEXT_PUBLIC_MINIO_URL}
  environment:
    # These are for SSR only
    API_URL: http://api:8000
```

### Solution

1. Set `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_BASE_URL` and `NEXT_PUBLIC_MINIO_URL` in `.env` (the compose file passes them as `build.args`)
2. Rebuild the frontend: `docker compose build frontend`
3. Restart: `docker compose up -d frontend`

### Prevention

- Always check the `NEXT_PUBLIC_*` values in `.env` when deploying to a new domain
- After changing `NEXT_PUBLIC_*` values, you **must rebuild**, not just restart
- Test by checking Network tab in browser DevTools

---

## Public vs Authenticated Endpoints

### Overview

The platform uses a "public read, authenticated write" security model. Most read operations are public to allow embedding and sharing. Write operations require authentication.

### Public Endpoints (No Auth Required)

These endpoints allow anonymous access for GET requests:

| Endpoint | Purpose |
|----------|---------|
| `/api/timeline` | Message timeline |
| `/api/channels` | Channel list and details |
| `/api/messages` | Message content and media |
| `/api/events` | Detected events |
| `/api/map/*` | Map data (GeoJSON, clusters, heatmap) |
| `/api/search` | Full-text and semantic search |
| `/api/analytics` | Dashboard statistics |
| `/api/about/*` | Platform information |
| `/api/rss/*` | RSS feeds |

### Authenticated Endpoints

These require a valid session (via Ory Kratos):

| Operation | Example |
|-----------|---------|
| POST/PUT/DELETE on any endpoint | Creating, updating, deleting |
| `/api/admin/*` | Admin operations |
| Channel management | Adding/removing channels |
| User preferences | Saved searches, alerts |

### Security Considerations

- **Sensitive data**: The platform assumes all archived content is meant to be publicly accessible. Don't archive private channels if you need access control.
- **Rate limiting**: Public endpoints have rate limits to prevent abuse
- **Write protection**: All mutations require authentication, preventing unauthorized changes

See also: [Authorization Guide](../security-guide/authorization.md)

---

## Stack Manager

### Overview

The platform includes a Python-based stack manager (`scripts/stack_manager/`) that provides better control over Docker Compose operations.

### Basic Usage

```bash
# Start core services
./scripts/stack-manager-py.sh start

# Stop services
./scripts/stack-manager-py.sh stop --all

# View logs (-f to stream, Ctrl+C safe)
./scripts/stack-manager-py.sh logs -f api
./scripts/stack-manager-py.sh logs -f processor-worker

# Check hardware tier
./scripts/stack-manager-py.sh hardware show
```

### Key Features

- **Streaming logs**: Real-time log output with proper Ctrl+C handling
- **Profile-aware**: Automatically discovers services based on active profiles
- **Production overlay**: `-e production` adds `docker-compose.production.yml`
- **Hardware detection**: Detects the hardware tier and writes sizing defaults into `.env`

### Multi-File Compose Support

For production deployments with multiple compose files:

```bash
./scripts/stack-manager-py.sh -e production start
```

See also: [Stack Manager Reference](stack-manager.md)

---

## Storage Boxes Default Entry

### Problem

Media archival requires a default storage box entry in the database. Without it, media archival fails silently.

### Symptoms

- Messages are processed but media URLs show Telegram CDN (not your MinIO)
- No files appearing in MinIO storage
- No errors in processor logs (fails silently)

### Root Cause

The `storage_boxes` table must have at least one row (the placeholder row with id `default`). `init.sql` and migration 017 create it for new installs, but older installations may be missing it. There is no `is_default` column; boxes are chosen by `is_active`, `is_full`, `is_readonly` and `priority`. Update the placeholder host values for your real storage, see [Hetzner Storage](hetzner-storage.md).

### Solution

Check if you have a default storage box:

```sql
-- Connect to database
docker compose exec postgres psql -U osint_user -d osint_platform

-- Check for default storage box
SELECT id, account_region, is_active, is_full, is_readonly, priority FROM storage_boxes;
```

If no rows are returned, run `./scripts/migrate.sh` or insert the default row:

```sql
INSERT INTO storage_boxes (id, hetzner_host, hetzner_user, hetzner_port, mount_path,
                           minio_endpoint, minio_port, capacity_gb, priority, account_region, is_active)
VALUES ('default', 'your-storagebox.de', 'u000000', 23, '/mnt/hetznerstorage',
        'minio', 9000, 5000, 100, 'eu', true)
ON CONFLICT (id) DO NOTHING;
```

### Prevention

- Run migrations after upgrading: `./scripts/migrate.sh`
- For fresh installs, this is handled automatically

---

## Session File Location and Names

### Problem

Telegram session files live in the `telegram_sessions/` directory of the repository (a bind-mounted volume, `/app/sessions` inside containers). Sessions copied from an older layout, or from another machine, are not picked up, and one session file shared by two processes gets revoked by Telegram (`AuthKeyDuplicated`).

### Symptoms

- Telegram authentication fails after upgrade
- "Session file not found" errors in listener logs
- Prompted to re-authenticate even though a session exists
- `AuthKeyDuplicated` after copying a session to a second host

### Solution

Each service role opens its own file: `listener`, `discovery`, `enrichment`, `analytics` for the default account, and `<role>-<account>` (for example `listener-russia`) for named accounts. Sessions are bound to their server and IP, so re-authenticate on the new host instead of copying files:

```bash
python3 scripts/telegram_auth.py --list     # which sessions exist
python3 scripts/telegram_auth.py            # default account (add --account NAME for others)
docker compose restart listener
```

See [Telegram Setup](telegram-setup.md) for the full layout.

### Prevention

- Back up `telegram_sessions/` before major upgrades (`./scripts/backup.sh` does this)
- Never run two processes on the same session file

---

## Contributing

Found a new gotcha? Add it to this page following this format:

```markdown
## Problem Title

### Problem
One sentence describing the issue.

### Symptoms
- Bullet points of what you'll observe
- Error messages, unexpected behavior

### Root Cause
Why this happens (technical explanation).

### Solution
Step-by-step fix with commands.

### Prevention
How to avoid this in the future.
```

Then submit a PR to the docs repository.
