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

Ask your administrator to import a new list; column mapping is configured per source format.

### OpenSanctions (Auto-synced)

**Sanctions Lists**
- EU sanctions
- US OFAC
- UK sanctions
- UN sanctions
- 10,000+ entities, with fuzzy name matching against the full OpenSanctions dataset

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

### Finding an Entity

There is no separate entity directory page. Entities are reached through search or from the badges on messages.

### Search Results

Use the unified search (**Search** in the top navigation) and open the **Entities** tab:

- Search by name or alias
- Fuzzy matching for typos and alternative spellings
- Filter by source (curated or OpenSanctions) and entity type through the API
- Click an entity to view its profile

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
- Will show related events

#### RSS Articles Tab

External news articles mentioning entity:

**Status:** Coming soon
- RSS article linking planned
- Will show aggregated news coverage

#### Relationships Tab

Interactive graph of entity relationships from multiple sources:

**Display:**
- Interactive network graph (drawn in your browser with Sigma.js)
- Entity nodes connected by relationship types
- Zoom in/out, a **Refresh** button, and PNG export

**Relationship Sources:**

!!! info "Wikidata Relationships"
    The platform automatically fetches structured relationships from Wikidata for entities with known Wikidata QIDs. These provide authoritative, verified connections.

| Category | Relationship Types | Examples |
|----------|-------------------|----------|
| **Corporate** | employer, owns, owned_by | "Works at Gazprom", "Owns Arsenal FC" |
| **Political** | position, party, member_of | "President of Russia", "Member of United Russia" |
| **Associates** | partner, associate | "Business partner with X" |

**Additional Relationship Types:**
- Co-mentioned in messages (platform-computed)
- Links reported by OpenSanctions

**Graph Controls:**
- Zoom in/out
- **Refresh** button to fetch the latest Wikidata relationships
- **Export** as PNG

**Node Colors:**
- Blue: the entity you are viewing (center)
- Other colors group related entities by relationship category (corporate, political, associates)
- Sanctioned related entities are marked

**Empty State:**
- "Needs enrichment" - relationships not yet computed
- Click "Refresh" to trigger Wikidata enrichment
- A background task also builds the graph over time

## Understanding Match Scores

Entity mentions are detected automatically. Each badge on a message shows how well the mention matched the entity, as a percentage:

- **High (90% and above)**: near-exact match, safe to rely on
- **Lower**: fuzzy or partial match, check the message text before trusting it

Short or ambiguous names (common surnames, short equipment designations) are the most likely to produce false matches.

## Entity Mention Context

When you click an entity badge in a message:

1. **Tooltip** - Shows entity type, source and match percentage
2. **Click Action** - Navigates to the entity profile

## Advanced Features

### Wikidata Integration

Automatically fetches additional data from the Wikidata knowledge base:

**Profile Data:**
- Profile images (for people, equipment)
- Multilingual descriptions
- Birth/founding dates
- Official websites
- Country/nationality
- Alternative names (aliases)
- Social media handles (Twitter, Telegram)

**Relationship Data (NEW):**

The platform now fetches structured relationships from Wikidata SPARQL queries:

| Category | What It Shows |
|----------|---------------|
| **Corporate** | Employers, companies owned, board positions |
| **Political** | Positions held, party membership, government roles |
| **Associates** | Business partners, personal associates |

Each relationship includes:
- Related entity name and Wikidata QID
- Start/end dates (if available)
- Cross-reference with OpenSanctions (shows if related entity is sanctioned)

**How It Works:**
1. An entity is created with basic information
2. A background task matches it to Wikidata by name (only close matches, 90% or better, are accepted)
3. Profile details and relationships are fetched and cross-checked against OpenSanctions
4. The entity profile is updated

**Data Freshness:**
- Relationships are cached for 7 days
- Entities are re-enriched periodically in the background
- Use the **Refresh** button on the Relationships tab to force a re-fetch

**Viewing Wikidata Data:**
- Profile header shows Wikidata description and image
- "📖 Wikidata" badge indicates enrichment present
- Click Wikidata link to view full entry
- Relationships Tab shows relationship graph

### OpenSanctions Sync

**Kept up to date automatically:**
1. The platform downloads the latest OpenSanctions dataset
2. Sanctions and PEP lists are processed and entities updated or added
3. Entities are linked to messages by a background task

**Data Included:**
- Personal information (name, DOB, nationality)
- Positions and roles
- Sanctions details (authority, date, reason)
- Related entities (family, associates)
- Source datasets (OFAC, EU, UN, etc.)

### Entity Relationship Graph

Relationships come from Wikidata, OpenSanctions links and co-mentions in messages. They are built on demand for the entities you view, cached, and refreshed in the background.

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
1. Search for the person or organization (filter by OpenSanctions source)
2. Look for the "Sanctioned" badge on the profile
3. Check Messages tab for recent mentions
4. View Relationships tab for networks
5. Cross-reference with events on the Events page

### Research Organizational Networks

**Workflow:**
1. Start with known organization (e.g., "Wagner Group")
2. Open Relationships tab
3. Explore connected entities (commanders, units)
4. Click nodes to navigate to related profiles
5. Export the graph as PNG for reporting

### Identify Emerging Entities

**Workflow:**
1. Browse recent high-importance messages
2. Look for names that carry no entity badge
3. Research via Wikidata and external sources
4. Ask your administrator to add them to a curated list

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

For programmatic access, see [API Reference](../reference/api-endpoints.md#entities):

**Endpoints:**
```
GET /api/entities/search?q=...
GET /api/entities/{source}/{id}
GET /api/entities/{source}/{id}/messages
GET /api/entities/{source}/{id}/relationships
```

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
- Cross-reference OpenSanctions with curated lists and events
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
- Contact admin to request entity addition

### Missing Wikidata

**Why it happens:**
- Entity name doesn't match Wikidata exactly
- Wikidata entry doesn't exist
- Enrichment task hasn't run yet
- API rate limit temporarily hit

**What to do:**
- Wait for background enrichment (can take a day or two)
- Check Wikidata manually and note QID
- Report to admin for manual enrichment

### Empty Relationships Tab

**Reasons:**
- Entity is newly added
- Insufficient mentions for co-occurrence
- Enrichment has not run yet

**What to do:**
- Click **Refresh** on the Relationships tab
- Check back later; the background task fills it in over time

---

**Next Steps:**
- [Search for entities](searching.md) in messages
- [View Social Graph](social-graph.md) for message-level relationships
- [Subscribe to RSS feeds](rss-feeds.md) for entity monitoring
