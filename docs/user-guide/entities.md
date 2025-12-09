# Entity Exploration

Browse and analyze tracked entities including military equipment, personnel, organizations, and locations.

## Overview

The platform maintains an **entity knowledge graph** from multiple intelligence sources, automatically detecting mentions in Telegram messages and linking them to enriched profiles with Wikidata integration.

### Entity Types

**Equipment** (🛡️)
- Tanks, APCs, artillery systems
- Aircraft, helicopters, UAVs
- Naval vessels
- Small arms and munitions
- Example: T-90M tank, HIMARS, Shahed-136

**Personnel** (👤)
- Military commanders
- Political figures
- Spokespersons and officials
- Sanctioned individuals (via OpenSanctions)
- Example: Valery Gerasimov, Oleksiy Reznikov

**Organizations** (🏢)
- Military units and formations
- Private military companies
- Government agencies
- NGOs and humanitarian groups
- Example: Wagner Group, 47th Mechanized Brigade

**Locations** (📍)
- Cities and settlements
- Military installations
- Geographical features
- Strategic infrastructure
- Example: Bakhmut, Crimean Bridge

**Units** (🎖️)
- Brigade-level formations
- Battalion tactical groups
- Special forces units
- Air force squadrons
- Example: 1st Guards Tank Army

## Entity Sources

### Custom CSV Imports

Import your own domain-specific entity lists via CSV. The platform supports:

**Equipment Databases**
- Global military equipment databases
- Technical specifications
- Service history and variants

**Personnel Lists**
- Military leadership
- Political figures
- Regional commanders

**Organization Databases**
- Military units and formations
- Private military companies
- Government agencies

The entity-ingestion service processes CSVs with configurable column mapping for different source formats.

### OpenSanctions (Auto-synced)

**Sanctions Lists**
- EU sanctions
- US OFAC
- UK sanctions
- UN sanctions
- 10,000+ entities

**Politically Exposed Persons (PEPs)**
- Government officials
- Military leadership
- State-owned enterprises
- Family members

### Wikidata Enrichment

Automatically enriches entities with:
- Profile images
- Birth/founding dates
- Descriptions in multiple languages
- Official websites
- Country/location data
- Alternative names and aliases

## Browsing Entities

### Entity Directory

Access the entity list at `/entities`.

**Filters:**
- Entity type (equipment, person, organization, etc.)
- Source (curated, opensanctions)
- Name search
- Alphabetical sorting

**Display:**
- Entity icon and type badge
- Primary name
- Brief description
- Link count (messages, events, RSS articles)
- Source indicators

### Search Results

Entities appear in unified search (`/search`) under the "Entities" tab:

- Search by name or alias
- Fuzzy matching for typos
- Alternative spellings recognized
- Click entity to view profile

## Entity Profile Pages

Each entity has a detailed profile at `/entities/{source}/{id}`.

### Profile Header

**Left Side: Image or Icon**
- Wikidata profile photo (if available)
- Default icon for type (🛡️ equipment, 👤 person, etc.)
- 128x128 pixel display

**Right Side: Information**

**Badges:**
```
Equipment Type           (for curated entities)
⚠️ Sanctioned           (OpenSanctions, not PEP)
👔 Politically Exposed  (OpenSanctions PEP)
📚 Curated Entity       (from ArmyGuide, Root.NK, ODIN)
📖 Wikidata            (has Wikidata enrichment)
```

**Name and Description:**
- Primary name (bold, large)
- Wikidata description (if available, in multiple languages)
- Fallback to entity description from source

**Quick Facts:**
- 📅 Birth/founding date (from Wikidata)
- 🌍 Country/nationality
- 📍 Birth place
- 📋 First seen in database

### Key Information Card

For OpenSanctions entities, displays structured data:

**Positions/Roles**
- Current and past positions
- Government offices
- Military ranks
- Corporate roles

**Education**
- Universities attended
- Degrees and qualifications
- Military academies

**Citizenship**
- Nationalities
- Dual citizenship

**Birth Place**
- City/region
- Country

**Title**
- Official titles
- Honorifics
- Academic degrees

**Name Variants (Expandable)**
- Latin script aliases (prioritized)
- Alternative spellings
- Transliterations
- Click to expand full list

### External Links

**OpenSanctions Link**
- Full entity profile on OpenSanctions.org
- Source datasets
- Additional metadata

**Wikidata Link**
- Full Wikidata entry
- Multilingual descriptions
- Structured data browser

**Official Website**
- Entity's homepage (if available from Wikidata)
- Verified organizational sites

**Data Sources**
- List of contributing datasets
- EU sanctions, OFAC, etc.
- Limited to 5, shows "+X more" for additional

### Content Tabs

#### Messages Tab

Lists Telegram messages mentioning this entity:

**Display:**
- Message snippet (2 lines max)
- Channel name with icon (📢)
- Post date (🕐)
- Click to view full message

**Pagination:**
- Shows 10 most recent
- Link to "View all messages" with entity filter pre-applied

**Empty State:**
- "No linked messages found"
- Suggests searching for entity manually

#### Events Tab

Curated intelligence events linked to entity:

**Status:** Coming soon
- Event linking not yet implemented
- Will show ODIN event references

#### RSS Articles Tab

External news articles mentioning entity:

**Status:** Coming soon
- RSS article linking planned
- Will show aggregated news coverage

#### Relationships Tab

Interactive graph of entity relationships:

**Display:**
- Cytoscape network visualization
- Entity nodes connected by relationship types
- Force-directed layout (default)
- Zoom, pan, and layout controls

**Relationship Types:**
- Co-mentioned in messages
- Organizational hierarchy
- Geographic proximity
- Temporal correlation
- OpenSanctions relationships

**Graph Controls:**
- Layout selector (fCoSE, grid, circle, etc.)
- Zoom in/out
- Reset view
- Re-run layout
- Export as PNG

**Node Colors:**
- Blue: Selected entity (center)
- Green: Related entities
- Purple: Organizations
- Orange: Locations
- Pink: Events

**Empty State:**
- "Needs enrichment" - relationships not yet computed
- Background task runs periodically to build graph

## Understanding Confidence Scores

Entity mentions are detected automatically with confidence levels:

### Extraction Methods

**NER (Named Entity Recognition)**
- AI model extracts entities from text
- Confidence: 0.7-0.95
- May have false positives

**Dictionary Matching**
- Exact match against curated database
- Confidence: 0.95-1.0
- High precision

**Fuzzy Matching**
- Handles typos and variants
- Confidence: 0.6-0.85
- Requires manual review

### Confidence Indicators

In message views, entities display badges:

```
High Confidence (≥0.9)
  └─ Solid badge, no indicator

Medium Confidence (0.7-0.9)
  └─ Badge with "~" symbol

Low Confidence (<0.7)
  └─ Badge with "?" symbol, faded color
```

## Entity Mention Context

When you click an entity badge in a message:

1. **Inline Tooltip** - Shows entity type and source
2. **Click Action** - Navigates to entity profile
3. **Highlight** - Message text highlights entity mention
4. **Context Preview** - See surrounding text

## Advanced Features

### Wikidata Integration

Automatically fetches additional data:

**What It Provides:**
- Profile images (for people, equipment)
- Multilingual descriptions
- Birth/death dates
- Official websites
- Country/nationality
- Alternative names (aliases)

**How It Works:**
1. Entity created with basic info
2. Background enrichment task runs
3. Queries Wikidata API by name/identifier
4. Stores enrichment in `metadata.wikidata`
5. Updates entity profile

**Data Freshness:**
- Enriched on first view (lazy loading)
- Re-enriched monthly (background task)
- Manual refresh (admin action)

### OpenSanctions Sync

**Automatic Daily Updates:**
1. Downloads latest OpenSanctions dataset
2. Processes sanctions and PEP lists
3. Upserts entities (updates existing, adds new)
4. Links to messages via background task

**Data Included:**
- Personal information (name, DOB, nationality)
- Positions and roles
- Sanctions details (authority, date, reason)
- Related entities (family, associates)
- Source datasets (OFAC, EU, UN, etc.)

### Entity Relationship Graph

**Graph Building:**
1. Co-occurrence analysis (entities mentioned together)
2. Temporal proximity (mentioned in similar timeframes)
3. Channel clustering (same channels discuss related entities)
4. OpenSanctions links (explicit relationships)

**Update Frequency:**
- Built on-demand for viewed entities
- Cached for performance
- Refreshed weekly (background task)

## Using Entities for Analysis

### Track Equipment Losses

**Workflow:**
1. Find equipment entity (e.g., "T-90M tank")
2. View Messages tab
3. Filter by:
   - Topic: Combat, Equipment
   - Media Type: Photo (visual confirmation)
   - Importance: High
4. Sort by date descending
5. Subscribe to RSS feed for updates

### Monitor Sanctioned Individuals

**Workflow:**
1. Browse OpenSanctions entities
2. Filter by "Sanctioned" badge
3. Check Messages tab for recent mentions
4. View Relationships tab for networks
5. Cross-reference with ODIN events

### Research Organizational Networks

**Workflow:**
1. Start with known organization (e.g., "Wagner Group")
2. Open Relationships tab
3. Explore connected entities (commanders, units)
4. Click nodes to navigate to related profiles
5. Export graph for reporting

### Identify Emerging Entities

**Workflow:**
1. Browse messages with high NER confidence
2. Look for unknown entity mentions
3. Research via Wikidata and external sources
4. Submit to admin for curated entity creation

## Filtering by Entities

In Browse Messages, filter by entity mentions:

1. Apply filters as normal
2. Add search term for entity name
3. Messages containing entity are prioritized
4. Use semantic search for concept matching

**Example:**
```
Search: "Wagner Group"
Topic: Combat
Importance: High
Date: Last 30 days
```

## Entity API Access

For programmatic access, see [API Reference](../api/entities.md):

**Endpoints:**
```
GET /api/entities
GET /api/entities/{source}/{id}
GET /api/entities/{source}/{id}/messages
GET /api/entities/{source}/{id}/relationships
```

**Use Cases:**
- Build custom dashboards
- Export entity data
- Integration with other tools
- Automated reporting

## Best Practices

### Effective Entity Research

**Start Broad, Then Narrow:**
1. Use unified search for initial discovery
2. Click entity to view profile
3. Check Messages tab for context
4. Explore Relationships for connections
5. Subscribe to RSS for monitoring

**Verify Confidence:**
- Prioritize high-confidence mentions
- Cross-reference low-confidence with media
- Check multiple messages for confirmation

**Use Multiple Sources:**
- Compare curated vs. auto-detected entities
- Cross-reference OpenSanctions with ODIN
- Validate with Wikidata and external sources

### Responsible Use

**Sanctions Compliance:**
- OpenSanctions data is for research only
- Consult legal counsel for compliance questions
- Verify sanctions status with official sources

**Privacy Considerations:**
- PEPs are public figures by definition
- Data is sourced from public records
- Respect data protection regulations

**Accuracy:**
- AI extraction may have errors
- Always verify critical information
- Report inaccuracies to administrators

## Troubleshooting

### Entity Not Found

**Possible reasons:**
- Not yet added to curated database
- Different name/spelling used
- Entity is auto-detected but not prominent
- Search query too specific

**Solutions:**
- Try alternative spellings
- Use semantic search for concept matching
- Browse entity directory manually
- Contact admin to request entity addition

### Missing Wikidata

**Why it happens:**
- Entity name doesn't match Wikidata exactly
- Wikidata entry doesn't exist
- Enrichment task hasn't run yet
- API rate limit temporarily hit

**What to do:**
- Wait for background enrichment (24-48 hours)
- Check Wikidata manually and note QID
- Report to admin for manual enrichment

### Empty Relationships Tab

**Reasons:**
- Entity is newly added
- Insufficient mentions for co-occurrence
- Enrichment task not yet run

**Wait time:**
- Background task runs weekly
- On-demand computation available (admin)
- Check back in 7 days

---

**Next Steps:**
- [Search for entities](searching.md) in messages
- [View Social Graph](social-graph.md) for message-level relationships
- [Subscribe to RSS feeds](rss-feeds.md) for entity monitoring
