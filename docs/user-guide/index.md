# User Guide

Welcome to the OSINT Intelligence Platform user guide. This documentation will help you get the most out of the platform's features for intelligence gathering, analysis, and monitoring.

## Overview

The OSINT Intelligence Platform is a comprehensive system for archiving, enriching, and analyzing Telegram content with AI-powered intelligence assessment. This guide covers all user-facing features for analysts and researchers.

## Key Features

### Intelligence Analysis
- **Smart Search** - Full-text and AI-powered semantic search across all content
- **Entity Tracking** - Curated and automatically detected entities (people, equipment, organizations)
- **Social Graph** - Visualize relationships, forwards, and influence patterns
- **AI Classification** - Automatic topic detection and importance scoring

### Content Management
- **Filtered Views** - Filter by channel, date, topic, importance, and media type
- **RSS Feeds** - Subscribe to any search with custom filters
- **Multi-format Support** - RSS 2.0, Atom 1.0, and JSON Feed

### Monitoring & Alerts
- **Real-time Updates** - Live notifications via ntfy.sh
- **Custom Subscriptions** - Get alerts for specific topics or channels
- **Mobile Support** - Push notifications to your phone

## Who This Is For

- **Intelligence Analysts** - Track military activities, equipment, and personnel
- **Researchers** - Study propaganda, influence operations, and information warfare
- **Content Moderators** - Monitor channels for policy violations
- **Data Scientists** - Export structured data for analysis

## Quick Navigation

### Getting Started

1. **[Searching Messages](searching.md)** - Learn advanced search techniques
2. **[Entity Exploration](entities.md)** - Understand entity types and profiles
3. **[RSS Feeds](rss-feeds.md)** - Set up custom feeds for your workflow

### Advanced Features

4. **[Social Graph Analysis](social-graph.md)** - Analyze channel relationships
5. **[Notifications](notifications.md)** - Configure real-time alerts

## Common Workflows

### Find High-Priority Intelligence

1. Go to the Browse Messages page
2. Set **Importance Level** to "High Priority"
3. Filter by **Topic** (e.g., "Combat", "Equipment")
4. Sort by **Date** to see newest first
5. Subscribe to the RSS feed to monitor ongoing

### Track a Specific Entity

1. Use **Search** to find mentions (e.g., "T-90 tank")
2. Click on entity badges in messages
3. View the **Entity Profile** page
4. Explore **Relationships** tab for connections
5. Check **Messages** tab for all mentions

### Monitor Channel Activity

1. Browse messages and filter by **Channel**
2. Use **Country Filter** for Ukrainian or Russian channels
3. Set **Minimum Views** to find viral content
4. Enable **Semantic Search** for concept-based matching

## Platform Structure

```
Home (/)
├── Browse Messages - Main feed with advanced filters
├── Search (/search) - Unified search across all content
├── Entities (/entities) - Browse tracked entities
└── Settings
    └── Feed Tokens - Manage RSS authentication
```

## Understanding Content Types

### Messages
- Telegram channel posts archived in real-time
- Includes text, media, translations, and metadata
- AI-enriched with topics, entities, and importance scores

### Events
- Curated intelligence reports from ODIN
- Structured event data with timestamps and locations
- Linked to relevant entities

### RSS Articles
- External news sources monitored for keywords
- Filtered by relevance and source credibility

### Entities
- **Curated** - Manually verified (ArmyGuide, Root.NK, ODIN)
- **OpenSanctions** - Sanctions lists and politically exposed persons (PEPs)
- **Auto-detected** - Extracted from messages via AI

## Data Freshness

- **Real-time** - New messages appear within seconds
- **AI Enrichment** - Processed within 1-5 minutes
- **Translations** - Generated on-demand (DeepL Pro)
- **Entity Linking** - Background task, runs every 15 minutes

## Getting Help

### Interface Questions
- Hover over field labels for tooltips
- Look for "?" icons for explanations
- Check filter descriptions under dropdown menus

### Technical Issues
- Check the [Operator Guide](../operator-guide/index.md) for admin tasks
- Review [Architecture](../developer-guide/architecture.md) for system design
- See [API Reference](../reference/api-endpoints.md) for programmatic access

## Privacy & Ethics

This platform archives **public** Telegram channels only. No private chats or groups are monitored. All data is:

- Sourced from publicly accessible channels
- Stored with full metadata for verification
- Available for research and analysis
- Subject to responsible use guidelines

---

**Ready to start?** Begin with [Searching Messages](searching.md) to learn the core features.
