# Social Graph Analysis

Explore message relationships, channel networks, and content propagation through interactive graph visualization.

## Overview

The social graph feature reveals how messages, channels, and content relate to each other, helping analysts understand:

- **Message Relationships** - Forwards, replies, and comment threads
- **Channel Networks** - Who shares content from whom
- **Influence Patterns** - Viral content and coordination
- **Author Connections** - User posting behavior

## Social Network Graph (Per-Message)

Each message has its own social network showing direct relationships.

### Accessing the Graph

1. Open any message detail page
2. Scroll to **Social Network Graph** card
3. Graph loads automatically if relationships exist
4. Interactive visualization with controls

### Graph Structure

**Node Types:**

**Message Node** (📄 Blue, Large)
- The central message being viewed
- Largest node in the graph
- Shows views, forwards, comments count
- Content preview on hover

**Author Node** (👤 Sage Green, Ellipse)
- Channel or user who posted
- Links to message via "authored" edge
- Displays user ID if available

**Forward Source Node** (⤴️ Lavender, Rectangle)
- Original channel message was forwarded from
- Shows channel ID and original message ID
- Connected via "forwarded_from" dashed edge

**Parent Message Node** (↩️ Copper, Rectangle)
- Message being replied to
- Shows parent message ID
- Connected via "reply_to" edge

**Comment Nodes** (💬 Dusty Rose, Small Circles)
- Individual comments on the message
- Size varies by comment engagement
- Shows commenter name and text preview

**Reaction Nodes** (👍 Warm Amber, Circles)
- Individual reaction types (🔥, ❤️, 👍, 💯, etc.)
- Size scales with reaction count (1-10,000)
- Different colors per reaction type
- Paid reactions (⭐) highlighted in gold

**Edge Types:**

```
authored            → Solid green line (author to message)
forwarded_from      → Dashed purple line (forward source)
reply_to            → Solid copper line (parent message)
commented_on        → Thin pink line (comments)
reacted             → Dotted gray line (reactions)
```

### Graph Metrics

**Engagement Metrics Bar:**
- Views (👁️ Blue) - Total views count
- Forwards (↗️ Green) - Times shared
- Engagement Rate (📈 Purple) - Percentage of viewers who engaged

**Virality Badge:**
- Very High (🔥 Red) - Exceptional spread
- High (🔥 Orange) - Wide distribution
- Medium (🔥 Yellow) - Moderate sharing
- Low (🔥 Blue) - Limited spread
- None - Not viral

**Reach Indicator:**
- Very High - 100,000+ views
- High - 10,000-100,000 views
- Medium - 1,000-10,000 views
- Low - <1,000 views

### Reaction Display

**Standard Reactions:**
Grouped by emoji with counts:
```
🔥 1,234    ❤️ 892    👍 567
💯 234      😡 123    👎 45
```

**Paid Reactions:**
Displayed separately with gold highlighting:
```
⭐ 89 Paid
```

These are Telegram's premium reactions that cost money.

### Social Relationship Indicators

**Status Grid:**

**Author** (👤)
- Green if present, gray if unknown
- Displays channel/user name
- Truncated if too long (hover for full)

**Forwarded** (↗️)
- Purple if message was forwarded
- Gray if original content
- Shows "Yes" or "No"

**Reply** (💬 Orange)
- Orange if replying to another message
- Gray if standalone
- Shows "Yes" or "No"

**Comments** (💬 Pink)
- Pink if has discussion
- Gray if no comments
- Shows comment count

### Graph Controls

The graph is laid out automatically with a force-directed layout (connected nodes pull together, others push apart) and drawn in your browser.

**View Controls:**

- **Zoom In / Zoom Out** - Step the zoom in or out
- **Reset View** - Fits the entire graph to the canvas
- **Re-run Layout** (🔄) - Recomputes node positions
- **Export as PNG** (📥) - Saves the graph as an image (theme-aware background), useful for reports

### Interacting with the Graph

**Navigation:**
- **Drag canvas** - Pan around the graph
- **Scroll wheel** - Zoom in/out
- **Hover node** - Highlights it
- **Click node** - View detailed info panel

**Node Selection:**

When you click a node, a detail panel appears showing:

**Message Node:**
- Views, forwards, comments count (grid)
- Content preview (up to 200 chars)

**Author Node:**
- Description: "Channel or user who posted"
- User ID (if available)

**Forward Source Node:**
- Description: "Message was forwarded from this channel"
- Channel ID and original message ID

**Parent Message Node:**
- Description: "This message is a reply to another message"
- Parent message ID

**Comment Node:**
- Commenter name
- Full comment text (scrollable)
- Post timestamp

**Reaction Node:**
- Reaction emoji (large display)
- Total count
- "Paid Reaction" badge (if applicable)

**Close Panel:**
- Click X button
- Click anywhere on canvas
- Click another node

### Empty States

**No Relationships:**
```
Message has none of:
- Author information
- Forward source
- Reply parent
- Comments
```

**What's captured:**
- Author (if available from Telegram)
- Forward source (channel + original message)
- Reply parent (conversation threading)
- Comments from linked discussion groups

**Why it might be empty:**
- Message is standalone (no forwards/replies)
- Channel doesn't expose author info
- No linked discussion group
- Older messages archived before relationship tracking was added

## Channel Social Graph (Coming Soon)

Channel-level analysis (which channels forward from which, communities of related channels, influence ranking) is not yet available in the web interface.

## Use Cases

### Verify Content Origin

**Goal:** Determine if message is original or forwarded

**Steps:**
1. Open message social graph
2. Look for "Forward Source" node
3. Check channel ID and original message ID
4. Navigate to source if needed
5. Compare timestamps

**Indicators:**
- No forward source = likely original
- Forward source present = redistributed content
- Multiple hops = viral spread

### Track Discussion Threads

**Goal:** Follow conversation across comments

**Steps:**
1. Check for "Comments" indicator (💬)
2. Open social graph
3. View comment nodes
4. Read comment text in detail panel
5. Identify key participants

**Use for:**
- Audience sentiment analysis
- Key influencer identification
- Misinformation tracking

### Analyze Message Impact

**Goal:** Understand reach and engagement

**Steps:**
1. Review engagement metrics (views, forwards)
2. Check virality badge
3. Count reactions and types
4. Examine comment volume
5. Compare to channel average

**Metrics:**
- High views + low forwards = passive audience
- Low views + high forwards = active amplification
- Many reactions = strong sentiment
- Many comments = controversial/engaging

### Detect Coordination

**Goal:** Identify potential coordinated behavior

**Steps:**
1. Check multiple related messages
2. Compare forward sources
3. Look for timing patterns
4. Check channel clusters
5. Note similar reaction patterns

**Red flags:**
- Same messages forwarded by many channels simultaneously
- Unusual reaction patterns (same emojis, counts)
- Identical comment text across channels

## Performance & Limitations

### Graph Size

Per-message graphs are normally small (a handful of nodes up to a few dozen) and render instantly. Messages with very many comments or reactions produce larger graphs that take a few seconds to lay out.

### Browser Performance

**Recommended:**
- Modern browser (Chrome, Firefox, Safari, Edge)
- 4GB+ RAM available
- Hardware acceleration enabled

**Slow performance:**
- Close other tabs
- Reduce window size
- Use a browser with hardware acceleration enabled (the graph uses WebGL)

### Data Freshness

- Message metadata (views, forwards), author information and forward sources are available as soon as the message is archived
- Comments, reactions and engagement figures are collected in the background and refresh periodically, so recent messages may show partial numbers

## Advanced Features

### Graph Export for Analysis

**PNG Export:**
- Use for reports and documentation
- Includes all visible nodes and edges
- Theme-aware (matches light/dark mode)

**Data export (GraphML, JSON, CSV)** is not available in the interface. Programmatic access is through the API; see [API Reference](../reference/api-endpoints.md).

### Theme Adaptation

Graph automatically adapts to light/dark theme:

**Dark Mode:**
- Dark background (#0f172a)
- Light node borders
- Muted colors for eyes

**Light Mode:**
- Light background (#f8fafc)
- Darker node borders
- Vibrant colors

## Troubleshooting

### Graph Not Loading

**Check:**
- JavaScript enabled
- Modern browser (updated)
- No browser extensions blocking canvas

**Solutions:**
- Refresh page
- Clear browser cache
- Try a different browser
- Check that WebGL is enabled (the graph will not draw without it)

### Empty Graph

**Reasons:**
- Message truly has no relationships
- Data not yet computed (new message)
- Temporary API error

**Verify:**
- Check "Social Relationship Indicators" section
- See if any show "Yes" status
- Wait a few minutes and refresh

### Performance Issues

**Symptoms:**
- Slow rendering
- Laggy interactions
- Browser freeze

**Solutions:**
- Close other browser tabs
- Use zoom controls to reduce visible area
- Re-run the layout once after the graph loads

### Graph Too Cluttered

**Techniques:**
- Use zoom controls to focus on region
- Click nodes to read details instead of relying on labels
- Re-run the layout to separate overlapping nodes

## Best Practices

### Effective Graph Analysis

**Start with Metrics:**
1. Check engagement metrics first
2. Identify if content is viral
3. Look for unusual patterns

**Explore Relationships:**
1. Identify node types
2. Follow edges to understand flow
3. Click nodes for details

**Compare Across Messages:**
1. Check multiple related messages
2. Look for patterns
3. Note differences

### Research Workflow

**Content Verification:**
```
1. Open suspicious message
2. Check forward source
3. Verify original if present
4. Compare timestamps
5. Document chain of custody
```

**Influence Analysis:**
```
1. Identify high-forward messages
2. Map forward sources
3. Calculate reach
4. Identify amplifiers
5. Report findings
```

**Sentiment Analysis:**
```
1. Review reaction distribution
2. Read comment sample
3. Note dominant sentiment
4. Compare to similar content
5. Track changes over time
```

---

**Next Steps:**
- [Search Messages](searching.md) to find content to analyze
- [Explore Entities](entities.md) for entity-level relationships
- [Subscribe to RSS](rss-feeds.md) to monitor viral content
