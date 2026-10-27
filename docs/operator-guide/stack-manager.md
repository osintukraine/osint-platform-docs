# Stack Manager

Python-based command-line tool for managing the OSINT Intelligence Platform Docker Compose stack.

## Overview

The stack manager (`scripts/stack-manager-py.sh`) is a modern replacement for manual Docker Compose commands, providing:

- **Rich UI**: Beautiful terminal output with colors and progress indicators
- **Environment-aware**: Auto-detects development vs production
- **Profile discovery**: Dynamically discovers available profiles from docker-compose.yml
- **Streaming logs**: Real-time log output with proper Ctrl+C handling
- **Hardware detection**: Detects the hardware tier and writes tier defaults (worker replicas, PostgreSQL and Redis sizing) into `.env`
- **Presets**: Hardware-aware profile presets (Minimal, Laptop, Small Server, Production)

## Quick Start

```bash
# Interactive menu (no arguments)
./scripts/stack-manager-py.sh

# Direct commands
./scripts/stack-manager-py.sh status
./scripts/stack-manager-py.sh start
./scripts/stack-manager-py.sh logs -f api
```

## Available Commands

| Command | Description |
|---------|-------------|
| `start` | Start services (`-p/--profile` to enable profiles, `-b/--build` to build first) |
| `stop` | Stop services (`--all` to stop everything) |
| `restart` | Restart services (`-f/--force` to recreate containers) |
| `status` | Show current stack status |
| `logs` | View service logs (`-f` to follow, `-n/--tail`, default 100) |
| `build` | Build service images (`--no-cache`) |
| `health` | Check service health and connectivity |
| `profiles` | Manage service profiles and presets (interactive) |
| `hardware` | Hardware detection and configuration (`detect`, `show`, `--tier`, `--auto`) |
| `shell` | Open a shell in a running container (`-c` to choose the command, default `/bin/sh`) |
| `urls` | Show or open service URLs |
| `metrics` | Check metrics endpoints |
| `monitoring` | Open monitoring dashboards |
| `stats` | Show container resource usage |
| `scale` | Scale a service: `scale SERVICE COUNT` (default service `processor-worker`) |
| `clean` | Remove stopped containers |
| `nuclear` | Remove EVERYTHING (dangerous!) |

## Common Operations

### Starting the Stack

```bash
# Start core services
./scripts/stack-manager-py.sh start

# Start with specific profiles (repeat -p)
./scripts/stack-manager-py.sh start -p enrichment-standard -p monitoring

# Start in production mode (adds docker-compose.production.yml)
./scripts/stack-manager-py.sh -e production start
```

### Viewing Logs

```bash
# Stream logs from API service
./scripts/stack-manager-py.sh logs api

# Follow logs from multiple services
./scripts/stack-manager-py.sh logs -f processor-worker listener

# Show last 200 lines
./scripts/stack-manager-py.sh logs api --tail 200
```

The logs command streams in real-time and handles Ctrl+C gracefully (unlike raw `docker compose logs`).

### Checking Status

```bash
# Quick status overview
./scripts/stack-manager-py.sh status

# Detailed health check
./scripts/stack-manager-py.sh health
```

### Hardware Configuration

```bash
# Detect hardware and update .env (interactive)
./scripts/stack-manager-py.sh hardware

# Show current hardware configuration
./scripts/stack-manager-py.sh hardware show

# Non-interactive, or force a tier (laptop, laptop-gpu, server, server-gpu)
./scripts/stack-manager-py.sh hardware --auto
./scripts/stack-manager-py.sh hardware --tier=server
```

Detection writes a `HARDWARE-ADAPTIVE CONFIG` block into `.env` (worker replicas, `REDIS_MAXMEMORY`, `POSTGRES_SHARED_BUFFERS`, `POSTGRES_WORK_MEM`, ...). Restart the stack afterwards to apply it.

!!! note
    The tier block also contains legacy `LLM_MODEL`/`OLLAMA_*` values. LLM provider selection is made with `LLM_PROVIDER`, see [LLM Gateway and Providers](external-llm.md); review those lines before relying on them.

### Shell Access

```bash
# Open a shell in the API container (default /bin/sh)
./scripts/stack-manager-py.sh shell api

# Run a different command (psql in the postgres container)
./scripts/stack-manager-py.sh shell postgres -c "psql -U osint_user -d osint_platform"
```

## Profile Management

Profiles control which services start. The stack manager discovers them from `docker-compose.yml`:

```bash
# Interactive profile menu (status, presets, custom picker, stop all optional)
./scripts/stack-manager-py.sh profiles

# Show profile status
./scripts/stack-manager-py.sh profiles status

# Pick a preset or individual profiles
./scripts/stack-manager-py.sh profiles presets
./scripts/stack-manager-py.sh profiles custom
```

There are no `profiles enable`/`disable` subcommands; toggling happens in the menus (or use `start -p NAME`).

### Presets

| Preset | Profiles | For hardware tier |
|--------|----------|-------------------|
| Minimal Archive | none (core only) | any |
| Laptop Mode | `enrichment-lite`, `monitoring` | laptop |
| Small Server | `enrichment-standard`, `opensanctions`, `monitoring` | laptop-gpu, server |
| Production | `enrichment-full`, `opensanctions`, `monitoring`, `tools` | server-gpu |

### Available Profiles

The authoritative list is in [Docker Services](../reference/docker-services.md#profiles).

| Profile | Description |
|---------|-------------|
| (default) | Core: postgres, redis, minio, ollama (search embeddings), listener, processor-worker, media-worker, media-sync, api, frontend, rss-ingestor |
| `enrichment-lite` | CPU-only workers: router, fast-pool, telegram, decision, maintenance, cluster-detection, analytics |
| `enrichment-standard`, `enrichment-full` | Lite plus the LLM workers (ai-tagging, event-detection, fact-check, geolocation-llm, cluster-validation, rss-validation), plus opensanctions workers and ollama-batch |
| `llm-gateway` | Workers that use the LLM gateway (backlog-*, AI enrichment) |
| `monitoring` | Prometheus, Grafana, Loki, Promtail, Alertmanager, exporters, ntfy |
| `opensanctions` | Yente, OpenSanctions, entity-ingestion |
| `auth` | Ory Kratos, Oathkeeper, Caddy, CrowdSec |
| `tools` | NocoDB, MkDocs |
| `multi-account` | Per Telegram account (russia, ukraine): listener, media-worker, enrichment-telegram, analytics (use `TELEGRAM_MODE=multi` / `scripts/switch-telegram-mode.sh`) |
| `dr` | shadow-sync (disaster recovery) |

## Environment Modes

The stack manager supports three environments. The default comes from `ENVIRONMENT` (or `ENV`) in `.env`, falling back to development; `-e` overrides it:

```bash
./scripts/stack-manager-py.sh start                  # development / from .env
./scripts/stack-manager-py.sh -e staging start
./scripts/stack-manager-py.sh -e production start
```

### Environment Differences

| Setting | Development | Production |
|---------|-------------|------------|
| Compose files | `docker-compose.yml` | `docker-compose.yml` + `docker-compose.production.yml` |
| Service URLs (`urls`, `monitoring`) | `localhost` ports | Subdomains of the configured domain |

## Multi-File Compose Support

For production with multiple compose files:

```bash
# Production adds the overlay automatically
./scripts/stack-manager-py.sh -e production start
```

In production mode the manager passes `docker-compose.yml` followed by `docker-compose.production.yml` (CrowdSec, Caddy with TLS, auth overrides). Enable monitoring through the `monitoring` profile rather than the standalone `docker-compose.monitoring.yml`.

## Troubleshooting

### Command Not Found

```bash
# Ensure you're in the project root
cd /path/to/osint-intelligence-platform

# Run from there
./scripts/stack-manager-py.sh status
```

### Missing Python Dependencies

The script auto-installs dependencies, but if issues occur:

```bash
pip3 install click rich pyyaml
```

### Permission Denied

```bash
chmod +x scripts/stack-manager-py.sh
```

### Docker Socket Access

```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Log out and back in, or:
newgrp docker
```

## Comparison with Docker Compose

| Task | Docker Compose | Stack Manager |
|------|----------------|---------------|
| Start all | `docker compose up -d` | `./scripts/stack-manager-py.sh start` |
| View logs | `docker compose logs -f api` | `./scripts/stack-manager-py.sh logs api` |
| Status | `docker compose ps` | `./scripts/stack-manager-py.sh status` |
| Health | Manual checks | `./scripts/stack-manager-py.sh health` |
| Shell | `docker compose exec api bash` | `./scripts/stack-manager-py.sh shell api` |

**Benefits of Stack Manager:**

- Pretty output with colors and tables
- Profile-aware service discovery
- Environment detection
- Hardware tier information
- Integrated health checks
- Proper signal handling for logs

## See Also

- [Installation Guide](installation.md) - Initial setup
- [Configuration Guide](configuration.md) - Environment variables
- [Troubleshooting Guide](troubleshooting.md) - Common issues
- [Production Gotchas](production-gotchas.md) - Deployment lessons
