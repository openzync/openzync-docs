# OpenZync Documentation

**Comprehensive documentation for the OpenZync agent memory platform.**

<p align="center">
  <img src="https://img.shields.io/badge/sphinx-7.0+-green" alt="Sphinx">
  <img src="https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey" alt="CC BY 4.0">
  <img src="https://readthedocs.org/projects/openzync/badge/?version=latest" alt="ReadTheDocs">
</p>

Hosted on [ReadTheDocs](https://openzync.readthedocs.io). Built with [Sphinx](https://www.sphinx-doc.org), [Furo](https://pradyunsg.me/furo/) theme, and [MyST](https://myst-parser.readthedocs.io) Markdown parser.

## Content

Covering all components of the OpenZync platform:

### Getting Started (5 guides, ~2,045 lines)
- **Overview** — platform architecture and design philosophy
- **Quickstart** — get up and running in minutes
- **Architecture** — deep dive into system design
- **Deployment** — Docker Compose and Helm production setup
- **Contributing** — development workflow and standards

### Core Infrastructure (4 domains, ~19,344 lines)
- **Core** — config, DB, Redis, ARQ, events, exceptions, logging
- **LLM** — multi-provider LLM orchestration (OpenAI, Anthropic, Ollama, Azure, OpenRouter)
- **Graph Backends** — PostgreSQL-native, FalkorDB, SurrealDB
- **Reranker** — cross-encoder + label propagation community detection

### Domain Services (4 domains)
- **Auth** — JWT, API keys, OTP, multi-tenant auth
- **Memory & Context** — episode ingestion, context retrieval, hybrid search
- **Admin & Webhooks** — admin API, webhook delivery, audit logging
- **Idempotency** — idempotent request handling

### API & Workers (2 domains)
- **API Layer** — FastAPI routers, middleware, dependencies
- **Workers** — ARQ background job definitions

### SDK, MCP & Frontend (3 domains)
- **Python SDK** — client library and LangChain integrations
- **MCP Server** — Model Context Protocol tools
- **Frontend** — admin dashboard

### API Reference (auto-generated)
Auto-generated reference from docstrings for all core, routers, models, schemas, services, repositories, middleware, dependencies, workers, and utility packages.

## Build Locally

```bash
# Install build dependencies
pip install -r requirements.txt

# Build the docs
sphinx-build -b html . _build/html

# Open in browser
open _build/html/index.html

# Auto-reload on changes
sphinx-autobuild . _build/html
```

## Structure

```
openzync-docs/
├── conf.py                 # Sphinx configuration
├── index.rst               # Root toctree
├── .readthedocs.yaml       # ReadTheDocs build config
├── requirements.txt        # Python build dependencies
├── guides/                 # Getting started guides (RST)
│   ├── overview.md
│   ├── quickstart.md
│   ├── architecture.md
│   ├── deployment.md
│   └── contributing.md
├── domains/                # Domain documentation (RST/MD)
│   ├── core.md
│   ├── auth.md
│   ├── memory_context.md
│   ├── llm.md
│   ├── graph_backends.md
│   ├── api_layer.md
│   ├── workers.md
│   ├── admin_webhooks.md
│   ├── idempotency.md
│   ├── reranker.md
│   ├── sdk_python.md
│   ├── mcp_server.md
│   └── frontend.md
├── api/                    # Auto-generated API reference
└── _build/                 # Build artifacts (gitignored — built dynamically by ReadTheDocs)
```

The `conf.py` imports `openzync-core` packages at build time for autodoc by adding the sibling `openzync-core/` directory to `sys.path`.

## License

CC BY 4.0 — attribution required. See [LICENSE](./LICENSE).
