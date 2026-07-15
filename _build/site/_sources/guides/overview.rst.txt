Overview
========

What is OpenZync?
-----------------

OpenZync is an open-source agent memory platform designed to give AI agents
**persistent, queryable, graph-based memory**. It sits between your LLM
application and your data, providing:

- **Hybrid retrieval** — semantic (vector) search combined with graph traversal
  for context-aware memory recall.  See :doc:`/domains/memory_context` for the
  retrieval architecture.
- **Knowledge graphs** — entity extraction, relationship detection, and
  community detection over conversation history.  The graph abstraction supports
  multiple backends: see :doc:`/domains/graph_backends`.
- **Temporal queries** — time-aware memory retrieval that respects when events
  occurred.
- **Multi-tenant isolation** — organizations, projects, and users are fully
  isolated at the data layer.  See :doc:`/domains/auth` for the auth and org
  model.
- **Pluggable LLM backends** — OpenAI, Anthropic, Azure OpenAI, OpenRouter,
  and local Ollama.  See :doc:`/domains/llm` for the strategy/registry pattern.
- **Prompt caching** — provider-side caching for Anthropic (``cache_control``
  markers), OpenAI/Azure (automatic prefix caching), and OpenRouter (session
  stickiness).  Configurable per-org via system settings.
- **PII detection & redaction** — automatic identification and redaction of
  personally identifiable information before storage.
- **Idempotency** — all ingestion and mutation endpoints support idempotency
  keys to prevent duplicate side effects on retry.

.. note::

   OpenZync is currently a **monolith** — all backend domains live in a single
   repository (``openzync-core``). Future work may decompose into separate
   microservices, but today everything shares a single FastAPI process, a
   single PostgreSQL database, and a single Redis instance.

Architecture
------------

The system is built as a **FastAPI** application with:

- **PostgreSQL 15+ with pgvector** for relational data, vector embeddings
  (``HNSW`` and ``IVFFlat`` indexes), and the default graph backend
  (``PostgresGraphBackend``).
- **Redis 7+** for caching, rate limiting, ARQ task queues, and pub/sub event
  broadcasting.
- **Pluggable graph backend** — a common abstraction (see
  :doc:`/domains/graph_backends`) supporting PostgreSQL-native (default),
  FalkorDB, or SurrealDB for entity-relation storage.  The backend is
  resolved per-organisation at runtime.
- **OpenBao** (Vault-compatible secrets manager) as the **sole source of
  truth** for all runtime configuration and secrets.  See
  :doc:`/domains/infrastructure` and :doc:`/adr/003-openbao-zero-fallback` for
  the architectural rationale.
- **ARQ worker pool** with **high- and low-priority queues** for background
  tasks: entity extraction, fact extraction, dialog classification, embedding
  computation, episode enrichment, entity merging, and user summarisation.
  See :doc:`/domains/workers`.

Layer structure follows strict separation of concerns:

.. list-table:: Layer responsibilities
   :header-rows: 1

   * - Layer
     - Responsibility
   * - ``routers/``
     - HTTP adapter only — validates input, delegates to services
   * - ``services/``
     - All business logic — orchestrates call chains
   * - ``repositories/``
     - Database access — SQLAlchemy async queries only
   * - ``models/``
     - ORM definitions — no business logic
   * - ``schemas/``
     - Pydantic request/response models — no ORM imports

See :doc:`/domains/api_layer` for the full router/service/repository pattern
and lifecycle.

Key features
------------

- **Agent sessions** with memory across conversations.  See
  :doc:`/domains/memory_context`.
- **Message ingestion** with automatic entity/fact extraction and enrichment
  pipeline (LLM-based classification, entity linking, community detection).
- **Graph-based memory** with community detection and temporal queries.
  Backends: PostgreSQL (built-in), FalkorDB, SurrealDB.
- **Hybrid retrieval** — multi-leg search combining vector similarity, keyword
  BM25, graph traversal, and temporal filtering with a configurable reranker.
  See :doc:`/domains/reranker`.
- **Prompt templates** versioned and stored as Jinja2 files in a manifest
  system with per-organisation overrides (``custom_instructions``).
- **Webhook system** with 14 event types and HMAC-SHA256 (Svix-compatible)
  signed delivery.  See :doc:`/domains/admin_webhooks`.
- **SDKs** — Python (``openzync`` package) with optional LangChain
  integration.  See :doc:`/domains/sdk_python`.
- **MCP server** — Model Context Protocol server for AI tool integration
  (tools for sessions, memory, facts, graph, and users).  See
  :doc:`/domains/mcp_server`.
- **OpenBao-based secrets management** — zero-fallback bootstrap, AppRole
  machine auth, Transit engine for encrypting API keys and PII at rest.
- **Idempotency** — request-level idempotency keys across mutation endpoints.
- **Audit logging** — structured audit trail of configuration changes and
  sensitive operations.
- **OTP & email** — one-time password authentication flow delivered via SMTP
  (``aiosmtplib``).
- **Grafana dashboard** — pre-built ``OpenZync-overview`` dashboard with API
  request rate, latency percentiles, worker queue depth, LLM token usage, graph
  node growth, and service health panels.

Tech stack
----------

.. list-table::
   :header-rows: 1

   * - Layer
     - Technology
   * - **Runtime**
     - Python 3.11+ / FastAPI / uvicorn
   * - **Web server**
     - NGINX (reverse proxy with optional TLS via Cloudflare origin certs)
   * - **Database**
     - PostgreSQL 15+ with pgvector
   * - **Cache / Queue**
     - Redis 7+ with hiredis, ARQ for async job queues
   * - **LLM backends**
     - OpenAI, Anthropic, Azure OpenAI, OpenRouter, Ollama via pluggable
       strategy/registry pattern
   * - **Graph backends**
     - PostgreSQL-native, FalkorDB, SurrealDB (pluggable via
       :doc:`/domains/graph_backends`)
   * - **Secrets management**
     - OpenBao (Vault-compatible) with AppRole auth, Transit engine
       (encryption-as-a-service)
   * - **Observability**
     - Prometheus, Grafana, Alloy (OpenTelemetry collector)
   * - **Structured logging**
     - structlog with JSON output (production), PII redaction, request-context
       binding
   * - **Frontend**
     - Next.js 16 / React 19 / TypeScript 5 / Tailwind CSS 4 / Radix UI
   * - **Python SDK**
     - ``openzync`` package (PyPI) with optional LangChain integration
   * - **MCP Server**
     - ``openzync-mcp`` — independently deployable model context protocol
       server
   * - **Deployment**
     - Docker Compose (full backend stack), Helm chart (Kubernetes 1.25+),
       GitHub Actions CI/CD

Related documentation
---------------------

Core infrastructure:

- :doc:`/domains/core` — Configuration system, async DB/Redis/ARQ connection
  management, exception hierarchy, structured logging, OpenBao integration
- :doc:`/domains/llm` — LLM backend abstraction, providers, caching
- :doc:`/domains/graph_backends` — Pluggable graph backend abstraction
- :doc:`/domains/reranker` — Cross-encoder reranker for hybrid retrieval

Domain services:

- :doc:`/domains/auth` — Authentication, authorisation, org model, JWT, OTP
- :doc:`/domains/memory_context` — Session memory, context retrieval, hybrid
  search
- :doc:`/domains/admin_webhooks` — Webhook system, event types, delivery
- :doc:`/domains/infrastructure` — Docker Compose, OpenBao secrets bootstrap,
  Helm chart, observability stack, production considerations

API & workers:

- :doc:`/domains/api_layer` — Router/service/repository pattern, middleware,
  exception handlers
- :doc:`/domains/workers` — ARQ worker architecture, task registry,
  enrichment pipeline

SDK, MCP & frontend:

- :doc:`/domains/sdk_python` — Python SDK, LangChain integration
- :doc:`/domains/mcp_server` — MCP server tools and transport
- :doc:`/domains/frontend` — Next.js frontend architecture
