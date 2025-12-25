# External LLM Endpoint Configuration

This guide explains how to configure the platform to use an external Ollama-compatible LLM endpoint instead of the local Docker service.

## Overview

By default, the platform runs Ollama locally in Docker. However, you can offload LLM processing to a dedicated external server for:

- **Better resource isolation**: LLM doesn't compete with database/API for CPU/RAM
- **Larger models**: External server can have more RAM for bigger models (32b, 70b)
- **Easier scaling**: Scale LLM infrastructure independently
- **Cost optimization**: Use specialized LLM hosting (e.g., Contabo DeepSeek)

## Supported Providers

Any Ollama-compatible endpoint works:

| Provider | Description | Cost |
|----------|-------------|------|
| **Self-hosted VM** | Run Ollama on your own VPS | VPS cost only |
| **Contabo DeepSeek** | 1-click DeepSeek-R1 image | €45-120/mo |
| **Ollama Cloud** | Official Ollama hosting | Varies |

## Configuration

### Environment Variables

Add these to your `.env` file:

```bash
# Required: External endpoint URL
OLLAMA_BASE_URL=https://your-llm-server:11434

# Optional: API key for authenticated endpoints
OLLAMA_API_KEY=your-api-key-here

# Required: Enable external mode (adds retry logic)
OLLAMA_EXTERNAL_MODE=true

# Optional: Retry settings for network resilience
OLLAMA_MAX_RETRIES=3        # Max retries on failure (default: 3)
OLLAMA_RETRY_DELAY=1.0      # Base delay in seconds (default: 1.0)

# Recommended: Increase timeout for network latency
OLLAMA_TIMEOUT=60           # Seconds (default: 30)

# Optional: Use a different model on external server
OLLAMA_MODEL=deepseek-r1:14b
```

### Local vs External Mode

| Setting | Local (Default) | External |
|---------|-----------------|----------|
| `OLLAMA_BASE_URL` | `http://ollama:11434` | `https://your-server:11434` |
| `OLLAMA_EXTERNAL_MODE` | `false` | `true` |
| Retry on failure | No (fail fast) | Yes (exponential backoff) |
| Authentication | None | Optional API key |

## Setup Guide

### Option 1: Self-Hosted VM

1. **Provision a VM** with adequate resources:
   - 8+ cores, 24+ GB RAM for 14b models
   - 16+ cores, 48+ GB RAM for 32b models

2. **Install Ollama**:
   ```bash
   curl -fsSL https://ollama.com/install.sh | sh
   ```

3. **Pull your model**:
   ```bash
   ollama pull deepseek-r1:14b
   # or
   ollama pull qwen2.5:3b
   ```

4. **Configure Ollama for external access**:
   ```bash
   # /etc/systemd/system/ollama.service.d/override.conf
   [Service]
   Environment="OLLAMA_HOST=0.0.0.0"
   ```

5. **Secure with HTTPS** (recommended):
   - Use Caddy, nginx, or cloud load balancer
   - Add API key authentication if exposed to internet

6. **Update platform `.env`**:
   ```bash
   OLLAMA_BASE_URL=https://your-vm-ip:11434
   OLLAMA_EXTERNAL_MODE=true
   OLLAMA_TIMEOUT=60
   ```

### Option 2: Contabo Hosted DeepSeek

1. **Order a VDS** from [Contabo](https://contabo.com/en/hosted-deepseek-ai-enterprise-cloud/):
   - VDS M (€45/mo): DeepSeek-R1:14b
   - VDS L (€64/mo): Larger models

2. **Select the 1-click DeepSeek image** during setup

3. **Get your server IP** after provisioning

4. **Update platform `.env`**:
   ```bash
   OLLAMA_BASE_URL=http://your-contabo-ip:11434
   OLLAMA_EXTERNAL_MODE=true
   OLLAMA_MODEL=deepseek-r1:14b
   OLLAMA_TIMEOUT=60
   ```

5. **Restart processor**:
   ```bash
   docker-compose restart processor-worker
   ```

## Health Check

Verify external LLM connectivity:

```bash
# API endpoint
curl http://localhost:8000/api/system/health/llm
```

Expected response:
```json
{
  "endpoint": "https://your-llm-server:11434",
  "external_mode": true,
  "configured_model": "deepseek-r1:14b",
  "auth_configured": false,
  "status": "healthy",
  "models_available": ["deepseek-r1:14b"],
  "configured_model_available": true
}
```

### Status Values

| Status | Meaning | Action |
|--------|---------|--------|
| `healthy` | Connected, model available | None |
| `auth_failed` | 401 response | Check `OLLAMA_API_KEY` |
| `unreachable` | Connection failed | Check URL, firewall |
| `timeout` | Request timed out | Increase `OLLAMA_TIMEOUT` |
| `degraded` | Connected but unexpected response | Check server logs |

## Retry Behavior

When `OLLAMA_EXTERNAL_MODE=true`, the platform handles transient failures:

1. **First attempt** fails → wait 1s
2. **Second attempt** fails → wait 2s
3. **Third attempt** fails → wait 4s
4. **All retries exhausted** → fall back to rules (if `LLM_FALLBACK_TO_RULES=true`)

Configure with:
```bash
OLLAMA_MAX_RETRIES=3      # Number of retries
OLLAMA_RETRY_DELAY=1.0    # Base delay (exponential backoff)
```

## Model Recommendations

### For OSINT Classification

| Model | Size | RAM | Use Case |
|-------|------|-----|----------|
| `qwen2.5:3b` | 2 GB | 6 GB | Best for RU/UK content |
| `deepseek-r1:7b` | 4.7 GB | 10 GB | Good reasoning |
| `deepseek-r1:14b` | 9 GB | 20 GB | High quality |
| `deepseek-r1:32b` | 20 GB | 40 GB | Best quality |

### DeepSeek vs Qwen

| Aspect | DeepSeek-R1 | Qwen 2.5 |
|--------|-------------|----------|
| Reasoning | Excellent (chain-of-thought) | Good |
| RU/UK languages | Good | Excellent |
| Speed | Slower (reasoning overhead) | Faster |
| Context | 128K tokens | 32K tokens |

**Recommendation**: Start with `qwen2.5:3b` for best RU/UK OSINT classification. Switch to DeepSeek for complex reasoning tasks.

## Security Considerations

### Network Security

1. **Use HTTPS** for external endpoints
2. **Firewall rules**: Only allow platform IP to access LLM port
3. **VPN/Private network**: Best for production

### Authentication

If your endpoint requires auth:
```bash
OLLAMA_API_KEY=your-secret-key
```

The platform sends this as:
```
Authorization: Bearer your-secret-key
```

### Never Expose Ollama Directly

Ollama doesn't have built-in authentication. Always:
- Use a reverse proxy (Caddy, nginx) with auth
- Or restrict access via firewall/VPN

## Monitoring

### Prometheus Metrics

When using external LLM, monitor:

```promql
# Request latency (will be higher for external)
histogram_quantile(0.95, rate(llm_request_duration_seconds_bucket[5m]))

# Retry rate (should be low)
rate(llm_external_retry_total[5m])

# Error rate
rate(llm_external_errors_total[5m])
```

### Grafana Dashboard

Add panel for external LLM latency:
- Warning: p95 > 5s
- Critical: p95 > 15s

## Troubleshooting

### Connection Refused

```
status: unreachable
error: Connection refused
```

**Fix**: Check that Ollama is listening on `0.0.0.0`:
```bash
# On LLM server
curl http://localhost:11434/api/tags  # Should work
curl http://YOUR_IP:11434/api/tags    # Should also work
```

### Authentication Failed

```
status: auth_failed
error: Authentication failed - check OLLAMA_API_KEY
```

**Fix**: Verify API key matches server configuration.

### Timeout

```
status: timeout
error: Request timed out after 5 seconds
```

**Fix**: Increase timeout:
```bash
OLLAMA_TIMEOUT=120
```

### Model Not Found

```
configured_model_available: false
```

**Fix**: Pull the model on the external server:
```bash
ssh your-llm-server
ollama pull qwen2.5:3b
```

## Rollback to Local

If external LLM causes issues:

```bash
# Comment out external settings
# OLLAMA_BASE_URL=...
# OLLAMA_EXTERNAL_MODE=true

# Restart
docker-compose restart processor-worker
```

The platform will use local `http://ollama:11434` by default.

---

*See also: [Configuration](configuration.md) | [Troubleshooting](troubleshooting.md) | [Performance Tuning](performance-tuning.md)*
