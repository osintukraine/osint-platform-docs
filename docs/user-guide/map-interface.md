# Map Interface

Visualize geocoded Telegram messages and detected event clusters on an interactive map.

## Overview

The Map Interface transforms Telegram messages into geospatial intelligence by:
- Extracting location names from message content (4-stage pipeline)
- Detecting event clusters from message velocity spikes
- Showing real-time updates via WebSocket

**Access**: Navigate to `/map` in the frontend

## Map Layers

Toggle layers in the sidebar under **Map Layers**. The sidebar also has a region switch (**Conflict Zone** or **Worldwide**), source and confidence filters, a **Hot Locations (24h)** list and a **Recent Messages** list.

!!! warning "Experimental"
    The map shows a banner because locations are detected automatically from message text. Treat positions as approximate and check the source message.

### Message Markers

Individual geocoded Telegram posts with a location mention. The marker shows a flag for the source channel (Ukrainian, Russian or unknown affiliation), with a halo that gets larger the less precise the location is.

**Hover or click a marker to see**:
- Message content preview
- Channel name and time
- Location name and a confidence label (High, Medium or Low)

**How confidence is set**, by how the location was found:
- **High (0.95)**: found in the offline place-name database (GeoNames)
- **Medium-high (0.85)**: resolved by OpenStreetMap search (Nominatim)
- **Lower (0.75)**: worked out from a relative description such as "10km north of X"

**Zoomed out**, nearby markers merge into numbered groups. Zoom in to see individual posts.

### Event Clusters

Groups of posts about the same event, detected when several channels report similar content close together in time and place.

**Color-coded tiers**:

| Color | Tier | Description |
|-------|------|-------------|
| Red | Rumor | 1 channel reporting |
| Yellow | Unconfirmed | 2-3 channels, same affiliation |
| Green | Confirmed | 3+ channels, cross-affiliation |
| Blue | Verified | 2+ independent RSS news sources report it (automatic), or an analyst verified it |

**Click a cluster** to see its tier, summary, message and channel counts and detection time, then expand it to fan out the individual messages.

**Tier progression**: tiers upgrade automatically as more channels or news sources report the same event.

### Verified Events

Shows confirmed events as structured event records.

### Heat Map

Geographic density of message activity, from blue (low) through green and yellow to red (high). Radius and opacity can be adjusted in the sidebar.

**Use cases**:
- Identify geographic hotspots
- Track activity shifts over time
- Find coverage gaps

### Trajectories and Vessels

- **Trajectories**: drone flight paths reconstructed from reports
- **Vessels**: tracked shadow-fleet vessels

## Controls

### Timeline Slider

Filter messages and clusters by time range.

**Quick presets**: Last 24h, Last 7d, Last 30d, All time. Drag the handles to choose a custom range, or press play to replay activity over time. The map updates as you move.

### Sources and Confidence

Use the sidebar filters to show only Ukrainian, Russian or unknown-affiliation channels, and to hide low-confidence locations.

### Finding a Place

Click a location in **Hot Locations (24h)** to jump there, or click a post in **Recent Messages**. To search for a message near a named place, use the location filter on the [Search](searching.md) page.

## Real-Time Updates

With **Live Updates** on, new geocoded messages appear on the map as they arrive. The sidebar shows the connection status; if it drops, the map reconnects automatically.

## Troubleshooting

### No Messages Appearing
- Check the region switch: **Conflict Zone** only covers Ukraine and its border regions, use **Worldwide** for elsewhere
- Widen the timeline (for example "All time")
- Lower the confidence filter and enable all source filters

### Clusters Not Showing
- Zoom out to see the aggregated view
- Enable the Event Clusters layer
- Widen the timeline (cluster formation takes time)

### Live Updates Not Connecting
- Reload the page
- Corporate firewalls can block live connections; the map still works but needs a reload to refresh

### Slow Performance
- Apply a shorter timeline (for example 7 days)
- Zoom in or out to reduce the number of markers

## Related Documentation

- [API Endpoints](../reference/api-endpoints.md) - Map API endpoints
- [Database Tables](../reference/database-tables.md) - Geolocation tables
- [Environment Variables](../reference/environment-variables.md) - Configuration
