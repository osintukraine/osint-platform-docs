# Code Documentation Plan - 2025-12-19

## Objective

Add comprehensive docstrings to all API endpoints and key service functions to improve maintainability and match documentation terminology.

## Scope

### API Routers (Priority 1 - User-facing)

| Router | Functions | Priority | Notes |
|--------|-----------|----------|-------|
| `messages.py` | 6 | HIGH | Core functionality |
| `search.py` | 6 | HIGH | Core functionality |
| `entities.py` | 9 | HIGH | Complex entity system |
| `events.py` | 7 | HIGH | Event detection output |
| `channels.py` | 4 | HIGH | Channel management |
| `analytics.py` | 7 | MEDIUM | Dashboard data |
| `map.py` | 17 | MEDIUM | Already well documented |
| `social_graph.py` | 6 | MEDIUM | Graph features |
| `system.py` | 44 | LOW | Mostly health checks |

### Admin Routers (Priority 2 - Admin-facing)

| Router | Functions | Priority |
|--------|-----------|----------|
| `admin/export.py` | 11 | HIGH |
| `admin/feeds.py` | 10 | HIGH |
| `admin/config.py` | 10 | MEDIUM |
| `admin/spam.py` | 8 | MEDIUM |
| `admin/prompts.py` | 7 | MEDIUM |
| `admin/users.py` | 12 | LOW |

### Services (Priority 3 - Internal)

| Service | Key Files | Priority |
|---------|-----------|----------|
| Processor | `message_processor.py`, `llm_classifier.py` | HIGH |
| Enrichment | All task files | MEDIUM |
| Listener | `main.py`, `message_handler.py` | LOW |

## Documentation Standard

### Endpoint Docstrings

```python
@router.get("/example")
async def get_example(
    param: str = Query(..., description="Parameter description")
) -> ExampleResponse:
    """
    Short description of what the endpoint does.

    Longer description if needed, explaining:
    - Business logic
    - Side effects
    - Performance considerations

    Args:
        param: Description of the parameter

    Returns:
        ExampleResponse with fields:
        - field1: Description
        - field2: Description

    Raises:
        HTTPException(404): When resource not found
        HTTPException(403): When permission denied
    """
```

### Helper Function Docstrings

```python
async def helper_function(arg1: str, arg2: int) -> dict:
    """
    Brief description of what the function does.

    Args:
        arg1: Description
        arg2: Description

    Returns:
        Dict containing keys: key1, key2
    """
```

## Execution Plan

### Phase 1: Core API Routers (6 files)
1. messages.py - Message detail, album, network endpoints
2. search.py - Unified search, suggestions
3. entities.py - Entity CRUD, relationships
4. events.py - Event listing, details, timeline
5. channels.py - Channel listing, stats
6. analytics.py - Timeline, distributions, heatmap

### Phase 2: Admin Routers (6 files)
1. admin/export.py - Data export functionality
2. admin/feeds.py - RSS feed management
3. admin/config.py - Platform configuration
4. admin/spam.py - Spam management
5. admin/prompts.py - LLM prompt management
6. admin/users.py - User management

### Phase 3: Remaining Routers (10 files)
- social_graph.py, map.py, media.py, rss.py, stream.py
- validation.py, semantic.py, auth.py, comments.py, bookmarks.py

### Phase 4: Service Core Files (5 files)
- processor/message_processor.py
- processor/llm_classifier.py
- enrichment key tasks
- listener/main.py

## Progress Tracking

**Last Updated**: 2025-12-21

### Phase 1: Core API Routers (6/6 complete) ✅
- [x] channels.py ✅ **COMPLETE** - Gold standard with Args/Returns/Raises
- [x] messages.py ✅ **COMPLETE** - Full-text search, message detail, album endpoints
- [x] search.py ✅ **COMPLETE** - Unified search with text/semantic modes, clustering
- [x] entities.py ✅ **COMPLETE** - Entity CRUD, Yente integration, Wikidata relationships
- [x] events.py ✅ **COMPLETE** - Event listing, tiers, timeline, archive/major status
- [x] analytics.py ✅ **COMPLETE** - Timeline, distributions, heatmap, channel/entity analytics

### Phase 2: Admin Routers (6/6 complete) ✅
- [x] export.py ✅ **COMPLETE** - 88% comprehensive docstrings
- [x] feeds.py ✅ **COMPLETE** - RSS feed CRUD, polling, articles (10 endpoints)
- [x] config.py ✅ **COMPLETE** - Platform config, model configs, env vars (12 endpoints)
- [x] spam.py ✅ **COMPLETE** - Queue, review, bulk ops, purge (8 endpoints)
- [x] prompts.py ✅ **COMPLETE** - Prompt versioning, activation, history (7 endpoints)
- [x] users.py ✅ **COMPLETE** - Ory Kratos users/sessions, recovery links (12 endpoints)

### Phase 3: Remaining Routers (0/10)
- [ ] social_graph.py
- [ ] map.py
- [ ] media.py
- [ ] rss.py
- [ ] stream.py
- [ ] validation.py
- [ ] semantic.py
- [ ] auth.py
- [ ] comments.py
- [ ] bookmarks.py

### Phase 4: Service Core (2/3 complete)
- [x] llm_classifier.py ✅ **COMPLETE** - 85% coverage, excellent docs
- [ ] message_processor.py ⚠️ 70% - good structure, business logic gaps
- [x] listener/main.py ✅ **COMPLETE** - 115-line module docstring, 12-step init sequence

## Estimated Effort

- Phase 1: ~60 functions × 5 min = 5 hours
- Phase 2: ~60 functions × 5 min = 5 hours
- Phase 3: ~50 functions × 3 min = 2.5 hours
- Phase 4: ~40 functions × 5 min = 3.5 hours

Total: ~16 hours of work

## Notes

- Focus on user-facing endpoints first
- Health check functions can have minimal docstrings
- Match terminology from docs repo documentation
- Include parameter descriptions in FastAPI Query/Path decorators
