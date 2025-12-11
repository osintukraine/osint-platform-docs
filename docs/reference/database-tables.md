# Database Tables Reference

Complete reference documentation for all database tables in the OSINT Intelligence Platform.

**Source**: `/infrastructure/postgres/init.sql`
**Database**: PostgreSQL 16+ with pgvector, pg_trgm, btree_gin extensions
**Last Updated**: 2025-12-09

---

## Table of Contents

- [Core Data Tables](#core-data-tables)
- [User Management & Authentication](#user-management--authentication)
- [Message Processing & Classification](#message-processing--classification)
- [Events & Incidents (V2)](#events--incidents-v2)
- [Entity Knowledge Graph](#entity-knowledge-graph)
- [RSS Intelligence Layer](#rss-intelligence-layer)
- [Social Graph & Engagement](#social-graph--engagement)
- [Configuration & Runtime Settings](#configuration--runtime-settings)
- [Enrichment & Background Tasks](#enrichment--background-tasks)
- [Audit & Decision Tracking](#audit--decision-tracking)

---

## Core Data Tables

### `channels`

Telegram channels being monitored by the platform.

**Purpose**: Central registry of all Telegram channels with folder-based management rules.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY): Internal channel ID
- `telegram_id` (BIGINT UNIQUE): Telegram's channel ID
- `username` (VARCHAR): Channel @username (nullable for private channels)
- `name` (VARCHAR): Display name
- `folder` (VARCHAR): Telegram folder name (e.g., "Archive-UA", "Monitor-RU")
- `rule` (VARCHAR): Processing rule (`archive_all`, `selective_archive`, `discovery`)
- `source_type` (VARCHAR): Channel categorization (state_media, military_unit, journalist, osint_aggregator, etc.)
- `affiliation` (VARCHAR): russia, ukraine, neutral, unknown
- `source_account` (VARCHAR): Which Telegram session monitors this (for multi-account)
- `discovery_status` (VARCHAR): For auto-discovered channels (discovered, evaluating, promoted, rejected)
- `quality_metrics` (JSONB): Spam/quality tracking for discovered channels
- `active` (BOOLEAN): Whether currently monitoring

**Important Indexes**:
- `idx_channels_telegram_id` - Fast lookup by Telegram ID
- `idx_channels_folder` - Folder-based queries
- `idx_channels_rule` - Processing rule filtering
- `idx_channels_discovery_status` - Discovery workflow
- `idx_channels_source_type` - Source categorization

**Foreign Keys**: None (root table)

**Related Tables**: `messages`, `channel_interactions`, `folder_rules`

---

### `messages`

All Telegram messages after spam filtering.

**Purpose**: Core message storage with content, metadata, AI enrichment, and semantic search.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY): Internal message ID
- `message_id` (BIGINT): Telegram's message ID within channel
- `channel_id` (INTEGER FK → channels): Source channel
- `content` (TEXT): Original message text
- `content_translated` (TEXT): Translated content (DeepL/Google)
- `telegram_date` (TIMESTAMPTZ): When posted on Telegram
- `media_type` (VARCHAR): photo, video, document, etc.
- `is_spam` (BOOLEAN): LLM spam classification
- `spam_type` (VARCHAR): financial, promotional, off_topic, forwarding
- `osint_topic` (VARCHAR): combat, equipment, casualties, diplomatic, etc. (12 categories)
- `importance_level` (VARCHAR): high, medium, low
- `content_embedding` (vector(384)): Semantic search embedding
- `search_vector` (TSVECTOR): Full-text search (auto-generated)
- `entities` (JSONB): Extracted hashtags, mentions, locations, military units
- `author_user_id` (BIGINT): Message author's Telegram ID
- `forward_from_channel_id` (BIGINT): Original channel if forwarded
- `has_comments` (BOOLEAN): Whether channel has discussion enabled
- `comments_count` (INTEGER): Number of comments
- `views` (INTEGER): View count from Telegram
- `forwards` (INTEGER): Forward count

**Important Indexes**:
- `idx_messages_content_embedding` (HNSW) - Vector similarity search (m=16, ef_construction=64)
- `idx_messages_search_vector` (GIN) - Full-text search
- `idx_messages_telegram_date` - Date-based queries
- `idx_messages_importance_level` - Filter by importance
- `idx_messages_osint_topic` - Topic filtering
- `idx_messages_is_spam` - Spam filtering
- `idx_messages_content_hash` - Deduplication
- `uq_messages_channel_message` (UNIQUE) - Prevent duplicates per channel

**Foreign Keys**:
- `channel_id` → `channels(id)` ON DELETE CASCADE

**Related Tables**: `message_media`, `message_tags`, `message_entities`, `message_quarantine`, `event_messages`

---

### `message_quarantine`

Off-topic or suspected spam held for human review.

**Purpose**: 7-day holding area for messages that fail Ukraine relevance checks. Human review creates LLM training feedback.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `channel_id` (INTEGER FK → channels)
- `telegram_message_id` (BIGINT): Original Telegram ID
- `content` (TEXT): Message content
- `quarantine_reason` (VARCHAR): off_topic, spam_suspected, low_confidence
- `quarantine_details` (TEXT): LLM reasoning
- `is_ukraine_relevant` (BOOLEAN): LLM classification
- `review_status` (VARCHAR): pending, approved, rejected, expired
- `reviewed_by` (VARCHAR): Reviewer username
- `review_notes` (TEXT): Human feedback
- `expires_at` (TIMESTAMPTZ): Auto-cleanup after 7 days
- `feedback_sent_to_llm` (BOOLEAN): Whether used for LLM training

**Important Indexes**:
- `idx_quarantine_review_status` - Filter by review state
- `idx_quarantine_expires_at` - Cleanup expired entries
- `idx_quarantine_channel` - Per-channel quarantine view
- `uq_quarantine_channel_message` (UNIQUE) - Prevent duplicates

**Foreign Keys**:
- `channel_id` → `channels(id)` ON DELETE CASCADE

**Related Tables**: `messages`, `decision_log`

---

### `media_files`

Content-addressed media storage (SHA-256 deduplication).

**Purpose**: Stores media files with SHA-256 deduplication. Files archived to MinIO S3.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `sha256` (VARCHAR(64) UNIQUE): Content hash for deduplication
- `s3_key` (TEXT): MinIO path (media/{hash[:2]}/{hash[2:4]}/{hash}.ext)
- `file_size` (BIGINT): Bytes
- `mime_type` (VARCHAR): image/jpeg, video/mp4, etc.
- `telegram_file_id` (TEXT): Telegram's file reference
- `telegram_url` (TEXT): Original Telegram URL (expires)
- `reference_count` (INTEGER): How many messages reference this file
- `first_seen` (TIMESTAMP): When first archived

**Important Indexes**:
- `idx_media_files_sha256` - Fast deduplication lookups

**Foreign Keys**: None (referenced by `message_media`)

**Related Tables**: `message_media`

---

### `message_media`

Junction table linking messages to media files (many-to-many).

**Purpose**: Handles albums (multiple media per message) and deduplication (same media in multiple messages).

**Key Columns**:
- `message_id` (BIGINT FK → messages)
- `media_id` (INTEGER FK → media_files)
- PRIMARY KEY (`message_id`, `media_id`)

**Important Indexes**:
- `idx_message_media_message_id` - Find all media for a message
- `idx_message_media_media_id` - Find all messages using a media file

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE
- `media_id` → `media_files(id)` ON DELETE CASCADE

---

## User Management & Authentication

### `users`

Application users (legacy - pre-Kratos).

**Purpose**: Basic user authentication. Being replaced by Ory Kratos.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `username` (VARCHAR(50) UNIQUE)
- `email` (VARCHAR(255) UNIQUE)
- `hashed_password` (VARCHAR)
- `is_active` (BOOLEAN)
- `is_admin` (BOOLEAN)
- `created_at` (TIMESTAMPTZ)
- `last_login` (TIMESTAMPTZ)

**Important Indexes**:
- `idx_users_username`
- `idx_users_email`

---

### `user_roles`

Role-based access control mapped to Kratos identities.

**Purpose**: Maps Ory Kratos identity UUIDs to application roles (admin, moderator, authenticated, anonymous).

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `kratos_identity_id` (UUID UNIQUE): Foreign key to kratos.identities
- `role` (VARCHAR): anonymous, authenticated, admin, moderator
- `is_active` (BOOLEAN): Can deactivate without deleting Kratos identity
- `banned_until` (TIMESTAMP): Temporary ban expiry
- `ban_reason` (TEXT)

**Important Indexes**:
- `idx_user_roles_kratos_id`
- `idx_user_roles_role`

**Related Tables**: `kratos.identities` (external schema)

---

### `feed_tokens`

Authenticated RSS/Atom/JSON feed subscriptions.

**Purpose**: Generate secure tokens for users to subscribe to personalized RSS feeds with filters.

**Key Columns**:
- `id` (UUID PRIMARY KEY)
- `user_id` (INTEGER FK → users)
- `token_hash` (BYTEA UNIQUE): SHA-256 of token
- `token_prefix` (CHAR(8)): User-visible identifier (e.g., "ft_a1b2")
- `signing_secret` (BYTEA): 32-byte HMAC signing key
- `label` (VARCHAR): User-defined label ("My Feedly")
- `revoked_at` (TIMESTAMPTZ)
- `last_used_at` (TIMESTAMPTZ)
- `use_count` (BIGINT): Usage tracking

**Important Indexes**:
- `idx_feed_tokens_user_active` (WHERE revoked_at IS NULL)
- `idx_feed_tokens_prefix`
- `idx_feed_tokens_last_used`

**Foreign Keys**:
- `user_id` → `users(id)` ON DELETE CASCADE

---

### `export_jobs`

Background data export jobs (async CSV/JSON exports).

**Purpose**: Handles large data exports with progress tracking and secure download tokens.

**Key Columns**:
- `id` (UUID PRIMARY KEY)
- `user_id` (INTEGER FK → users)
- `export_type` (VARCHAR): messages, channels, entities, audit_log
- `format` (VARCHAR): json, csv, jsonl
- `profile` (VARCHAR): minimal, standard, full, custom
- `filters` (JSONB): Query filters (date range, channels, importance)
- `columns` (JSONB): Column selection for custom profile
- `status` (VARCHAR): pending, processing, completed, failed, cancelled
- `total_rows` (INTEGER)
- `processed_rows` (INTEGER)
- `s3_key` (VARCHAR): MinIO export file path
- `download_token` (UUID): Secure download link
- `download_token_expires_at` (TIMESTAMPTZ)
- `max_downloads` (INTEGER): Limit to prevent abuse

**Important Indexes**:
- `idx_export_jobs_user_status`
- `idx_export_jobs_pending` (WHERE status = 'pending')
- `idx_export_jobs_expires`
- `idx_export_jobs_download_token` (UNIQUE)

**Foreign Keys**:
- `user_id` → `users(id)` ON DELETE SET NULL

---

## Message Processing & Classification

### `message_tags`

AI-generated tags for messages (many-to-many).

**Purpose**: LLM-generated semantic tags for filtering and discovery.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `message_id` (BIGINT FK → messages)
- `tag` (VARCHAR(100)): Tag name
- `tag_type` (VARCHAR): keyword, topic, entity, emotion, urgency
- `confidence` (NUMERIC(3,2)): 0.00-1.00
- `generated_by` (VARCHAR): ollama:qwen2.5:3b, rule_based, manual
- `created_at` (TIMESTAMPTZ)

**Important Indexes**:
- `idx_message_tags_message_id`
- `idx_message_tags_tag`
- `idx_message_tags_type`
- `uq_message_tag` (UNIQUE on message_id, tag, tag_type)

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE

**Related Tables**: `tag_stats`

---

### `tag_stats`

Tag popularity tracking for autocomplete and trending tags.

**Purpose**: Denormalized tag statistics for UI autocomplete and tag clouds.

**Key Columns**:
- `tag` (VARCHAR(100))
- `tag_type` (VARCHAR(50))
- `usage_count` (INTEGER): How many times used
- `avg_confidence` (NUMERIC(3,2))
- `first_seen` (TIMESTAMPTZ)
- `last_seen` (TIMESTAMPTZ)
- PRIMARY KEY (`tag`, `tag_type`)

**Important Indexes**:
- `idx_tag_stats_usage` (usage_count DESC)
- `idx_tag_stats_last_seen`

---

## Events & Incidents 
### `events`

Real-world incidents detected from RSS articles and Telegram clustering.

**Purpose**: Track actual events (strikes, advances, political developments) with tiered confidence validation.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `title` (TEXT): Event headline
- `summary` (TEXT): LLM-generated summary
- `event_type` (VARCHAR): strike, advance, retreat, political, humanitarian, etc.
- `location_name` (TEXT): "Pokrovsk", "Kherson Oblast"
- `location_coords` (POINT): Lat/lon if available
- `event_date` (DATE): Primary date
- `tier_status` (VARCHAR): breaking, developing, confirmed, verified
- `is_major` (BOOLEAN): Pinned historical events (never auto-archive)
- `rss_source_count` (INTEGER): Number of news sources validating
- `telegram_channel_count` (INTEGER): Number of channels discussing
- `telegram_message_count` (INTEGER): Total messages linked
- `content_embedding` (vector(384)): For similarity/deduplication
- `search_vector` (TSVECTOR): Full-text search
- `archived_at` (TIMESTAMPTZ): NULL = active

**Important Indexes**:
- `idx_events_tier_status` (WHERE archived_at IS NULL)
- `idx_events_is_major` (WHERE is_major = TRUE)
- `idx_events_last_activity`
- `idx_events_event_type`
- `idx_events_embedding` (IVFFLAT)
- `idx_events_search_vector` (GIN)

**Related Tables**: `event_messages`, `event_sources`, `event_config`

---

### `event_messages`

Junction table linking Telegram messages to events.

**Purpose**: Links Telegram messages to real-world events with confidence scores.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `event_id` (BIGINT FK → events)
- `message_id` (BIGINT FK → messages)
- `match_confidence` (NUMERIC(4,3)): 0.000-1.000
- `match_method` (VARCHAR): embedding_similarity, location_match, llm_confirmed, cluster_detection
- `matched_at` (TIMESTAMPTZ)

**Important Indexes**:
- `idx_event_messages_message`
- `idx_event_messages_event`
- `uq_event_messages` (UNIQUE on event_id, message_id)

**Foreign Keys**:
- `event_id` → `events(id)` ON DELETE CASCADE
- `message_id` → `messages(id)` ON DELETE CASCADE

---

### `event_sources`

Links RSS articles to events (validation layer).

**Purpose**: Links authoritative news sources to events for tiered confidence (breaking → verified).

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `event_id` (BIGINT FK → events)
- `rss_article_id` (BIGINT FK → external_news)
- `is_primary_source` (BOOLEAN): The article that created/seeded the event
- `linked_at` (TIMESTAMPTZ)

**Important Indexes**:
- `idx_event_sources_event`
- `idx_event_sources_article`
- `idx_event_sources_primary` (WHERE is_primary_source = TRUE)
- `uq_event_sources` (UNIQUE on event_id, rss_article_id)

**Foreign Keys**:
- `event_id` → `events(id)` ON DELETE CASCADE
- `rss_article_id` → `external_news(id)` ON DELETE CASCADE

---

### `event_config`

Configurable thresholds for event detection.

**Purpose**: Runtime-editable event detection parameters (no restarts required).

**Key Columns**:
- `key` (VARCHAR(100) PRIMARY KEY): Config key
- `value` (TEXT): Config value
- `description` (TEXT)
- `updated_at` (TIMESTAMPTZ)

**Default Values**:
- `breaking_to_developing_hours`: 2
- `developing_to_confirmed_rss`: 1
- `confirmed_to_verified_rss`: 3
- `auto_archive_inactive_days`: 7
- `cluster_min_channels`: 3
- `cluster_time_window_hours`: 2
- `embedding_match_threshold`: 0.85
- `novelty_threshold`: 0.88

---

## Entity Knowledge Graph

### `curated_entities`

Curated entity knowledge graph (ArmyGuide, Root.NK, ODIN).

**Purpose**: Military equipment, individuals, organizations, locations from trusted CSV sources.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `entity_type` (VARCHAR): equipment, individual, organization, location, military_unit, ship, aircraft, military_vehicle, military_weapon, electronic_warfare, component
- `name` (TEXT): Entity name
- `aliases` (TEXT[]): Alternative names/spellings
- `description` (TEXT): Full description
- `latitude` (FLOAT): For locations
- `longitude` (FLOAT): For locations
- `source_reference` (TEXT): armyguide, odin_sanctions, etc.
- `metadata` (JSONB): Original source data preserved
- `content_hash` (VARCHAR(64) UNIQUE): SHA-256 for deduplication
- `embedding` (vector(384)): Semantic search
- `search_vector` (TSVECTOR GENERATED): Full-text search

**Important Indexes**:
- `idx_curated_entities_type`
- `idx_curated_entities_source`
- `idx_curated_entities_hash`
- `idx_curated_entities_search` (GIN)
- `idx_curated_entities_embedding` (IVFFLAT)
- `idx_curated_entities_coords` (WHERE lat/lon NOT NULL)
- `idx_curated_entities_name_trgm` (GIN) - Fuzzy name search

**Related Tables**: `message_entities`

---

### `message_entities`

Junction table linking messages to curated entities.

**Purpose**: Knowledge graph relationships between messages and military/political entities.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `message_id` (BIGINT FK → messages)
- `entity_id` (BIGINT FK → curated_entities)
- `similarity_score` (FLOAT): 0.0-1.0 confidence
- `match_type` (VARCHAR): semantic, exact_name, alias, hashtag
- `context_snippet` (TEXT): Where in message entity was found
- `matched_at` (TIMESTAMPTZ)

**Important Indexes**:
- `idx_message_entities_message`
- `idx_message_entities_entity`
- `idx_message_entities_similarity`
- `idx_message_entities_network` (message_id, entity_id, similarity_score)
- UNIQUE (message_id, entity_id)

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE
- `entity_id` → `curated_entities(id)` ON DELETE CASCADE

---

### `opensanctions_entities`

OpenSanctions entities (sanctions, PEPs, criminals).

**Purpose**: Sanctions lists, politically exposed persons, criminal organizations from OpenSanctions API.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `opensanctions_id` (VARCHAR(255) UNIQUE): e.g., "ofac-13661"
- `entity_type` (VARCHAR): Person, Organization, Vessel, Aircraft
- `name` (VARCHAR(500))
- `aliases` (TEXT[]): All known names
- `description` (TEXT): Full bio
- `properties` (JSONB): birth_date, nationalities, positions, addresses
- `risk_classification` (VARCHAR): sanctioned, pep, criminal, corporate
- `datasets` (TEXT[]): ["us_ofac_sdn", "eu_fsf"]
- `entity_embedding` (vector(384)): Semantic search
- `mention_count` (INTEGER): How many times mentioned

**Important Indexes**:
- `idx_opensanctions_entities_external_id` (UNIQUE)
- `idx_opensanctions_entities_name`
- `idx_opensanctions_entities_type`
- `idx_opensanctions_entities_risk`
- `idx_opensanctions_entities_datasets_gin` (GIN)
- `idx_opensanctions_entities_embedding` (HNSW)
- `idx_opensanctions_name_trgm` (GIN) - Fuzzy search

**Related Tables**: `opensanctions_message_entities`, `entity_relationships`

---

### `opensanctions_message_entities`

Junction table linking messages to OpenSanctions entities.

**Purpose**: Tracks mentions of sanctioned individuals/organizations in messages.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `message_id` (BIGINT FK → messages)
- `entity_id` (INTEGER FK → opensanctions_entities)
- `match_score` (NUMERIC(3,2)): 0.00-1.00
- `match_method` (VARCHAR): real_time, async_enrichment, manual
- `context_snippet` (TEXT)
- `match_features` (JSONB): OpenSanctions API match details
- `matched_at` (TIMESTAMPTZ)

**Important Indexes**:
- `idx_opensanctions_message_entities_message_id`
- `idx_opensanctions_message_entities_entity_id`
- `idx_opensanctions_message_entities_match_score`
- UNIQUE (message_id, entity_id)

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE
- `entity_id` → `opensanctions_entities(id)` ON DELETE CASCADE

---

### `entity_relationships`

Entity relationship graphs (family, business, ownership).

**Purpose**: Network analysis of entity connections from OpenSanctions data.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `from_entity_id` (INTEGER FK → opensanctions_entities)
- `to_entity_id` (INTEGER FK → opensanctions_entities)
- `relationship_type` (VARCHAR): family, business, ownership, associate
- `relationship_subtype` (VARCHAR): spouse, child, director, shareholder
- `relationship_properties` (JSONB)
- `confidence` (NUMERIC(3,2)): 0.00-1.00
- `source_datasets` (TEXT[])

**Important Indexes**:
- `idx_entity_relationships_from`
- `idx_entity_relationships_to`
- `idx_entity_relationships_type`
- `idx_entity_relationships_both` (from_entity_id, to_entity_id)
- UNIQUE (from_entity_id, to_entity_id, relationship_type)

**Foreign Keys**:
- `from_entity_id` → `opensanctions_entities(id)` ON DELETE CASCADE
- `to_entity_id` → `opensanctions_entities(id)` ON DELETE CASCADE

---

## RSS Intelligence Layer

### `rss_feeds`

External news RSS/Atom feeds.

**Purpose**: Configured RSS feeds for fact-checking and event validation.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `name` (VARCHAR): Feed display name
- `url` (TEXT UNIQUE): RSS feed URL
- `category` (VARCHAR): aggregator, news_agency, military, etc.
- `trust_level` (INTEGER): 1-5 (5 = highest)
- `language` (VARCHAR): en, ru, uk
- `country` (VARCHAR)
- `active` (BOOLEAN)
- `last_polled_at` (TIMESTAMPTZ)
- `articles_fetched_total` (INTEGER)

**Important Indexes**:
- `idx_rss_feeds_active` (active, last_polled_at)
- `idx_rss_feeds_category`

**Related Tables**: `external_news`

---

### `news_sources`

Individual news source trust levels.

**Purpose**: Per-domain trust configuration for aggregator feeds (e.g., separate trust for kyivpost.com vs pravda.com.ua).

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `domain` (VARCHAR(255) UNIQUE): Source domain
- `name` (VARCHAR): Display name
- `trust_level` (INTEGER): 1-5
- `category` (VARCHAR): neutral, pro_ukraine, pro_russia
- `bias` (VARCHAR): neutral, pro_ukraine, pro_russia, mixed
- `verified` (BOOLEAN): Manually verified source
- `articles_count` (INTEGER): Total articles seen

**Important Indexes**:
- `idx_news_sources_domain`
- `idx_news_sources_trust_level`
- `idx_news_sources_category`

**Related Tables**: `external_news`

---

### `external_news`

RSS articles for cross-correlation and fact-checking.

**Purpose**: Stores RSS articles with embeddings for semantic correlation with Telegram messages.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `feed_id` (INTEGER FK → rss_feeds)
- `source_id` (INTEGER FK → news_sources): Individual source
- `source_domain` (VARCHAR): Extracted from URL
- `title` (TEXT)
- `content` (TEXT)
- `url` (TEXT UNIQUE)
- `url_hash` (VARCHAR(64)): Deduplication
- `published_at` (TIMESTAMPTZ)
- `author` (VARCHAR)
- `tags` (TEXT[])
- `entities` (JSONB)
- `embedding` (vector(384)): Semantic search
- `search_vector` (TSVECTOR): Full-text search
- `source_trust_level` (INTEGER): Copied from news_sources
- `correlation_count` (INTEGER): How many Telegram messages correlated

**Important Indexes**:
- `idx_external_news_feed`
- `idx_external_news_published`
- `idx_external_news_url_hash`
- `idx_external_news_source_id`
- `idx_external_news_source_domain`
- `idx_external_news_embedding` (IVFFLAT)
- `idx_external_news_search_vector` (GIN)

**Foreign Keys**:
- `feed_id` → `rss_feeds(id)` ON DELETE SET NULL
- `source_id` → `news_sources(id)` ON DELETE SET NULL

**Related Tables**: `message_news_correlations`, `event_sources`

---

### `message_news_correlations`

Links Telegram messages to RSS articles (fact-checking layer).

**Purpose**: Semantic correlation between Telegram claims and news sources for validation.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `message_id` (BIGINT FK → messages)
- `news_id` (BIGINT FK → external_news)
- `similarity_score` (NUMERIC): pgvector cosine similarity
- `entity_overlap_score` (INTEGER)
- `time_proximity_hours` (NUMERIC)
- `correlation_type` (VARCHAR): semantic, temporal, entity_based
- `validation_type` (VARCHAR): confirms, contradicts, context, alternative
- `relevance_explanation` (TEXT): LLM reasoning
- `confidence` (FLOAT): 0.0-1.0
- `perspective_difference` (BOOLEAN): Different viewpoints on same event

**Important Indexes**:
- `idx_correlations_message`
- `idx_correlations_news`
- `idx_correlations_similarity`
- `idx_correlations_validation_type`
- UNIQUE (message_id, news_id)

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE
- `news_id` → `external_news(id)` ON DELETE CASCADE

---

### `message_validations`

Cached LLM validation summaries.

**Purpose**: Cache expensive LLM fact-checking summaries to avoid repeated processing.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `message_id` (BIGINT FK → messages UNIQUE)
- `summary` (TEXT): LLM-generated validation summary
- `confidence_score` (FLOAT)
- `total_articles_found` (INTEGER)
- `cached_at` (TIMESTAMPTZ)
- `expires_at` (TIMESTAMPTZ): Cache TTL

**Important Indexes**:
- `idx_message_validations_message`
- `idx_message_validations_expires`

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE

---

## Social Graph & Engagement

### `telegram_users`

Telegram user profiles who interact with messages.

**Purpose**: Track users who author, forward, or comment on messages for influence mapping.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `telegram_id` (BIGINT UNIQUE): Telegram user ID
- `first_name` (VARCHAR)
- `last_name` (VARCHAR)
- `username` (VARCHAR)
- `is_bot` (BOOLEAN)
- `is_verified` (BOOLEAN)
- `is_premium` (BOOLEAN)
- `interaction_count` (INTEGER): How many times seen
- `first_seen` (TIMESTAMPTZ)
- `last_seen` (TIMESTAMPTZ)

**Important Indexes**:
- `idx_telegram_users_telegram_id`
- `idx_telegram_users_username` (WHERE username IS NOT NULL)
- `idx_telegram_users_last_seen`

**Related Tables**: `messages` (author_user_id), `message_comments`

---

### `message_reactions`

Emoji reactions to messages (sentiment proxy).

**Purpose**: Track emoji reactions for sentiment analysis and engagement.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `message_id` (BIGINT FK → messages)
- `emoji` (VARCHAR(20)): 👍, 👎, ❤️, 🔥, 💩, etc.
- `count` (INTEGER): Aggregated or individual count
- `user_id` (BIGINT FK → telegram_users): Optional individual tracking
- `reacted_at` (TIMESTAMPTZ)

**Important Indexes**:
- `idx_message_reactions_message`
- `idx_message_reactions_emoji`
- `idx_message_reactions_count`
- UNIQUE (message_id, emoji, user_id)

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE
- `user_id` → `telegram_users(id)` ON DELETE CASCADE

---

### `message_comments`

Comments from Telegram discussion groups.

**Purpose**: Track discussion/comment threads on channel posts.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `parent_message_id` (BIGINT FK → messages): Original channel post
- `comment_id` (BIGINT): Telegram message_id in discussion group
- `author_user_id` (BIGINT): Commenter's Telegram ID
- `content` (TEXT): Comment text
- `translated_content` (TEXT)
- `telegram_date` (TIMESTAMPTZ)
- `is_pinned` (BOOLEAN)
- `views` (INTEGER)
- `reactions_count` (INTEGER)
- `replies_count` (INTEGER): Thread depth
- `reply_to_comment_id` (BIGINT): Nested replies

**Important Indexes**:
- `idx_message_comments_parent` (parent_message_id, telegram_date DESC)
- `idx_message_comments_author`
- UNIQUE (parent_message_id, comment_id)

**Foreign Keys**:
- `parent_message_id` → `messages(id)` ON DELETE CASCADE

---

### `message_engagement_timeline`

Time-series engagement snapshots (virality tracking).

**Purpose**: Track engagement metrics over time to detect viral posts and propagation patterns.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `message_id` (BIGINT FK → messages)
- `snapshot_at` (TIMESTAMPTZ): When polled
- `views_count` (INTEGER): Snapshot
- `forwards_count` (INTEGER)
- `reactions_count` (INTEGER)
- `comments_count` (INTEGER)
- `views_delta` (INTEGER): Change since last snapshot
- `propagation_rate` (NUMERIC): Forwards per hour
- `engagement_rate` (NUMERIC): (reactions + comments) / views * 100

**Important Indexes**:
- `idx_engagement_timeline_message` (message_id, snapshot_at DESC)
- `idx_engagement_timeline_virality` (propagation_rate DESC NULLS LAST)
- UNIQUE (message_id, snapshot_at)

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE

---

### `viral_posts`

High-engagement posts for enhanced comment polling.

**Purpose**: Flag viral posts for more frequent comment polling (every 4h vs standard schedule).

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `message_id` (BIGINT FK → messages UNIQUE)
- `viral_reason` (VARCHAR): high_views, high_forwards, high_comments, velocity
- `viral_score` (INTEGER): Composite score
- `views_at_detection` (INTEGER)
- `forwards_at_detection` (INTEGER)
- `channel_avg_views` (INTEGER): Relative comparison
- `last_comment_check` (TIMESTAMPTZ)
- `is_active` (BOOLEAN): FALSE after 30 days or engagement plateau
- `deactivation_reason` (VARCHAR): age_limit, engagement_plateau, manual

**Important Indexes**:
- `idx_viral_posts_active` (is_active, last_comment_check)
- `idx_viral_posts_message`

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE

---

### `channel_interactions`

Cross-channel forwarding patterns (influence mapping).

**Purpose**: Pre-computed metrics for who forwards from whom, coordination detection.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `from_channel_id` (INTEGER FK → channels)
- `to_channel_id` (INTEGER FK → channels)
- `forward_count` (INTEGER): How many messages forwarded
- `first_forward_at` (TIMESTAMPTZ)
- `last_forward_at` (TIMESTAMPTZ)
- `avg_forward_delay_hours` (NUMERIC): Time from original → forward
- `synchronized_forwards` (INTEGER): Forwards within 1 hour (coordination indicator)
- `topic_distribution` (JSONB): Which topics get forwarded

**Important Indexes**:
- `idx_channel_interactions_from` (from_channel_id, forward_count DESC)
- `idx_channel_interactions_to`
- `idx_channel_interactions_synchronized` (WHERE synchronized_forwards > 0)
- UNIQUE (from_channel_id, to_channel_id)
- CHECK (from_channel_id != to_channel_id)

**Foreign Keys**:
- `from_channel_id` → `channels(id)` ON DELETE CASCADE
- `to_channel_id` → `channels(id)` ON DELETE CASCADE

---

### `message_replies`

Reply relationship tracking for conversation threading.

**Purpose**: Tracks which messages are replies to other messages, enabling conversation threading and discussion analysis.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `parent_message_id` (INTEGER FK → messages): The message being replied to
- `reply_message_id` (INTEGER FK → messages UNIQUE): The reply message itself
- `reply_to_msg_id` (INTEGER): Telegram API's reply_to_msg_id field
- `discussion_group_id` (BIGINT): Telegram discussion group ID for channel comments
- `author_user_id` (BIGINT): Who wrote the reply
- `created_at` (TIMESTAMPTZ)

**Important Indexes**:
- `idx_message_replies_parent` - Find all replies to a message
- `idx_message_replies_reply` - Lookup by reply message
- `idx_message_replies_discussion_group` - Filter by discussion group

**Foreign Keys**:
- `parent_message_id` → `messages(id)` ON DELETE CASCADE
- `reply_message_id` → `messages(id)` ON DELETE CASCADE

**Related Tables**: `messages`, `message_comments`

---

### `message_forwards`

Forward chain tracking for propagation analysis.

**Purpose**: Tracks message forwarding relationships for virality analysis, information propagation studies, and coordination detection.

**Key Columns**:
- `id` (BIGSERIAL PRIMARY KEY)
- `original_message_id` (INTEGER FK → messages): The original message
- `forwarded_message_id` (INTEGER FK → messages): The forward
- `forward_date` (TIMESTAMPTZ): When forwarded
- `forward_from_id` (BIGINT): Original Telegram channel/user ID
- `forward_from_name` (TEXT): Original channel/user name
- `forward_signature` (TEXT): Forward signature if any
- `propagation_seconds` (INTEGER): Seconds from original post to forward
- `created_at` (TIMESTAMPTZ)
- UNIQUE (`original_message_id`, `forwarded_message_id`)

**Important Indexes**:
- `idx_message_forwards_original` - Find all forwards of a message
- `idx_message_forwards_forwarded` - Lookup by forwarded message
- `idx_message_forwards_from_id` - Filter by source channel/user
- `idx_message_forwards_date` - Time-based queries

**Foreign Keys**:
- `original_message_id` → `messages(id)` ON DELETE CASCADE
- `forwarded_message_id` → `messages(id)` ON DELETE CASCADE

**Related Tables**: `messages`, `channel_interactions`

**Usage Note**: The `propagation_seconds` column is useful for virality metrics - shorter propagation times indicate faster information spread.

---

## Configuration & Runtime Settings

### `platform_config`

Runtime platform configuration (NocoDB-editable).

**Purpose**: All feature toggles and thresholds editable via NocoDB without code changes.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `category` (VARCHAR): system, features, thresholds, enrichment, rss, etc.
- `key` (VARCHAR(100) UNIQUE): Config key
- `value` (TEXT): Config value
- `description` (TEXT)
- `data_type` (VARCHAR): string, integer, boolean, float
- `is_secret` (BOOLEAN)
- `restart_required` (BOOLEAN)

**Example Keys**:
- `features.spam_filter_enabled`
- `features.llm_scoring_enabled`
- `thresholds.monitoring_min_importance`
- `rss.correlation_similarity_threshold`
- `enrichment.embedding_batch_size`

**Important Indexes**:
- `idx_platform_config_category`
- `idx_platform_config_key`

---

### `model_configuration`

Multi-model LLM architecture configuration.

**Purpose**: Maps tasks to LLM models with fallback priorities (NocoDB-editable).

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `task` (VARCHAR): embedding, tag_generation, classification, message_classification
- `model_id` (VARCHAR): all-minilm, qwen2.5:3b, llama3.2:3b
- `enabled` (BOOLEAN)
- `priority` (INTEGER): 1=primary, 2=fallback
- `override_config` (JSONB): {'temperature': 0.5, 'max_tokens': 100}

**Important Indexes**:
- `idx_model_config_task` (task, enabled, priority)
- UNIQUE (task, model_id)

**Default Models**:
- embedding: all-minilm (priority 1)
- message_classification: qwen2.5:3b (priority 1), llama3.2:3b (priority 2)

---

### `llm_prompts`

LLM prompts (NocoDB runtime-editable).

**Purpose**: All LLM prompts editable via NocoDB. Changes take effect immediately without restarts.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `task` (VARCHAR): message_classification, event_extract, etc.
- `name` (VARCHAR): Human-readable identifier
- `prompt_type` (VARCHAR): system, user_template
- `content` (TEXT): The actual prompt text
- `version` (INTEGER): For rollback
- `is_active` (BOOLEAN): Only one active version per task
- `model_name` (VARCHAR): Per-prompt model override
- `model_parameters` (JSONB): {'temperature': 0.3, 'max_tokens': 2048}
- `variables` (TEXT[]): Template variables like {{content}}, {{channel_name}}
- `expected_output_format` (TEXT): json, text
- `usage_count` (INTEGER): Performance tracking
- `avg_latency_ms` (INTEGER)

**Important Indexes**:
- `idx_llm_prompts_task_active` (task, is_active)
- UNIQUE (task, version)

**Current Active Prompts**:
- message_classification v6 (chain-of-thought with <analysis> tags)
- event_extract, event_match, event_summarize

---

### `military_slang`

Military slang dictionary (NocoDB-editable).

**Purpose**: Ukrainian/Russian military slang injected into LLM prompts at runtime via {{MILITARY_SLANG}} placeholder.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `term` (VARCHAR(100)): The slang term (e.g., "прилетіло")
- `language` (VARCHAR(10)): uk, ru
- `meaning` (TEXT): English translation
- `topic_hint` (VARCHAR): combat, casualties, equipment
- `category` (VARCHAR): slang, abbreviation, location, derogatory
- `notes` (TEXT): Context (e.g., "Often mistranslated by DeepL")
- `is_active` (BOOLEAN)

**Important Indexes**:
- `idx_military_slang_active` (is_active, language)
- UNIQUE (term, language)

**Example Entries**:
- "прилетіло" (uk): "strike LANDED, got HIT" (combat)
- "бавовна" (uk): "explosion (lit. cotton)" (combat)
- "двохсотий" (uk): "KIA" (casualties)
- "ЗСУ" (uk): "Armed Forces of Ukraine" (abbreviation)

---

### `folder_rules`

Folder-based channel processing rules (NocoDB-editable).

**Purpose**: Maps Telegram folder patterns to processing rules. Previously hardcoded, now database-driven.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `folder_pattern` (VARCHAR(50) UNIQUE): Regex pattern (e.g., "^Archive")
- `rule` (VARCHAR): archive_all, selective_archive, discovery, test
- `description` (TEXT)
- `min_importance` (VARCHAR): For selective_archive (high, medium, low)
- `active` (BOOLEAN)

**Default Rules**:
- `^Archive` → archive_all
- `^Monitor` → selective_archive (high importance only)
- `^Discover` → discovery (auto-joined channels)

**Important Indexes**:
- `idx_folder_rules_active`

**Related Tables**: `channels`

---

### `translation_config`

Translation configuration per channel.

**Purpose**: Per-channel translation settings (global default when channel_id IS NULL).

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `channel_id` (INTEGER UNIQUE FK → channels): NULL = global default
- `enabled` (BOOLEAN)
- `provider` (VARCHAR): google, deepl, manual
- `target_language` (VARCHAR): en
- `translate_from_languages` (VARCHAR[]): ['ru', 'uk'] or NULL

**Foreign Keys**:
- `channel_id` → `channels(id)` ON DELETE CASCADE

---

### `translation_usage`

Translation API usage tracking.

**Purpose**: Daily cost tracking for translation providers.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `date` (DATE)
- `provider` (VARCHAR): google, deepl
- `characters_translated` (INTEGER)
- `cost_usd` (NUMERIC(10,4))
- `message_count` (INTEGER)
- UNIQUE (date, provider)

**Important Indexes**:
- `idx_translation_usage_date`
- `idx_translation_usage_provider`

---

## Enrichment & Background Tasks

### `enrichment_progress`

Enrichment task progress tracking.

**Purpose**: Resume-able backfill tasks with progress tracking (translation, embeddings, entity matching).

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `task_name` (VARCHAR(50) UNIQUE): translation, entity_matching, embeddings
- `last_processed_id` (BIGINT): Last message.id processed
- `messages_processed` (BIGINT)
- `messages_total` (BIGINT): Estimated total
- `last_run_at` (TIMESTAMPTZ)
- `status` (VARCHAR): idle, running, completed, paused, failed
- `error_message` (TEXT)

**Important Indexes**:
- `idx_enrichment_progress_status`
- `idx_enrichment_progress_task`

---

## Audit & Decision Tracking

### `decision_log`

Full audit trail for all LLM/processing decisions.

**Purpose**: Records ALL LLM classification decisions with chain-of-thought reasoning for verification and reprocessing.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `message_id` (INTEGER FK → messages)
- `channel_id` (INTEGER FK → channels)
- `telegram_message_id` (BIGINT): Backup if message deleted
- `decision_type` (VARCHAR): spam_filter, ukraine_relevance, importance, topic_classification, archive_decision
- `decision_value` (JSONB): Full decision JSON
- `decision_source` (VARCHAR): llm_v6, rule_based, fallback, human
- `llm_analysis` (TEXT): Chain-of-thought from <analysis> tags
- `llm_reasoning` (TEXT): Short reasoning field
- `llm_raw_response` (TEXT): Full LLM response for debugging
- `processing_time_ms` (INTEGER)
- `model_used` (VARCHAR): qwen2.5:3b, llama3.2:3b
- `prompt_version` (VARCHAR): v6, v5, etc.
- `verification_status` (VARCHAR): unverified, verified_correct, verified_incorrect, flagged, reprocessed
- `verified_by` (VARCHAR): automated:rule_name, human:username
- `reprocess_requested` (BOOLEAN)
- `reprocess_priority` (INTEGER)
- `previous_decision_id` (INTEGER FK → decision_log): Reprocessing chain

**Important Indexes**:
- `idx_decision_log_message`
- `idx_decision_log_channel` (channel_id, created_at DESC)
- `idx_decision_log_type` (decision_type, created_at DESC)
- `idx_decision_log_verification`
- `idx_decision_log_reprocess` (WHERE reprocess_requested = TRUE)
- `idx_decision_log_message_type` (message_id, decision_type, created_at DESC)

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE
- `channel_id` → `channels(id)` ON DELETE SET NULL
- `previous_decision_id` → `decision_log(id)` (self-referential)

---

### `admin_audit_log`

Admin action audit trail.

**Purpose**: Tracks all admin actions for security and compliance.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `kratos_identity_id` (UUID): Admin user
- `action` (VARCHAR): Action performed
- `resource_type` (VARCHAR): messages, channels, users
- `resource_id` (INTEGER)
- `details` (JSONB): Full action details
- `ip_address` (INET)
- `user_agent` (TEXT)
- `created_at` (TIMESTAMP)

**Important Indexes**:
- `idx_audit_log_user`
- `idx_audit_log_action`
- `idx_audit_log_created`

---

### `user_bookmarks`

User-saved messages.

**Purpose**: Allow users to bookmark messages for later review.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `kratos_identity_id` (UUID): User
- `message_id` (BIGINT FK → messages)
- `notes` (TEXT): User notes
- `created_at` (TIMESTAMP)
- UNIQUE (kratos_identity_id, message_id)

**Important Indexes**:
- `idx_user_bookmarks_user`
- `idx_user_bookmarks_message`

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE

---

### `user_comments`

User comments on messages (analyst notes).

**Purpose**: Allow authenticated users to add analysis notes to messages.

**Key Columns**:
- `id` (SERIAL PRIMARY KEY)
- `kratos_identity_id` (UUID): User
- `message_id` (BIGINT FK → messages)
- `content` (TEXT): Comment text
- `created_at` (TIMESTAMP)
- `is_deleted` (BOOLEAN): Soft delete

**Important Indexes**:
- `idx_user_comments_message`
- `idx_user_comments_user`

**Foreign Keys**:
- `message_id` → `messages(id)` ON DELETE CASCADE

---

## Materialized Views

### `message_social_graph`

Pre-computed social graph data for each message.

**Purpose**: Fast queries for message author, forwards, replies, reactions, comments. Refresh hourly.

**Refresh Command**: `REFRESH MATERIALIZED VIEW message_social_graph;`

**Key Columns**: message_id, channel_id, author_user_id, forward_from_channel_id, replies, reactions, comments_count, engagement metrics

---

### `channel_influence_network`

Pre-computed channel-to-channel influence map.

**Purpose**: Who forwards from whom, coordination patterns, propagation speed. Refresh daily.

**Refresh Command**: `REFRESH MATERIALIZED VIEW channel_influence_network;`

**Key Columns**: from_channel_id, to_channel_id, forward_count, coordination_level, propagation_speed

---

### `top_influencers`

Top 1000 influential Telegram users.

**Purpose**: User influence rankings by activity, reach, and engagement. Refresh daily.

**Refresh Command**: `REFRESH MATERIALIZED VIEW top_influencers;`

**Key Columns**: telegram_id, username, messages_authored, avg_views_per_message, influence_score

---

### `unified_message_entities`

Unified view of curated + OpenSanctions entities.

**Purpose**: Fast unified queries across both entity sources. Refresh periodically.

**Refresh Command**: `REFRESH MATERIALIZED VIEW unified_message_entities;`

**Key Columns**: message_id, entity_id, entity_source (curated/opensanctions), name, score, match_method

---

## Database Functions

### `find_similar_messages(query_embedding, threshold, limit, min_importance)`

Find semantically similar messages using pgvector.

**Returns**: message_id, content, similarity_score, importance_level, channel_name

---

### `hybrid_search(search_query, query_embedding, filters, limit)`

Combines full-text search + vector similarity.

**Returns**: message_id, content, text_rank, vector_similarity, combined_score, tags

**Weighting**: 40% text rank, 60% vector similarity

---

### `find_similar_events(embedding, threshold, limit, hours_lookback)`

Find similar events for novelty detection (deduplication).

**Returns**: event_id, title, similarity

---

### `get_user_role(kratos_identity_id)`

Returns user role: anonymous, authenticated, admin, moderator

---

## Triggers

### Auto-update triggers

- `update_channels_updated_at` - Auto-update channels.updated_at on UPDATE
- `update_messages_updated_at` - Auto-update messages.updated_at on UPDATE
- `messages_search_vector_trigger` - Auto-populate search_vector from content + translated_content
- `events_search_vector_trigger` - Auto-populate events.search_vector
- `external_news_search_vector_trigger` - Auto-populate external_news.search_vector

### Statistics triggers

- `trigger_update_event_v2_messages` - Update event denormalized counts when messages linked
- `trigger_update_event_v2_sources` - Update event counts when RSS sources linked

### Social graph triggers

- `update_user_last_seen_on_message` - Update telegram_users.last_seen when message authored
- `update_user_last_seen_on_comment` - Update telegram_users.last_seen when comment posted

---

## Extensions Used

- **pgvector**: Vector similarity search (384-dim embeddings)
- **pg_trgm**: Fuzzy text search with trigrams
- **btree_gin**: Compound GIN indexes
- **pg_stat_statements**: Query performance tracking

---

## Key Design Patterns

### Content-Addressed Storage

Media files use SHA-256 deduplication with reference counting.

**Path**: `media/{hash[:2]}/{hash[2:4]}/{hash}.{ext}`

---

### Folder-Based Channel Management

Channels organized via Telegram native folders. Rules editable in `folder_rules` table.

**Pattern**: `^Archive` → archive_all, `^Monitor` → selective_archive

---

### Multi-Model LLM Fallback

`model_configuration` table defines primary/fallback models per task with priority ranking.

**Example**: message_classification uses qwen2.5:3b (priority 1) with llama3.2:3b fallback (priority 2)

---

### Tiered Event Validation

Events progress through tiers based on source count:

- **breaking**: Initial detection
- **developing**: 2+ hours old
- **confirmed**: 1+ RSS sources
- **verified**: 3+ RSS sources

Configuration in `event_config` table.

---

### Chain-of-Thought Decision Logging

All LLM decisions stored in `decision_log` with full reasoning for verification and reprocessing.

**Enables**: A/B testing prompts, quality metrics, reprocessing with improved models

---

## Index Strategy

### HNSW Indexes (pgvector)

Used for frequent, high-quality searches (messages, opensanctions_entities).

**Parameters**: m=16, ef_construction=64

---

### IVFFLAT Indexes (pgvector)

Used for large datasets with less frequent searches (events, curated_entities, external_news).

**Parameters**: lists=100

---

### GIN Indexes

- Full-text search (TSVECTOR columns)
- JSONB columns (entities, metadata)
- Array columns (tags, datasets, aliases)
- Trigram fuzzy search (name columns)

---

### Partial Indexes

WHERE clauses optimize storage and query performance:

- `WHERE archived_at IS NULL` - Active events only
- `WHERE is_major = TRUE` - Pinned events
- `WHERE revoked_at IS NULL` - Active tokens
- `WHERE username IS NOT NULL` - Named users only

---

## Performance Notes

### Vector Search Performance

- **messages.content_embedding**: HNSW for <100ms semantic search on 1M+ messages
- **events.content_embedding**: IVFFLAT for event deduplication (less frequent)
- **Similarity thresholds**: 0.85 for auto-linking, 0.75 for LLM confirmation, 0.88 for novelty detection

### Full-Text Search Performance

- GIN indexes on search_vector: 10-100x faster than ILIKE
- Weighted search: 'A' for titles, 'B' for content, 'C' for notes
- Combined with pgvector for hybrid search (40% text, 60% vector)

### Materialized View Refresh Strategy

- **message_social_graph**: Hourly
- **channel_influence_network**: Daily
- **top_influencers**: Daily
- **unified_message_entities**: As needed (after enrichment runs)

---

## Schema Versions

This documentation reflects the schema as of:

- **Message Classification**: v6 (chain-of-thought with <analysis> tags)
- **Events**: v2 (tiered validation with RSS sources)
- **Entities**: Unified curated + OpenSanctions
- **Social Graph**: Full implementation with materialized views

For prompt evolution history, see `/docs/architecture/LLM_PROMPTS.md`

---

**File**: `/home/rick/code/osintukraine/osint-platform-docs/docs/reference/database-tables.md`
