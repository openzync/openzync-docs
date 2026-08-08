Architecture Overview
=====================

.. note::

   This document describes the **current shipped architecture** — not an
   aspirational plan.  OpenZync is currently a backend monolith with separate
   frontend, SDK, MCP, and docs repos.  Future decomposition into microservices
   is a roadmap goal, not today's reality.


Repository Landscape
--------------------

OpenZync consists of **6 repositories** (not the 16 once planned):

.. list-table::
   :header-rows: 1

   * - Repository
     - Purpose
     - Tech
   * - ``openzync-core``
     - **Backend monolith** — all domain logic in one repo
     - Python / FastAPI / SQLAlchemy / ARQ
   * - ``openzync-frontend``
     - Dashboard UI
     - Next.js / TypeScript / Tailwind / D3
   * - ``openzync-landing``
     - Marketing site
     - Next.js / Tailwind
   * - ``openzync-sdk-python``
     - Client SDK for Python
     - Python / httpx / Pydantic
   * - ``openzync-mcp``
     - MCP server for LLM tool integration
     - Python / FastMCP
   * - ``openzync-docs``
     - Technical documentation (this site)
     - Sphinx / RST / Furo

The following planned repos **do not exist** as separate packages — they are
directories or modules inside the ``openzync-core`` monolith:

- ``openzync-llm`` → ``core/llm.py`` + ``core/llm_backends.py``
- ``openzync-graph`` → ``packages/graph_backend/``
- ``openzync-reranker`` → ``packages/reranker/``
- ``openzync-auth`` → ``services/auth_service.py`` + ``routers/auth.py``
- ``openzync-context`` → ``services/context_service.py`` + related
- ``openzync-webhooks`` → ``services/webhook_service.py``
- ``openzync-admin`` → ``routers/admin_*.py`` + ``services/org*_service.py``
- ``openzync-api`` → ``routers/`` + ``middleware/`` (the FastAPI app shell)
- ``openzync-worker`` → ``workers/``
- ``openzync-infra`` → ``infra/``
- ``openzync-build`` → does not exist

Documentation for each domain is in the :doc:`/domains/core` section of
this site.  Cross-repo relationships:

.. code-block:: text

    openzync-frontend ──HTTP──┐
    openzync-sdk-python ──────┤
    openzync-mcp ──(via SDK)──┤
                              ▼
                      openzync-core  (FastAPI + ARQ + DB)
                              │
                              ▼
                    ┌─────────────────┐
                    │  PostgreSQL 15   │
                    │  + pgvector      │
                    └─────────────────┘


High-Level Architecture
-----------------------

The system follows a **layered monolith** pattern with an async background
worker tier:

.. mermaid::

   flowchart LR
       subgraph Clients["Client Layer"]
           Frontend["Frontend"]
           SDK["SDK"]
           MCP["MCP"]
           Curl["curl"]
       end

       subgraph API["FastAPI Application (openzync-core)"]
           Middleware["Middleware<br/>RateLimit, Auth, Logging,<br/>Audit, CORS, RequestID"]
           Routers["Routers<br/>auth, memory, context, search,<br/>graph, admin, webhooks, health"]
           Services["Services<br/>(business logic)"]
           Repositories["Repositories<br/>(DB access)"]
           Middleware --> Routers --> Services --> Repositories
       end

       Clients -->|HTTP REST| Middleware

       Repositories --> PG[("PostgreSQL 15<br/>+ pgvector<br/>(primary DB)")]
       Repositories --> Redis[("Redis<br/>(cache + queues)")]
       Repositories --> Bao["OpenBao<br/>(secrets + config)"]

       Redis -->|ARQ job queue| Workers["ARQ Workers<br/>enrichment, webhook delivery,<br/>community detection"]
       Workers --> PG

**Key design decisions:**

1. **Layered separation** — strict ``routers → services → repositories → models``
   dependency direction.  No business logic in routers, no DB queries in services.
2. **Async throughout** — FastAPI async handlers, async SQLAlchemy sessions,
   async Redis, async ARQ workers.
3. **Zero env-var fallback** — all runtime configuration and secrets are
   stored in OpenBao.  The only env vars are the bootstrap credentials
   needed to reach OpenBao.
4. **Monolith simplicity** — all domains share a single database, a single
   Redis instance, and a single FastAPI process.  No network calls between
   services, no eventual-consistency headaches for synchronous reads.


Request Lifecycle
-----------------

Tracing a ``POST /v1/projects/{project_id}/memory`` request:

.. mermaid::

   sequenceDiagram
       autonumber
       participant Client
       participant Middleware as Middleware stack (10)
       participant Router
       participant MemoryService
       participant Repository
       participant Redis as Redis (queues + cache)
       participant Workers as ARQ Workers

       Client->>Middleware: POST /v1/projects/{project_id}/memory
       Middleware->>Middleware: rate limit, auth (JWT/API key), request ID, audit
       Middleware->>Router: forward validated request
       Router->>MemoryService: ingest(payload)
       MemoryService->>Repository: check project membership
       Repository-->>MemoryService: member
       MemoryService->>Repository: content-dedup (SHA-256 hash)
       MemoryService->>Repository: create Episode
       Repository-->>MemoryService: Episode created
       MemoryService->>Redis: enqueue ARQ enrichment jobs
       MemoryService->>Redis: invalidate context cache
       MemoryService-->>Router: IngestMemoryResponse
       Router-->>Middleware: HTTP response
       Middleware-->>Client: 201 Created

       Note over Client,Workers: Background
       Redis->>Workers: enrichment jobs (classify, entities/facts, embed)
       Workers->>Repository: persist results to PostgreSQL

The middleware stack runs in order: ``MetricsMiddleware`` (RED metrics),
``CORSMiddleware`` (origin validation), ``RequestIDMiddleware`` (generates or
propagates ``X-Request-ID``), ``LoggingMiddleware`` (binds ``request_id`` to
structlog context), ``TracingMiddleware`` (OTLP span),
``RateLimitMiddleware`` (IP-based, Redis sorted set), ``AuthMiddleware``
(validates JWT or API key, resolves user + org), ``AuditMiddleware`` (queues
an audit event after the response), ``TrustedHostMiddleware`` (validates
``Host`` header), ``GZipMiddleware`` (compresses responses).

The full lifecycle is documented in :doc:`/domains/api_layer` and
:doc:`/domains/memory_context`.


Async Enrichment Pipeline
-------------------------

The enrichment pipeline transforms raw conversation messages into a structured
knowledge graph:

.. mermaid::

   stateDiagram-v2
       [*] --> ingested: POST /memory persists Episode
       ingested --> queued: enqueue ARQ jobs
       queued --> classifying: classify_dialog
       classifying --> entities_extracted: extract_entities → GraphEntity nodes
       entities_extracted --> facts_extracted: extract_facts → Fact + Relation edges
       facts_extracted --> embedded: embed_episode / embed_facts → pgvector
       embedded --> graph_synced: link_entities_to_episode → temporal edges + observations
       graph_synced --> reconciled: reconcile_enrichment (cron safety net)
       reconciled --> queued: re-queue stuck episodes
       reconciled --> [*]: enrichment complete

Each task checks and sets a bit in the episode's ``enrichment_status`` bitmask:

.. list-table::
   :header-rows: 1

   * - Bit
     - Task
     - Description
   * - 0
     - ``classify_dialog``
     - Conversation type classification
   * - 1
     - ``extract_entities``
     - Named entity extraction from messages
   * - 2
     - ``extract_facts``
     - Fact triple extraction (subject-predicate-object)
   * - 3
     - ``embed_episode`` / ``embed_fact``
     - Vector embedding generation
   * - 4
     - ``link_entities_to_episode``
     - Graph sync + observation computation
   * - 6
     - ``reconcile_enrichment``
     - Reconciliation (re-processes stuck episodes)

This bitmask ensures **idempotent execution** — if a worker crashes mid-pipeline,
the next run skips already-completed steps.

The full pipeline is documented in :doc:`/domains/workers`.


Context Retrieval Pipeline
--------------------------

When a client requests context for an LLM prompt, the system performs a
**multi-leg hybrid search**:

.. code-block:: text

    POST /v1/projects/{project_id}/context
           │
           ▼
    ┌─────────────────────────────────────────────┐
    │           ContextService.assemble()          │
    │                                              │
    │   1. Check context cache (Redis, keyed by    │
    │      query + project_id, TTL configurable)   │
    │                                              │
    │   2. If miss, run multi-leg search:          │
    │      ├─ Vector search  (pgvector <=>)        │
    │      ├─ BM25 search    (PostgreSQL FTS)      │
    │      ├─ Graph BFS      (graph backend)       │
    │      └─ Fact search    (SQL + embedding)     │
    │                                              │
    │   3. RRF fusion (Reciprocal Rank Fusion,     │
    │      RRF_K=60)                               │
    │                                              │
    │   4. Optional: Cross-encoder reranker        │
    │      (sentence-transformers or Cohere)       │
    │                                              │
    │   5. ContextFormatter formats results        │
    │      (text or JSON mode)                     │
    │                                              │
    │   6. Cache the result                        │
    └──────────────────────────────────────────────┘

The retrieval pipeline is documented in :doc:`/domains/memory_context` and
the reranker in :doc:`/domains/reranker`.


Multi-Tenancy Model
-------------------

OpenZync uses a three-level hierarchy for data isolation:

.. mermaid::

   erDiagram
       Organization ||--o{ User : has
       Organization ||--o{ Project : owns
       Organization ||--o{ OrgConfig : has
       Project ||--o{ Episode : contains
       Project ||--o{ GraphEntity : "knowledge graph (entities)"
       Project ||--o{ Fact : "knowledge graph (facts)"
       Project ||--o{ Session : has
       Project ||--o{ WebhookSubscription : has

- **Organization** — billing and administrative boundary.  Has its own
  OpenBao namespace for per-org secrets (LLM API keys).
- **User** — belongs to exactly one organization.  Authenticates via JWT
  (dashboard) or API key (programmatic access).
- **Project** — data isolation boundary.  All memory, graph, and session
  data is scoped to a project.  Users can be members of multiple projects.

Auth is dual-mode:

- **JWT tokens** — for dashboard users.  Short-lived (default 30 min) with
  refresh token rotation.
- **API keys** — for programmatic access (SDK, MCP).  Prefix ``oz_live_`` or
  ``oz_test_``.  Stored as bcrypt hashes.

See :doc:`/domains/auth` for details.


OpenBao Secrets Architecture
----------------------------

OpenZync implements a **zero-fallback secrets architecture**:

.. code-block:: text

    Boot sequence:
    ┌──────────┐    ┌───────────┐    ┌─────────────┐
    │  .env    │───▶│  OpenBao  │───▶│  AppRole     │
    │ (seal    │    │  (init +  │    │  (auth +     │
    │  key)    │    │  unseal)  │    │   policy)    │
    └──────────┘    └───────────┘    └──────┬──────┘
                                            │
                                            ▼
                                    ┌───────────────┐
                                    │  OpenBao KV    │
                                    │  system/config │
                                    │  ├─ DATABASE_URL│
                                    │  ├─ SECRET_KEY │
                                    │  ├─ REDIS_URL  │
                                    │  └─ ...        │
                                    └───────────────┘
                                            │
                                OpenBao Agent sidecar
                                    (renders to file)
                                            │
                                            ▼
                                    ┌───────────────┐
                                    │  api / worker  │
                                    │  (sources file │
                                    │   at startup)  │
                                    └───────────────┘

Key properties:

- **No ``.env`` file** contains runtime secrets.  Only bootstrap credentials
  (``BAO_STATIC_SEAL_KEY``, ``OZ_OPENBAO_ROLE_ID``, etc.) live in ``.env``.
- **AppRole authentication** — API and worker each have their own AppRole
  with least-privilege policies.
- **Auto-generated credentials** — database passwords are generated at
  bootstrap and written directly to OpenBao.  No human ever sees them.
- **Transit encryption** — sensitive per-org data is encrypted via OpenBao's
  Transit secrets engine before storage.
- **Fail-fast** — if OpenBao is unreachable at startup, the process exits
  immediately.  No silent fallback to env vars or defaults.

See :doc:`/domains/core` (Configuration System and OpenBao sections) and
:doc:`/domains/infrastructure` for operational details.


Infrastructure Stack
--------------------

.. list-table::
   :header-rows: 1

   * - Component
     - Technology
     - Purpose
   * - API Server
     - FastAPI / uvicorn
     - HTTP API (port 8000)
   * - Database
     - PostgreSQL 15+ with pgvector
     - Primary data store
   * - Cache / Queues
     - Redis 7+
     - Context cache, rate limiting, ARQ queues
   * - Secrets / Config
     - OpenBao 2.5+
     - All runtime config and secrets
   * - Workers
     - ARQ (Redis-based job queue)
     - Background enrichment, webhook delivery
   * - Graph Backend
     - PostgreSQL / FalkorDB / SurrealDB
     - Knowledge graph storage
   * - Reverse Proxy
     - NGINX
     - TLS termination, routing, static files
   * - Observability
     - Prometheus + Grafana + Alloy
     - Metrics, dashboards, logs, traces
   * - Dashboard
     - Next.js (separate container)
     - Admin UI (port 3000)

All components run in Docker Compose for development and small-scale
production.  Kubernetes/Helm is available for larger deployments.

See :doc:`/domains/infrastructure` and :doc:`/guides/deployment`.


Current State vs Target Architecture
-------------------------------------

**Current state (shipped):**

- Backend monolith: all domains in ``openzync-core``
- Single FastAPI process, single PostgreSQL, single Redis
- ARQ workers in a separate container
- 3 graph backends (PostgreSQL, FalkorDB, SurrealDB)
- 5 LLM providers (OpenAI, Anthropic, Azure, OpenRouter, Ollama)
- 1 reranker implementation (sentence-transformers + Cohere)
- 1 community detection algorithm (Label Propagation)
- Python SDK, MCP server, Next.js frontend, Sphinx docs

**Not yet shipped (roadmap):**

- Decomposition into separate microservices (``openzync-llm``, ``openzync-auth``, etc.)
- Human-in-the-loop approval gates
- Public plugin/tool API
- Deterministic workflow engine
- TypeScript SDK
- Managed cloud service
- Credit/billing system
- Migration tools for common sources

All planned features are described honestly on the landing page and in the
documentation — nothing fictional is claimed as shipped.


Cross-Repo Dependencies
-----------------------

.. code-block:: text

    openzync-core
        ↑ (REST API)        → openzync-frontend
        ↑ (REST API)        → openzync-sdk-python
        ↑ (documents)       → openzync-docs
        ↑ (marketing copy)  → openzync-landing (no code dependency)
    openzync-sdk-python
        ↑ (as dependency)   → openzync-mcp
    openzync-docs
        ↑ (autodoc imports) → openzync-core (at doc-build time)
