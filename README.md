# OSINT Intelligence Platform Documentation

This repository contains the complete documentation for the OSINT Intelligence Platform, built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

## Quick Start

### Prerequisites

- Python 3.8+
- pip

### Installation

```bash
# Install dependencies
pip install -r requirements.txt
```

### Development

```bash
# Serve documentation locally with live reload
mkdocs serve

# The site will be available at http://127.0.0.1:8000
```

### Building

```bash
# Build static site
mkdocs build

# Output will be in the site/ directory
```

### Deployment

```bash
# Deploy to GitHub Pages
mkdocs gh-deploy
```

## Documentation Structure

- **Getting Started**: Quick introduction and core concepts
- **User Guide**: End-user features and workflows
- **Operator Guide**: Installation, configuration, and operations
- **Developer Guide**: Architecture, services, and development
- **Security Guide**: Authentication, authorization, and hardening
- **Tutorials**: Step-by-step guides for common tasks
- **Reference**: API docs, environment variables, database schema

## Contributing

When adding new documentation:

1. Create markdown files in the appropriate `docs/` subdirectory
2. Add the page to `nav:` section in `mkdocs.yml`
3. Use proper heading hierarchy (`#`, `##`, `###`)
4. Include code examples with language-specific syntax highlighting
5. Use admonitions for important notes, warnings, and tips
6. Test locally with `mkdocs serve` before committing

## Admonition Examples

```markdown
!!! note
    This is a note admonition.

!!! warning
    This is a warning admonition.

!!! tip
    This is a tip admonition.

!!! danger
    This is a danger admonition.
```

## Code Block Examples

```markdown
\`\`\`python
# Python code with syntax highlighting
def hello_world():
    print("Hello, World!")
\`\`\`

\`\`\`bash
# Bash commands
docker-compose up -d
\`\`\`
```

## Mermaid Diagrams

```markdown
\`\`\`mermaid
graph LR
    A[Telegram] --> B[Listener]
    B --> C[Redis]
    C --> D[Processor]
    D --> E[PostgreSQL]
\`\`\`
```

## License

Same as the main OSINT Intelligence Platform repository.
