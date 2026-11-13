# Searching Messages

Master the platform's powerful search and filtering capabilities to find the intelligence you need.

## Overview

The platform offers two complementary search interfaces:

1. **Browse Messages** (`/`) - Filter-first approach with live updates
2. **Unified Search** (`/search`) - Search-first approach across all content types

## Browse Messages Interface

### Search Bar

Located at the top of the Browse Messages page, the search bar supports:

**Full-Text Search**
```
Search across original content and translations.
Searching in English also finds posts that exist only in Russian or
Ukrainian ("Bakhmut" finds "Бахмут"), and the other way round.
Examples:
- "T-90 tank destroyed"
- "Avdiivka offensive"
- "HIMARS strike"
```

**Semantic Search** (AI-Powered)
- Toggle "Use Semantic Search" checkbox
- Finds messages by meaning, not just keywords
- Works for single names too (e.g. "Bakhmut"), though keyword search is still more exact
- Example: "military vehicle damage" finds mentions of tanks, APCs, artillery

```
Regular search:  "tank" → finds only "tank"
Semantic search: "tank" → finds "T-90", "armored vehicle", "BMP", "BTR"
```

### Country Filter

Prominent quick filter for channel groups:

- **All Channels** - No filtering (default)
- **Ukrainian Channels** - Only channels in `Archive-UA` and `Monitor-UA` folders
- **Russian Channels** - Only channels in `Archive-RU` and `Monitor-RU` folders

Selecting a country filter automatically clears individual channel selection.

### Filter Panel

Click "Show Filters" to expand the full filter panel. Active filters are shown with a badge count.

#### Channel Filters

**Individual Channel**
- Dropdown organized by country (Ukrainian, Russian, Other)
- Shows channel name, username, and folder
- Verified channels marked with ✓
- Selecting a channel clears country filter

**Channel Folder**
- Internal grouping (Archive-*, Monitor-*, Discover-*)
- Primarily used by country filter

#### Content Filters

**Media Type**
- **Any** - All messages (with or without media)
- **With Media** - Photos, videos, documents, audio
- **Text Only** - No attachments

**Specific Media Type**
- Photo (📷)
- Video (🎥)
- Document (📄)
- Audio (🔊)
- Voice (🎤)

#### Intelligence Filters

**Importance Level** (AI-assessed)
- **High Priority** (🔴) - Critical intelligence, combat reports
- **Medium Priority** (🟡) - Relevant but not urgent
- **Low Priority** (⚪) - General updates, noise

**Topic** (AI-detected)
- Combat ⚔️
- Equipment 🛡️
- Casualties 💀
- Movements 🚛
- Infrastructure ⚡
- Humanitarian 🏘️
- Diplomatic 🤝
- Intelligence 🔍
- Propaganda 📢
- Units 🎖️
- Locations 📍
- General 📰

**Language** (Auto-detected)
- Ukrainian 🇺🇦
- Russian 🇷🇺
- English 🇬🇧
- German 🇩🇪
- French 🇫🇷
- Polish 🇵🇱

#### Quality Filters

**Spam Filter**
- **Exclude Spam** (default) - Hides spam messages
- **Only Non-Spam** - Guaranteed legitimate
- **Only Spam** - Review flagged content
- **Include All** - No spam filtering

**Human Review**
- **Any Status** - All messages
- **Needs Review** (⚠️) - Flagged for analyst review
- **Reviewed/OK** (✅) - Already checked

**Discussion Thread**
- **Any** - All messages
- **Has Comments** (💬) - Posts with discussion
- **No Comments** - Standalone posts

#### Engagement Filters

**Minimum Views**
- Numeric threshold (e.g., 1000)
- Find viral/popular content
- Press Enter or click away to apply

**Minimum Forwards**
- Numeric threshold (e.g., 50)
- Identify widely shared messages
- Indicates influence/reach

#### Date Filters

**Quick Filters**
- Last 24 Hours
- Last 3 Days
- Last Week
- Last Month
- Last 3 Months
- All Time (default)

**Custom Date Range**
- **From Date** - Start date (inclusive)
- **To Date** - End date (inclusive)
- Mutually exclusive with quick filters

### Sorting Options

**Sort By**
- **Date (Telegram)** - When posted on Telegram (default)
- **Date Added** - When archived to platform
- **Importance Level** - AI priority score
- **Urgency Level** - Content urgency
- **Topic** - Alphabetical by topic
- **Media Type** - Group by media
- **Language** - Group by language
- **Message ID** - Chronological by ID

**Sort Order**
- **Descending** (⬇️) - Newest/highest first (default)
- **Ascending** (⬆️) - Oldest/lowest first

### Active Filters Summary

Below the filter panel, see all active filters as removable badges:
```
Search: T-90 tank
Country: 🇺🇦 Ukraine
Priority: 🔴 High
Topic: combat
Min Views: 1000
```

Click "Clear All" to reset all filters at once.

## Unified Search Interface

Access via `/search` or the top navigation bar.

### Search Modes

**Keywords Mode** (📝)
- Fast PostgreSQL full-text search
- Matches words in original or translated content, in English, Russian and Ukrainian
- Good for specific terms, names, locations

**Semantic Mode** (🧠)
- AI-powered similarity search via embeddings
- Finds conceptually related content
- Slower but more comprehensive

Toggle between modes before searching. Below the search box you can also pick a location (an optional filter): results are limited to messages geolocated around that place.

### Search Tabs

Results are organized by content type:

**All Tab**
- Shows top 5 results from each type
- Click "See all →" to view full results
- Total count displayed per type

**Messages Tab**
- Telegram channel posts
- Up to 20 results in focused view

**Events Tab**
- Events detected from RSS news articles, with timestamps and locations

**RSS Articles Tab**
- External news articles
- Filtered by relevance

**Entities Tab**
- People, organizations, equipment
- From curated and OpenSanctions sources

### Search Results

Each result card shows:
- **Icon** - Type indicator (💬 message, 📰 event, 🌐 RSS, 👤 entity)
- **Title** - Message snippet or entity name
- **Snippet** - Contextual preview (up to 2 lines)
- **Score** - Relevance percentage (semantic mode only)
- **Metadata** - Channel name, source domain, or entity type

Click a result to view the full content.

### Search Performance

Search timing displayed in top-right:
```
250ms - Fast text search
1500ms - Semantic search (normal)
```

## Advanced Search Techniques

### Multi-Word Queries

Keyword search treats what you type as plain words: a message must contain **all** of them, in any order. Boolean operators (`&`, `|`, `!`), quoted phrases and wildcards are not interpreted, so use filters (channel, topic, date) to narrow results instead.

```
tank destroyed       → messages containing both "tank" and "destroyed"
```

### Semantic Search Tips

**Good semantic queries:**
- Concepts: "military casualties"
- Descriptions: "damaged armored vehicles"
- Situations: "urban combat footage"

**Less effective:**
- Single common words: "tank" (use text search)
- Exact quotes: Use text search instead
- Very short queries return fewer matches because semantic results must reach a minimum similarity

### Combining Filters

**Find viral combat footage:**
```
Topic: Combat
Media Type: Video
Min Views: 5000
Sort: Views descending
```

**Monitor high-priority Ukrainian sources:**
```
Country: Ukraine
Importance: High
Date: Last 24 Hours
Sort: Date descending
```

**Research specific equipment:**
```
Search: "T-90M"
Topic: Equipment
Media Type: Photo
Sort: Date descending
```

## Saving Searches as RSS Feeds

Once you've configured your search and filters, subscribe to updates:

1. Configure all desired filters
2. Click "Subscribe Feed" button
3. Choose format:
   - **RSS 2.0** - Most compatible (Feedly, Inoreader)
   - **Atom 1.0** - Better content structure
   - **JSON Feed** - For developers/APIs
4. Copy URL to clipboard
5. Add to your feed reader

See [RSS Feeds](rss-feeds.md) for detailed feed management.

## Pagination

Browse messages with pagination controls at the bottom:

- **Items per page**: 20 by default
- **Navigation**: Previous, Next, Jump to page
- **Total count**: Shows filtered results count
- **Depth limit**: Browsing stops after the first 10,000 results. Narrow the filters (for example the date range) to reach older items.

## Performance Tips

### Fast Searches
- Use text search for specific terms
- Filter by channel first (reduces dataset)
- Use date ranges to limit scope

### Comprehensive Searches
- Use semantic search for concepts
- Allow more time for AI processing
- Consider RSS feeds for ongoing monitoring

### Large Result Sets
- Apply topic filter to narrow scope
- Use importance filter to prioritize

## Troubleshooting

### No Results Found

**Check your filters:**
- Clear all filters and try again
- Verify date range isn't too narrow
- Try different search terms

**Semantic search limitations:**
- Requires messages to have embeddings
- Not all old messages are embedded
- Try text search as fallback

### Search Too Slow

**Optimize your query:**
- Reduce date range
- Filter by channel first
- Use text search instead of semantic

### Unexpected Results

**Understand content matching:**
- Searches both original AND translated text
- Semantic mode finds related concepts
- Check "Active Filters" summary

## Keyboard Shortcuts

- `Enter` in search box → Perform search
- Page controls work with keyboard navigation

---

**Next Steps:**
- [Create RSS Feeds](rss-feeds.md) from your searches
- [Explore Entities](entities.md) mentioned in results
- [View Social Graph](social-graph.md) for message relationships
