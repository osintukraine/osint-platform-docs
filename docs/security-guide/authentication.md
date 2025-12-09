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
JWT_SECRET_KEY=<generate-random-256-bit-key>
JWT_ALGORITHM=HS256
JWT_EXPIRATION_MINUTES=60
JWT_REFRESH_EXPIRATION_DAYS=30
```

**How It Works:**

1. User logs in with email/password via `/api/auth/login`
2. API returns JWT access token + refresh token
3. Frontend stores token in localStorage
4. All API requests include `Authorization: Bearer <token>` header
5. API validates JWT signature and expiration

**Generate JWT Secret:**

```bash
# Generate strong 256-bit secret key
openssl rand -hex 32
```

**Features:**

- Simple setup (no external dependencies)
- Stateless (no session storage required)
- Fast validation (HMAC signature check)
- Refresh token support

**Limitations:**

- Manual user management
- No password recovery flow
- No OAuth2 providers
- No MFA support

### 3. Cloudron Mode (Cloudron App Store)

**OAuth authentication** using Cloudron's built-in identity provider.

```bash
# .env configuration (automatically set by Cloudron)
AUTH_PROVIDER=cloudron
AUTH_REQUIRED=true
```

**How It Works:**

1. Cloudron authenticates user
2. Cloudron injects headers: `Cloudron-UserId`, `Cloudron-Email`
3. API reads headers and creates session

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

# OAuth providers (optional)
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret

# SMTP (required for production)
SMTP_CONNECTION_URI=smtps://apikey:YOUR_SENDGRID_API_KEY@smtp.sendgrid.net:465
SMTP_FROM_ADDRESS=noreply@yourdomain.com
SMTP_FROM_NAME=OSINT Platform
```

**Architecture:**

```
User → Frontend → Ory Kratos (authentication) → Session Cookie
                                                      ↓
                                              Ory Oathkeeper (access proxy)
                                                      ↓
                                                   API Server
```

**Features:**

- **Email/password authentication** with bcrypt (cost 12)
- **OAuth2 providers** (Google, GitHub)
- **Password recovery** via email verification codes
- **Email verification** required for registration
- **MFA support** (TOTP, backup codes)
- **Session management** with secure cookies
- **Account settings UI** (password change, MFA setup)
- **Self-service flows** (registration, login, recovery)

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

**Step 3: Create Admin User**

```bash
# Use API to create admin user (development)
curl -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@localhost",
    "password": "change-this-password",
    "role": "admin"
  }'
```

**Step 4: Test Login**

```bash
# Get JWT token
TOKEN=$(curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@localhost","password":"change-this-password"}' \
  | jq -r '.access_token')

# Test authenticated endpoint
curl http://localhost:8000/api/user/me \
  -H "Authorization: Bearer $TOKEN"
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

**Google OAuth:**

1. Go to [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
2. Create OAuth 2.0 Client ID
3. Set authorized redirect URI: `http://localhost:4433/self-service/methods/oidc/callback/google`
4. Add to `.env`:

```bash
GOOGLE_CLIENT_ID=your_client_id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your_client_secret
```

**GitHub OAuth:**

1. Go to [GitHub Settings > Developer Settings](https://github.com/settings/developers)
2. Create OAuth App
3. Set authorization callback URL: `http://localhost:4433/self-service/methods/oidc/callback/github`
4. Add to `.env`:

```bash
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret
```

**Step 3: Configure SMTP (Production)**

For production, configure SMTP for email verification and password recovery:

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
# Start Kratos and Oathkeeper with auth profile
docker-compose --profile auth up -d kratos oathkeeper
```

**Step 5: Verify Setup**

```bash
# Check Kratos health
curl http://localhost:4433/health/ready

# Check Oathkeeper health
curl http://localhost:4455/health/ready
```

## Feed Token Authentication

Feed tokens enable **secure RSS/Atom/JSON feed subscriptions** without requiring user login in feed readers.

### How Feed Tokens Work

1. User generates feed token in account settings
2. Platform creates HMAC-signed feed URL with token
3. Feed reader makes request with token + signature
4. API validates HMAC signature and token status
5. Feed data is returned if valid

**Feed URL Format:**

```
http://localhost:8000/api/rss/search?q=ukraine&token=<token-id>&sig=<hmac-signature>
```

### Enable Feed Authentication

```bash
# Add to .env
FEED_AUTH_REQUIRED=true  # Require tokens for all feeds
```

### Generate Feed Token

```bash
# Via API (authenticated user)
curl -X POST http://localhost:8000/api/feed-tokens \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My RSS Reader",
    "description": "Personal feed subscription",
    "scopes": ["read:feeds"]
  }'
```

**Response:**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "My RSS Reader",
  "token_prefix": "ft_1234",
  "created_at": "2025-01-01T00:00:00Z",
  "last_used_at": null,
  "feed_url_template": "http://localhost:8000/api/rss/search?token={token}&sig={signature}&q={query}"
}
```

### Revoke Feed Token

```bash
# Revoke token
curl -X DELETE http://localhost:8000/api/feed-tokens/<token-id> \
  -H "Authorization: Bearer $TOKEN"
```

## Session Management

### Redis-Based Sessions

Sessions are stored in Redis for fast access and automatic expiration:

```bash
# Redis configuration in .env
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=<strong-random-password>
REDIS_DB=0
```

**Session Expiration:**

- **Ory Kratos**: 24 hours (configurable in `kratos.yml`)
- **JWT**: 60 minutes access token + 30 days refresh token

### Session Security

**Session Cookies (Ory Kratos):**

- **HttpOnly** - Prevents JavaScript access (XSS protection)
- **Secure** - HTTPS only (production)
- **SameSite=Lax** - CSRF protection
- **Signed** - HMAC signature prevents tampering

**JWT Tokens:**

- **Signature** - HMAC-SHA256 prevents tampering
- **Expiration** - Short-lived access tokens (60 minutes)
- **Refresh tokens** - Stored HttpOnly cookies (not localStorage)

## Password Policies

### Ory Kratos Password Requirements

Configured in `/home/rick/code/osintukraine/osint-intelligence-platform/infrastructure/kratos/kratos.yml`:

- **Minimum length**: 8 characters
- **Hashing algorithm**: bcrypt with cost 12
- **Password history**: Not enforced (can be added)
- **Expiration**: No automatic expiration

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
TOKEN=$(curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password"}' \
  | jq -r '.access_token')

# Use token for authenticated requests
curl http://localhost:8000/api/bookmarks \
  -H "Authorization: Bearer $TOKEN"

# Refresh token when expired
REFRESH_TOKEN=$(curl -X POST http://localhost:8000/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token":"<your-refresh-token>"}' \
  | jq -r '.access_token')
```

### Feed Token Authentication

```bash
# Feed URL includes token + HMAC signature
curl "http://localhost:8000/api/rss/search?q=ukraine&token=550e8400-e29b-41d4-a716-446655440000&sig=a1b2c3d4e5f6..."
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

Protect authentication endpoints from brute force:

```bash
# API rate limiting (in .env)
API_RATE_LIMIT_PER_MINUTE=60
API_RATE_LIMIT_BURST=100
```

### Audit Logging

Enable authentication event logging:

```bash
# All authentication events are logged
docker-compose logs -f api | grep authentication
```

**Logged Events:**

- Successful logins
- Failed login attempts
- Password changes
- MFA enrollment
- Session creation/destruction
- Token generation/revocation

## Troubleshooting

### JWT Issues

**Token Invalid:**

```bash
# Verify JWT_SECRET_KEY matches in .env
grep JWT_SECRET_KEY .env

# Check token expiration
curl http://localhost:8000/api/auth/verify \
  -H "Authorization: Bearer $TOKEN"
```

**Token Expired:**

```bash
# Use refresh token to get new access token
curl -X POST http://localhost:8000/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token":"<refresh-token>"}'
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

# Test with MailSlurper (development)
# View emails at http://localhost:4436
```

**OAuth Provider Not Working:**

```bash
# Verify client ID and secret
grep GOOGLE_CLIENT_ID .env
grep GITHUB_CLIENT_ID .env

# Check Kratos OAuth configuration
docker-compose exec kratos cat /etc/config/kratos/kratos.yml | grep -A 10 oidc
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
curl http://localhost:8000/api/feed-tokens \
  -H "Authorization: Bearer $TOKEN"
```

## Migration Between Auth Providers

### None → JWT

```bash
# 1. Generate JWT secret
JWT_SECRET=$(openssl rand -hex 32)

# 2. Update .env
AUTH_PROVIDER=jwt
AUTH_REQUIRED=true
JWT_SECRET_KEY=$JWT_SECRET

# 3. Restart API
docker-compose restart api

# 4. Create admin user
curl -X POST http://localhost:8000/api/auth/register \
  -d '{"email":"admin@localhost","password":"<password>","role":"admin"}'
```

### JWT → Ory

```bash
# 1. Generate Ory secrets
KRATOS_SECRET_COOKIE=$(openssl rand -base64 32)
KRATOS_SECRET_CIPHER=$(openssl rand -base64 24)

# 2. Update .env
AUTH_PROVIDER=ory
KRATOS_SECRET_COOKIE=$KRATOS_SECRET_COOKIE
KRATOS_SECRET_CIPHER=$KRATOS_SECRET_CIPHER

# 3. Start Ory services
docker-compose --profile auth up -d

# 4. Users must re-register (password migration not supported)
```

## Related Documentation

- [Authorization Guide](authorization.md) - Role-based access control
- [API Endpoint Protection](https://github.com/osintukraine/osint-intelligence-platform/blob/master/services/api/ENDPOINT_PROTECTION.md) - Detailed endpoint security
- [Configuration Reference](../operator-guide/configuration.md) - Environment variables

---

!!! tip "Choose the Right Auth Mode"
    - **Development**: Use `AUTH_PROVIDER=none` for quick iteration
    - **Small Teams**: Use `AUTH_PROVIDER=jwt` for simple setup
    - **Production**: Use `AUTH_PROVIDER=ory` for enterprise-grade security
