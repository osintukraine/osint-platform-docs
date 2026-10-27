# Caddy Reverse Proxy Configuration

Caddy serves as the single entry point for all platform services, handling routing, authentication integration, media serving, and TLS termination.

## Configuration Files

The platform includes several Caddyfiles in `infrastructure/caddy/`:

| File | Environment | Auth System | TLS |
|------|-------------|-------------|-----|
| `Caddyfile.local` | Local development | Keycloak + OAuth2-proxy (`docker-compose.development.yml`) | Off |
| `Caddyfile.ory` | Development with Ory | Kratos session checked by the API | Off |
| `Caddyfile.production` | Production (mounted by the `caddy` service of the `auth` profile in `docker-compose.yml`) | Ory stack, CrowdSec bouncer | Auto (Let's Encrypt) |
| `Caddyfile.security` | Production overlay (mounted by `docker-compose.production.yml`, container `osint-caddy-secured`) | Same, stricter headers (`X-Frame-Options DENY`) | Auto (Let's Encrypt) |
| `Caddyfile.example` | Template | n/a | n/a |

Both production files need `DOMAIN` and `CROWDSEC_BOUNCER_API_KEY` in `.env`, and use the CrowdSec bouncer image of Caddy. `Caddyfile.production` also needs `SITE_DOMAIN` for the media hotlink check. Route documentation lives next to the files (`PRODUCTION_ROUTES.md`, `ROUTING.md`).

## Architecture Overview

```
                                    ┌─────────────────┐
                                    │   Frontend      │
                                    │   (Next.js)     │
                                    └────────▲────────┘
                                             │
┌──────────┐      ┌─────────┐      ┌─────────┴─────────┐
│  Client  │─────▶│  Caddy  │─────▶│   API (FastAPI)   │
└──────────┘      └────┬────┘      └───────────────────┘
                       │
                       │           ┌─────────────────┐
                       ├──────────▶│   Oathkeeper    │ (auth proxy)
                       │           └─────────────────┘
                       │
                       │           ┌─────────────────┐
                       └──────────▶│   Media Files   │
                                   │ (local + remote)│
                                   └─────────────────┘
```

## Route Summary

### All Configurations

| Path | Target | Description |
|------|--------|-------------|
| `/api/*` | API service | REST API endpoints |
| `/rss/*` | API service | RSS feed endpoints |
| `/_next/*`, `/static/*` | Frontend | Next.js static assets |
| `/*` | Frontend | All other routes (SPA fallback) |

### Caddyfile.local (Keycloak)

| Path | Target | Auth |
|------|--------|------|
| `/auth/*` | Keycloak | Public |
| `/oauth2/*` | OAuth2-proxy | Public |
| `/grafana/*` | Grafana | OAuth2 forward auth |
| `/nocodb/*` | NocoDB | OAuth2 forward auth |
| `/ntfy/*` | ntfy | OAuth2 forward auth |
| `/prometheus/*` | Prometheus | Direct (dev only) |
| `/alertmanager/*` | Alertmanager | Direct (dev only) |
| `/media/*` | MinIO | Public |

### Caddyfile.ory (Ory Stack)

| Path | Target | Auth |
|------|--------|------|
| `/auth/*` | Frontend | Self-service UI |
| `/kratos/*` | Kratos | Public API |
| `/api/*` | API | Session cookie (OryAuthMiddleware) |
| `/rss/*` | Oathkeeper | Public + authenticated feeds |
| `/media/*` | Hybrid routing | Public (see below) |

### Caddyfile.production

| Path | Target | Auth |
|------|--------|------|
| `/kratos/*` | Kratos (prefix stripped) | Public self-service API |
| `/api/admin/check-role` | none | Blocked (403), internal to Oathkeeper |
| `/api/*` | API | Session cookie validated by the API (OryAuthMiddleware); `X-User-*` headers stripped to prevent spoofing |
| `/rss/*` | API | Public + token feeds; `X-User-*` stripped |
| `/grafana/*`, `/prometheus/*`, `/alertmanager/*`, `/ntfy/*` | Oathkeeper | Admin only |
| `/nocodb/*` | Oathkeeper | Analyst and above |
| `/dozzle/*`, `/dashy/*` | Oathkeeper | Viewer and above |
| `/media/*` | Local buffer, Hetzner mount, then API redirect | Public, referer check |
| `/minio-storage/*` | MinIO | Internal redirect target |
| `/platform-docs/*` | MkDocs (`tools`/`docs` profile) | Public |
| `/health` | Frontend/API health | Public |

## Media Serving (Hybrid Architecture)

The Ory and production configurations implement a tiered media serving strategy:

```
Request: /media/ab/cd/abcdef123456.jpg
                │
                ▼
┌─────────────────────────────────┐
│  Check local SSD buffer         │
│  /var/cache/osint-media-buffer  │
│  (MEDIA_BUFFER_PATH on host)    │
└─────────────┬───────────────────┘
              │
    ┌─────────┴─────────┐
    │ File exists?      │
    │                   │
   YES                  NO
    │                   │
    ▼                   ▼
┌─────────┐     ┌─────────────────┐
│ Serve   │     │ (production)    │
│ locally │     │ try Hetzner     │
│ (fast)  │     │ SSHFS mount,    │
└─────────┘     │ then rewrite to │
                │ /api/media/     │
                │ internal/...    │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ API checks Redis│
                │ for storage box │
                │ routing         │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │ 302 Redirect to │
                │ Hetzner Storage │
                └─────────────────┘
```

**Benefits:**
- Hot files (recently uploaded) served instantly from local SSD
- Cold files redirected to Hetzner (bandwidth offload)
- Redis cache gives 99%+ hit rate on routing lookups
- Content-addressed storage enables aggressive browser caching

**Simplified configuration (`Caddyfile.ory`; `Caddyfile.production` adds a Hetzner mount tier at `/mnt/hetzner-media/osint-media` and a referer-based hotlink block):**
```caddyfile
handle /media/* {
    root * /var/cache/osint-media-buffer/osint-media

    @local_file file {path}

    handle @local_file {
        header Cache-Control "public, max-age=31536000, immutable"
        header X-Media-Source "local-buffer"
        file_server
    }

    handle {
        uri strip_prefix /media
        rewrite * /api/media/internal/media-redirect{uri}
        reverse_proxy api:8000 {
            header_up X-Internal-Request "caddy-media-fallback"
        }
    }
}
```

## Security Headers (Production)

The production Caddyfiles add HSTS (`max-age=63072000; includeSubDomains; preload`), `X-Content-Type-Options`, `X-Frame-Options` (`SAMEORIGIN` in `Caddyfile.production`, `DENY` in `Caddyfile.security`), a `Referrer-Policy` and a Content-Security-Policy. The CSP allow-lists the third-party hosts the frontend uses (unpkg, jsDelivr, Cloudflare challenges, Ko-fi, Google Fonts, t.me frames). If you embed a new third-party script or iframe, add its host to the CSP in the Caddyfile your deployment mounts, otherwise the browser blocks it silently. Check the browser console first when a widget does not load.

Real client IPs: the production files trust the Docker network and Cloudflare ranges (`trusted_proxies`, `Cf-Connecting-Ip`), so CrowdSec and the API see the real address. Access logs are written as JSON to `/var/log/caddy/access.log` (the `caddy_logs` volume) for CrowdSec.

## Compression

All configurations enable compression for responses over 1KB:

```caddyfile
encode {
    gzip 6
    zstd
    minimum_length 1024
}
```

## Switching Configurations

The Caddyfile is bind-mounted read-only by the compose file, so there is nothing to copy. `docker-compose.yml` (profile `auth`) mounts `Caddyfile.production`; `docker-compose.production.yml` swaps in `Caddyfile.security`. Pick the stack and recreate the container:

```bash
# Production with the security overlay (CrowdSec + stricter headers)
export DOMAIN=yourdomain.com
docker compose -f docker-compose.yml -f docker-compose.production.yml up -d caddy

# Base compose with the auth profile (Caddyfile.production)
docker compose --profile auth up -d caddy
```

Editing a mounted Caddyfile: save it, then reload (below) or restart `caddy`. Note that a bind-mounted file replaced by an editor that swaps inodes may need `docker compose up -d --force-recreate caddy`.

After changing `storage_boxes`, regenerate the per-box MinIO routes with `python scripts/generate-caddy-storage-routes.py > infrastructure/caddy/storage-routes.snippet` and add them to the Caddyfile.

## Direct Service Access (Development)

For debugging, services are also available directly:

| Service | Direct Port | Through Caddy |
|---------|-------------|---------------|
| API | `localhost:8000` | `localhost/api/` |
| Frontend | `localhost:3000` | `localhost/` |
| Keycloak (`docker-compose.development.yml`) | `localhost:8180` | `localhost/auth/` |
| Grafana | `localhost:3001` (`GRAFANA_PORT`) | `localhost/grafana/` |
| Prometheus | `localhost:9090` | `localhost/prometheus/` |
| NocoDB | `localhost:8080` | `localhost/nocodb/` |
| ntfy | `localhost:8090` | `localhost/ntfy/` |

## Troubleshooting

### Check Caddy logs
```bash
docker compose logs -f caddy
```

### Reload configuration without restart
```bash
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
```

### Verify routing
```bash
# Check which upstream handles a path
curl -I http://localhost/api/health
curl -I http://localhost/media/test.jpg
```

### Common issues

**502 Bad Gateway**: Target service not running
```bash
docker compose ps  # Check service status
```

**Media not loading**: Check local buffer mount
```bash
docker compose exec caddy ls -la /var/cache/osint-media-buffer/   # host path: MEDIA_BUFFER_PATH (default ./data/media-buffer)
```

**Auth redirect loops**: Check Kratos/Oathkeeper health
```bash
curl http://localhost:4433/health/ready  # Kratos
curl http://localhost:4456/health/ready  # Oathkeeper
```

### HTTPS Redirect Issues (Production)

!!! warning "Critical Production Issue"
    This is the #1 cause of production deployment failures. Read carefully.

**Symptom**: API redirects go to `http://` instead of `https://`

When FastAPI performs automatic trailing slash redirects (e.g., `/api/channels` → `/api/channels/`), the redirect URL uses `http://` even though the original request was `https://`.

```bash
# Test for this issue:
curl -I https://yoursite.com/api/channels

# BAD - redirect goes to http:
HTTP/2 307
location: http://yoursite.com/api/channels/

# GOOD - redirect stays https:
HTTP/2 307
location: https://yoursite.com/api/channels/
```

**Root Cause**: FastAPI doesn't automatically respect `X-Forwarded-Proto` header from Caddy.

**Solution**: This is **already fixed** in the codebase. The fix adds `ProxyHeadersMiddleware`:

```python
# services/api/src/main.py
from uvicorn.middleware.proxy_headers import ProxyHeadersMiddleware

app.add_middleware(ProxyHeadersMiddleware, ...)  # reads X-Forwarded-Proto / -For only
```

**If you're on the latest code**, you don't need to do anything. Just `git pull` and rebuild.

**Caddy automatically sends** `X-Forwarded-Proto: https` for HTTPS requests. The middleware makes FastAPI respect this header.

**See also**: [Production Gotchas](production-gotchas.md#https-redirect-issues)
