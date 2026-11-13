# Authentication

User authentication and identity management for the OSINT Intelligence Platform.

## Overview

The platform supports four authentication modes to accommodate different deployment scenarios - from local development to production environments requiring enterprise-grade identity management.

## Authentication Modes

The platform supports four authentication providers configurable via the `AUTH_PROVIDER` environment variable:

### 1. None Mode (Development Only)

**No authentication required** - All endpoints are publicly accessible.

```bash
# .env configuration
AUTH_PROVIDER=none
AUTH_REQUIRED=false
```

**Use Cases:**

- Local development and testing
- Private VPN deployments with network-level security
- Internal research environments with trusted users

!!! danger "Security Risk"
    Never use `AUTH_PROVIDER=none` in production. All data becomes publicly accessible without authentication.

### 2. JWT Mode (Simple Deployments)

**Token-based authentication** using JSON Web Tokens for small teams and simple deployments.

```bash
# .env configuration
AUTH_PROVIDER=jwt
AUTH_REQUIRED=true
JWT_SECRET_KEY=<generate-random-256-bit-key>   # at least 32 characters, API refuses to start otherwise
JWT_ADMIN_PASSWORD=<admin-password>            # without it the built-in admin user cannot log in
JWT_ALGORITHM=HS256
JWT_EXPIRATION_MINUTES=60
```

**How It Works:**

1. User logs in with username/password via `POST /auth/login` on the API
2. API returns a JWT access token (`access_token`, `token_type`, `expires_in`)
3. The client sends `Authorization: Bearer <token>` with every request
4. API validates JWT signature and expiration

!!! note "Path prefix"
    The auth router is mounted at `/auth` (not `/api/auth`). Behind the production Caddy, `/auth/*` is routed to the frontend, so JWT login is intended for direct API access (development or small private deployments), not the production Caddy setup.

**Generate JWT Secret:**

```bash
# Generate strong 256-bit secret key
openssl rand -hex 32
```

**Features:**

- Simple setup (no external dependencies)
- Stateless tokens (HMAC signature check)
- Admins can create users with `POST /auth/users`; users can change their password with `POST /auth/users/me/password`

**Limitations:**

- Small local user store (a built-in `admin` user plus users added via the API), not a database-backed identity system
- No refresh-token or token-verify endpoint (log in again when the token expires)
- No password recovery flow, OAuth2 providers or MFA

### 3. Cloudron Mode (Cloudron App Store)

**OAuth authentication** using Cloudron's built-in identity provider.

```bash
# .env configuration (automatically set by Cloudron)
AUTH_PROVIDER=cloudron
AUTH_REQUIRED=true
```

**How It Works:**

1. Cloudron authenticates user
2. Cloudron injects the user's identity into the request
3. The API's Cloudron provider reads it and builds the user context

!!! info "Cloudron Only"
    This mode only works when deployed via Cloudron App Store. Headers are injected by Cloudron's reverse proxy.

### 4. Ory Mode (Production - Recommended)

**Production-grade identity management** using Ory Kratos for authentication and Ory Oathkeeper for authorization.

```bash
# .env configuration
AUTH_PROVIDER=ory
AUTH_REQUIRED=true

# Ory Kratos secrets (REQUIRED)
KRATOS_SECRET_COOKIE=<openssl rand -base64 32>
KRATOS_SECRET_CIPHER=<openssl rand -base64 24>  # Must be exactly 32 chars

KRATOS_BROWSER_URL=https://your-domain.com   # public URL Kratos renders into redirects/CORS

# Optional first admin (created once by the kratos-init container)
INITIAL_ADMIN_EMAIL=admin@yourdomain.com
INITIAL_ADMIN_PASSWORD=<strong-password>

# OAuth providers (optional, see Step 2 below)
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret

# SMTP (required for recovery/verification emails in production)
SMTP_CONNECTION_URI=smtps://apikey:YOUR_SENDGRID_API_KEY@smtp.sendgrid.net:465
SMTP_FROM_ADDRESS=noreply@yourdomain.com
SMTP_FROM_NAME=OSINT Platform
```

**Architecture:**

```
Browser → Caddy → /kratos/*  → Ory Kratos (login, registration, sessions)
                → /api/*, /  → Oathkeeper → validates ory_kratos_session cookie → API / Frontend
```

Oathkeeper injects `X-User-ID`, `X-User-Email` and `X-User-Role` after validating the session. The API only trusts these headers from trusted proxy addresses, and also validates the `ory_kratos_session` cookie against Kratos itself. The role comes from the identity's `metadata_public.role`.

**Features:**

- **Email/password authentication** with bcrypt (cost 12)
- **OAuth2 providers** (Google, GitHub), disabled until client credentials are configured
- **Password recovery and email verification** via one-time codes
- **MFA support** (TOTP and lookup/backup codes; settings require the highest available assurance level)
- **Session management** with secure cookies
- **Self-service flows** (registration, login, recovery, settings), served by the Next.js frontend under `/auth/*`

!!! warning "Review the shipped Kratos defaults before going live"
    - `docker-entrypoint.sh` starts Kratos with `--dev`. Dev mode relaxes cookie and URL security checks; decide whether that is acceptable for your deployment.
    - The `require_verified_address` login hook is commented out in `kratos.yml.template`, so unverified emails can log in. Enable it if you need verified accounts.
    - Without `SMTP_CONNECTION_URI`, Kratos points at a `mailslurper` host that is not started by default, so recovery/verification emails are never delivered.
    - Anyone who can reach the registration page can create an account (role `authenticated`). Restrict registration at the proxy or identity schema if the platform is not meant to be open.

## Setup Instructions

### JWT Authentication Setup

**Step 1: Generate JWT Secret**

```bash
# Generate random 256-bit secret
JWT_SECRET=$(openssl rand -hex 32)
echo "JWT_SECRET_KEY=$JWT_SECRET" >> .env
```

**Step 2: Configure Environment**

```bash
# Add to .env
AUTH_PROVIDER=jwt
AUTH_REQUIRED=true
JWT_SECRET_KEY=<your-generated-secret>
JWT_ALGORITHM=HS256
JWT_EXPIRATION_MINUTES=60
```

**Step 3: Log in as the built-in admin**

```bash
# The built-in user is "admin"; its password is JWT_ADMIN_PASSWORD
TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"<JWT_ADMIN_PASSWORD>"}' \
  | jq -r '.access_token')
```

**Step 4: Create more users and test**

```bash
# Create a user (admin only; default role is viewer)
curl -X POST http://localhost:8000/auth/users \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"username":"analyst1","password":"change-this-password","email":"analyst1@example.com","roles":["analyst"]}'

# Check who you are
curl http://localhost:8000/auth/users/me -H "Authorization: Bearer $TOKEN"
```

### Ory Kratos Setup (Production)

**Step 1: Generate Kratos Secrets**

```bash
# Generate cookie secret (any length)
COOKIE_SECRET=$(openssl rand -base64 32)

# Generate cipher secret (MUST be exactly 32 characters)
CIPHER_SECRET=$(openssl rand -base64 24)

# Add to .env
echo "KRATOS_SECRET_COOKIE=$COOKIE_SECRET" >> .env
echo "KRATOS_SECRET_CIPHER=$CIPHER_SECRET" >> .env
```

!!! warning "Cipher Secret Length"
    `KRATOS_SECRET_CIPHER` must be exactly 32 characters (not more, not less). This is an AES-256 encryption requirement. Using `openssl rand -base64 24` produces exactly 32 characters.

**Step 2: Configure OAuth Providers (Optional)**

!!! note "Verify OAuth is actually enabled"
    `kratos.yml.template` defines the `google` and `github` providers with `client_id: disabled`. `docker-compose.yml` passes `GOOGLE_CLIENT_ID` etc. to the Kratos container, but Kratos cannot override list entries such as `providers` from plain environment variables. To enable a provider, put the client ID and secret into the provider entry of the template, restart Kratos, and test the login flow.

**Google OAuth:**

1. Go to [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
2. Create OAuth 2.0 Client ID
3. Set authorized redirect URI: `https://your-domain.com/kratos/self-service/methods/oidc/callback/google`
4. Add to `.env`:

```bash
GOOGLE_CLIENT_ID=your_client_id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your_client_secret
```

**GitHub OAuth:**

1. Go to [GitHub Settings > Developer Settings](https://github.com/settings/developers)
2. Create OAuth App
3. Set authorization callback URL: `https://your-domain.com/kratos/self-service/methods/oidc/callback/github`
4. Add to `.env`:

```bash
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret
```

**Step 3: Configure SMTP (Production)**

For production, configure SMTP for email verification and password recovery (compose maps these to Kratos' `COURIER_SMTP_*`):

```bash
# SendGrid example
SMTP_CONNECTION_URI=smtps://apikey:YOUR_SENDGRID_API_KEY@smtp.sendgrid.net:465
SMTP_FROM_ADDRESS=noreply@yourdomain.com
SMTP_FROM_NAME=OSINT Platform

# AWS SES example
SMTP_CONNECTION_URI=smtps://YOUR_SMTP_USERNAME:YOUR_SMTP_PASSWORD@email-smtp.us-east-1.amazonaws.com:465
```

**Step 4: Start Ory Services**

```bash
# Kratos, its migration, the admin bootstrap and Oathkeeper are in the "auth" profile
docker-compose --profile auth up -d kratos-migrate kratos kratos-init oathkeeper
```

If `INITIAL_ADMIN_EMAIL` is set, `kratos-init` creates an admin identity (`metadata_public.role = admin`) once. With `INITIAL_ADMIN_PASSWORD` empty it prints a recovery link instead. It is idempotent and skips creation when an admin already exists.

**Step 5: Verify Setup**

```bash
# Check Kratos health (published on the host by docker-compose.yml)
curl http://localhost:4433/health/ready

# Oathkeeper's API port (4456) serves the health endpoint; 4455 is the proxy
curl http://localhost:4456/health/ready
```

## Feed Token Authentication

Feed tokens enable **secure RSS/Atom/JSON feed subscriptions** without requiring user login in feed readers.

### How Feed Tokens Work

1. A logged-in user creates a feed token (web UI: Settings -> Feed Tokens)
2. The platform signs the feed URL with the token's secret (HMAC)
3. The feed reader requests the URL with `token` and `sig`
4. The API validates the token (active, not revoked) and recomputes the signature over the query parameters
5. Feed data is returned if valid

**Feed URL Format:**

```
https://your-domain.com/rss/search?q=ukraine&token=<token-id>&sig=<hmac-signature>
```

Feeds live under `/rss/*` (not `/api/rss/*`). Caddy routes `/rss/*` directly to the API, bypassing Oathkeeper.

### Enable Feed Authentication

```bash
# .env (docker-compose.yml defaults to true; .env.example ships true)
FEED_AUTH_REQUIRED=true  # Require token + signature for all feeds
```

With `false`, feeds are public and tokens are optional. A maximum of 10 active tokens per user is enforced.

### Generate Feed Token

Tokens are created by an authenticated user (session cookie, or a bearer token/API key accepted by your auth provider):

```bash
# 1. Create the token. The plaintext token is returned ONCE.
curl -X POST https://your-domain.com/api/feed-tokens \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"label": "My RSS Reader"}'

# 2. Sign a feed URL with that token
curl -X POST https://your-domain.com/api/feed-tokens/<token-id>/sign-url \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"endpoint": "/rss/search", "params": {"q": "ukraine"}}'
```

The response of step 2 contains the full signed `url`. Changing any query parameter invalidates the signature. Tokens carry no scopes.

### Revoke Feed Token

```bash
# Revoke token
curl -X DELETE https://your-domain.com/api/feed-tokens/<token-id> \
  -H "Authorization: Bearer $TOKEN"
```

## Session Management

### Ory Kratos Sessions

Kratos stores identities and sessions in PostgreSQL (the `kratos-migrate` container creates its tables in the platform database); Redis is not used for sessions.

- **Ory Kratos**: browser session cookie `ory_kratos_session`; the lifespan is the Kratos default unless you set `session.lifespan` in `kratos.yml.template`
- **JWT**: access token only, lifetime `JWT_EXPIRATION_MINUTES` (default 60)

Admins can list and revoke a user's Kratos sessions via `GET/DELETE /api/admin/users/{user_id}/sessions`.

### Session Security

**Session Cookies (Ory Kratos):**

- **HttpOnly** - Prevents JavaScript access (XSS protection)
- **Secure** - set when served over HTTPS
- **SameSite=Lax** - CSRF protection (set explicitly in the template)

**JWT Tokens:**

- **Signature** - HMAC (`JWT_ALGORITHM`, default HS256) prevents tampering
- **Expiration** - short-lived access tokens (`JWT_EXPIRATION_MINUTES`)

### API Keys

For scripts and integrations, users can create API keys (`/api/api-keys`; tokens start with `ak_` and are sent as `Authorization: Bearer ak_...`). Scopes: `read`, `write`, `media`, `export`, `admin` (only admins may create `admin`-scope keys). Keys can expire (`expires_in_days`, 1-365) and be revoked with `DELETE /api/api-keys/{id}`. API key authentication has the highest priority in the auth middleware.

## Password Policies

### Ory Kratos Password Requirements

Configured in `infrastructure/kratos/kratos.yml.template` (rendered to `/tmp/kratos.yml` at container start):

- **Password policy**: Kratos defaults (minimum length 8, breach-list check); the template sets no custom policy
- **Hashing algorithm**: bcrypt with cost 12
- **Password history / expiration**: not enforced

### Password Recovery Flow

1. User clicks "Forgot Password" on login page
2. User enters email address
3. Kratos sends verification code via email
4. User enters code to verify identity
5. User sets new password
6. Password is hashed with bcrypt (cost 12)
7. User is logged in with new password

## Multi-Factor Authentication (MFA)

### TOTP (Time-Based One-Time Password)

Ory Kratos supports TOTP for second-factor authentication:

**Enable TOTP:**

1. User navigates to Settings page
2. User clicks "Enable Two-Factor Authentication"
3. Kratos displays QR code
4. User scans QR code with authenticator app (Google Authenticator, Authy)
5. User enters verification code
6. TOTP is enabled

**Login with MFA:**

1. User enters email/password
2. Kratos prompts for TOTP code
3. User enters 6-digit code from authenticator app
4. Session is created

### Backup Codes

When MFA is enabled, Kratos provides recovery codes:

- 10 single-use backup codes
- Used if TOTP device is lost
- Can be regenerated

## OAuth2 Flows

### Google OAuth Flow

```mermaid
sequenceDiagram
    User->>Frontend: Click "Login with Google"
    Frontend->>Kratos: Initiate Google OAuth
    Kratos->>Google: Redirect to Google login
    Google->>User: Show consent screen
    User->>Google: Approve consent
    Google->>Kratos: Return auth code
    Kratos->>Google: Exchange code for token
    Google->>Kratos: Return user profile
    Kratos->>Frontend: Create session + redirect
    Frontend->>API: Request with session cookie
```

### GitHub OAuth Flow

Identical to Google flow, but uses GitHub as identity provider.

## API Authentication Examples

### JWT Authentication

```bash
# Login to get token
TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"analyst1","password":"password"}' \
  | jq -r '.access_token')

# Use token for authenticated requests
curl http://localhost:8000/api/bookmarks/ \
  -H "Authorization: Bearer $TOKEN"
```

When the token expires, log in again.

### Feed Token Authentication

```bash
# Feed URL includes token + HMAC signature
curl "https://your-domain.com/rss/search?q=ukraine&token=550e8400-e29b-41d4-a716-446655440000&sig=a1b2c3d4e5f6..."
```

## Security Best Practices

### Credential Management

!!! danger "Critical Security"
    - Never commit secrets to git
    - Use `.env` file (gitignored)
    - Generate random secrets (256-bit minimum)
    - Rotate credentials regularly

```bash
# Generate strong secrets
openssl rand -hex 32  # JWT_SECRET_KEY
openssl rand -base64 32  # KRATOS_SECRET_COOKIE
openssl rand -base64 24  # KRATOS_SECRET_CIPHER (exactly 32 chars)
```

### HTTPS/TLS Only

**Development:**

HTTP is acceptable for localhost development.

**Production:**

HTTPS is mandatory. Configure Caddy with Let's Encrypt:

```bash
# Caddy automatically obtains TLS certificates
DOMAIN=yourdomain.com
```

### Rate Limiting

Brute-force protection is applied at Caddy (`infrastructure/caddy/Caddyfile.security`, per client IP):

| Zone | Limit | Applies to |
|------|-------|------------|
| `auth_attempts` | 10 per 5 minutes | `/kratos/*` (login, registration, recovery) |
| `api_calls` | 60 per minute | `/api/*` |
| `rss_feeds` | 30 per 2 minutes | `/rss/*` |
| `admin_access` | 20 per 10 minutes | admin routes |

CrowdSec adds behaviour-based bans on top (see [CrowdSec](crowdsec.md)). The API itself rate limits specific endpoints (map, media redirect) via `*_RATE_LIMIT` variables in `services/api/src/utils/rate_limit.py`.

### Audit Logging

Admin actions are recorded in the `admin_audit_log` table; Kratos logs authentication flows and the API logs rejected requests:

```bash
docker-compose logs -f kratos | grep -i "login\|registration"
docker-compose logs -f api | grep -i "auth"
```

Kratos logs at `info` with `leak_sensitive_values: false`. Oathkeeper ships with `log.level: debug` in `infrastructure/oathkeeper/oathkeeper.yml`; lower it to `info` for production.

## Troubleshooting

### JWT Issues

**API fails to start:** `JWT_SECRET_KEY not configured or too short` means the key is missing or under 32 characters.

**Cannot log in as admin:** `JWT_ADMIN_PASSWORD` is not set.

**Token invalid or expired:**

```bash
curl -i http://localhost:8000/auth/users/me -H "Authorization: Bearer $TOKEN"
# 401 -> log in again
```

### Ory Kratos Issues

**Kratos Service Not Starting:**

```bash
# Check Kratos logs
docker-compose logs kratos

# Verify secrets are exactly 32 characters
echo -n "$KRATOS_SECRET_CIPHER" | wc -c
# Should output: 32
```

**Email Verification Not Sending:**

```bash
# Check SMTP configuration
docker-compose logs kratos | grep smtp

# With the default SMTP URI (smtp://mailslurper:1025) no mail server exists;
# set SMTP_CONNECTION_URI to a real SMTP server
```

**OAuth Provider Not Working:**

```bash
# Verify client ID and secret
grep GOOGLE_CLIENT_ID .env
grep GITHUB_CLIENT_ID .env

# Check Kratos OAuth configuration
docker-compose exec kratos grep -A 10 oidc /tmp/kratos.yml
```

### Feed Token Issues

**Feed URL Invalid:**

```bash
# Regenerate feed URL in account settings
# Old signatures become invalid when token is rotated
```

**Feed Token Revoked:**

```bash
# Check token status
curl https://your-domain.com/api/feed-tokens \
  -H "Authorization: Bearer $TOKEN"
```

## Migration Between Auth Providers

### None → JWT

```bash
# 1. Update .env
AUTH_PROVIDER=jwt
AUTH_REQUIRED=true
JWT_SECRET_KEY=$(openssl rand -hex 32)
JWT_ADMIN_PASSWORD=<password>

# 2. Recreate the API so .env changes are applied
docker-compose up -d api
```

### JWT → Ory

```bash
# 1. Generate Ory secrets
KRATOS_SECRET_COOKIE=$(openssl rand -base64 32)
KRATOS_SECRET_CIPHER=$(openssl rand -base64 24)

# 2. Update .env
AUTH_PROVIDER=ory
AUTH_REQUIRED=true
KRATOS_SECRET_COOKIE=$KRATOS_SECRET_COOKIE
KRATOS_SECRET_CIPHER=$KRATOS_SECRET_CIPHER

# 3. Also set DEPLOYMENT_MODE, KRATOS_BROWSER_URL, INITIAL_ADMIN_EMAIL, and rebuild the
#    frontend (AUTH_PROVIDER is baked in as NEXT_PUBLIC_AUTH_PROVIDER at build time)
docker-compose --profile auth up -d --build frontend

# 4. Users must re-register (password migration not supported)
```

## Developer Guide: User Identity in Database

When building features that need to track user ownership (bookmarks, tokens, audit logs), you must use the correct identity pattern.

### The Kratos Identity UUID Pattern

!!! warning "Critical: Use kratos_identity_id, NOT users.id"
    The database still contains a **legacy `users` table** with integer IDs; it should NOT be used for new features. Always use `kratos_identity_id` (UUID) directly.

**Correct Pattern (new tables):**

```sql
CREATE TABLE my_feature (
    id SERIAL PRIMARY KEY,
    kratos_identity_id UUID NOT NULL,  -- ✅ Correct: Kratos identity UUID
    -- ... other columns
);

CREATE INDEX idx_my_feature_user ON my_feature(kratos_identity_id);
```

**Incorrect Pattern (legacy - DO NOT USE):**

```sql
CREATE TABLE my_feature (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),  -- ❌ Wrong: Legacy table reference
    -- ... other columns
);
```

### Why This Matters

The legacy `users` table was designed for local authentication with password hashing. With Ory Kratos, user identities are managed externally and identified by UUIDs.

**Converting a UUID to an integer causes overflow errors:**

```python
# ❌ WRONG - This causes int32 overflow!
user_id = int(kratos_user_id)  # UUID → huge 128-bit integer

# ✅ CORRECT - Use UUID directly
kratos_identity_id = user.user_id  # Already a UUID
```

### Tables Using the Correct Pattern

These tables correctly use `kratos_identity_id`:

| Table | Purpose |
|-------|---------|
| `user_roles` | Application roles per Kratos identity |
| `user_bookmarks` | User-saved messages |
| `user_comments` | User comments on messages |
| `admin_audit_log` | Admin action audit trail |
| `feed_tokens` | RSS/Atom feed authentication |
| `api_keys` | Programmatic API access |

### ORM Model Pattern

When creating SQLAlchemy models:

```python
from uuid import UUID
from sqlalchemy.dialects.postgresql import UUID as PG_UUID

class MyFeature(Base):
    __tablename__ = "my_feature"

    id: Mapped[int] = mapped_column(primary_key=True)

    # ✅ Correct: UUID column for Kratos identity
    kratos_identity_id: Mapped[UUID] = mapped_column(
        PG_UUID(as_uuid=True),
        nullable=False,
        index=True,
    )
```

### Router/Service Pattern

When accessing the current user in FastAPI routes:

```python
from ..dependencies.auth import AuthenticatedUser

@router.get("/my-items")
async def list_items(user: AuthenticatedUser, db: AsyncSession = Depends(get_db)):
    # ✅ Correct: Use user.user_id directly (already a UUID)
    items = await service.get_user_items(user.user_id)

    # ❌ Wrong: Never convert to int
    # items = await service.get_user_items(int(user.user_id))
```

## Related Documentation

- [Authorization Guide](authorization.md) - Role-based access control
- [Database Schema](../developer-guide/database-schema.md) - Table definitions
- [API Endpoint Protection](https://github.com/osintukraine/osint-intelligence-platform/blob/master/services/api/ENDPOINT_PROTECTION.md) - Detailed endpoint security
- [Configuration Reference](../operator-guide/configuration.md) - Environment variables

---

!!! tip "Choose the Right Auth Mode"
    - **Development**: Use `AUTH_PROVIDER=none` for quick iteration
    - **Small Teams**: Use `AUTH_PROVIDER=jwt` for simple setup
    - **Production**: Use `AUTH_PROVIDER=ory` for enterprise-grade security
