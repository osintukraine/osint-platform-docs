# Multi-Storage Box Setup

This guide covers adding multiple Hetzner Storage Boxes for scalable media storage. The platform supports database-driven multi-box routing where **adding new boxes requires zero code changes**.

## Overview

The multi-storage architecture enables:

- **Horizontal scaling** - Add storage boxes as data grows
- **Region partitioning** - Route channels to specific storage by region
- **Zero downtime expansion** - Add boxes without service interruption
- **Automatic load balancing** - Round-robin selection within 5% tolerance band

!!! info "Prerequisites"
    Complete the [single-box Hetzner setup](./hetzner-storage.md) first. Multi-storage extends that foundation.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MULTI-STORAGE ARCHITECTURE                                │
└─────────────────────────────────────────────────────────────────────────────┘

                                 ┌──────────────────┐
                                 │   Box Selector   │
                                 │ (round-robin 5%) │
                                 └────────┬─────────┘
                                          │
              ┌───────────────────────────┼───────────────────────────┐
              │                           │                           │
              ▼                           ▼                           ▼
    ┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
    │  minio-default   │        │  minio-russia-1  │        │  minio-ukraine-1 │
    │  (port 9000)     │        │  (port 9000)     │        │  (port 9000)     │
    └────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
             │                           │                           │
             ▼                           ▼                           ▼
    ┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
    │  SSHFS Mount     │        │  SSHFS Mount     │        │  SSHFS Mount     │
    │  /mnt/hetzner/   │        │  /mnt/hetzner/   │        │  /mnt/hetzner/   │
    │    default       │        │    russia-1      │        │    ukraine-1     │
    └────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
             │                           │                           │
             ▼                           ▼                           ▼
    ┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
    │  Hetzner Box 1   │        │  Hetzner Box 2   │        │  Hetzner Box 3   │
    │  (5TB, EU)       │        │  (10TB, EU)      │        │  (10TB, EU)      │
    └──────────────────┘        └──────────────────┘        └──────────────────┘

Each storage box has:
- One Hetzner Storage Box (remote storage)
- One SSHFS mount (local mount point)
- One MinIO container (S3 gateway)
- One Caddy route (/minio-{id}/*)
```

## Quick Start

### Adding a New Storage Box

```bash
# 1. Add box to database
./scripts/storage-admin.sh add russia-1 \
    u283231.your-storagebox.de \
    u283231 \
    /mnt/hetzner/russia-1 \
    5000 \
    russia

# 2. Regenerate Caddy routes and Docker Compose
./scripts/storage-admin.sh regen

# 3. Setup SSHFS mount (see below)
sudo ./scripts/setup-hetzner-storage.sh --box-id russia-1

# 4. Restart affected services
docker-compose -f docker-compose.yml -f docker-compose.storage.yml up -d
docker-compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

## storage-admin.sh Commands

The `storage-admin.sh` script is your primary tool for multi-box management:

| Command | Description |
|---------|-------------|
| `./scripts/storage-admin.sh list` | List all storage boxes with usage |
| `./scripts/storage-admin.sh add <id> <host> <user> <mount> <capacity> [region]` | Add new box |
| `./scripts/storage-admin.sh regen` | Regenerate Caddy routes + Docker Compose |
| `./scripts/storage-admin.sh health` | Check all box health status |

### Example: List Storage Boxes

```bash
$ ./scripts/storage-admin.sh list
=== Storage Boxes ===
 id        | account_region | used_gb | capacity_gb |  pct  | is_active | is_full | is_readonly | minio_endpoint
-----------+----------------+---------+-------------+-------+-----------+---------+-------------+----------------
 default   | eu             |  234.50 |        5000 |  4.7  | t         | f       | f           | minio
 russia-1  | russia         | 1245.00 |       10000 | 12.5  | t         | f       | f           | minio-russia-1
 ukraine-1 | ukraine        |  567.80 |       10000 |  5.7  | t         | f       | f           | minio-ukraine-1
```

### Example: Add New Box

```bash
$ ./scripts/storage-admin.sh add russia-2 \
    u999999.your-storagebox.de \
    u999999 \
    /mnt/hetzner/russia-2 \
    10000 \
    russia
Adding storage box: russia-2
INSERT 0 1
Added. Now run: ./scripts/storage-admin.sh regen
```

## Step-by-Step: Adding a Storage Box

### Step 1: Order Hetzner Storage Box

1. Log into [Hetzner Robot Console](https://robot.hetzner.com)
2. Order a new Storage Box (BX21 5TB or larger recommended)
3. Note the connection details:
   - Host: `uXXXXXX.your-storagebox.de`
   - User: `uXXXXXX`
   - Port: `23`

### Step 2: Add SSH Key to New Box

```bash
# Add your public key to the new storage box
sftp -P 23 -o PreferredAuthentications=password \
  uXXXXXX@uXXXXXX.your-storagebox.de << 'EOF'
mkdir .ssh
put ~/.ssh/id_ed25519.pub .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
chmod 700 .ssh
exit
EOF

# Verify key authentication works
ssh -p 23 -i ~/.ssh/id_ed25519 \
  uXXXXXX@uXXXXXX.your-storagebox.de "df -h ."
```

### Step 3: Add Box to Database

```bash
./scripts/storage-admin.sh add russia-2 \
    uXXXXXX.your-storagebox.de \
    uXXXXXX \
    /mnt/hetzner/russia-2 \
    10000 \
    russia
```

Parameters:
- `id`: Unique identifier (e.g., `russia-2`, `ukraine-1`)
- `hetzner_host`: Storage box hostname
- `hetzner_user`: Storage box username
- `mount_path`: Local mount point (will be created)
- `capacity_gb`: Total capacity in GB
- `region`: Logical region for channel routing (`russia`, `ukraine`, `eu`)

### Step 4: Regenerate Configuration

```bash
./scripts/storage-admin.sh regen
```

This generates:
- `infrastructure/caddy/storage-routes.snippet` - Caddy reverse proxy routes
- `docker-compose.storage.yml` - MinIO container definitions

!!! warning "Critical: Verify Docker Mount"
    The generated `storage-routes.snippet` file **must** be mounted into the Caddy container.
    Check your `docker-compose.yml` has this volume under the `caddy` service:

    ```yaml
    volumes:
      - ./infrastructure/caddy/Caddyfile.production:/etc/caddy/Caddyfile:ro
      - ./infrastructure/caddy/storage-routes.snippet:/etc/caddy/storage-routes.snippet:ro  # ← Required!
    ```

    Without this mount, Caddy imports an empty/missing snippet and `/minio-{box-id}/*` routes won't work.

### Step 5: Setup SSHFS Mount

Create mount point and systemd units:

```bash
# Create mount directory
sudo mkdir -p /mnt/hetzner/russia-2

# Create systemd mount unit
sudo tee /etc/systemd/system/mnt-hetzner-russia\\x2d2.mount << 'EOF'
[Unit]
Description=SSHFS mount for Hetzner Storage Box russia-2
After=network-online.target
Wants=network-online.target

[Mount]
What=uXXXXXX@uXXXXXX.your-storagebox.de:/
Where=/mnt/hetzner/russia-2
Type=fuse.sshfs
Options=port=23,IdentityFile=/home/youruser/.ssh/id_ed25519,allow_other,reconnect,ServerAliveInterval=15,ServerAliveCountMax=3

[Install]
WantedBy=multi-user.target
EOF

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable mnt-hetzner-russia\\x2d2.mount
sudo systemctl start mnt-hetzner-russia\\x2d2.mount

# Verify mount
df -h /mnt/hetzner/russia-2
```

### Step 6: Start New MinIO Container

```bash
# Start with multi-storage profile
docker-compose -f docker-compose.yml -f docker-compose.storage.yml up -d

# Verify container is running
docker-compose -f docker-compose.yml -f docker-compose.storage.yml ps minio-russia-2
```

### Step 7: Reload Caddy

```bash
docker-compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

### Step 8: Verify Setup

```bash
# Check box health
./scripts/storage-admin.sh health

# Test MinIO connectivity
curl -I http://localhost/minio-russia-2/osint-media/test

# Check processor can select new box
docker-compose logs processor | grep "Selected storage box"
```

## Box Selection Algorithm

The platform uses intelligent round-robin selection with a 5% tolerance band:

```
1. Filter eligible boxes:
   - is_active = true
   - is_full = false
   - is_readonly = false
   - usage < high_water_mark (default 90%)

2. Filter by region (if specified)

3. Find lowest usage percentage

4. Select boxes within 5% of lowest usage

5. Round-robin among selected boxes
```

**Why 5% tolerance?**

Without tolerance, all writes would go to the "least full" box, causing:
- Hot-spotting on single disk
- Uneven IOPS distribution
- Potential bottleneck

With 5% tolerance, boxes at 50%, 52%, and 54% usage are treated equally, distributing writes across all three.

## Configuration Options

### Storage Box Table Columns

| Column | Type | Description |
|--------|------|-------------|
| `id` | VARCHAR(50) | Unique box identifier (e.g., `russia-1`) |
| `hetzner_host` | VARCHAR(255) | Storage box hostname |
| `hetzner_user` | VARCHAR(50) | Storage box username |
| `hetzner_port` | INTEGER | SSH port (always 23) |
| `mount_path` | VARCHAR(255) | Local SSHFS mount point |
| `minio_endpoint` | VARCHAR(255) | Docker service name (e.g., `minio-russia-1`) |
| `minio_port` | INTEGER | MinIO port (default 9000) |
| `capacity_gb` | INTEGER | Total capacity in GB |
| `used_bytes` | BIGINT | Actual usage in bytes |
| `high_water_mark` | INTEGER | Stop writes at this % (default 90) |
| `is_readonly` | BOOLEAN | Accept no new writes |
| `is_active` | BOOLEAN | Accepting traffic |
| `is_full` | BOOLEAN | Auto-set when above high water mark |
| `priority` | INTEGER | Box selection priority (lower = higher) |
| `account_region` | VARCHAR(20) | Logical region for routing |

### High Water Mark

When a box reaches `high_water_mark` percentage:
- `is_full` is automatically set to `true`
- Box stops receiving new uploads
- Existing files remain accessible

To clear (after freeing space):
```sql
UPDATE storage_boxes
SET is_full = false
WHERE id = 'russia-1';
```

### Read-Only Mode

Put a box in maintenance mode:
```sql
UPDATE storage_boxes
SET is_readonly = true
WHERE id = 'russia-1';
```

Existing files remain accessible; new uploads go to other boxes.

### Priority

Lower priority = preferred for selection:
```sql
-- Prefer SSDs (priority 50) over HDDs (priority 100)
UPDATE storage_boxes SET priority = 50 WHERE id LIKE '%ssd%';
UPDATE storage_boxes SET priority = 100 WHERE id LIKE '%hdd%';
```

## Monitoring

### Health Check Task

The enrichment service runs a storage health check every 5 minutes:

```bash
# View health check logs
docker-compose logs enrichment | grep storage_health

# Check in database
SELECT id, last_health_check,
       ROUND(usage_percent::numeric, 1) as usage_pct,
       is_full
FROM storage_boxes
WHERE is_active = true;
```

### Prometheus Metrics

| Metric | Description |
|--------|-------------|
| `storage_boxes_checked_total` | Health checks performed |
| `storage_boxes_healthy_total` | Boxes with healthy mounts |
| `storage_boxes_unhealthy_total` | Boxes with mount issues |

### Alerting

Add to Prometheus alert rules:
```yaml
- alert: StorageBoxUnhealthy
  expr: storage_boxes_unhealthy_total > 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Storage box mount unhealthy"
    description: "{{ $value }} storage boxes have unhealthy mounts"

- alert: StorageBoxFull
  expr: |
    (storage_box_used_bytes / storage_box_capacity_bytes) > 0.90
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Storage box above 90% capacity"
```

## Troubleshooting

### New box not receiving uploads

1. Check box is active and not full:
```sql
SELECT id, is_active, is_full, is_readonly, high_water_mark,
       ROUND((used_bytes::float / (capacity_gb * 1024 * 1024 * 1024)) * 100, 1) as pct
FROM storage_boxes WHERE id = 'russia-2';
```

2. Check MinIO container is running:
```bash
docker-compose -f docker-compose.yml -f docker-compose.storage.yml ps minio-russia-2
```

3. Verify SSHFS mount:
```bash
ls /mnt/hetzner/russia-2
df -h /mnt/hetzner/russia-2
```

### Media returns 404 or blank images

**Symptom**: `/minio-{box-id}/osint-media/...` returns 404, media thumbnails fail to load.

**Cause**: Caddy snippet file not mounted or empty.

**Fix**:

1. Verify snippet exists and has content:
```bash
# On host
cat infrastructure/caddy/storage-routes.snippet | head -20

# Should show routes like:
# handle /minio-default/* {
#     uri strip_prefix /minio-default
#     ...
```

2. If empty, regenerate:
```bash
source .env 2>/dev/null
DATABASE_URL="postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@localhost:5432/${POSTGRES_DB}" \
    python3 scripts/generate-caddy-storage-routes.py > infrastructure/caddy/storage-routes.snippet
```

3. Verify mount in docker-compose.yml:
```yaml
caddy:
  volumes:
    - ./infrastructure/caddy/storage-routes.snippet:/etc/caddy/storage-routes.snippet:ro
```

4. Restart Caddy with mount:
```bash
docker-compose up -d caddy
docker exec osint-caddy cat /etc/caddy/storage-routes.snippet  # Verify in container
docker exec osint-caddy caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
```

### MinIO connection refused

1. Check container logs:
```bash
docker-compose logs minio-russia-2
```

2. Verify Caddy routes regenerated:
```bash
cat infrastructure/caddy/storage-routes.snippet | grep russia-2
```

3. Reload Caddy:
```bash
docker-compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

### SSHFS mount stale

Symptoms: `ls /mnt/hetzner/russia-2` hangs

Fix:
```bash
# Force unmount
sudo fusermount -uz /mnt/hetzner/russia-2

# Restart mount
sudo systemctl restart mnt-hetzner-russia\\x2d2.mount
```

### Box usage not updating

The health check task reconciles usage every 5 minutes. Force update:
```sql
UPDATE storage_boxes
SET used_bytes = (
    SELECT COALESCE(SUM(file_size), 0)
    FROM media_files
    WHERE storage_box_id = 'russia-2'
)
WHERE id = 'russia-2';
```

## Related Documentation

- [Hetzner Storage Setup](./hetzner-storage.md) - Single-box setup (prerequisite)
- [Scaling](./scaling.md) - Platform scaling strategies
- [Monitoring](./monitoring.md) - Prometheus and Grafana setup
- [Media Storage Architecture](../developer-guide/media-storage.md) - Technical architecture
