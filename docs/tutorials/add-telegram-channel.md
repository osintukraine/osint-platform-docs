# Tutorial: Add a Telegram Channel

Learn how to add and monitor a new Telegram channel using the folder-based management system.

## Learning Objectives

By the end of this tutorial, you will:

- Understand the folder-based channel management system
- Add a new channel to monitoring
- Configure monitoring rules
- Verify the channel is being archived

## Prerequisites

- Platform installed and running
- Telegram account with active session
- Access to Telegram Desktop/Web
- Target channel identified

## Estimated Time

10-15 minutes

## Step 1: Understand Folder Types

**TODO: Content to be generated from codebase analysis**

The platform uses three folder types:

### Archive-* Folders

Channels in these folders have all messages archived:

- `Archive-Russia`
- `Archive-Ukraine`
- `Archive-OSINT`

### Monitor-* Folders

Channels in these folders use selective archiving based on rules:

- `Monitor-Important`
- `Monitor-News`

### Discover-* Folders

Auto-joined channels with 14-day probation:

- `Discover-New`

## Step 2: Create or Use a Folder

**TODO: Add screenshots and step-by-step instructions:**

1. Open Telegram Desktop/Web
2. Go to Settings → Folders
3. Create new folder or use existing Archive/Monitor folder
4. Configure folder settings

## Step 3: Add Channel to Folder

**TODO: Add screenshots and step-by-step instructions:**

1. Find the target channel
2. Right-click channel
3. Select "Add to Folder"
4. Choose appropriate folder (e.g., `Archive-Russia`)
5. Confirm addition

## Step 4: Wait for Sync

**TODO: Document sync timing:**

The listener service checks for folder changes every:

- X minutes (TODO: check actual interval)

You can also restart the listener to force immediate sync:

```bash
docker-compose restart listener
```

## Step 5: Verify Channel is Monitored

**TODO: Add verification steps:**

### Check Listener Logs

```bash
docker-compose logs -f listener | grep "channel_name"
```

### Check Database

```bash
docker-compose exec postgres psql -U osint_user -d osint_platform

SELECT id, username, title, folder FROM channels
WHERE username = '@channel_username';
```

### Check Frontend

1. Open frontend at http://localhost:3000
2. Go to Admin → Channels
3. Verify new channel appears in list

## Step 6: Verify Messages are Being Archived

**TODO: Add verification steps:**

1. Wait for new messages in the channel
2. Check message count in database
3. Search for messages in frontend

```bash
# Check message count
docker-compose exec postgres psql -U osint_user -d osint_platform

SELECT COUNT(*) FROM messages
WHERE channel_id = (SELECT id FROM channels WHERE username = '@channel_username');
```

## Troubleshooting

**TODO: Common issues and solutions:**

### Channel Not Appearing

- Verify folder name starts with `Archive-`, `Monitor-`, or `Discover-`
- Check listener logs for errors
- Verify Telegram session is active

### Messages Not Being Archived

- Check processor logs
- Verify spam filter isn't blocking
- Check intelligence rules

### Permission Issues

- Ensure you're a member of the channel
- For private channels, verify access rights

## Next Steps

After adding a channel:

- [Configure custom intelligence rules](../operator-guide/configuration.md)
- [Create RSS feed for the channel](create-custom-rss-feed.md)
- [Set up Discord alerts](setup-discord-alerts.md)

---

!!! tip "Bulk Adding Channels"
    You can add multiple channels at once by adding them all to the same folder in Telegram.

!!! note "Documentation Status"
    This page is a placeholder. Content will be generated from listener service code and folder management logic.
