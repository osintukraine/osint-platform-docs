# Map Interface

Visualize geocoded Telegram messages and detected event clusters on an interactive map.

## Overview

The Map Interface transforms Telegram messages into geospatial intelligence by:
- Extracting location names from message content (4-stage pipeline)
- Detecting event clusters from message velocity spikes
- Showing real-time updates via WebSocket

**Access**: Navigate to `/map` in the frontend

## Map Layers

### Messages Layer (Blue Markers)

Individual geocoded Telegram posts with location mentions.

**Click a marker to see**:
- Message content preview
- Channel name and time
- Location name and confidence score
- Extraction method (gazetteer, nominatim, llm_relative)

**Confidence Levels**:
- **0.95**: Gazetteer match (offline database)
- **0.85**: Nominatim API (OpenStreetMap)
- **0.75**: LLM relative location ("10km north of X")

**Server-Side Clustering**: At zoom < 12, nearby markers automatically cluster with count badges.

### Clusters Layer (Colored Markers)

Detected event clusters from message velocity spikes.

**Color-Coded Tiers**:

| Color | Tier | Description |
|-------|------|-------------|
| Red | Rumor | 1 channel reporting |
| Yellow | Unconfirmed | 2-3 channels, same affiliation |
| Orange | Confirmed | 3+ channels, cross-affiliation |
| Green | Verified | Human analyst confirmed |

**Click a cluster to**:
- View summary and claim type
- See channel count and detection time
- Expand to show constituent messages
- Promote to verified event (admin)

**Tier Progression**: Tiers automatically upgrade as more channels report the same event.

### Heatmap Layer

Geographic density visualization of message activity.

**Color Gradient**:
- Blue → Green → Yellow → Red (low to high activity)

**Use Cases**:
- Identify geographic hotspots
- Track activity shifts over time
- Find coverage gaps

## Controls

### Timeline Slider

Filter messages and clusters by time range.

**Quick Presets**: Last 24h, 7 days, 30 days, custom range

**Usage**: Drag handles to select start/end dates. Map updates in real-time.

### Polygon Filter

Draw custom geographic boundaries to focus on specific areas.

**How to Draw**:
1. Click "Polygon Tool" button
2. Click to place vertices (minimum 3)
3. Double-click to close polygon
4. Only content inside boundary is shown

**Preset Polygons**: Donetsk, Luhansk, Kherson, Zaporizhzhia oblasts, Crimea

### Location Search

Navigate to specific locations using autocomplete.

**How to Use**:
1. Type location name (min 2 characters)
2. Select from suggestions (ordered by population)
3. Map pans and zooms to location

**Data Source**: 30,000+ UA/RU locations from GeoNames (searches both ASCII and Cyrillic names)

## Real-Time Updates

The map receives new geocoded messages via WebSocket.

**Connection Status** (top right):
- Green: Connected, receiving live updates
- Yellow: Reconnecting
- Red: Disconnected (falls back to HTTP polling)

**When a new location is geocoded**:
1. Marker fades in with pulse animation
2. Toast notification shows location name

## Cluster Interaction

### Expanding Clusters

Click cluster marker → "Expand Messages" → Shows spider pattern with all constituent messages.

### Cluster Actions (Admin)

- **Promote to Event**: Convert confirmed cluster to structured event record
- **Change Tier**: Manually override tier (requires verification notes)
- **Archive**: Hide false positives (soft delete, can restore)

## Export & Sharing

### Share Current View

Click "Share" to generate URL with:
- Bounding box and zoom level
- Enabled layers
- Timeline range
- Polygon filter (if active)

### Export Data

Download filtered data as:
- **GeoJSON**: For QGIS, ArcGIS, Leaflet
- **CSV**: For Excel, R, Python
- **KML**: For Google Earth

**Limit**: Maximum 10,000 messages per export

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `P` | Toggle polygon draw mode |
| `M` | Toggle messages layer |
| `C` | Toggle clusters layer |
| `H` | Toggle heatmap layer |
| `S` | Focus search box |
| `Esc` | Clear filters / Exit draw mode |
| `[` `]` | Previous/next day |

## Troubleshooting

### No Messages Appearing
- Check bounding box (default: Ukraine region)
- Expand timeline to "Last 30 days"
- Verify Messages layer is enabled
- Lower minimum confidence slider

### Clusters Not Showing
- Zoom out to see aggregated view
- Enable all tier filters
- Expand timeline (cluster formation takes time)

### WebSocket Not Connecting
- Check browser console for errors
- System falls back to HTTP polling automatically
- May be blocked by corporate firewalls

### Slow Performance
- Apply timeline filter (reduce to last 7 days)
- Use polygon filter to limit area
- Enable server-side clustering (zoom out)

## Related Documentation

- [API Endpoints](../reference/api-endpoints.md) - Map API endpoints
- [Database Schema](../reference/database-schema.md) - Geolocation tables
- [Environment Variables](../reference/environment-variables.md) - Configuration
