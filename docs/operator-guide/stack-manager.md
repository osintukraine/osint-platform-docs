# Stack Manager

Python-based command-line tool for managing the OSINT Intelligence Platform Docker Compose stack.

## Overview

The stack manager (`scripts/stack-manager-py.sh`) is a modern replacement for manual Docker Compose commands, providing:

- **Rich UI**: Beautiful terminal output with colors and progress indicators
- **Environment-aware**: Auto-detects development vs production
- **Profile discovery**: Dynamically discovers available profiles from docker-compose.yml
- **Streaming logs**: Real-time log output with proper Ctrl+C handling
- **Hardware detection**: Shows current hardware tier and recommendations

## Quick Start

```bash
# Interactive menu (no arguments)
./scripts/stack-manager-py.sh

# Direct commands
./scripts/stack-manager-py.sh status
./scripts/stack-manager-py.sh start
./scripts/stack-manager-py.sh logs api
```

## Available Commands

| Command | Description |
|---------|-------------|
| `start` | Start services (with profile selection) |
| `stop` | Stop services |
| `restart` | Restart services |
| `status` | Show current stack status |
| `logs` | View service logs (streaming) |
| `build` | Build service images |
| `health` | Check service health and connectivity |
| `profiles` | Manage service profiles |
| `hardware` | Hardware detection and configuration |
| `shell` | Open a shell in a running container |
| `urls` | Show or open service URLs |
| `metrics` | Check metrics endpoints |
| `monitoring` | Open monitoring dashboards |
| `stats` | Show container resource usage |
| `scale` | Scale a service to N replicas |
| `clean` | Remove stopped containers |
| `nuclear` | Remove EVERYTHING (dangerous!) |

## Common Operations

### Starting the Stack

```bash
# Start with default profile (core services)
./scripts/stack-manager-py.sh start

# Start with specific profiles
./scripts/stack-manager-py.sh start --profiles enrichment,monitoring

# Start in production mode
./scripts/stack-manager-py.sh -e production start
```

### Viewing Logs

```bash
# Stream logs from API service
./scripts/stack-manager-py.sh logs api

# Follow logs from multiple services
./scripts/stack-manager-py.sh logs processor listener

# Show last 100 lines
./scripts/stack-manager-py.sh logs api --tail 100
```

The logs command streams in real-time and handles Ctrl+C gracefully (unlike raw `docker-compose logs`).

### Checking Status

```bash
# Quick status overview
./scripts/stack-manager-py.sh status

# Detailed health check
./scripts/stack-manager-py.sh health
```

### Hardware Configuration

```bash
# Show detected hardware tier
./scripts/stack-manager-py.sh hardware show

# Expected output:
# Hardware Tier: server-gpu
# CPU Cores: 16
# RAM: 32GB
# GPU: NVIDIA RTX 3080 (10GB VRAM)
# Recommended LLM: qwen2.5:7b
```

### Shell Access

```bash
# Open bash in API container
./scripts/stack-manager-py.sh shell api

# Open psql in postgres container
./scripts/stack-manager-py.sh shell postgres psql -U osint_user -d osint_platform
```

## Profile Management

Profiles control which services start. The stack manager discovers profiles from `docker-compose.yml`:

```bash
# List available profiles
./scripts/stack-manager-py.sh profiles list

# Enable a profile
./scripts/stack-manager-py.sh profiles enable enrichment

# Disable a profile
./scripts/stack-manager-py.sh profiles disable monitoring
```

### Available Profiles

| Profile | Services | Description |
|---------|----------|-------------|
| (default) | Core services | Listener, processor, API, frontend, postgres, redis |
| `enrichment` | AI enrichment | Ollama, embeddings, AI tagging, event detection |
| `monitoring` | Observability | Prometheus, Grafana, Loki, alertmanager |
| `opensanctions` | Entity matching | Yente API, OpenSanctions data |
| `auth` | Authentication | Ory Kratos, Oathkeeper |

## Environment Modes

The stack manager supports three environments:

```bash
# Development (default)
./scripts/stack-manager-py.sh start

# Staging
./scripts/stack-manager-py.sh -e staging start

# Production
./scripts/stack-manager-py.sh -e production start
```

### Environment Differences

| Setting | Development | Production |
|---------|-------------|------------|
| Log level | DEBUG | INFO |
| Hot reload | Enabled | Disabled |
| Volume mounts | Source code | None |
| Resource limits | None | Configured |

## Multi-File Compose Support

For production with multiple compose files:

```bash
# Set compose file path
export COMPOSE_FILE="docker-compose.yml:docker-compose.production.yml:docker-compose.monitoring.yml"

# Stack manager respects COMPOSE_FILE
./scripts/stack-manager-py.sh start
```

The stack manager handles service deduplication automatically when multiple compose files define the same service.

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
| Start all | `docker-compose up -d` | `./scripts/stack-manager-py.sh start` |
| View logs | `docker-compose logs -f api` | `./scripts/stack-manager-py.sh logs api` |
| Status | `docker-compose ps` | `./scripts/stack-manager-py.sh status` |
| Health | Manual checks | `./scripts/stack-manager-py.sh health` |
| Shell | `docker-compose exec api bash` | `./scripts/stack-manager-py.sh shell api` |

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
