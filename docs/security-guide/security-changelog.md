# Security Changelog

This document tracks significant security improvements and vulnerability fixes in the OSINT Intelligence Platform.

## 2025-12-27: Comprehensive Security Hardening

This release includes a major security audit and hardening effort, addressing multiple vulnerability classes and improving the overall security posture.

### Summary of Changes

| Category | Changes | Severity |
|----------|---------|----------|
| Access Control | Role-filtered OpenAPI docs, Admin endpoint protection | High |
| Injection Prevention | ILIKE pattern escaping across 4 routers | Medium |
| Dependencies | Updated 13 requirements.txt files with security patches | High |
| Media Security | Pre-signed URL support for MinIO | Medium |
| Configuration | Production hardening (DEBUG, logging, verbose) | Low |

---

### Access Control Improvements

#### Role-Filtered OpenAPI Documentation

**Commit**: `444dead0 feat(api): implement role-filtered OpenAPI documentation`

**Problem**: The full OpenAPI specification was exposed to anonymous users, potentially revealing admin-only endpoints and internal API structure.

**Solution**: Implemented role-aware OpenAPI generation that filters endpoints based on user authentication:

| Role | Visible Endpoints |
|------|-------------------|
| Anonymous | 78 public endpoints |
| User | + User profile endpoints |
| Analyst | + Search, export endpoints |
| Admin | All endpoints (194 total) |

**Files Changed**:
- `services/api/src/routers/docs.py` - New role-filtered docs router
- `services/api/src/main.py` - Disabled default docs, integrated new router

**Usage**:
- Anonymous users see `/docs` with filtered endpoints
- Authenticated users see additional endpoints based on role
- Full docs available at `/docs?include_admin=true` for admins

---

#### Admin Endpoint Protection

**Commit**: `6bbca6f1 fix(auth): enforce admin role on user management endpoints`

**Problem**: Some admin endpoints in `auth.py` were using incorrect dependency injection, potentially allowing unauthorized access.

**Solution**: Fixed endpoints to properly use `AdminUser` dependency from the dependencies module:

```python
# BEFORE (incorrect)
async def create_user_endpoint(
    config: AuthConfig = Depends(get_auth_config)  # Wrong!
):

# AFTER (correct)
from ..dependencies import AdminUser

async def create_user_endpoint(
    current_user: AdminUser,  # Requires admin role
    config: AuthConfig = Depends(get_auth_config)
):
```

**Affected Endpoints**:
- `POST /api/auth/users` - Create user
- `PUT /api/auth/users/{id}` - Update user
- `DELETE /api/auth/users/{id}` - Delete user
- `PUT /api/auth/users/{id}/activate` - Activate/deactivate user

---

### Injection Prevention

#### ILIKE Pattern Injection

**Commits**:
- `17b67097 fix(channels): escape ILIKE wildcards to prevent pattern injection`
- `d95970b1 fix(security): escape ILIKE wildcards across all routers`

**Problem**: User-controlled strings were passed directly to PostgreSQL `ILIKE` queries, allowing pattern injection attacks:

```sql
-- Attacker input: "test%'; DROP TABLE users; --"
WHERE folder ILIKE '%test%'; DROP TABLE users; --%'
```

While SQL injection was blocked by parameterized queries, ILIKE wildcards (`%`, `_`) could be exploited for:
- Information disclosure (matching unintended patterns)
- Performance attacks (complex regex patterns)

**Solution**: Created `escape_ilike_pattern()` utility to escape special characters:

```python
# services/api/src/utils/sql_safety.py
import re

def escape_ilike_pattern(value: str) -> str:
    """Escape ILIKE special characters to prevent pattern injection."""
    return re.sub(r'([\\%_])', r'\\\1', value)
```

**Affected Files**:
- `services/api/src/routers/channels.py` - folder filter
- `services/api/src/routers/rss.py` - query, channel_folder filters
- `services/api/src/routers/admin/feeds.py` - search filter
- `services/api/src/routers/admin/export.py` - search_query filter

**Testing**: Verified with malicious patterns:
```bash
# Before: matched unintended patterns
curl "/api/channels?folder=%25admin%25"  # % = wildcard

# After: treats as literal text
curl "/api/channels?folder=%25admin%25"  # Escaped, no match
```

---

### Dependency Security Updates

**Commit**: `edd3586e security: update all Python dependencies for security patches`

**Problem**: Multiple dependencies had known CVEs:

| Package | Old Version | New Version | CVE |
|---------|-------------|-------------|-----|
| httpx | 0.27.0 | 0.28.0+ | CVE-2024-47874 (HTTP/2 request smuggling) |
| aiohttp | 3.9.1 | 3.11.0+ | CVE-2024-52304, CVE-2024-42367 |
| fastapi | 0.109.0 | 0.127.0+ | Security patches |
| uvicorn | 0.25.0 | 0.40.0+ | Security updates |

**Solution**: Updated all 13 requirements.txt files across services:

- `services/api/requirements.txt`
- `services/listener/requirements.txt`
- `services/processor/requirements.txt`
- `services/enrichment/requirements.txt`
- `services/rss-ingestor/requirements.txt`
- `services/notifier/requirements.txt`
- `services/analytics/requirements.txt`
- `services/entity-ingestion/requirements.txt`
- `services/opensanctions/requirements.txt`
- `services/media-sync/requirements.txt`
- `services/shadow-sync/requirements.txt`
- `services/translation-backfill/requirements.txt`
- `shared/python/requirements.txt`

**Additional Updates**:
| Package | Old | New | Reason |
|---------|-----|-----|--------|
| sqlalchemy | 2.0.23 | 2.0.36+ | Latest stable |
| pydantic | 2.6.0 | 2.10.0+ | Bug fixes |
| minio | 7.2.0 | 7.2.10+ | Security patches |
| redis | 5.0.1 | 5.2.0+ | Security updates |
| deepl | 1.16.1 | 1.21.0+ | API compatibility |
| prometheus-client | 0.19.0 | 0.21.0+ | Latest |
| sentence-transformers | 2.x | 3.3.0+ | Performance |

---

### Media Security

#### Pre-Signed URL Support

**Commits**:
- `89b715fe security: add MinIO pre-signed URL support for media access`
- `f2934014 feat: add pre-signed URL config + cluster-validation to build scripts`

**Problem**: Media files were accessible via predictable public URLs. Anyone with a URL could access media indefinitely.

**Solution**: Implemented configurable pre-signed URLs:

```python
# services/api/src/utils/minio_client.py
def get_presigned_url(s3_key: str, expiry_hours: int = 4) -> str:
    """Generate time-limited signed URL for media access."""
```

**Configuration**:
```bash
# .env
USE_PRESIGNED_URLS=true           # Enable (default: false)
PRESIGNED_URL_EXPIRY_HOURS=4      # Expiry in hours
```

**Benefits**:
- URLs expire after configured period
- Cannot be guessed or tampered with
- MinIO doesn't need public access
- Can audit who requested access

**Files Changed**:
- `services/api/src/utils/minio_client.py` - New utility
- `services/api/src/routers/messages.py` - Use new utility
- `.env.example`, `.env.development`, `.env.production.template` - New variables

---

### Configuration Hardening

**Commit**: `f6bef784 chore(security): production hardening for config files`

**Changes**:

| File | Change | Reason |
|------|--------|--------|
| `.env.example` | `DEBUG=false` | Prevent debug info leakage |
| `infrastructure/oathkeeper/oathkeeper.yml` | `verbose: false` | Reduce auth log exposure |
| `infrastructure/kratos/kratos.yml.template` | `level: info` | Reduce debug logging |

---

### Security Audit Reports

**Commit**: `ed244ceb docs(security): add comprehensive security audit reports`

Created detailed security audit documentation:

- `docs/audits/2025-12-27-code-security-audit.md` - Static code analysis results
- `docs/audits/2025-12-27-live-penetration-test.md` - Production penetration test results

**Audit Findings Summary**:
- 1 Critical (SQL injection vector - analyzed, mitigated)
- 4 High (access control, dependencies)
- 6 Medium (injection, config, headers)
- 5 Low (logging, hardening)

All findings addressed in this release.

---

## Security Testing Verification

### Rate Limiting
```bash
# Verify rate limiting works
for i in {1..15}; do curl -s -o /dev/null -w "%{http_code}\n" "https://v2.osintukraine.com/api/messages"; done
# Expected: 200s then 429 (Too Many Requests)
```

### Authentication
```bash
# Verify admin endpoints require auth
curl -s "https://v2.osintukraine.com/api/admin/channels" | jq .detail
# Expected: "Authentication required" or "Not authenticated"
```

### ILIKE Escaping
```bash
# Verify wildcards are escaped
curl "https://v2.osintukraine.com/api/channels?folder=%25test%25"
# Expected: Literal match, not wildcard
```

### OpenAPI Filtering
```bash
# Verify admin endpoints hidden from anonymous
curl -s "https://v2.osintukraine.com/openapi.json" | jq '.paths | keys | length'
# Expected: ~78 (not 194)
```

---

## Recommendations for Operators

### Immediate Actions

1. **Update dependencies**: Pull latest code and rebuild images
   ```bash
   git pull
   ./scripts/build-pytorch-services.sh
   docker-compose up -d
   ```

2. **Enable pre-signed URLs** (production):
   ```bash
   # Add to .env
   USE_PRESIGNED_URLS=true
   PRESIGNED_URL_EXPIRY_HOURS=4
   ```

3. **Verify configuration**:
   ```bash
   # Check DEBUG is false
   grep "^DEBUG=" .env
   # Should be: DEBUG=false
   ```

### Ongoing Security

- Review `docs/audits/` for detailed findings
- Run security scans after updates: `trivy image <image>`
- Monitor authentication logs for anomalies
- Rotate credentials quarterly

---

## Related Documentation

- [Security Hardening Guide](hardening.md) - Full production hardening
- [Authentication Guide](authentication.md) - Ory Kratos/Oathkeeper setup
- [Authorization Guide](authorization.md) - Role-based access control
- [CrowdSec Integration](crowdsec.md) - Intrusion prevention
