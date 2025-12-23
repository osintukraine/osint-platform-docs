# Caddy Reverse Proxy Configuration

Caddy serves as the single entry point for all platform services, handling routing, authentication integration, media serving, and TLS termination.

## Configuration Files

The platform includes three Caddyfile configurations for different environments:

| File | Environment | Auth System | TLS |
|------|-------------|-------------|-----|
| `Caddyfile.local` | Local development | Keycloak + OAuth2-proxy | Off |
| `Caddyfile.ory` | Development with Ory | Kratos + Oathkeeper | Off |
| `Caddyfile.production` | Production | Ory stack | Auto (Let's Encrypt) |

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
| `/auth/*` | Kratos | Public |
| `/api/*` | Oathkeeper | Zero-trust |
| `/rss/*` | Oathkeeper | Zero-trust |
| `/grafana/*` | Oathkeeper | Zero-trust |
| `/nocodb/*` | Oathkeeper | Zero-trust |
| `/prometheus/*` | Oathkeeper | Zero-trust |

## Media Serving (Hybrid Architecture)

The Ory configuration implements a sophisticated media serving strategy:

```
Request: /media/ab/cd/abcdef123456.jpg
                │
                ▼
┌─────────────────────────────────┐
│  Check local SSD buffer         │
│  /var/cache/osint-media-buffer  │
└─────────────┬───────────────────┘
              │
    ┌─────────┴─────────┐
    │ File exists?      │
    │                   │
   YES                  NO
    │                   │
    ▼                   ▼
┌─────────┐     ┌─────────────────┐
│ Serve   │     │ Rewrite to API  │
│ locally │     │ /api/media/     │
│ (fast)  │     │ internal/...    │
└─────────┘     └────────┬────────┘
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

**Configuration in Caddyfile.ory:**
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
        rewrite * /api/media/internal/media-redirect{path}
        reverse_proxy api:8000 {
            header_up X-Internal-Request "caddy-media-fallback"
        }
    }
}
```

## Security Headers (Production)

The production Caddyfile adds comprehensive security headers:

```caddyfile
header {
    Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    X-Content-Type-Options "nosniff"
    X-Frame-Options "SAMEORIGIN"
    X-XSS-Protection "1; mode=block"
    Referrer-Policy "strict-origin-when-cross-origin"
    Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';"
}
```

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

To switch between configurations:

```bash
# Development with Keycloak
cp infrastructure/caddy/Caddyfile.local infrastructure/caddy/Caddyfile
docker-compose restart caddy

# Development with Ory
cp infrastructure/caddy/Caddyfile.ory infrastructure/caddy/Caddyfile
docker-compose restart caddy

# Production
export DOMAIN=yourdomain.com
cp infrastructure/caddy/Caddyfile.production infrastructure/caddy/Caddyfile
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## Direct Service Access (Development)

For debugging, services are also available directly:

| Service | Direct Port | Through Caddy |
|---------|-------------|---------------|
| API | `localhost:8000` | `localhost/api/` |
| Frontend | `localhost:3000` | `localhost/` |
| Keycloak | `localhost:8180` | `localhost/auth/` |
| Grafana | `localhost:3001` | `localhost/grafana/` |
| Prometheus | `localhost:9090` | `localhost/prometheus/` |
| NocoDB | `localhost:8080` | `localhost/nocodb/` |
| ntfy | `localhost:8090` | `localhost/ntfy/` |

## Troubleshooting

### Check Caddy logs
```bash
docker-compose logs -f caddy
```

### Reload configuration without restart
```bash
docker-compose exec caddy caddy reload --config /etc/caddy/Caddyfile
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
docker-compose ps  # Check service status
```

**Media not loading**: Check local buffer mount
```bash
docker-compose exec caddy ls -la /var/cache/osint-media-buffer/
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

**Solution**: This is **already fixed** in the codebase (December 2025). The fix adds `ProxyHeadersMiddleware`:

```python
# services/api/src/main.py
from uvicorn.middleware.proxy_headers import ProxyHeadersMiddleware

app.add_middleware(ProxyHeadersMiddleware, trusted_hosts=["*"])
```

**If you're on the latest code**, you don't need to do anything. Just `git pull` and rebuild.

**Caddy automatically sends** `X-Forwarded-Proto: https` for HTTPS requests. The middleware makes FastAPI respect this header.

**See also**: [Production Gotchas](production-gotchas.md#https-redirect-issues)
