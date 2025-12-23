# Production Gotchas

Hard-won lessons from deploying and operating the OSINT Intelligence Platform. **Check this page before your first production deployment.**

!!! warning "Last Updated: December 2025"
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
| Session file not found | Telegram auth fails after upgrade | [Session Naming](#session-file-location-changes) |

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

This is **already fixed** in the codebase (December 2025). The fix adds `ProxyHeadersMiddleware` from uvicorn:

```python
# services/api/src/main.py
from uvicorn.middleware.proxy_headers import ProxyHeadersMiddleware

app.add_middleware(ProxyHeadersMiddleware, trusted_hosts=["*"])
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
      NEXT_PUBLIC_API_URL: https://example.com
      NEXT_PUBLIC_BASE_URL: https://example.com
  environment:
    # These are for SSR only
    API_URL: http://api:8000
```

### Solution

1. Add your `NEXT_PUBLIC_*` variables to `build.args` in `docker-compose.yml`
2. Rebuild the frontend: `docker-compose build frontend`
3. Restart: `docker-compose up -d frontend`

### Prevention

- Always check `build.args` when deploying to a new domain
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
# Start all services
./scripts/stack-manager-py.sh start

# Stop all services
./scripts/stack-manager-py.sh stop

# View logs (with streaming and Ctrl+C support)
./scripts/stack-manager-py.sh logs api
./scripts/stack-manager-py.sh logs processor

# Check hardware tier
./scripts/stack-manager-py.sh hardware show
```

### Key Features

- **Streaming logs**: Real-time log output with proper Ctrl+C handling
- **Profile-aware**: Automatically discovers services based on active profiles
- **Service deduplication**: Handles multiple compose files without duplicate services
- **Hardware detection**: Shows current hardware tier and recommended settings

### Multi-File Compose Support

For production deployments with multiple compose files:

```bash
export COMPOSE_FILE="docker-compose.yml:docker-compose.production.yml"
./scripts/stack-manager-py.sh start
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

The `storage_boxes` table must have at least one entry with `is_default = true`. Migration 017 adds this automatically for new installs, but older installations may be missing it.

### Solution

Check if you have a default storage box:

```sql
-- Connect to database
docker-compose exec postgres psql -U osint_user -d osint_platform

-- Check for default storage box
SELECT id, name, is_default FROM storage_boxes WHERE is_default = true;
```

If no rows returned, add one:

```sql
INSERT INTO storage_boxes (name, box_type, is_default, is_active, created_at)
VALUES ('local', 'local', true, true, NOW());
```

### Prevention

- Run migrations after upgrading: `./scripts/migrate.sh`
- For fresh installs, this is handled automatically

---

## Session File Location Changes

### Problem

Telegram session file location changed in December 2025. If you're upgrading from an older version, your session files may be in the wrong location.

### Changes Made

| Item | Old | New |
|------|-----|-----|
| Session directory | `/app/data/` | `/data/sessions/` |
| Mount propagation | `:rshared` flag | Removed (incompatible with SSHFS) |

### Symptoms

- Telegram authentication fails after upgrade
- "Session file not found" errors in listener logs
- Prompted to re-authenticate even though session exists

### Solution

Move your existing session files:

```bash
# On your server
cd /path/to/osint-intelligence-platform

# Check where your old sessions are
ls -la data/*.session

# Create new directory if needed
mkdir -p data/sessions

# Move session files
mv data/*.session data/sessions/
```

Then restart the listener:

```bash
docker-compose restart listener
```

### Prevention

- When upgrading, check release notes for breaking changes
- Back up your session files before major upgrades

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
