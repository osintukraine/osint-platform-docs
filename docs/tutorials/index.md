# Tutorials

**Step-by-step guides for common tasks and workflows.**

These tutorials are designed to get you productive with the OSINT Intelligence Platform through hands-on, practical guidance. Each tutorial builds confidence progressively, starting with basics and advancing to complex scenarios.

---

## What You'll Learn

Our tutorials cover four essential areas of the platform:

1. **Channel Management** - How to add and monitor Telegram channels using folder-based discovery
2. **Data Distribution** - Creating dynamic RSS feeds with advanced filtering and search
3. **Real-Time Alerts** - Setting up Discord notifications for high-value intelligence
4. **Production Deployment** - Complete server setup, SSL configuration, and first-time monitoring

---

## Tutorials

### 1. Add a Telegram Channel (~5 minutes)

Learn the folder-based channel management system that makes adding channels incredibly simple.

**What You'll Learn:**
- How to create Telegram folders on mobile/desktop
- Adding channels to folders for monitoring
- Understanding archive vs. monitoring rules
- Verifying that channels are being captured

**Prerequisites:**
- Telegram account with admin access to a test account
- Platform running and operational (docker-compose up -d)

[:octicons-arrow-right-24: Start Tutorial](add-telegram-channel.md)

---

### 2. Create a Custom RSS Feed (~10 minutes)

Turn any search query into a shareable RSS feed that you can subscribe to in any reader.

**What You'll Learn:**
- How to build search queries with filters
- Generating RSS feed URLs with authentication tokens
- Subscribing in popular RSS readers (Feedly, Inoreader, Thunderbird)
- Understanding importance filtering for high-value feeds

**Prerequisites:**
- Platform API running (http://localhost:8000)
- At least 10 messages indexed in the platform

[:octicons-arrow-right-24: Start Tutorial](create-custom-rss-feed.md)

---

### 3. Setup Discord Alerts (~15 minutes)

Configure real-time notifications for critical intelligence using Discord webhooks and n8n workflows.

**What You'll Learn:**
- Creating a Discord webhook for your server
- Setting up an n8n workflow to process RSS feeds
- Filtering by importance level (high, medium, low)
- Testing the alert pipeline end-to-end

**Prerequisites:**
- Discord server with admin permissions
- Platform running with at least 50 messages
- n8n service available (http://localhost:5678)

[:octicons-arrow-right-24: Start Tutorial](setup-discord-alerts.md)

---

### 4. Deploy to Production (~30 minutes)

Complete walkthrough of deploying the OSINT platform to a production server with SSL, proper configuration, and first-time monitoring setup.

**What You'll Learn:**
- Server requirements and cost optimization
- DNS setup and SSL certificates with Caddy
- Environment configuration for production
- Telegram session authentication
- First channel monitoring and verification
- Health monitoring with Grafana

**Prerequisites:**
- A VPS server (Ubuntu 22.04 LTS recommended)
- Domain name pointed to your server
- Basic knowledge of SSH and terminal commands
- 30 minutes of uninterrupted time

[:octicons-arrow-right-24: Start Tutorial](deploy-to-production.md)

---

## Tutorial Features

All tutorials include:

- **Time Estimate** - How long the tutorial takes to complete
- **What You'll Learn** - Clear learning objectives
- **Prerequisites** - Required knowledge and setup
- **Step-by-Step Instructions** - Numbered steps with expected output
- **Expected Results** - What you should see at each checkpoint
- **Troubleshooting** - Common errors and how to fix them
- **What You Learned** - Summary of key concepts
- **Next Steps** - Where to go from here

---

## Learning Paths

Choose a learning path based on your goals:

### For Intelligence Analysts

1. Start with [Add a Telegram Channel](add-telegram-channel.md) to get familiar with the interface
2. Create [Custom RSS Feeds](create-custom-rss-feed.md) for personalized intelligence
3. Set up [Discord Alerts](setup-discord-alerts.md) for real-time notifications
4. Explore the full feature set in the [User Guide](../user-guide/index.md)

### For Systems Administrators

1. Start with [Deploy to Production](deploy-to-production.md) to set up your server
2. Review [Operations Guide](../operator-guide/index.md) for monitoring and maintenance
3. Set up [Discord Alerts](setup-discord-alerts.md) for team notifications
4. Configure [Advanced Monitoring](../operator-guide/monitoring.md) with Grafana

### For Developers

1. Review the [API Reference](../reference/api-endpoints.md) for available endpoints
2. Create [Custom RSS Feeds](create-custom-rss-feed.md) to understand the data structure
3. Explore [Architecture Documentation](../developer-guide/architecture.md) for deep technical details
4. Review [Deployment Guide](deploy-to-production.md) to understand infrastructure

---

## Tips for Success

### Avoid Getting Stuck

- **Read the prerequisites carefully** - Missing setup can cause confusion
- **Type out commands** - You'll learn more by typing than copying/pasting
- **Check expected output** - If your output differs, read the troubleshooting section
- **Take notes** - Jot down configuration values you'll need later

### Common Mistakes to Avoid

1. **Telegram folders** - Must use exact naming patterns (Archive-*, Monitor-*)
2. **API URLs** - Always use the base URL (e.g., http://localhost:8000)
3. **RSS subscriptions** - Copy the entire URL including the feed token
4. **Discord webhooks** - Keep webhook URLs private (never commit to git)
5. **Production DNS** - Ensure your domain DNS is propagated before SSL setup

---

## Getting Help

If you get stuck:

1. **Check the Troubleshooting section** - Each tutorial has a dedicated troubleshooting section
2. **Review the logs** - Most issues show up in service logs:
   ```bash
   docker-compose logs -f service-name
   ```
3. **Check the Operator Guide** - Common issues and solutions: [Troubleshooting](../operator-guide/troubleshooting.md)
4. **Ask for help** - Contact the team or community forums with:
   - What you were trying to do
   - The exact error message
   - The command or step that failed

---

## Tutorial Philosophy

Our tutorials are designed with these principles:

- **Learn by Doing** - Each tutorial is hands-on with real commands you run
- **Progressive Complexity** - Start simple, build to advanced features
- **Error Anticipation** - We predict common mistakes and show how to fix them
- **Multiple Perspectives** - Concepts explained several different ways
- **Immediate Validation** - You run code and see results frequently

---

## Feedback

Have suggestions for improving these tutorials? We'd love to hear from you:

- Unclear sections
- Missing topics
- Errors or outdated information
- Topics you'd like to see covered

Please file an issue or reach out to the documentation team.

---

**Ready to get started?** Choose a tutorial above and dive in!
