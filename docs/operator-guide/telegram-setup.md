# Telegram Setup

Configure Telegram API access, session management, and channel monitoring for the OSINT Intelligence Platform.

## Overview

The platform monitors Telegram channels for intelligence collection. This guide covers obtaining API credentials, creating sessions, configuring multi-account setups, and managing channels through Telegram folders.

**Key Concepts**:

- **API Credentials**: Required from my.telegram.org for bot/user access
- **Session Files**: Persistent authentication state (avoid re-login)
- **Multi-Account**: Scale with separate Russia/Ukraine Telegram accounts
- **Folder-Based Management**: Organize channels by Telegram folders (Archive-*, Monitor-*, Discover-*)

## Prerequisites

Before starting, you need:

- [ ] A Telegram account (phone number required)
- [ ] Access to https://my.telegram.org
- [ ] Ability to receive SMS/calls for verification
- [ ] Docker Compose environment running

## Obtaining Telegram API Credentials

### Step 1: Register Application

1. **Visit Telegram API portal**:
   - Go to https://my.telegram.org
   - Click "API development tools"

2. **Log in with your phone number**:
   - Enter your phone number with country code (e.g., `+1234567890`)
   - You'll receive a verification code via SMS or call

3. **Create Application**:
   - Click "Create a new application"
   - Fill in application details:
     - **App title**: `OSINT Intelligence Platform`
     - **Short name**: `osint-platform`
     - **URL**: `https://github.com/osintukraine/osint-intelligence-platform` (or your fork)
     - **Platform**: `Desktop`
   - Submit the form

4. **Copy Credentials**:
   - **API ID**: 8-digit number (e.g., `12345678`)
   - **API Hash**: 32-character hex string (e.g., `a1b2c3d4e5f6...`)

### Step 2: Configure Environment Variables

Add credentials to `.env` file:

```bash
# Single Account Mode (Default)
TELEGRAM_API_ID=12345678
TELEGRAM_API_HASH=a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
TELEGRAM_PHONE=+1234567890              # Your phone number with country code
```

**CRITICAL**: Keep these credentials secret. Never commit to git.

### Security Best Practices

- **Use a dedicated account**: Don't use your personal Telegram account
- **Limit application access**: Only grant necessary permissions
- **Rotate credentials**: If compromised, delete application and create new one
- **Backup session files**: Sessions are as valuable as passwords

## Creating Telegram Sessions

### Single Account Session

#### Option 1: Interactive Session Creation (Recommended)

```bash
# Start session creation process
docker-compose run --rm listener python -c "
from telethon import TelegramClient
import os

api_id = int(os.getenv('TELEGRAM_API_ID'))
api_hash = os.getenv('TELEGRAM_API_HASH')
phone = os.getenv('TELEGRAM_PHONE')

client = TelegramClient('/app/sessions/osint_platform', api_id, api_hash)
client.start(phone=phone)
print('Session created successfully!')
client.disconnect()
"
```

**Interactive Prompts**:

1. **Phone Number**: Enter phone (with +country code)
2. **Verification Code**: Enter code from Telegram app
3. **2FA Password** (if enabled): Enter your 2FA password
4. **Success**: Session saved to `telegram_sessions/osint_platform.session`

#### Option 2: Using Telegram Auth Script

```bash
# Run authentication script
python3 scripts/telegram_auth.py

# Follow prompts:
# Enter phone number: +1234567890
# Enter verification code: 12345
# Enter 2FA password (if required): ******
```

### Multi-Account Sessions

For scaled deployments with separate Russia/Ukraine accounts:

#### Step 1: Configure Multi-Account Credentials

Add to `.env`:

```bash
# Russia account
TELEGRAM_API_ID_RUSSIA=11111111
TELEGRAM_API_HASH_RUSSIA=a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1
TELEGRAM_PHONE_RUSSIA=+1234567890

# Ukraine account
TELEGRAM_API_ID_UKRAINE=22222222
TELEGRAM_API_HASH_UKRAINE=b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2b2
TELEGRAM_PHONE_UKRAINE=+0987654321
```

#### Step 2: Authenticate Each Account

```bash
# Authenticate Russia account
python3 scripts/telegram_auth.py --account russia
# Enter phone: +1234567890
# Enter code: 12345

# Authenticate Ukraine account
python3 scripts/telegram_auth.py --account ukraine
# Enter phone: +0987654321
# Enter code: 54321
```

**Session Files Created**:
- `telegram_sessions/listener_russia.session`
- `telegram_sessions/listener_ukraine.session`

#### Step 3: Start Multi-Account Listeners

```bash
# Start both listeners
docker-compose --profile multi-account up -d listener-russia listener-ukraine

# Verify running
docker-compose ps | grep listener

# Expected output:
# osint-listener-russia    running   8011/tcp
# osint-listener-ukraine   running   8012/tcp

# (Optional) Stop default single-account listener
docker-compose stop listener
```

### Session File Structure

```bash
telegram_sessions/
├── osint_platform.session          # Single account (default)
├── listener_russia.session         # Russia account
├── listener_ukraine.session        # Ukraine account
└── enrichment.session              # Enrichment worker (separate to avoid SQLite locks)
```

**Never delete session files** - they contain authentication state. Losing them requires re-authentication and may trigger security checks.

## Session Management

### Verifying Session Health

```bash
# Check if session is valid
docker-compose exec listener python -c "
from telethon import TelegramClient
import os

client = TelegramClient('/app/sessions/osint_platform',
                       int(os.getenv('TELEGRAM_API_ID')),
                       os.getenv('TELEGRAM_API_HASH'))

async def check():
    await client.start(phone=os.getenv('TELEGRAM_PHONE'))
    me = await client.get_me()
    print(f'Session valid for: {me.first_name} (@{me.username})')
    await client.disconnect()

import asyncio
asyncio.run(check())
"
```

Expected output:
```
Session valid for: John (@johndoe)
```

### Session Expiration

Telegram sessions can expire due to:

- **Long inactivity**: Sessions may expire after ~1 year of no use
- **Security triggers**: Suspicious activity, password changes, or account compromise
- **Manual logout**: Logging out from Telegram app revokes all sessions

**Symptoms of Expired Session**:
- `AuthKeyUnregistered` error in logs
- `SessionPasswordNeeded` error
- Listener service exits immediately

**Solution**: Re-authenticate:

```bash
# Remove old session
rm telegram_sessions/osint_platform.session

# Create new session (follow prompts)
docker-compose run --rm listener python -m telethon_session_creator
```

### Session Security

**Protect session files like passwords**:

```bash
# Set strict permissions (owner read/write only)
chmod 600 telegram_sessions/*.session

# Backup sessions securely
tar -czf sessions-backup-$(date +%Y%m%d).tar.gz telegram_sessions/
gpg -c sessions-backup-$(date +%Y%m%d).tar.gz  # Encrypt with password
rm sessions-backup-$(date +%Y%m%d).tar.gz      # Remove unencrypted backup
```

## Folder-Based Channel Management

The platform uses Telegram app folders to organize and manage channels. No database configuration required.

### Folder Patterns

| Folder Pattern | Rule | Behavior |
|----------------|------|----------|
| `Archive-*` | `archive_all` | Store all non-spam messages |
| `Monitor-*` | `selective_archive` | Only messages with OSINT score ≥ 70 |
| `Discover-*` | `discovery` | Auto-joined channels, 14-day probation |

**Examples**:
- `Archive-Russia` - Archive all messages from Russian military channels
- `Archive-Ukraine` - Archive all messages from Ukrainian OSINT channels
- `Monitor-Important` - Only high-value messages from news channels
- `Discover-New` - Recently discovered channels (auto-pruned after 14 days if inactive)

### Creating Folders in Telegram

#### Desktop App (Telegram Desktop)

1. Open Telegram Desktop
2. Click "Settings" → "Folders"
3. Click "Create New Folder"
4. Name folder (e.g., `Archive-Russia`)
5. Add channels to folder:
   - Click "Add Chats"
   - Select channels to monitor
   - Click "Done"
6. Save folder

#### Mobile App (Android/iOS)

1. Open Telegram app
2. Tap "Settings" → "Folders"
3. Tap "Create New Folder"
4. Name folder (e.g., `Archive-Ukraine`)
5. Tap "Add Chats"
6. Select channels
7. Tap "Done"

### Folder Sync Process

The listener service automatically syncs folders every 5 minutes:

```bash
# Check folder sync interval (default: 300 seconds)
grep FOLDER_SYNC_INTERVAL .env
# FOLDER_SYNC_INTERVAL=300
```

**Sync Workflow**:

1. Listener reads all folders from Telegram account
2. Matches folder names against patterns (`Archive-*`, `Monitor-*`, `Discover-*`)
3. Creates/updates channels in database
4. Assigns intelligence rules based on folder pattern
5. Starts monitoring matched channels

**View sync logs**:

```bash
docker-compose logs listener | grep -i folder

# Expected output:
# INFO: Discovered folder: Archive-Russia (12 channels)
# INFO: Discovered folder: Monitor-Important (3 channels)
# INFO: Assigned rule archive_all to 12 channels in Archive-Russia
```

### Managing Channels

#### Adding Channels

**Method 1: Add to Telegram Folder**

1. Join channel in Telegram app
2. Add channel to folder (e.g., `Archive-Russia`)
3. Wait for sync (max 5 minutes)
4. Verify in logs:

```bash
docker-compose logs listener | grep "Assigned rule"
```

**Method 2: Auto-Discovery (Discover-* folders)**

1. Create `Discover-New` folder
2. Platform auto-joins channels found in forward chains
3. Channels stay for 14-day probation
4. Move to `Archive-*` or `Monitor-*` to keep permanently

#### Removing Channels

**Method 1: Remove from Folder**

1. Open Telegram app
2. Remove channel from folder
3. Wait for sync (max 5 minutes)
4. Channel monitoring stops automatically

**Method 2: Database Cleanup (for stale channels)**

```bash
# View stale channels (no messages in 30+ days)
docker-compose exec postgres psql -U osint_user -d osint_platform -c "
SELECT name, last_message_at
FROM channels
WHERE last_message_at < NOW() - INTERVAL '30 days'
ORDER BY last_message_at;
"

# Disable stale channels (mark inactive, stop monitoring)
docker-compose exec postgres psql -U osint_user -d osint_platform -c "
UPDATE channels
SET is_active = false
WHERE last_message_at < NOW() - INTERVAL '30 days';
"
```

### Folder-Specific Intelligence Rules

#### Archive-All Rule (Archive-* folders)

Stores **all non-spam messages** regardless of OSINT score:

- **Spam filter**: YES (filters donation scams, off-topic)
- **OSINT scoring**: YES (for analytics)
- **Archival threshold**: 0 (all non-spam archived)

**Use cases**:
- Critical primary sources (Ukrainian military, Russian MoD)
- High-signal channels (DeepStateUA, Rybar)
- Historical archiving (preserve everything for research)

#### Selective Archive Rule (Monitor-* folders)

Stores **only high-value messages** (OSINT score ≥ 70):

- **Spam filter**: YES
- **OSINT scoring**: YES
- **Archival threshold**: 70 (configurable via `MONITORING_OSINT_THRESHOLD`)

**Use cases**:
- News aggregators (too noisy for archive-all)
- Secondary sources (only significant updates)
- Test channels (evaluate before archiving)

**Adjust threshold**:

```bash
# In .env file
MONITORING_OSINT_THRESHOLD=70  # Default: 70
# Lower = more messages archived (e.g., 50)
# Higher = fewer messages archived (e.g., 85)

# Restart processor to apply
docker-compose restart processor-worker
```

#### Discovery Rule (Discover-* folders)

Auto-joins and evaluates channels for 14 days:

- **Auto-join**: YES (from forward chains)
- **Probation period**: 14 days
- **Auto-removal**: After 14 days if inactive or spam

**Workflow**:

1. Platform detects new channel in forwards
2. Auto-joins and adds to `Discover-*` folder
3. Monitors for 14 days
4. **Decision after 14 days**:
   - **Active + valuable**: Move to `Archive-*` or `Monitor-*`
   - **Inactive or spam**: Auto-removed

**View discovery candidates**:

```bash
docker-compose exec postgres psql -U osint_user -d osint_platform -c "
SELECT name, folder, message_count, created_at
FROM channels
WHERE folder LIKE 'Discover-%'
ORDER BY created_at DESC;
"
```

## Multi-Account Architecture

### When to Use Multi-Account

**Use multi-account if**:

- Monitoring **200+ channels** (approaching single-account rate limits)
- Need **higher throughput** (double the API quota)
- Want **risk isolation** (one account banned doesn't stop everything)
- Have **organizational split** (separate Russia/Ukraine teams)

**Stick to single-account if**:

- Monitoring **<200 channels**
- API rate limits not a problem
- Simpler operations preferred

### Account Assignment by Folder

| Account | Folders | Use Case |
|---------|---------|----------|
| **Default** | `Archive-*`, `Monitor-*`, `Discover-*` | Neutral channels, general OSINT |
| **Russia** | `Archive-RU-*`, `Monitor-RU-*`, `Discover-RU` | Russian military, state media |
| **Ukraine** | `Archive-UA-*`, `Monitor-UA-*`, `Discover-UA` | Ukrainian military, government |

**Example Folder Structure**:

```
Default Account:
- Archive-Neutral (international news)
- Monitor-Analysis (OSINT analysts)

Russia Account:
- Archive-RU-Military (Russian MoD, Wagner)
- Archive-RU-Propaganda (RT, TASS)
- Monitor-RU-Regional (regional channels)

Ukraine Account:
- Archive-UA-Military (Armed Forces, DeepState)
- Archive-UA-Government (Ukrainian officials)
- Monitor-UA-Regional (oblast channels)
```

### Multi-Account Performance

**Rate Limits** (per account):

- **Message fetching**: 20 requests/second per account
- **Channel joins**: 20 joins/day per account
- **Downloads**: Unlimited (but throttled by Telegram)

**With 2 accounts** (Russia + Ukraine):
- **Effective throughput**: 40 requests/second
- **Channel capacity**: 400+ channels monitored
- **Fault tolerance**: 50% capacity if one account down

## Rate Limiting & Flood Control

Telegram enforces strict API rate limits. The platform handles this automatically.

### Rate Limit Types

#### 1. Global Rate Limit

**Limit**: 20 requests/second per account

**Platform Handling**:
- Worker pool distributes requests across workers
- Per-channel rate limiting (1 request per 2 seconds)
- Redis-based distributed rate limiter

**Configuration**:

```bash
# In .env (defaults are safe)
TELEGRAM_RATE_LIMIT_PER_CHANNEL=20  # Messages per minute
```

#### 2. FloodWait Errors

**Symptom**: `FloodWaitError: Must wait 300 seconds before next request`

**Platform Handling**:
- Automatic backoff with exponential delay
- Re-queue message for later processing
- Continue processing other channels (worker pool isolates failures)

**View FloodWait in logs**:

```bash
docker-compose logs listener | grep FloodWait

# Example output:
# WARNING: Channel @example flood wait 300s, re-queuing
```

**Configuration**:

```bash
# Flood wait multiplier (backoff aggressiveness)
TELEGRAM_FLOOD_WAIT_MULTIPLIER=2  # Default: 2 (doubles wait time each retry)
```

#### 3. Channel Join Limits

**Limit**: 20 joins/day per account

**Platform Handling**:
- Discovery feature only joins high-potential channels
- Manual approval workflow for new channels
- Join queue with daily limit tracking

**View join queue**:

```bash
docker-compose exec postgres psql -U osint_user -d osint_platform -c "
SELECT name, join_attempted_at, join_status
FROM channels
WHERE join_status = 'pending'
ORDER BY join_attempted_at;
"
```

### Handling Rate Limit Errors

#### Temporary Rate Limit

**Error**: `FloodWaitError: A wait of X seconds is required`

**Action**: Wait and retry (automatic)

```bash
# Platform handles automatically, but you can monitor:
docker-compose logs listener | grep -i "wait\|flood"
```

#### Permanent Ban

**Error**: `UserDeactivatedError: The user has been deleted/deactivated`

**Action**: Account is banned, must use different account

```bash
# Check if account is valid
docker-compose exec listener python -c "
from telethon import TelegramClient
import os

async def check():
    client = TelegramClient('/app/sessions/osint_platform',
                           int(os.getenv('TELEGRAM_API_ID')),
                           os.getenv('TELEGRAM_API_HASH'))
    try:
        await client.start(phone=os.getenv('TELEGRAM_PHONE'))
        print('Account active')
    except Exception as e:
        print(f'Account error: {e}')
    await client.disconnect()

import asyncio
asyncio.run(check())
"
```

### Rate Limit Best Practices

1. **Use folder-based management** - Avoid manual channel additions (triggers rate limits)
2. **Enable worker pool** - Isolates failures, continues processing other channels
3. **Monitor FloodWait frequency** - If frequent, reduce `TELEGRAM_RATE_LIMIT_PER_CHANNEL`
4. **Use multi-account** - Doubles effective rate limit
5. **Avoid bulk operations** - Spread channel additions over days, not hours

## Historical Backfill

Fetch historical messages from channels (e.g., messages from Feb 24, 2022 to now).

### Backfill Configuration

```bash
# Enable backfill
BACKFILL_ENABLED=true

# Start date (ISO format: YYYY-MM-DD)
BACKFILL_START_DATE=2022-02-24    # Day of Russian invasion

# Backfill mode
BACKFILL_MODE=manual              # manual, on_discovery, scheduled

# Rate-limit friendly settings
BACKFILL_BATCH_SIZE=100           # Messages per batch
BACKFILL_DELAY_MS=1000            # 1 second between batches

# Media handling
BACKFILL_MEDIA_STRATEGY=download_available  # download_available, skip, download_all
```

### Backfill Modes

#### Manual Backfill (Recommended)

Start backfill via API call:

```bash
# Trigger backfill for specific channel
curl -X POST http://localhost:8000/api/admin/backfill \
  -H "Content-Type: application/json" \
  -d '{
    "channel_id": 12345,
    "start_date": "2024-01-01"
  }'
```

#### On-Discovery Backfill

Automatically backfill when channel added to folder:

```bash
BACKFILL_MODE=on_discovery
```

**Workflow**:
1. Add channel to `Archive-*` folder
2. Listener detects new channel
3. Automatically starts backfill from `BACKFILL_START_DATE`

**WARNING**: Can trigger rate limits if adding many channels at once. Use manual mode for bulk additions.

### Backfill Performance

**Example**: Backfill 1 year of messages from active channel

- **Messages**: ~10,000 messages (30 msgs/day average)
- **Batch size**: 100 messages
- **Delay**: 1 second between batches
- **Estimated time**: 100 batches × 1 second = **~2 minutes**

**With media download**:
- **+Media time**: ~5-10 seconds per batch (network dependent)
- **Total time**: **~10-15 minutes**

### Monitoring Backfill Progress

```bash
# View backfill status
docker-compose exec postgres psql -U osint_user -d osint_platform -c "
SELECT name, backfill_status, backfill_progress, backfill_started_at
FROM channels
WHERE backfill_status = 'in_progress';
"

# View backfill logs
docker-compose logs listener | grep -i backfill
```

### Backfill Errors

#### Media Not Available

**Error**: `404: Media not found (Telegram expired URLs)`

**Action**: Set `BACKFILL_MEDIA_STRATEGY=download_available`

This gracefully handles expired media:
- Try to download
- If 404, mark as `media_unavailable=true`
- Continue backfill (don't fail)

#### Rate Limit During Backfill

**Error**: `FloodWaitError: Must wait 300 seconds`

**Action**: Increase `BACKFILL_DELAY_MS`

```bash
# More conservative (slower but safer)
BACKFILL_DELAY_MS=2000  # 2 seconds between batches
```

## Troubleshooting

### Session Creation Fails

**Error**: `RuntimeError: No running event loop`

**Solution**: Use `asyncio.run()` wrapper

```bash
docker-compose run --rm listener python -c "
import asyncio
from telethon import TelegramClient
import os

async def create_session():
    client = TelegramClient('/app/sessions/osint_platform',
                           int(os.getenv('TELEGRAM_API_ID')),
                           os.getenv('TELEGRAM_API_HASH'))
    await client.start(phone=os.getenv('TELEGRAM_PHONE'))
    print('Session created!')
    await client.disconnect()

asyncio.run(create_session())
"
```

### Verification Code Not Received

**Problem**: SMS/call not arriving

**Solutions**:

1. **Check phone number format**: Must include country code (`+1234567890`)
2. **Try calling instead**: Telegram offers voice call option
3. **Wait 5 minutes**: Telegram throttles verification requests
4. **Use different account**: Try another phone number

### Channel Not Monitored

**Problem**: Channel in folder but not showing in database

**Diagnostics**:

```bash
# Check folder sync
docker-compose logs listener | grep -i "folder\|channel"

# Verify channel exists
docker-compose exec postgres psql -U osint_user -d osint_platform -c "
SELECT name, folder, is_active
FROM channels
WHERE name LIKE '%channel_name%';
"

# Force folder re-sync
docker-compose restart listener
```

### FloodWait Errors Frequent

**Problem**: `FloodWaitError` appearing multiple times per hour

**Solutions**:

1. **Reduce request rate**:

```bash
TELEGRAM_RATE_LIMIT_PER_CHANNEL=10  # Lower from 20
```

2. **Enable multi-account** (doubles capacity):

```bash
# Configure second account in .env
TELEGRAM_API_ID_UKRAINE=...
# ... follow multi-account setup
```

3. **Check for misbehaving channels**:

```bash
# View channels with highest message volume
docker-compose exec postgres psql -U osint_user -d osint_platform -c "
SELECT name, message_count, last_message_at
FROM channels
ORDER BY message_count DESC
LIMIT 10;
"

# Move high-volume channels to Monitor-* (selective archival)
```

### Session Expired

**Error**: `AuthKeyUnregistered` or `SessionPasswordNeeded`

**Solution**: Re-authenticate

```bash
# Remove old session
rm telegram_sessions/osint_platform.session

# Create new session
docker-compose run --rm listener python -c "
from telethon import TelegramClient
import os

async def auth():
    client = TelegramClient('/app/sessions/osint_platform',
                           int(os.getenv('TELEGRAM_API_ID')),
                           os.getenv('TELEGRAM_API_HASH'))
    await client.start(phone=os.getenv('TELEGRAM_PHONE'))
    print('Re-authenticated successfully')
    await client.disconnect()

import asyncio
asyncio.run(auth())
"

# Restart listener
docker-compose restart listener
```

## Next Steps

- [Configuration](configuration.md) - Environment variables and service configuration
- [Monitoring](monitoring.md) - Set up monitoring and alerts
- [Backup & Restore](backup-restore.md) - Backup Telegram sessions
- [Troubleshooting](troubleshooting.md) - Additional Telegram-related issues

## References

- Telegram API Documentation: https://core.telegram.org/api
- Telethon Library: https://docs.telethon.dev
- CLAUDE.md - Critical Rule #1 (Telegram Session Management)
- `.env.example` - Telegram configuration variables
