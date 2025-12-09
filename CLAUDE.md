# CLAUDE.md - Documentation Repository

This file provides guidance to Claude Code when working with the OSINT Platform documentation.

## Repository Overview

**Purpose**: MkDocs Material documentation for the OSINT Intelligence Platform
**Status**: 🚧 In Progress | **Phase**: 5 of 5 | **Last Updated**: 2025-12-09

## Quick Facts

- **Platform**: MkDocs Material with persona-based navigation
- **Source Repo**: `osintukraine/osint-intelligence-platform`
- **Audiences**: Analysts (users), Operators (deploy/maintain), Developers (extend), Security
- **Generation Method**: Agents analyzing actual codebase (not copying old docs)

## Documentation Structure

```
docs/
├── getting-started/     # ✅ COMPLETE - Platform intro, quick-start, concepts
├── user-guide/          # ⏳ PENDING - For OSINT analysts
├── operator-guide/      # ⏳ PENDING - For platform operators
├── developer-guide/     # 🚧 IN PROGRESS
│   ├── architecture.md  # ✅ COMPLETE (545 lines)
│   └── services/        # 🚧 Core services being documented
├── security-guide/      # ⏳ PENDING - Kratos, CrowdSec
├── tutorials/           # ⏳ PENDING - Step-by-step guides
└── reference/           # ⏳ PENDING - API, env vars, schema
```

## Generation Progress

### Phase 1: Foundation ✅ COMPLETE
- [x] MkDocs skeleton (mkdocs.yml, 40 stub files)
- [x] Architecture overview (545 lines)
- [x] Getting-started section (4 files, ~900 lines)

### Phase 2: Core Services ✅ COMPLETE
- [x] Listener service (813 lines, 24KB)
- [x] Processor service (738 lines, 19KB)
- [x] API service (1,157 lines, 41KB)
- [x] Frontend service (888 lines, 24KB)

### Phase 3: Supporting Services ✅ COMPLETE
- [x] Enrichment service (1,325 lines - 26 tasks, 8 workers)
- [x] RSS-Ingestor service (1,136 lines)
- [x] Notifier service (802 lines)
- [x] OpenSanctions service (1,062 lines)
- [x] Entity-Ingestion service (1,157 lines)
- [x] Analytics service (761 lines)
- [x] Migration service (863 lines)
- [x] Translation-Backfill service (686 lines)

### Phase 4: Specialized Content ✅ COMPLETE
- [x] Tutorials (5 files, 2,585 lines)
- [x] Security Guide (5 files, 3,158 lines)
- [x] Operator Guide (7 files, 6,128 lines)
- [x] User Guide (6 files, 2,706 lines)

### Phase 5: Reference & Polish 🚧 IN PROGRESS
- [ ] API endpoints reference
- [ ] Environment variables reference
- [ ] Database schema reference
- [ ] Docker services reference
- [ ] Consistency review

## Content Standards

### Service Documentation Template
Every service doc in `developer-guide/services/` must include:
1. Overview (purpose, pipeline position, technologies)
2. Architecture (key files, diagrams)
3. Configuration (env vars table)
4. Key Features
5. Metrics (Prometheus)
6. Running Locally
7. Troubleshooting
8. Related docs

### Style Guide
- Use MkDocs Material admonitions: `!!! note`, `!!! warning`, `!!! tip`
- Include Mermaid diagrams for architecture/flow
- Tables for configuration, metrics
- Copy-pasteable commands
- Link between docs: `[Processor](services/processor.md)`

### Principles
1. **Source of truth is the code** - Don't copy from old `/docs/`
2. **Persona-based** - Write for specific audience (analyst vs developer)
3. **Scannable** - Use headers, tables, lists
4. **Actionable** - Include verification steps, troubleshooting

## Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Serve locally (hot reload)
mkdocs serve

# Build static site
mkdocs build

# Deploy to GitHub Pages
mkdocs gh-deploy
```

## Source Repository Reference

When documenting, analyze code from:
- **Services**: `/home/rick/code/osintukraine/osint-intelligence-platform/services/`
- **Shared**: `/home/rick/code/osintukraine/osint-intelligence-platform/shared/python/`
- **Infrastructure**: `/home/rick/code/osintukraine/osint-intelligence-platform/infrastructure/`
- **Context Layers**: `/home/rick/code/osintukraine/osint-intelligence-platform/.claude/context/`

## Current Task

Generating Phase 2: Core service documentation (Listener, Processor, API, Frontend)
using parallel technical-writer agents.

---

*This file is updated as documentation generation progresses.*
