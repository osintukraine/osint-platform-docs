# Telegram Setup

Configure Telegram monitoring, session management, and channel organization.

## Overview

**TODO: Content to be generated from codebase analysis**

This page will cover:

- Telegram API credentials
- Session creation and management
- Multi-account setup
- Folder-based channel management
- Channel monitoring rules
- Rate limiting and flood control
- Session persistence and backup

## Telegram API Credentials

**TODO: Document how to obtain Telegram API credentials**

### Creating API Credentials

1. Visit https://my.telegram.org
2. Log in with your phone number
3. Navigate to "API Development Tools"
4. Create a new application
5. Copy API ID and API Hash

### Configuring Credentials

**TODO: Document environment variable configuration**

```bash
TELEGRAM_API_ID=your_api_id
TELEGRAM_API_HASH=your_api_hash
TELEGRAM_PHONE=your_phone_number
```

## Session Management

**TODO: Explain Telegram session lifecycle**

### Creating a Session

**TODO: Document interactive session creation process**

```bash
docker-compose run --rm listener python -m telethon_session_creator
```

### Session Storage

**TODO: Explain where sessions are stored and how to back them up**

### Multi-Account Setup

**TODO: Document multi-account configuration for scaling**

## Folder-Based Channel Management

**TODO: Explain the folder-based channel management system**

### Archive Folders

**TODO: Document Archive-* folder behavior**

- `Archive-Russia` - Archive all messages from Russian channels
- `Archive-Ukraine` - Archive all messages from Ukrainian channels
- `Archive-OSINT` - Archive all messages from OSINT channels

### Monitor Folders

**TODO: Document Monitor-* folder behavior**

- `Monitor-Important` - Selective archiving with rules

### Discover Folders

**TODO: Document Discover-* folder behavior**

- `Discover-New` - Auto-joined channels, 14-day probation

## Channel Monitoring Rules

**TODO: Document intelligence rules configuration**

### Archive All Rule

**TODO: Explain archive_all rule for Archive-* folders**

### Selective Archive Rule

**TODO: Explain selective_archive rule for Monitor-* folders**

### Custom Rules

**TODO: Document how to create custom intelligence rules**

## Rate Limiting

**TODO: Document Telegram rate limits and how to avoid them**

- Message retrieval limits
- Join/leave limits
- API flood protection
- Backoff strategies

## Session Persistence

**TODO: Document session backup and restore procedures**

### Backing Up Sessions

```bash
# TODO: Add backup commands
```

### Restoring Sessions

```bash
# TODO: Add restore commands
```

## Troubleshooting

**TODO: Common Telegram-related issues:**

- Session expiration
- Flood wait errors
- Channel access issues
- Folder synchronization

---

!!! warning "Critical Rule"
    NEVER create standalone Telegram clients in task classes. Always pass the client from main.py to avoid session conflicts.

!!! note "Documentation Status"
    This page is a placeholder. Content will be generated from listener service, session management code, and CLAUDE.md guidance.
