# Documentation Audit Implementation Plan

**Date:** 2025-12-09
**Audit Grade:** B+ (82/100)
**Goal:** Close all documentation gaps identified in audit.md

---

## Plan Overview

| Aspect | Decision |
|--------|----------|
| **Scope** | Full plan (all audit items) |
| **Organization** | Hybrid: Critical fixes first, then section-by-section |
| **Depth** | Practical (200-400 lines per new doc) |
| **Verification** | Read source code for all items |
| **Process** | Add docs policy + glossary; defer CI and changelogs |
| **CLAUDE.md** | Sync after each phase |

**Deliverables:**

- **5 phases** of work
- **12 new files** created
- **~18 files** updated
- **5 CLAUDE.md updates** (one per phase)
- **4 items deferred** to future work

---

## Phase 1: Critical Fixes

**Priority:** 🔴 CRITICAL
**Goal:** Fix the 4 critical issues that are actively misleading users.

### Task 1.1: Document Wikidata Enrichment in Enrichment Service

**File:** `docs/developer-guide/services/enrichment.md`
**Action:** UPDATE - Add "Wikidata Enrichment" section

**Source files to verify:**
- `services/enrichment/src/tasks/wikidata_enrichment.py`
- `services/enrichment/src/workers/maintenance_worker.py`
- `shared/python/models/entity.py`

**Content to add:**
- What Wikidata enrichment does (fetches relationships, properties)
- How it's triggered (maintenance worker, refresh parameter)
- Data model (entity_relationships table)
- Configuration options
- Example SPARQL queries used

**Acceptance criteria:**
- [ ] WikidataEnrichmentTask class documented
- [ ] Integration with maintenance_worker explained
- [ ] entity_relationships table referenced
- [ ] Refresh mechanism documented

---

### Task 1.2: Document Missing Database Tables

**File:** `docs/reference/database-tables.md`
**Action:** UPDATE - Add 12 missing tables

**Source file to verify:**
- `infrastructure/postgres/init.sql`

**Tables to document:**

| Table | Purpose | Priority |
|-------|---------|----------|
| `entity_relationships` | Wikidata relationship storage | CRITICAL |
| `export_jobs` | Export job tracking | Medium |
| `events_v2` | Event detection V2 | Medium |
| `event_messages_v2` | Event-message links | Medium |
| `event_sources_v2` | Event sources | Medium |
| `event_config` | Event configuration | Medium |
| `translation_config` | Translation settings | Low |
| `translation_usage` | DeepL usage tracking | Low |
| `news_sources` | RSS news sources | Medium |
| `external_news` | External news items | Medium |
| `message_replies` | Reply tracking | Low |
| `message_forwards` | Forward tracking | Low |

**Acceptance criteria:**
- [ ] All 12 tables documented with columns
- [ ] Primary keys and foreign keys noted
- [ ] Purpose of each table explained
- [ ] Relationships between tables shown

---

### Task 1.3: Document Missing API Endpoints

**File:** `docs/reference/api-endpoints.md`
**Action:** UPDATE - Add missing routers

**Source files to verify:**
- `services/api/src/main.py` (router imports)
- `services/api/src/routers/channel_network.py`
- `services/api/src/routers/news_timeline.py`
- `services/api/src/routers/admin/kanban.py`
- `services/api/src/routers/admin/config.py`
- `services/api/src/routers/admin/dashboard.py`

**Routers to document:**

| Router | Endpoints | Priority |
|--------|-----------|----------|
| `channel_network_router` | Channel content network graphs | High |
| `news_timeline_router` | RSS + Telegram timeline | High |
| `admin_kanban_router` | Admin urgency kanban | Medium |
| `admin_config_router` | Admin configuration | Medium |
| `admin_dashboard_router` | Admin dashboard stats | Medium |

**Also update:**
- `bookmarks_router` - Add missing details
- `flowsint_export_router` - Document all endpoints
- `validation_router` - Document RSS validation endpoints

**Acceptance criteria:**
- [ ] All 5 missing routers documented
- [ ] Each endpoint has: method, path, description, parameters, response
- [ ] Example requests included
- [ ] Authentication requirements noted

---

### Task 1.4: Document Wikidata Relationships for Users

**File:** `docs/user-guide/entities.md`
**Action:** UPDATE - Add "Wikidata Relationships" section

**Source files to verify:**
- `services/frontend-nextjs/app/entities/[source]/[id]/page.tsx`
- `services/api/src/routers/entities.py`

**Content to add:**
- What relationship graphs show
- How to access them (entity profile page)
- How to interpret connections
- How to force refresh stale data
- Example: exploring a person's corporate connections

**Acceptance criteria:**
- [ ] Feature explained in user terms
- [ ] Screenshots or diagrams showing UI
- [ ] Refresh mechanism documented
- [ ] Example walkthrough included

---

### Task 1.5: Fix Service Counts in Main Index

**File:** `docs/index.md`
**Action:** UPDATE - Replace fixed "29 containers" with profile-based explanation

**Source file to verify:**
- `docker-compose.yml` (all profiles)

**Content to change:**

FROM:
```
Production-ready system with 29 containers...
```

TO:
```
The platform uses Docker Compose profiles for modular deployment:

- **Minimal** (~8 containers): Core platform only
- **Standard** (~15 containers): Core + enrichment workers
- **Full** (~25 containers): Standard + monitoring + auth
- **Maximum** (40+ containers): All services including dev tools
```

**Acceptance criteria:**
- [ ] Fixed container count removed
- [ ] Profile-based explanation added
- [ ] Accurate counts per profile

---

### Task 1.6: Update Docker Services Reference

**File:** `docs/reference/docker-services.md`
**Action:** UPDATE - Fix mkdocs entry, update counts

**Source file to verify:**
- `docker-compose.yml`

**Changes needed:**
1. Fix mkdocs service (builds from local repo, not pre-built image)
2. Update service counts to match profile-based model
3. Add profile information to service entries

**Acceptance criteria:**
- [ ] mkdocs entry shows local build
- [ ] Service counts match docker-compose.yml
- [ ] Profiles documented per service

---

### Task 1.7: Cross-Reference Environment Variables

**File:** `docs/reference/environment-vars.md`
**Action:** UPDATE - Sync with .env.example

**Source file to verify:**
- `.env.example`

**Variables to add (in .env.example but not docs):**
- `HF_HOME` - HuggingFace cache directory
- `HF_HUB_OFFLINE` - Offline mode for sentence-transformers
- `YENTE_UPDATE_TOKEN` - Yente index update authentication
- `SOURCE_ACCOUNT` - Multi-account listener identifier

**Variables to mark as deprecated/legacy (in docs but not .env.example):**
- `PROCESSOR_REPLICAS` - Uses docker-compose deploy.replicas instead
- `ENRICHMENT_TASKS` - Configured per-worker in docker-compose
- `ENRICHMENT_INTERVAL` - Workers have individual intervals

**Acceptance criteria:**
- [ ] All .env.example vars documented
- [ ] Deprecated vars marked clearly
- [ ] Advanced/Docker-only vars in separate section

---

### Task 1.8: Update CLAUDE.md (Phase 1 Sync)

**File:** `/home/rick/code/osintukraine/osint-intelligence-platform/CLAUDE.md`
**Action:** UPDATE - Sync critical changes

**Changes:**
1. Add Wikidata enrichment to "Key Patterns" section
2. Update service counts to profile-based model
3. Add new environment variables
4. Reference entity_relationships table

**Acceptance criteria:**
- [ ] CLAUDE.md reflects all Phase 1 changes
- [ ] Committed to platform repo (not docs repo)

---

## Phase 2: Operator Guide Completion

**Priority:** 🟠 HIGH
**Goal:** Give operators production-ready guides.

### Task 2.1: Create Scaling Guide

**File:** `docs/operator-guide/scaling.md`
**Action:** NEW

**Source files to verify:**
- `docker-compose.yml` (deploy sections, replicas)
- `CLAUDE.md` (Ollama CPU notes)
- `infrastructure/prometheus/` (scaling metrics)

**Content:**
1. **When to Scale**
   - Signs you need more processor workers
   - Signs you need more enrichment workers
   - Redis stream backlog thresholds

2. **How to Scale**
   - `docker-compose up -d --scale processor-worker=4`
   - Adding enrichment workers
   - Ollama CPU/memory allocation

3. **Resource Planning**
   - RAM requirements by scale (10/50/100+ channels)
   - CPU requirements for LLM inference
   - Disk requirements for media archival

4. **Bottleneck Identification**
   - Grafana dashboards to watch
   - Prometheus queries for queue depth
   - When Ollama is the bottleneck

**Target length:** 250-350 lines

**Acceptance criteria:**
- [ ] Scaling thresholds documented
- [ ] Commands for scaling provided
- [ ] Resource tables included
- [ ] Bottleneck diagnosis covered

---

### Task 2.2: Create Performance Tuning Guide

**File:** `docs/operator-guide/performance-tuning.md`
**Action:** NEW

**Source files to verify:**
- `docker-compose.yml` (resource limits)
- `services/processor/` (config options)
- `infrastructure/` (PostgreSQL, Redis configs)

**Content:**
1. **Ollama Optimization**
   - CPU core allocation (was 2.0, now 6.0)
   - Memory limits
   - Model selection (qwen2.5:3b vs larger models)

2. **PostgreSQL Tuning**
   - Connection pooling
   - Index usage
   - Vacuum schedules

3. **Redis Stream Management**
   - Stream length limits
   - Consumer group settings
   - Backlog monitoring

4. **MinIO Deduplication**
   - Content-addressed storage benefits
   - Expected deduplication rates
   - Storage savings calculations

**Target length:** 200-300 lines

**Acceptance criteria:**
- [ ] Ollama tuning documented
- [ ] Database tuning covered
- [ ] Redis stream settings explained
- [ ] Storage optimization included

---

### Task 2.3: Update Monitoring Guide

**File:** `docs/operator-guide/monitoring.md`
**Action:** UPDATE - Add practical examples

**Source files to verify:**
- `infrastructure/prometheus/rules/`
- `infrastructure/grafana/dashboards/`
- `infrastructure/prometheus/prometheus.yml`

**Content to add:**
1. **Prometheus Query Examples**
   - Message processing rate
   - Spam filter effectiveness
   - LLM classification latency
   - Queue depth alerts

2. **Grafana Dashboard Tour**
   - Which dashboards exist
   - What each shows
   - How to interpret key panels

3. **Alert Threshold Recommendations**
   - When to alert on queue depth
   - When to alert on error rates
   - When to alert on disk usage

4. **Loki Log Aggregation**
   - How to query logs
   - Common log patterns
   - Troubleshooting with logs

**Target additions:** 150-200 lines

**Acceptance criteria:**
- [ ] PromQL examples included
- [ ] Dashboard descriptions added
- [ ] Alert thresholds recommended
- [ ] Loki usage documented

---

### Task 2.4: Create Upgrades Guide

**File:** `docs/operator-guide/upgrades.md`
**Action:** NEW

**Source files to verify:**
- `CLAUDE.md` ("Database Schema Workflow")
- `infrastructure/postgres/init.sql`

**Content:**
1. **Pre-Upgrade Checklist**
   - Backup database
   - Backup Telegram session
   - Note current versions

2. **Database Schema Changes**
   - No Alembic migrations (init.sql is source of truth)
   - How to test schema changes
   - Volume wipe procedure

3. **Rolling Updates**
   - Which services can be updated without downtime
   - Which require restart
   - Order of operations

4. **Rollback Procedures**
   - How to restore from backup
   - How to revert to previous image
   - Emergency procedures

**Target length:** 200-250 lines

**Acceptance criteria:**
- [ ] Backup procedures documented
- [ ] Schema change process explained
- [ ] Rolling update order specified
- [ ] Rollback steps included

---

### Task 2.5: Update Operator Guide Index

**File:** `docs/operator-guide/index.md`
**Action:** UPDATE - Add links to new guides

**Changes:**
- Add scaling.md to guide list
- Add performance-tuning.md to guide list
- Add upgrades.md to guide list
- Update section descriptions

**Acceptance criteria:**
- [ ] All new guides linked
- [ ] Descriptions accurate

---

### Task 2.6: Update CLAUDE.md (Phase 2 Sync)

**File:** `/home/rick/code/osintukraine/osint-intelligence-platform/CLAUDE.md`
**Action:** UPDATE

**Changes:**
1. Add scaling commands to "Essential Commands"
2. Reference new operator guides
3. Add performance tuning tips

**Acceptance criteria:**
- [ ] CLAUDE.md references new operator guides
- [ ] Committed to platform repo

---

## Phase 3: Developer Guide Completion

**Priority:** 🟡 MEDIUM
**Goal:** Help developers extend the platform correctly.

### Task 3.1: Update Adding Features Guide

**File:** `docs/developer-guide/adding-features.md`
**Action:** UPDATE - Add enrichment and API sections

**Source files to verify:**
- `services/enrichment/src/tasks/base.py`
- `services/enrichment/src/coordinator.py`
- `services/api/src/main.py`

**Content to add:**
1. **Adding an Enrichment Task**
   - Inherit from BaseEnrichmentTask
   - Implement required methods
   - Register in coordinator
   - Add to router logic
   - Configure worker pool

2. **Adding an API Endpoint**
   - Create router file
   - Register in main.py
   - Add schemas
   - Write tests

3. **Documentation Requirements**
   - What changes require docs updates
   - Where to update (which files)

**Target additions:** 150-200 lines

**Acceptance criteria:**
- [ ] Enrichment task workflow documented
- [ ] API endpoint workflow documented
- [ ] Docs requirements section added

---

### Task 3.2: Create Frontend API Patterns Guide

**File:** `docs/developer-guide/frontend-api-patterns.md`
**Action:** NEW

**Source files to verify:**
- `services/frontend-nextjs/lib/api.ts`
- `services/frontend-nextjs/app/admin/page.tsx`
- `CLAUDE.md` (Rule #6)

**Content:**
1. **NEXT_PUBLIC_API_URL Pattern**
   - Why relative paths don't work in Docker
   - How to use environment variable
   - Client-side vs server-side fetching

2. **Error Handling**
   - Standard error response format
   - How to display errors to users
   - Retry logic

3. **Loading States**
   - Skeleton components
   - Suspense boundaries
   - Optimistic updates

4. **Type Safety**
   - Shared types location
   - API response typing
   - Generated types (if any)

**Target length:** 200-250 lines

**Acceptance criteria:**
- [ ] API URL pattern explained with examples
- [ ] Error handling patterns shown
- [ ] Loading state patterns included
- [ ] Type safety approach documented

---

### Task 3.3: Create LLM Integration Guide

**File:** `docs/developer-guide/llm-integration.md`
**Action:** NEW

**Source files to verify:**
- `services/processor/src/llm_classifier.py`
- `infrastructure/postgres/init.sql` (llm_prompts table)
- `docs/architecture/LLM_PROMPTS.md` (in platform repo)

**Content:**
1. **Prompt Engineering**
   - Prompt structure (system, user, examples)
   - Chain-of-thought format
   - How to test new prompts

2. **Version Management**
   - llm_prompts table structure
   - How to create new version
   - How to activate/deactivate
   - Migration procedure

3. **Fallback Strategy**
   - 4-tier fallback (v7 → v6 → v5 → hardcoded)
   - When fallbacks trigger
   - How to debug failures

4. **Testing**
   - Unit tests for classifier
   - Testing with actual Ollama
   - Performance benchmarking

5. **Reference**
   - Link to LLM_PROMPTS.md for full history

**Target length:** 250-300 lines

**Acceptance criteria:**
- [ ] Prompt structure documented
- [ ] Version management explained
- [ ] Fallback strategy clear
- [ ] Testing approach included

---

### Task 3.4: Create Testing Guide

**File:** `docs/developer-guide/testing-guide.md`
**Action:** NEW

**Source files to verify:**
- `services/api/tests/`
- `services/processor/tests/`
- `services/enrichment/tests/`
- `pytest.ini` or `pyproject.toml`

**Content:**
1. **Test Structure**
   - Where tests live
   - Naming conventions
   - Fixtures location

2. **Unit Tests**
   - How to write
   - Mocking guidelines
   - Coverage expectations

3. **Integration Tests**
   - Database setup
   - Redis mocking
   - API endpoint testing

4. **LLM Classification Tests**
   - The 16 existing tests
   - How to add new classification tests
   - Testing without Ollama

5. **Running Tests**
   - pytest commands
   - Coverage reports
   - CI integration

**Target length:** 200-250 lines

**Acceptance criteria:**
- [ ] Test structure documented
- [ ] Unit test guidelines included
- [ ] Integration test setup explained
- [ ] LLM test patterns shown

---

### Task 3.5: Create Database Migrations Guide

**File:** `docs/developer-guide/database-migrations.md`
**Action:** NEW

**Source files to verify:**
- `CLAUDE.md` ("Database Schema Workflow")
- `infrastructure/postgres/init.sql`

**Content:**
1. **No Alembic - Why**
   - init.sql as source of truth
   - Simplicity for self-hosted deployments
   - When this approach works

2. **Making Schema Changes**
   - Edit init.sql directly
   - Test with volume wipe
   - Commands for clean rebuild

3. **Data Migration Strategies**
   - When data preservation matters
   - Manual migration scripts
   - Export/transform/import pattern

4. **Testing Schema Changes**
   - Local testing workflow
   - Verify with psql queries
   - Check ORM compatibility

**Target length:** 150-200 lines

**Acceptance criteria:**
- [ ] No-Alembic rationale explained
- [ ] Schema change workflow clear
- [ ] Data migration options covered
- [ ] Testing steps included

---

### Task 3.6: Update Architecture Document

**File:** `docs/developer-guide/architecture.md`
**Action:** UPDATE - Add new system diagrams

**Source files to verify:**
- Phase 1 research (Wikidata, events, validation)

**Content to add:**
1. **Wikidata Enrichment Flow**
   - Diagram showing entity → Wikidata → relationships

2. **Event System V2 Architecture**
   - How events are detected
   - RSS + Telegram correlation
   - Event lifecycle

3. **Validation Layer**
   - How RSS validation works
   - Confidence scoring
   - Cross-source verification

**Target additions:** 100-150 lines

**Acceptance criteria:**
- [ ] Wikidata flow diagram added
- [ ] Event system V2 explained
- [ ] Validation layer documented

---

### Task 3.7: Update Developer Guide Index

**File:** `docs/developer-guide/index.md`
**Action:** UPDATE - Add links to new guides

**Changes:**
- Add frontend-api-patterns.md
- Add llm-integration.md
- Add testing-guide.md
- Add database-migrations.md
- Update descriptions

**Acceptance criteria:**
- [ ] All new guides linked
- [ ] Descriptions accurate

---

### Task 3.8: Update CLAUDE.md (Phase 3 Sync)

**File:** `/home/rick/code/osintukraine/osint-intelligence-platform/CLAUDE.md`
**Action:** UPDATE

**Changes:**
1. Update "Context Layers" to reference new developer guides
2. Add testing commands to "Essential Commands"

**Acceptance criteria:**
- [ ] CLAUDE.md references new developer guides
- [ ] Committed to platform repo

---

## Phase 4: User Guide & Tutorials

**Priority:** 🟡 MEDIUM
**Goal:** Help analysts use new features and understand AI classifications.

### Task 4.1: Create Entity Relationships Tutorial

**File:** `docs/tutorials/exploring-entity-relationships.md`
**Action:** NEW

**Source files to verify:**
- `services/frontend-nextjs/app/entities/[source]/[id]/page.tsx`
- `services/api/src/routers/entities.py`

**Content:**
1. **What You'll Learn**
   - How to find entity relationships
   - How to interpret Wikidata connections
   - How to refresh stale data

2. **Prerequisites**
   - Platform running
   - Entities imported

3. **Step-by-Step**
   - Navigate to entity profile
   - View relationship graph
   - Understand connection types
   - Force refresh

4. **Example Walkthrough**
   - Concrete example with a known entity

5. **Troubleshooting**
   - No relationships showing
   - Stale data

**Target length:** 200-300 lines

**Acceptance criteria:**
- [ ] Clear step-by-step instructions
- [ ] Screenshots or diagrams
- [ ] Example walkthrough included
- [ ] Troubleshooting section

---

### Task 4.2: Create Understanding AI Tags Guide

**File:** `docs/user-guide/understanding-ai-tags.md`
**Action:** NEW

**Source files to verify:**
- `services/processor/src/llm_classifier.py`
- `shared/python/models/message.py` (OSINT_TOPICS enum)
- `infrastructure/postgres/init.sql` (llm_prompts)

**Content:**
1. **The 12 OSINT Topics**
   - What each topic means
   - Examples of each
   - When topics overlap

2. **Importance Levels**
   - How importance is assigned
   - high/medium/low criteria
   - How folder tier affects this

3. **LLM Reasoning**
   - How to view reasoning
   - What good reasoning looks like
   - When to trust/question

4. **The LLM-as-Arbiter Model**
   - Folder tier sets strictness
   - LLM makes final decision
   - Archive rates by tier

**Target length:** 200-250 lines

**Acceptance criteria:**
- [ ] All 12 topics explained
- [ ] Importance levels clear
- [ ] Reasoning interpretation covered
- [ ] LLM-as-arbiter explained

---

### Task 4.3: Create Unified Stream Guide

**File:** `docs/user-guide/unified-stream.md`
**Action:** NEW

**Source files to verify:**
- `services/api/src/routers/stream.py`
- `services/api/src/routers/news_timeline.py`

**Content:**
1. **What is the Unified Stream?**
   - RSS + Telegram in one view
   - How correlation works

2. **Using the Stream**
   - How to access
   - Filtering options
   - Sorting options

3. **Validation Indicators**
   - What validation badges mean
   - Confidence levels
   - Cross-source verification

4. **Best Practices**
   - When to use stream vs search
   - Filtering strategies

**Target length:** 150-200 lines

**Acceptance criteria:**
- [ ] Stream concept explained
- [ ] Filtering documented
- [ ] Validation indicators clear

---

### Task 4.4: Create Events Explained Guide

**File:** `docs/user-guide/events-explained.md`
**Action:** NEW

**Source files to verify:**
- `infrastructure/postgres/init.sql` (events_v2 tables)
- `services/enrichment/src/tasks/` (event detection)

**Content:**
1. **What are Events?**
   - How events are created
   - Event vs message distinction

2. **Event Sources**
   - RSS-detected events
   - Telegram-detected events
   - Manual events

3. **Event Lifecycle**
   - Detection → correlation → major marking

4. **Using Events**
   - Finding events
   - Marking as major
   - Event timeline

**Target length:** 150-200 lines

**Acceptance criteria:**
- [ ] Event concept clear
- [ ] Sources explained
- [ ] Lifecycle documented

---

### Task 4.5: Update RSS Feed Tutorial

**File:** `docs/tutorials/create-custom-rss-feed.md`
**Action:** UPDATE - Add validation layer explanation

**Source files to verify:**
- `services/api/src/routers/validation.py`

**Content to add:**
- What validation layer does
- How validation affects feeds
- Validation indicators in feed items

**Target additions:** 50-80 lines

**Acceptance criteria:**
- [ ] Validation layer explained
- [ ] Impact on feeds documented

---

### Task 4.6: Update User Guide Index

**File:** `docs/user-guide/index.md`
**Action:** UPDATE

**Changes:**
- Add understanding-ai-tags.md
- Add unified-stream.md
- Add events-explained.md

**Acceptance criteria:**
- [ ] All new guides linked

---

### Task 4.7: Update Tutorials Index

**File:** `docs/tutorials/index.md`
**Action:** UPDATE

**Changes:**
- Add exploring-entity-relationships.md

**Acceptance criteria:**
- [ ] New tutorial linked

---

### Task 4.8: Update CLAUDE.md (Phase 4 Sync)

**File:** `/home/rick/code/osintukraine/osint-intelligence-platform/CLAUDE.md`
**Action:** UPDATE

**Changes:**
- Update documentation links to include new tutorials

**Acceptance criteria:**
- [ ] CLAUDE.md references new tutorials
- [ ] Committed to platform repo

---

## Phase 5: Reference & Process

**Priority:** 🟢 MAINTENANCE
**Goal:** Complete reference docs and establish maintenance culture.

### Task 5.1: Create Glossary

**File:** `docs/reference/glossary.md`
**Action:** NEW

**Source:** Existing docs terminology, OSINT vocabulary

**Content:**
1. **Platform Concepts** (~20 terms)
   - Archive tier, Monitor tier, Discover tier
   - LLM-as-arbiter
   - Content-addressed storage
   - Enrichment task
   - etc.

2. **OSINT Terms** (~15 terms)
   - OSINT, HUMINT, SIGINT
   - Source reliability
   - Information corroboration
   - etc.

3. **Military/Conflict Terms** (~15 terms)
   - Common Ukrainian/Russian military terms
   - Equipment categories
   - Unit designations
   - etc.

4. **Technical Terms** (~10 terms)
   - pgvector
   - Semantic search
   - Embedding
   - etc.

**Target length:** 200-300 lines (50-80 terms)

**Acceptance criteria:**
- [ ] Platform concepts defined
- [ ] OSINT terms included
- [ ] Military terms covered
- [ ] Alphabetically organized

---

### Task 5.2: Update Contributing Guide with Docs Policy

**File:** `docs/developer-guide/contributing.md`
**Action:** UPDATE - Add documentation requirements

**Content to add:**

```markdown
## Documentation Requirements

All pull requests MUST include documentation updates if they:

- Add new API endpoints
- Add new environment variables
- Change user workflows
- Add new services or containers
- Modify database schema
- Add new enrichment tasks

### Where to Update

| Change Type | Documentation File |
|-------------|-------------------|
| New API endpoint | `docs/reference/api-endpoints.md` |
| New env variable | `docs/reference/environment-vars.md` |
| New service | `docs/reference/docker-services.md` |
| Schema change | `docs/reference/database-tables.md` |
| User workflow | `docs/user-guide/` or `docs/tutorials/` |
| Developer pattern | `docs/developer-guide/` |
```

**Target additions:** 50-80 lines

**Acceptance criteria:**
- [ ] Policy clearly stated
- [ ] Change types listed
- [ ] File mapping provided

---

### Task 5.3: Complete Database Tables Documentation

**File:** `docs/reference/database-tables.md`
**Action:** UPDATE - Document remaining tables from Phase 1

**Tables to complete (if not fully done in 1.2):**
- events_v2, event_messages_v2, event_sources_v2, event_config
- export_jobs
- translation_config, translation_usage
- news_sources, external_news
- message_replies, message_forwards

**Acceptance criteria:**
- [ ] All tables documented
- [ ] Complete column definitions

---

### Task 5.4: Update Processor Service Documentation

**File:** `docs/developer-guide/services/processor.md`
**Action:** UPDATE - Verify prompt version

**Source files to verify:**
- `services/processor/src/llm_classifier.py`
- Database llm_prompts table

**Changes:**
- Verify current active prompt version (v7?)
- Add note about NocoDB runtime configuration
- Update if any discrepancies found

**Acceptance criteria:**
- [ ] Prompt version accurate
- [ ] Runtime config noted

---

### Task 5.5: Complete Docker Services Documentation

**File:** `docs/reference/docker-services.md`
**Action:** UPDATE - Add missing explanations

**Content to add:**
1. **Multi-Account Listeners**
   - listener-russia and listener-ukraine services
   - Folder naming conventions per account
   - When to use multi-account setup

2. **Enrichment Router**
   - What enrichment-router does
   - How it distributes tasks
   - Phase 3 architecture context

**Target additions:** 80-100 lines

**Acceptance criteria:**
- [ ] Multi-account setup explained
- [ ] Enrichment router role clear

---

### Task 5.6: Update Reference Index

**File:** `docs/reference/index.md`
**Action:** UPDATE

**Changes:**
- Add glossary.md link
- Update descriptions

**Acceptance criteria:**
- [ ] Glossary linked

---

### Task 5.7: Final CLAUDE.md Sync

**File:** `/home/rick/code/osintukraine/osint-intelligence-platform/CLAUDE.md`
**Action:** UPDATE - Final sync

**Changes:**
1. Update "Documentation" section with complete file list
2. Add glossary reference
3. Ensure all new guides referenced

**Acceptance criteria:**
- [ ] Documentation section complete
- [ ] All phases reflected
- [ ] Committed to platform repo

---

## Items Deferred to Future Work

These items from the audit are **not included** in this plan:

| Item | Reason |
|------|--------|
| Documentation CI check | Requires GitHub Actions setup (infrastructure work) |
| Per-doc changelogs | Maintenance overhead; git history suffices |
| Security audit procedures guide | Low priority per audit (existing security docs adequate) |
| Secrets management guide | Low priority per audit |

These can be addressed in a future documentation iteration.

---

## Execution Notes

### Source Verification

Before writing each doc section, verify against source code:
- Read the actual implementation
- Check for edge cases the audit may have missed
- Note any discrepancies for resolution

### Commit Strategy

- One commit per phase to docs repo
- One commit per phase to platform repo (CLAUDE.md)
- Clear commit messages referencing this plan

### Review Points

After each phase:
1. Rebuild MkDocs container: `docker-compose build --no-cache && docker-compose up -d`
2. Verify changes live at http://localhost:8002
3. Spot-check links work
4. Commit and push

---

## Summary

| Phase | New Files | Updates | Priority |
|-------|-----------|---------|----------|
| 1: Critical Fixes | 0 | 8 | 🔴 Critical |
| 2: Operator Guide | 3 | 3 | 🟠 High |
| 3: Developer Guide | 4 | 4 | 🟡 Medium |
| 4: User Guide | 4 | 4 | 🟡 Medium |
| 5: Reference & Process | 1 | 6 | 🟢 Maintenance |
| **Total** | **12** | **25** | — |

**Plan created:** 2025-12-09
**Based on:** audit.md (Grade B+ 82/100)
**Target:** Close all documentation gaps, reach A grade (95+/100)
