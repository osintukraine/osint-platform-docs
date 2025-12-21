# Hetzner Storage Box Setup

This guide covers setting up a Hetzner Storage Box for cost-effective media storage with a local SSD buffer for optimal performance.

## Overview

The OSINT Platform uses a **hybrid storage architecture**:

- **Local Buffer (SSD)**: Fast writes, instant browser access for new uploads
- **Hetzner Storage Box**: Cost-effective long-term storage (~€3.80/TB/month)
- **Async Sync Worker**: Background upload from buffer to Hetzner

This provides:

- **Fast Response**: Browser gets media instantly from local SSD
- **Cost Efficiency**: Bulk storage at 10x less than cloud providers
- **Zero Data Loss**: Write to local buffer before async sync
- **High Availability**: Redis-cached routing for 99%+ cache hit rate

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         MEDIA REQUEST FLOW                               │
└─────────────────────────────────────────────────────────────────────────┘

Browser: GET /media/ab/cd/abcd1234.jpg
         │
         ▼
    ┌─────────┐
    │  Caddy  │ ────→ Check local buffer (/var/cache/osint-media-buffer)
    └─────────┘
         │
    ┌────┴────┐
    │         │
   HIT       MISS
    │         │
    ▼         ▼
  Serve    API redirect → Redis cache → DB lookup → Hetzner redirect
  from                        │
  SSD                    99%+ hit rate
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         UPLOAD FLOW                                      │
└─────────────────────────────────────────────────────────────────────────┘

Telegram Message
         │
         ▼
    ┌──────────┐
    │Processor │ ─────→ Download to .tmp/
    └──────────┘
         │
         ▼
    Atomic move to local buffer
    /var/cache/osint-media-buffer/osint-media/media/ab/cd/hash.jpg
         │
         ▼
    Insert to DB (synced_at = NULL)
         │
         ▼
    Queue to Redis: media:sync:pending
         │
         ▼ (async, background)
    ┌─────────────┐
    │ Media-Sync  │ ─────→ Upload to MinIO (Hetzner SSHFS)
    │   Worker    │ ─────→ Update DB (synced_at = NOW())
    └─────────────┘ ─────→ Delete local file
```

## Prerequisites

1. **Hetzner Robot account** - [robot.hetzner.com](https://robot.hetzner.com)
2. **Storage Box ordered** - Available sizes:
   - BX11 (1TB): €3.86/month
   - BX21 (5TB): €9.80/month *(recommended for starting)*
   - BX31 (10TB): €19.60/month
   - BX41 (20TB): €36.17/month *(recommended for production)*
3. **SSH key pair** on your server
4. **Local SSD** with 100GB+ free space for buffer

### Ubuntu/Debian Prerequisites

```bash
# Install required packages
sudo apt-get update
sudo apt-get install -y sshfs fuse3

# Verify systemd-escape is available
which systemd-escape || sudo apt-get install -y systemd
```

### Fedora/RHEL Prerequisites

```bash
# Install required packages
sudo dnf install -y fuse-sshfs fuse3
```

## Quick Setup

### Step 1: Configure Environment

Add to your `.env` file:

```bash
# Hetzner Storage Box connection
HETZNER_HOST=uXXXXXX.your-storagebox.de
HETZNER_USER=uXXXXXX
HETZNER_PORT=23
HETZNER_SSH_KEY=/home/youruser/.ssh/id_ed25519

# Mount path for SSHFS (no dashes - avoids systemd escaping issues)
HETZNER_MOUNT_PATH=/mnt/hetznerstorage

# Local buffer path (should be on SSD)
MEDIA_BUFFER_PATH=/var/cache/osint-media-buffer
```

Get your credentials from [Hetzner Robot Console](https://robot.hetzner.com) → Storage Boxes → Your Box → Access Data.

### Step 2: Add SSH Key to Storage Box

If you don't have an SSH key, generate one:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -C "osint-platform"
```

Add your public key to the storage box via SFTP:

```bash
# Using password authentication (one-time)
sftp -P 23 -o PreferredAuthentications=password uXXXXXX@uXXXXXX.your-storagebox.de << 'EOF'
mkdir .ssh
put ~/.ssh/id_ed25519.pub .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
chmod 700 .ssh
exit
EOF
```

Test key authentication:

```bash
ssh -p 23 -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes \
  uXXXXXX@uXXXXXX.your-storagebox.de "df -h ."
```

### Step 3: Run Setup Script

```bash
sudo ./scripts/setup-hetzner-storage.sh
```

The script will:

1. **Install SSHFS** if needed
2. **Configure FUSE** for Docker access (`user_allow_other`)
3. **Create mount point** at `/mnt/hetznerstorage`
4. **Test SSH connectivity**
5. **Test SSHFS mount**
6. **Generate systemd units** with properly escaped names
7. **Setup local buffer** at `/var/cache/osint-media-buffer`
8. **Enable automount** via systemd
9. **Start health check timer**

Expected output:

```
==========================================
  Hetzner Storage Box Setup
==========================================

ℹ Step 1/9: Loading configuration from .env...
✓ Configuration loaded
ℹ Step 2/9: Checking SSHFS installation...
✓ SSHFS is installed
...
ℹ Step 9/9: Enabling and starting services...
✓ Enabled mnt-hetznerstorage.automount
✓ Enabled hetzner-storage-health.timer

==========================================
  Setup Complete!
==========================================
```

### Step 4: Start Media Sync Worker

```bash
docker-compose up -d media-sync
```

### Step 5: Verify Setup

```bash
# Check mount
df -h /mnt/hetznerstorage

# Check local buffer
ls -la /var/cache/osint-media-buffer/

# Check sync queue
docker-compose exec redis redis-cli LLEN media:sync:pending

# Check worker logs
docker-compose logs -f media-sync
```

## Component Details

### Local Buffer

**Purpose**: Fast SSD storage for recently uploaded media

**Path Structure**:
```
/var/cache/osint-media-buffer/
├── osint-media/
│   └── media/
│       └── ab/
│           └── cd/
│               └── abcd1234567890.jpg
└── .tmp/                                    # Atomic download staging
```

**Sizing**: Recommend 100GB minimum. Files stay in buffer until sync worker uploads them (typically minutes to hours depending on load).

### Media-Sync Worker

**Purpose**: Background service that uploads buffered media to Hetzner

**Configuration** (in `docker-compose.yml`):
```yaml
media-sync:
  build:
    context: .
    dockerfile: services/media-sync/Dockerfile
  environment:
    REDIS_URL: redis://redis:6379/0
    DATABASE_URL: postgresql+asyncpg://...
    MINIO_ENDPOINT: minio:9000
  volumes:
    - ${MEDIA_BUFFER_PATH:-./data/media-buffer}:/var/cache/osint-media-buffer:rshared
```

**Scaling**: For high-volume deployments, scale the worker:
```bash
docker-compose up -d --scale media-sync=3
```

### Systemd Units

The setup creates these systemd units:

| Unit | Purpose |
|------|---------|
| `mnt-hetznerstorage.mount` | SSHFS mount definition |
| `mnt-hetznerstorage.automount` | On-demand mount trigger |
| `hetzner-storage-health.service` | Health check with auto-remount |
| `hetzner-storage-health.timer` | Runs health check every 5 minutes |

!!! note "Unit Naming"
    The unit is named `mnt-hetznerstorage` matching the mount path `/mnt/hetznerstorage`. We avoid dashes in the path name to prevent systemd escaping complexity.

Check status:

```bash
# Mount status
systemctl status 'mnt-hetzner\x2dstorage.automount'

# Health timer
systemctl status hetzner-storage-health.timer
journalctl -u hetzner-storage-health.service --since "1 hour ago"
```

### Caddy Media Routing

Caddy routes media requests with a two-tier strategy:

1. **Fast path**: Check local buffer, serve if found
2. **Fallback**: Call API for redirect to Hetzner

Configuration (in `infrastructure/caddy/Caddyfile.ory`):
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

## Monitoring

### Queue Depth

```bash
# Pending sync jobs
docker-compose exec redis redis-cli LLEN media:sync:pending

# Failed jobs (need manual review)
docker-compose exec redis redis-cli LLEN media:sync:failed
```

### Cache Statistics

```bash
curl http://localhost:8000/api/media/internal/media-stats
```

Response:
```json
{
  "cache_size": 15234,
  "estimated_hit_rate": 0.994,
  "cache_ttl_seconds": 86400
}
```

### Worker Statistics

The media-sync worker logs statistics on shutdown:
```
Worker stopped. Stats: synced=1234, failed=5, bytes=5678901234
```

### Storage Usage

```bash
# Hetzner storage
df -h /mnt/hetznerstorage

# Local buffer
df -h /var/cache/osint-media-buffer
```

## Troubleshooting

### Systemd Unit Fails: "bad unit file setting"

**Cause**: Unit filename doesn't match mount path (systemd escaping issue).

**Fix**: Regenerate units with proper escaping:
```bash
./scripts/generate-systemd-units.sh
sudo ./scripts/setup-hetzner-storage.sh
```

### Mount fails with "Permission denied"

Ensure `user_allow_other` is in `/etc/fuse.conf`:

```bash
grep user_allow_other /etc/fuse.conf || echo "user_allow_other" | sudo tee -a /etc/fuse.conf
```

### "Too many authentication failures"

SSH is trying multiple keys. Specify the exact key in `.env`:

```bash
HETZNER_SSH_KEY=/home/youruser/.ssh/id_ed25519
```

### Mount hangs or times out

Check network connectivity:

```bash
ping uXXXXXX.your-storagebox.de
ssh -v -p 23 -i ~/.ssh/id_ed25519 uXXXXXX@uXXXXXX.your-storagebox.de "ls"
```

### Local buffer filling up

If the sync queue is backing up:

```bash
# Check queue depth
docker-compose exec redis redis-cli LLEN media:sync:pending

# Scale up workers
docker-compose up -d --scale media-sync=3

# Check for errors
docker-compose logs media-sync | grep ERROR
```

### Files not appearing in browser

1. Check if file is in local buffer:
   ```bash
   ls /var/cache/osint-media-buffer/osint-media/media/ab/cd/
   ```

2. Check if file was synced (has `synced_at`):
   ```sql
   SELECT sha256, synced_at, local_path FROM media_files WHERE sha256 = 'abcd...';
   ```

3. Check Redis cache:
   ```bash
   docker-compose exec redis redis-cli GET "media:route:abcd..."
   ```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `HETZNER_HOST` | - | Storage box hostname (e.g., `uXXXXXX.your-storagebox.de`) |
| `HETZNER_USER` | - | Storage box username (e.g., `uXXXXXX`) |
| `HETZNER_PORT` | `23` | SSH port (always 23 for Hetzner) |
| `HETZNER_SSH_KEY` | `~/.ssh/id_ed25519` | SSH private key path |
| `HETZNER_MOUNT_PATH` | `/mnt/hetznerstorage` | Local mount point for SSHFS (no dashes!) |
| `MEDIA_BUFFER_PATH` | `/var/cache/osint-media-buffer` | Local SSD buffer path |

## Security Considerations

1. **SSH Key Security**: Use a dedicated key pair for the storage box, with `chmod 600` permissions
2. **No Shell Access**: Hetzner storage boxes only support SFTP (no shell)
3. **FUSE Access**: `user_allow_other` in `/etc/fuse.conf` allows Docker to access the mount
4. **Encryption**: Data in transit is encrypted via SSH; enable at-rest encryption in Hetzner Robot if needed
5. **Network**: Consider firewall rules limiting access to the storage box IP range

## Cost Comparison

| Storage Option | Cost/TB/Month | Egress Fees | Notes |
|----------------|---------------|-------------|-------|
| AWS S3 Standard | ~$23 | Yes ($0.09/GB) | Enterprise, global |
| Google Cloud Storage | ~$20 | Yes ($0.12/GB) | Enterprise, global |
| Cloudflare R2 | ~$15 | No | S3-compatible, global |
| Backblaze B2 | ~$5 | Yes ($0.01/GB) | Simple, US/EU |
| **Hetzner Storage Box** | **~$3.80** | **No** | **EU only, SSHFS** |

**Recommendation**: Hetzner for bulk storage if your servers are in EU. Use local SSD buffer (80GB-200GB) for hot cache.

## Related Documentation

- [Backup & Restore](./backup-restore.md) - Full platform backup strategies
- [Scaling](./scaling.md) - Scaling storage and processing
- [Configuration](./configuration.md) - Environment variables reference
- [Monitoring](./monitoring.md) - Prometheus metrics and Grafana dashboards
