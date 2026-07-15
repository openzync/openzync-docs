.. _api-layer:

=========
API Layer
=========

.. module:: openzync.services.api

The OpenZync API Layer is the HTTP gateway for the ``openzync-core`` monolith.
It is a `FastAPI <https://fastapi.tiangolo.com/>`_ application deployed behind
``uvicorn``, serving versioned REST endpoints for memory ingestion, knowledge
graph querying, session management, authentication, administration, and
observability.

.. contents:: On this page
   :local:
   :depth: 3
   :class: this-will-duplicate-information-and-it-is-still-useful-here

--------
Overview
--------

The API Layer follows strict **Domain-Driven Design (DDD)** layering enforced
at the architectural level:

.. code-block:: text

   routers/     → HTTP adapter (no business logic, no DB queries)
   services/    → All business logic, orchestration
   repositories/→ DB access only (returns domain models)
   models/      → SQLAlchemy ORM definitions (no business logic)
   schemas/     → Pydantic request/response models (no ORM imports)
   middleware/  → ASGI middleware for cross-cutting concerns
   dependencies/→ FastAPI ``Depends`` factories

**Never** does a router import a model or touch a DB session directly. **Never**
does a service perform inline SQL. **Never** does a schema import from an ORM
model.

.. code-block:: python

   # ✅ Correct: router → service → repository
   @router.get("/{user_id}", response_model=UserResponse)
   async def get_user(
       user_id: UUID,
       service: UserService = Depends(get_user_service),
   ) -> UserResponse:
       return await service.get_user(user_id=user_id)

   # ❌ Wrong: router touching DB directly
   @router.get("/{user_id}")
   async def get_user(user_id: UUID, db: AsyncSession = Depends(get_db)):
       result = await db.execute(select(User).where(User.id == user_id))
       return result.scalar_one()  # business logic in router — rejected at review

.. _api-layer-app-factory:

---------------------
Application Factory
---------------------

The FastAPI application is built by the ``create_app()`` factory function in
:file:`services/api/main.py` and exposed via the ASGI entry point in
:file:`services/api/asgi.py`.

ASGI Entry Point
================

:file:`services/api/asgi.py` is the module-level entry point for ``uvicorn``:

.. code-block:: bash

   uvicorn services.api.asgi:app --host 0.0.0.0 --port 8000

Before the application is created, ``asgi.py`` performs a **fail-fast OpenBao
bootstrap**:

#. Connects to OpenBao using ``BootstrapSettings`` (read from environment
   variables ``OZ_OPENBAO_ADDR``, ``OZ_OPENBAO_ROLE_ID``,
   ``OZ_OPENBAO_SECRET_ID``).
#. Loads all system settings from OpenBao KV under the ``system/`` namespace
   via :func:`init_settings`.
#. If OpenBao is unreachable, the process exits immediately — the container
   orchestrator (Docker/K8s) handles restart.

After bootstrap, it applies a monkey-patch for a FastAPI 0.115.x regression
(``is_body_allowed_for_status_code`` with ``-> None`` annotations), then
imports and calls ``create_app()``.

Application Factory
===================

:func:`create_app` in :file:`services/api/main.py` builds the complete
application in this order:

1. **Load settings** from the OpenBao-populated singleton.
2. **Configure logging** via :func:`setup_logging` (structlog-based).
3. **Register lifespan** — the :func:`lifespan` async context manager handles
   startup and shutdown of all infrastructure connections.
4. **Register exception handlers** — RFC 7807 ``application/problem+json`` for
   every exception in the domain hierarchy.
5. **Register middleware** — in reverse runtime order (Starlette LIFO).
6. **Include routers** — all domain routers plus the ``/metrics`` endpoint.

Lifespan — Startup
==================

During application startup (the ``yield`` point in the lifespan), the factory
initialises these infrastructure components:

.. list-table:: Lifespan Startup
   :header-rows: 1

   * - Component
     - Initialisation
     - Stored on ``app.state``
   * - **PostgreSQL engine**
     - :func:`init_db_engine` with ``postgresql+asyncpg://``, pool size 20,
       max overflow 10, pool recycle 3600s
     - ``db_engine``, ``db_session_factory``
   * - **Redis client**
     - :func:`init_redis` with connection pooling, ``decode_responses=True``
     - ``redis``
   * - **ARQ pool**
     - :func:`init_arq` — async Redis queue for background jobs
     - ``arq_pool``
   * - **Graph backend dispatcher**
     - :func:`init_dispatcher` — singleton registry of backend classes
     - ``graph_backend_dispatcher``
   * - **SurrealDB pool**
     - :class:`SurrealConnectionPool` — per-org connection pool (optional)
     - ``surreal_connection_pool``
   * - **FalkorDB client**
     - :class:`BlockingConnectionPool` from URL (optional, graceful skip)
     - ``falkordb_client``
   * - **OpenBao client**
     - Persistent :class:`OpenBaoClient` for runtime org-config lookups
     - ``openbao_client``

Lifespan — Shutdown
===================

Shutdown happens in reverse order: FalkorDB → OpenBao → SurrealDB → ARQ →
Redis → PostgreSQL.

Exception Handlers
==================

All domain exceptions inherit from :class:`AppError` and carry:

- ``status_code`` — the HTTP status code to return.
- ``code`` — a machine-readable string (e.g. ``"not_found"``).
- ``message`` — a human-readable description.
- ``detail`` — an optional dict for additional context.

Exception handlers are registered via
:func:`register_exception_handlers` and return `RFC 7807
<https://www.rfc-editor.org/rfc/rfc7807>`_ Problem Details JSON:

.. code-block:: json
   :caption: RFC 7807 Response Body

   {
     "type": "https://errors.openzync.tech/not_found",
     "title": "Not Found",
     "status": 404,
     "detail": "The requested resource was not found.",
     "instance": "/v1/projects/abc-123/sessions/def-456"
   }

.. list-table:: Exception → HTTP Status Mapping
   :header-rows: 1

   * - Exception
     - HTTP Status
     - Code
   * - :class:`NotFoundError`
     - 404
     - ``not_found``
   * - :class:`ValidationError`
     - 422
     - ``validation_error``
   * - :class:`AuthenticationError`
     - 401
     - ``authentication_error``
   * - :class:`AuthorizationError`
     - 403
     - ``authorization_error``
   * - :class:`ConflictError`
     - 409
     - ``conflict``
   * - :class:`RateLimitError`
     - 429
     - ``rate_limit_exceeded``
   * - :class:`InsufficientCreditsError`
     - 402
     - ``insufficient_credits``
   * - :class:`ExternalServiceError`
     - 502
     - ``external_service_error``
   * - :class:`LLMConfigurationError`
     - 502
     - ``llm_configuration_error``
   * - :class:`PayloadTooLargeError`
     - 413
     - ``payload_too_large``
   * - :class:`EntityNotFoundError`
     - 404
     - ``entity_not_found``
   * - :class:`EdgeNotFoundError`
     - 404
     - ``edge_not_found``
   * - :class:`EpisodeNotFoundError`
     - 404
     - ``episode_not_found``
   * - :class:`GraphTimeoutError`
     - 504
     - ``graph_timeout``
   * - :class:`ServiceUnavailableError`
     - 503
     - ``service_unavailable``
   * - :class:`CacheUnavailableError`
     - 503
     - ``cache_unavailable``
   * - :class:`GraphBackendUnavailableError`
     - 503
     - ``graph_backend_unavailable``
   * - :class:`RateLimitUnavailableError`
     - 503
     - ``rate_limit_unavailable``
   * - :class:`MetricsUnavailableError`
     - 503
     - ``metrics_unavailable``
   * - :class:`DatabaseUnavailableError`
     - 503
     - ``database_unavailable``
   * - :class:`SearchLegFailedError`
     - 503
     - ``search_leg_failed``

.. _api-layer-middleware:

-----------------
Middleware Stack
-----------------

The middleware chain uses **raw ASGI middleware** (no ``BaseHTTPMiddleware``
overhead). Starlette middleware is LIFO — the last ``add_middleware()`` call
wraps the outermost layer and runs **first** on every request.

The numbered comments in the factory code show the RUNTIME execution order
(outermost → innermost):

.. code-block:: text

   Runtime order (outermost first):
      0. MetricsMiddleware    — RED metrics (wraps everything, including 404s)
      1. CORSMiddleware       — intercepts OPTIONS preflight
      2. LoggingMiddleware    — request/response lifecycle logging
      3. TracingMiddleware    — OpenTelemetry span management
      4. RateLimitMiddleware  — per-IP/per-token sliding-window rate limit
      5. AuthMiddleware       — extract/validate JWT & API key
      6. AuditMiddleware      — post-response audit log enqueue
      7. GZipMiddleware       — compress responses >= 1 KB
      8. TrustedHostMiddleware— host-header attack prevention
      9. RequestIDMiddleware  — assign X-Request-ID (innermost, closest to router)

MetricsMiddleware
=================

.. module:: middleware.metrics

:file:`middleware/metrics.py`

**Registry**: ``METRICS_REGISTRY`` — an isolated ``CollectorRegistry`` that
excludes default process/GC metrics.

**Metrics exposed**:

.. list-table:: Prometheus Metrics
   :header-rows: 1

   * - Metric
     - Type
     - Labels
     - Description
   * - ``openzync_http_requests_total``
     - Counter
     - ``method``, ``path``, ``status``
     - Total HTTP requests (rate)
   * - ``openzync_http_errors_total``
     - Counter
     - ``method``, ``path``
     - Total 5xx responses (errors)
   * - ``openzync_http_request_duration_seconds``
     - Histogram
     - ``method``, ``path``
     - Latency buckets (duration)
   * - ``openzync_http_requests_in_progress``
     - Gauge
     - ``method``
     - Concurrent requests (saturation)
   * - ``openzync_http_request_size_bytes``
     - Histogram
     - ``method``
     - Request body size
   * - ``openzync_context_latency_seconds``
     - Histogram
     - ``type`` (cold|warm)
     - Context assembly latency
   * - ``openzync_graph_search_latency_seconds``
     - Histogram
     - (none)
     - Hybrid search latency
   * - ``openzync_reranker_latency_seconds``
     - Histogram
     - ``backend``
     - Re-ranker inference latency

**Position**: Outermost. Catches EVERY request including 404s for unknown
routes.

CORSMiddleware
==============

.. module:: fastapi.middleware.cors

Built-in FastAPI middleware. Configured from ``settings.CORS_ORIGINS``
(comma-separated string from OpenBao).

.. code-block:: python

   app.add_middleware(
       CORSMiddleware,
       allow_origins=settings.CORS_ORIGINS.split(","),
       allow_credentials=True,
       allow_methods=["*"],
       allow_headers=["*"],
   )

**Position**: Runtime 1 (after Metrics). Intercepts ``OPTIONS`` preflight
requests before AuthMiddleware can reject them.

LoggingMiddleware
=================

.. module:: middleware.logging

:file:`middleware/logging.py`

Structured request/response logging. Logs a ``"request.completed"`` message at
INFO level with:

- ``method`` (GET, POST, ...)
- ``path`` (URL path)
- ``status_code`` (HTTP response status)
- ``duration_ms`` (wall-clock time in milliseconds)
- ``request_id`` (from RequestIDMiddleware)

Uses ``structlog`` with a custom processor ``add_request_context`` that merges
bound context variables (``request_id``, ``method``, ``path``, ``status_code``,
``duration_ms``) into every log entry automatically.

TracingMiddleware
=================

.. module:: middleware.tracing

:file:`middleware/tracing.py`

OpenTelemetry distributed tracing. Initialises a ``TracerProvider`` with an
OTLP gRPC exporter when ``OZ_OTLP_ENDPOINT`` is set. If unset, the middleware
is a zero-overhead pass-through.

Configuration (via environment variables):

- ``OZ_OTLP_ENDPOINT`` — OTLP gRPC endpoint (e.g. ``http://localhost:4317``).
- ``OZ_OTLP_HEADERS`` — optional comma-separated ``key=value`` headers.
- ``OZ_TRACE_SAMPLE_RATE`` — sampling rate (default 0.05, i.e. 5%).
- ``OZ_SERVICE_NAME`` — OpenTelemetry service name (default ``"openzync"``).

Spans include HTTP semantic convention attributes (method, URL, status code,
host, request ID, org ID).

**Position**: Runtime 3. Inside rate-limit but outside auth, so traces exist
even for rejected requests.

RateLimitMiddleware
===================

.. module:: middleware.rate_limit

:file:`middleware/rate_limit.py`

Sliding-window rate limiting using Redis sorted sets
(``ZREMRANGEBYSCORE`` / ``ZADD`` / ``ZCOUNT``). Two tiers:

1. **IP-based** — for unauthenticated requests
   (key: ``rate:auth:{ip}``, default: 10 req / 60s).
2. **Org-based** — for authenticated API requests
   (key: ``rate:api:{org_id}``, configured per-org quota or fallback: 1000
   req / 60s).

**Behaviour**:

- Bypass for non-production environments (development, testing).
- Bypass for ``/health`` and ``/ready``.
- Fail-closed: if Redis is unreachable and a config read fails, raises
  :class:`RateLimitUnavailableError`.
- Fail-open: if Redis ping fails, passes through with a warning.
- Wraps the downstream response with ``X-RateLimit-Limit``,
  ``X-RateLimit-Remaining``, ``X-RateLimit-Reset`` headers.
- Returns RFC 7807 ``application/problem+json`` on 429.

**Position**: Runtime 4. Inside tracing but outside auth — rate-limits before
auth processing cost.

AuthMiddleware
==============

.. module:: middleware.auth

:file:`middleware/auth.py`

**The authentication gate for every protected endpoint.** A raw ASGI middleware
that extracts and validates credentials from the ``Authorization: Bearer
<token>`` header.

Supports two authentication modes:

**API Key mode** (for SDK clients):

#. Token starts with ``oz_live_`` or ``oz_test_`` prefix.
#. Computes an unsalted SHA-256 lookup hash via :func:`compute_lookup_hash`.
#. Checks Redis cache at ``auth:key:{lookup_hash}`` (TTL: 300s).
#. On miss, queries the ``api_keys`` table via :class:`ApiKeyRepository`.
#. Verifies the raw key against the salted hash from the DB.
#. Sets ``scope["state"]`` fields: ``auth_type="api_key"``, ``org_id``,
   ``user_id`` (``created_by``), ``api_key_scopes``, ``api_key_project_id``.
#. Updates ``last_used`` timestamp on the API key (fire-and-forget).
#. Caches the result in Redis (fire-and-forget).

**JWT mode** (for dashboard users):

#. Token is a three-segment JWT (starts with ``eyJ``).
#. Verifies signature with ``OZ_SECRET_KEY`` via HS256.
#. Extracts ``sub`` (user_id), ``org_id``, ``role``, ``type`` claims.
#. Rejects tokens where ``type != "access"``.
#. Sets ``scope["state"]`` fields: ``auth_type="jwt"``, ``org_id``,
   ``user_id``, ``role``, ``api_key_scopes`` (all scopes granted).

**Security features**:

- **Negative cache**: ``auth:neg:{lookup_hash}`` — if a key is not found in
  the DB, subsequent lookups are rejected for 60s without a DB round-trip.
- **Auth miss-rate limiting**: ``auth:miss_ip:{ip}`` — max 10 DB misses per
  IP per 60s to prevent credential-stuffing.
- **Public endpoint allowlist** — paths exempt from auth (see
  :data:`PUBLIC_ENDPOINTS`).
- **CORS preflight pass-through** — ``OPTIONS`` requests are always allowed
  as defense-in-depth.
- **Prometheus scrape exemption** — ``/metrics`` exact-path only.
- All 401/403 responses use RFC 7807 bodies.
- PostgreSQL RLS context is set by the ``get_db`` dependency, not the
  middleware (``set_config`` is session-local).

**Position**: Runtime 5. Inside rate-limit so throttled attackers never reach
auth processing.

AuditMiddleware
===============

.. module:: middleware.audit

:file:`middleware/audit.py`

Post-response audit logging. Enqueues an ARQ job (low-priority queue) for
every non-exempt mutating request. The ARQ worker writes the entry to the
``audit_logs`` table asynchronously.

**Behaviour**:

- Exempt paths: ``/health``, ``/ready``, ``/metrics``, ``/docs``,
  ``/openapi.json``, ``/redoc``, ``/favicon.ico``.
- Exempt methods: ``OPTIONS``, ``GET``.
- Resolves a semantic ``action`` and ``resource_type`` from a route-action
  mapping (e.g. ``("POST", "/v1/auth/signup")`` → ``("auth.signup", "user")``).
- Captures response body if the org's config has
  ``audit_log_response_body=true`` — the body is PII-redacted before storage.
- Extracts ``resource_id`` from the URL path via UUID regex.
- Fire-and-forget: audit job failures never fail the original request.

**Position**: Runtime 6. Inside auth, so ``scope["state"]`` is fully
populated, but outside the router to capture all responses.

GZipMiddleware
==============

.. module:: fastapi.middleware.gzip

Built-in FastAPI middleware. Compresses responses >= 1000 bytes.

**Position**: Runtime 7.

TrustedHostMiddleware
=====================

.. module:: fastapi.middleware.trustedhost

Built-in FastAPI middleware. Prevents host-header attacks. In production,
configured from ``settings.HOSTS_ALLOWED`` (comma-separated). In development,
accepts ``["*"]``.

**Position**: Runtime 8.

RequestIDMiddleware
===================

.. module:: middleware.request_id

:file:`middleware/request_id.py`

Ensures every request has a traceable ``X-Request-ID``:

#. Reads ``X-Request-ID`` from incoming headers, or generates a UUID if absent.
#. Stores it on ``scope["state"]["request_id"]``.
#. Binds it to ``structlog.contextvars`` for automatic inclusion in all log
   entries.
#. Injects ``X-Request-ID`` into response headers.
#. Clears context variables in the ``finally`` block.

**Position**: Runtime 9 (innermost). Closest to the router so every downstream
layer (logging, auth, rate-limit) has access to the request ID.

.. _api-layer-router-reference:

-------------------------
Endpoint Reference Table
-------------------------

Every router, every endpoint, with method, path, auth, and description.

.. tip::

   All project-scoped endpoints (under ``/v1/projects/{project_id}/``) use
   :func:`require_project_membership` for unified auth
   (JWT + API key + project membership verification). The ``project_id`` in
   the path is validated against the authenticated user's organisation and
   membership status.

Health
======

.. module:: routers.health

Prefix: *(none — registered at root level)*

.. list-table::
   :header-rows: 1
   :widths: 10 10 15 10 55

   * - Method
     - Path
     - Auth
     - Status
     - Description
   * - ``GET``
     - ``/health``
     - Public
     - 200
     - **Liveness probe**. Returns ``{"status": "ok", "service": "openzync-api"}``.
       Does **not** check downstream dependencies.
   * - ``GET``
     - ``/ready``
     - Public
     - 200 / 503
     - **Readiness probe**. Validates PostgreSQL and Redis connectivity. Returns
       200 if all healthy, 503 with per-check details on failure.

Authentication
==============

.. module:: routers.auth

Prefix: ``/v1/auth``

.. list-table::
   :header-rows: 1
   :widths: 10 25 15 50

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``POST``
     - ``/v1/auth/signup``
     - Public
     - Create a new organisation with an admin dashboard user. Returns 201 with
       a confirmation message. A verification OTP is sent via email.
   * - ``POST``
     - ``/v1/auth/verify-email``
     - Public
     - Verify email with 6-digit OTP. Returns access + refresh JWT tokens.
       Rate-limited to prevent brute-force OTP guessing.
   * - ``POST``
     - ``/v1/auth/resend-otp``
     - Public
     - Resend the email verification OTP. Rate-limited: 5 sends per hour per
       email.
   * - ``POST``
     - ``/v1/auth/login/otp/send``
     - Public
     - Send a passwordless login OTP to the user's email.
   * - ``POST``
     - ``/v1/auth/login/otp/verify``
     - Public
     - Verify passwordless login OTP. Returns JWT tokens. Auto-verifies email
       on first login.
   * - ``POST``
     - ``/v1/auth/forgot-password``
     - Public
     - Send a password-reset OTP. Returns same response whether email exists or
       not (prevents email enumeration).
   * - ``POST``
     - ``/v1/auth/reset-password``
     - Public
     - Reset password with OTP. Invalidates all existing sessions.
   * - ``POST``
     - ``/v1/auth/login``
     - Public
     - Authenticate by email+password. Returns JWT tokens (MFA off) or MFA
       challenge with ``mfa_session_token`` (MFA on).
   * - ``POST``
     - ``/v1/auth/mfa/verify``
     - Public
     - Complete MFA login with OTP + ``mfa_session_token``. Returns JWT tokens.
   * - ``POST``
     - ``/v1/auth/mfa/enable``
     - JWT
     - Enable email-based MFA. Requires current password for re-authentication.
   * - ``POST``
     - ``/v1/auth/mfa/disable``
     - JWT
     - Disable MFA. Requires current password + MFA OTP.
   * - ``POST``
     - ``/v1/auth/refresh``
     - Public
     - Rotate refresh token. Returns new access + refresh token pair. Previous
       refresh token is revoked.
   * - ``GET``
     - ``/v1/auth/me``
     - JWT
     - Get the current dashboard user's profile (email, name, role, org,
       verification status).
   * - ``PATCH``
     - ``/v1/auth/me``
     - JWT
     - Update profile (name, email, password). All fields optional. Requires
       current password for password change.

Users
=====

.. module:: routers.users

Prefix: ``/v1/users``

.. list-table::
   :header-rows: 1
   :widths: 10 25 15 50

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``POST``
     - ``/v1/users``
     - API Key / JWT
     - Create a new user. ``external_id`` is caller-defined, must be unique
       per org. Returns 409 on duplicate.
   * - ``GET``
     - ``/v1/users``
     - API Key / JWT
     - List users with cursor-based pagination, multi-field search, and date-range
       filtering. Default limit: 50 (max 200).
   * - ``GET``
     - ``/v1/users/{user_id}``
     - API Key / JWT
     - Get user by internal UUID. Includes aggregate statistics (message_count,
       fact_count, session_count).
   * - ``PATCH``
     - ``/v1/users/{user_id}``
     - API Key / JWT
     - Update user fields. ``metadata`` is deep-merged, not replaced.
       ``None`` means "set to null".
   * - ``DELETE``
     - ``/v1/users/{user_id}``
     - API Key / JWT
     - Soft-delete user (``is_deleted=true``). Invisible immediately. Hard-delete
       after 30-day grace period via ARQ worker.
   * - ``GET``
     - ``/v1/users/{user_id}/summary``
     - API Key / JWT
     - Get the LLM-generated user summary. Returns 404 if not yet generated.
   * - ``POST``
     - ``/v1/users/{user_id}/summary``
     - API Key / JWT
     - Trigger user summary generation. Returns 202. Rate-limited to once per
       5 minutes per user.
   * - ``GET``
     - ``/v1/users/{user_id}/summary-instructions``
     - API Key / JWT
     - List custom instructions for user summary generation.
   * - ``PUT``
     - ``/v1/users/{user_id}/summary-instructions``
     - API Key / JWT
     - Replace all summary instructions for a user.
   * - ``DELETE``
     - ``/v1/users/{user_id}/summary-instructions``
     - API Key / JWT
     - Clear all summary instructions for a user.

Sessions
========

.. module:: routers.sessions

Prefix: ``/v1/projects/{project_id}/sessions``

All endpoints require project membership.

.. list-table::
   :header-rows: 1
   :widths: 10 25 15 50

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``POST``
     - ``/v1/projects/{project_id}/sessions``
     - Project Member
     - Create a new session. ``external_id`` is caller-defined, must be unique
       per project. Returns 409 on duplicate.
   * - ``GET``
     - ``/v1/projects/{project_id}/sessions``
     - Project Member
     - List sessions with cursor-based pagination. Excludes closed sessions by
       default. ``include_closed=true`` to include them.
   * - ``GET``
     - ``/v1/projects/{project_id}/sessions/{session_id}``
     - Project Member
     - Get session details with aggregate stats (message count, fact count).
   * - ``GET``
     - ``/v1/projects/{project_id}/sessions/{session_id}/messages``
     - Project Member
     - Get paginated messages for a session, ordered by ``sequence_number``.
       Default limit: 100 (max 500).
   * - ``GET``
     - ``/v1/projects/{project_id}/sessions/{session_id}/facts``
     - Project Member
     - Get paginated facts extracted from a session. Newest first. Only
       non-invalidated facts.
   * - ``DELETE``
     - ``/v1/projects/{project_id}/sessions/{session_id}``
     - Project Member
     - Soft-delete a session. Episodes are unlinked but preserved as orphaned
       history.

Memory (Ingestion)
==================

.. module:: routers.memory

Prefix: ``/v1/projects/{project_id}/memory``

.. list-table::
   :header-rows: 1
   :widths: 10 30 15 45

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``POST``
     - ``/v1/projects/{project_id}/memory``
     - Project Member
     - **Ingest messages** into project memory. Accepts up to 1000 messages.
       Returns 202 immediately with a ``Location`` header for job tracking.
       Supports ``Idempotency-Key`` header (48h cache). Each message content
       limited to 64KB UTF-8. ``session_id`` is optional (auto-creates
       ``__default__`` session).
   * - ``DELETE``
     - ``/v1/projects/{project_id}/memory``
     - Project Member
     - **Wipe all project memory**. Soft-deletes all episodes and facts.
       Sessions are preserved. Data is hard-purged after 30 days.

Context
=======

.. module:: routers.context

Prefix: ``/v1/projects/{project_id}/context``

.. list-table::
   :header-rows: 1
   :widths: 10 30 15 45

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/v1/projects/{project_id}/context``
     - Project Member
     - **Assemble context block** for LLM injection. Hybrid retrieval pipeline:
       pgvector (semantic) + BM25 (keyword) + Graph BFS (entity-relationship).
       Results RRF-merged. Supports ``format=text|json``. Cached in Redis for
       30s. Sets ``X-Cache: HIT|MISS`` header.

Search
======

.. module:: routers.search

Prefix: ``/v1/projects/{project_id}/search``

.. list-table::
   :header-rows: 1
   :widths: 10 30 15 45

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/v1/projects/{project_id}/search``
     - Project Member
     - **Hybrid search** across project memory. Three-leg retrieval: vector +
       BM25 + RRF. Filter by ``types`` (episodes, facts, entities, communities).
       Returns query, results, total.

Classifications
===============

.. module:: routers.classifications

Prefix: ``/v1/projects/{project_id}/sessions/{session_id}/classifications``

.. list-table::
   :header-rows: 1
   :widths: 10 40 15 35

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/v1/projects/{project_id}/sessions/{session_id}/classifications``
     - Project Member
     - List all dialog classifications for episodes in a session.
       Returns empty list if ``classify_dialog`` worker hasn't run yet.
   * - ``GET``
     - ``/v1/projects/{project_id}/sessions/{session_id}/classifications/{episode_id}``
     - Project Member
     - Get classification for a specific episode. Returns 404 if not yet
       classified.

Structured Extractions
======================

.. module:: routers.structured_extractions

Prefix: ``/v1/projects/{project_id}/sessions/{session_id}/structured-extractions``

.. list-table::
   :header-rows: 1
   :widths: 10 45 15 30

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/v1/projects/{project_id}/sessions/{session_id}/structured-extractions``
     - Project Member
     - List structured extractions for all episodes in a session. Returns empty
       if ``extract_structured`` worker hasn't run or no schemas configured.
   * - ``GET``
     - ``/v1/projects/{project_id}/sessions/{session_id}/structured-extractions/{episode_id}``
     - Project Member
     - Get structured extraction for a specific episode. Returns 404 if not
       found.

Facts
=====

.. module:: routers.facts

Prefix: ``/v1/projects/{project_id}/facts``

.. list-table::
   :header-rows: 1
   :widths: 10 30 15 45

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``POST``
     - ``/v1/projects/{project_id}/facts``
     - Project Member
     - **Ingest business fact triples** (subject-predicate-object). Max 500
       triples per request. Returns 202 with job_id. ``session_id`` is optional.
       ``content`` auto-generated if omitted.

Knowledge Graph
===============

.. module:: routers.graph

Prefix: ``/v1/projects/{project_id}/graph``

.. list-table::
   :header-rows: 1
   :widths: 10 35 15 40

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/v1/projects/{project_id}/graph/nodes``
     - Project Member
     - List entity nodes. Optional ``entity_type`` filter and cursor
       pagination.
   * - ``GET``
     - ``/v1/projects/{project_id}/graph/nodes/{node_id}``
     - Project Member
     - Get a single entity node with all incident edges.
   * - ``DELETE``
     - ``/v1/projects/{project_id}/graph/nodes/{node_id}``
     - Project Member
     - Delete an entity node and all incident edges. Returns 404 if not found.
   * - ``GET``
     - ``/v1/projects/{project_id}/graph/edges``
     - Project Member
     - List relationship edges. Filter by ``subject_id``, ``subject_ids``
       (comma-separated), and ``predicate``.
   * - ``GET``
     - ``/v1/projects/{project_id}/graph/communities``
     - Project Member
     - List community summary nodes. Returns empty list until communities are
       computed (scheduled background task).

Projects
========

.. module:: routers.projects

Prefix: ``/v1/projects``

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``POST``
     - ``/v1/projects``
     - API Key / JWT
     - Create a new project. JWT users auto-added as owner.
   * - ``GET``
     - ``/v1/projects``
     - API Key / JWT
     - List non-archived projects. JWT users see only their projects; API keys
       see all org projects.
   * - ``GET``
     - ``/v1/projects/{project_id}``
     - Project Member
     - Get a single project by ID.
   * - ``PATCH``
     - ``/v1/projects/{project_id}``
     - Project Owner
     - Update project name and/or description.
   * - ``DELETE``
     - ``/v1/projects/{project_id}``
     - Project Owner
     - Archive a project (soft-delete, preserves all data).
   * - ``POST``
     - ``/v1/projects/{project_id}/members``
     - Project Owner
     - Add a user to the project.
   * - ``GET``
     - ``/v1/projects/{project_id}/members``
     - Project Member
     - List all project members.
   * - ``DELETE``
     - ``/v1/projects/{project_id}/members/{user_id}``
     - Project Owner
     - Remove a user from the project. Cannot remove the last owner.
   * - ``PATCH``
     - ``/v1/projects/{project_id}/members/{user_id}``
     - Project Owner
     - Change a member's role (``owner`` / ``member``). Cannot downgrade the
       last owner.

Project API Keys
================

.. module:: routers.project_api_keys

Prefix: ``/v1/projects/{project_id}/api-keys``

.. list-table::
   :header-rows: 1
   :widths: 10 30 15 45

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/v1/projects/{project_id}/api-keys``
     - Project Owner (JWT only)
     - List non-revoked API keys for the project.
   * - ``POST``
     - ``/v1/projects/{project_id}/api-keys``
     - Project Owner (JWT only)
     - Create a new project-scoped API key. Returns the raw key exactly once.
   * - ``DELETE``
     - ``/v1/projects/{project_id}/api-keys/{key_id}``
     - Project Owner (JWT only)
     - Revoke (soft-delete) a project API key.

API Key Self-Service
====================

.. module:: routers.api_key_self

Prefix: ``/v1/api-key``

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/v1/api-key/project-id``
     - API Key / JWT
     - Return the ``project_id`` scoped to the authenticating API key. Returns
       ``null`` for JWT dashboard sessions.

Admin — Bootstrap
=================

.. module:: routers.admin

Prefix: ``/admin``

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``POST``
     - ``/admin/organizations``
     - **Public** (bootstrap)
     - Create a new organization and generate an admin API key. One-shot
       bootstrap endpoint intended for first-use flows. Should be disabled in
       production.

Admin — Organization Config
===========================

.. module:: routers.admin_org_config

Prefix: ``/admin/org/config``

.. list-table::
   :header-rows: 1
   :widths: 10 25 15 50

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/admin/org/config/defaults``
     - **Public**
     - Return seeded onboarding defaults from ``config/defaults/org_config.yaml``.
       Secrets are returned as empty strings.
   * - ``GET``
     - ``/admin/org/config``
     - API Key / JWT
     - Get the stored configuration for the current organisation. Unset fields
       are ``null``.
   * - ``PATCH``
     - ``/admin/org/config``
     - ``admin:write``
     - Partially update organisation config. Only provided fields are changed.
       Set a field to ``null`` to clear it.
   * - ``PUT``
     - ``/admin/org/config``
     - ``admin:write``
     - Replace the entire organisation configuration. Omitted fields are
       removed.

Admin — Schemas
===============

.. module:: routers.admin_schemas

Prefix: ``/v1/admin/schemas``

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``POST``
     - ``/v1/admin/schemas``
     - ``admin`` scope
     - Create an extraction or classification schema. Name must be unique per
       org.
   * - ``GET``
     - ``/v1/admin/schemas``
     - API Key / JWT
     - List schemas. Filter by ``type`` (structured/classification) and
       ``is_active``.
   * - ``GET``
     - ``/v1/admin/schemas/{schema_id}``
     - API Key / JWT
     - Get a single schema by ID (org-scoped).
   * - ``PUT``
     - ``/v1/admin/schemas/{schema_id}``
     - ``admin`` scope
     - Update a schema. ``type`` field is immutable after creation.
   * - ``DELETE``
     - ``/v1/admin/schemas/{schema_id}``
     - ``admin`` scope
     - Soft-delete a schema (sets ``is_active=false``).

Admin — Webhooks
================

.. module:: routers.admin_webhooks

Prefix: ``/v1/admin/webhooks``

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/v1/admin/webhooks/events``
     - Public
     - List all subscribable webhook event types, grouped by category.
   * - ``GET``
     - ``/v1/admin/webhooks``
     - JWT
     - List all webhook endpoints for the organisation.
   * - ``GET``
     - ``/v1/admin/webhooks/{endpoint_id}``
     - JWT
     - Get a single webhook endpoint by ID.
   * - ``POST``
     - ``/v1/admin/webhooks``
     - JWT
     - Create a new webhook endpoint. Returns the signing secret (shown once).
   * - ``PATCH``
     - ``/v1/admin/webhooks/{endpoint_id}``
     - JWT
     - Update a webhook endpoint (name, URL, events, active).
   * - ``DELETE``
     - ``/v1/admin/webhooks/{endpoint_id}``
     - JWT
     - Delete a webhook endpoint.

Admin — Org Configuration (Prompt Templates + Custom Instructions)
==================================================================

.. module:: routers.admin_organizations

Prefix: ``/admin/org``

.. list-table::
   :header-rows: 1
   :widths: 10 35 15 40

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/admin/org/prompts``
     - JWT
     - List all prompt template names with override status, current version,
       and last-updated timestamp.
   * - ``GET``
     - ``/admin/org/prompts/system``
     - JWT
     - List system-default prompt templates grouped by base name.
   * - ``POST``
     - ``/admin/org/prompts/import``
     - JWT
     - Import a system-default prompt template into the org. Returns 409 if
       already imported, 404 if no active system default.
   * - ``POST``
     - ``/admin/org/prompts/{name}/set-default``
     - JWT
     - Mark a prompt template as the active default for its type.
   * - ``GET``
     - ``/admin/org/prompts/{name}``
     - JWT
     - Get the active template for an org. Returns 404 if not found.
   * - ``GET``
     - ``/admin/org/prompts/{name}/versions``
     - JWT
     - List all versions of a named template, newest first.
   * - ``PUT``
     - ``/admin/org/prompts/{name}``
     - JWT
     - Create a new org-specific version (max version + 1). Invalidates Redis
       cache.
   * - ``POST``
     - ``/admin/org/prompts/{name}/rollback/{version}``
     - JWT
     - Rollback to a previous version (creates a new version with old text).
   * - ``DELETE``
     - ``/admin/org/prompts/{name}``
     - JWT
     - Delete all org-specific versions. Cannot delete if it's the active
       default for its type.
   * - ``GET``
     - ``/admin/org/custom-instructions``
     - JWT
     - List all extraction custom instructions for the organisation.
   * - ``PUT``
     - ``/admin/org/custom-instructions``
     - JWT
     - Replace all extraction custom instructions atomically.
   * - ``DELETE``
     - ``/admin/org/custom-instructions``
     - JWT
     - Clear all extraction custom instructions.

Admin — Audit Logs
==================

.. module:: routers.audit_log

Prefix: ``/v1/admin/audit-logs``

.. list-table::
   :header-rows: 1
   :widths: 10 25 15 50

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/v1/admin/audit-logs``
     - JWT
     - List paginated audit log entries. Filterable by action, actor_id,
       actor_type, resource_type, resource_id, status_code, and date range.

Admin — Metrics
===============

.. module:: routers.admin_metrics

Prefix: ``/metrics``

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/metrics/summary``
     - API Key / JWT
     - Aggregated admin dashboard metrics (DB counts + Prometheus performance).
       ``status`` is ``"degraded"`` if Prometheus is unreachable.
   * - ``GET``
     - ``/metrics/query``
     - API Key / JWT
     - Run an arbitrary PromQL instant query. Returns 502 if Prometheus is
       unreachable.
   * - ``GET``
     - ``/metrics/targets``
     - API Key / JWT
     - List Prometheus scrape targets and their health.

Admin — Stats
=============

.. module:: routers.admin_stats

Prefix: ``/v1/admin/stats``

.. list-table::
   :header-rows: 1
   :widths: 10 20 15 55

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/v1/admin/stats/org``
     - JWT
     - Aggregate counts (users, sessions, episodes, facts, messages, API keys).
   * - ``GET``
     - ``/v1/admin/stats/usage``
     - JWT
     - Daily usage trends (messages and sessions per day). Default 30-day look-back,
       max 365.

Prometheus Metrics
==================

.. module:: routers.metrics

.. list-table::
   :header-rows: 1
   :widths: 10 10 15 65

   * - Method
     - Path
     - Auth / Scopes
     - Description
   * - ``GET``
     - ``/metrics``
     - **Public** (Prometheus scrape)
     - Exposes Prometheus metrics in text format from the isolated
       ``METRICS_REGISTRY``. Intentionally unauthenticated — Prometheus
       scrapers cannot carry bearer tokens. No PII or business data exposed.

.. _api-layer-idempotency:

-----------------------
Idempotency Model
-----------------------

.. module:: services.idempotency_service

The :class:`IdempotencyService` provides three layers of idempotency and
deduplication protection:

1. HTTP-level (``Idempotency-Key`` header)
2. Content-level (SHA-256 content hash)
3. Worker-level (bitmask on ``episodes.enrichment_status``)

HTTP-Level Idempotency
======================

Prevents duplicate processing when a client retries the same request with the
same ``Idempotency-Key`` header.

.. code-block:: python

   result = await service.check_idempotency_key(key, body_hash)
   if result.status == IdempotencyStatus.NEW:
       response_data = await process_fn()
       await service.store_idempotency_key(key, body_hash, response_data)
   elif result.status == IdempotencyStatus.REPLAY:
       return result.response_data  # cached response, no side effects
   elif result.status == IdempotencyStatus.CONFLICT:
       raise HTTPException(409, ...)  # same key, different body

**Key format**: ``OpenZync:{env}:idempotency:{key}`` (Redis, 48h TTL).

**States**:

- ``NEW`` — first use, caller should process and store.
- ``REPLAY`` — duplicate, cached response returned.
- ``CONFLICT`` — same key but different body hash (client error).

Content-Level Deduplication
===========================

Prevents the same ``(org_id, user_id, session_id, messages)`` combination
from being ingested more than once, even from different clients with different
``Idempotency-Key`` values.

.. code-block:: python

   if await service.check_content_hash(org_id, user_id, session_id, messages):
       return  # duplicate content, skip

   await service.store_content_hash(org_id, user_id, session_id, messages)
   # ... continue processing ...

**Hash computation**: SHA-256 over a canonical JSON document containing only
``org_id``, ``user_id``, ``session_id``, and ``(role, content)`` pairs —
metadata and ordering are excluded for deterministic dedup.

**Race safety**: Uses ``SETNX`` (``nx=True``) on Redis so that two concurrent
ingestions of the same content cannot both pass the check. The second caller
sees ``SETNX`` returns ``False`` and can detect a race.

Worker-Level Idempotency (Bitmask)
==================================

Prevents ARQ worker tasks from processing the same episode twice. Uses
``SELECT ... FOR UPDATE`` row-level locking and bitwise operations on the
integer ``episodes.enrichment_status`` column.

.. code-block:: python

   if await service.check_and_mark_worker(db, episode_id, ENRICHMENT_ENTITIES):
       await do_extract_entities(db, episode_id)

**Bitmask constants**:

.. list-table::
   :header-rows: 1

   * - Constant
     - Bit
     - Value
     - Task
   * - ``ENRICHMENT_ENTITIES``
     - Bit 0
     - ``1``
     - Entity extraction
   * - ``ENRICHMENT_EMBEDDING``
     - Bit 1
     - ``2``
     - Episode embedding
   * - ``ENRICHMENT_FACTS``
     - Bit 2
     - ``4``
     - Fact extraction
   * - ``ENRICHMENT_ENTITY_LINKS``
     - Bit 3
     - ``8``
     - Entity-episode linking
   * - ``ENRICHMENT_ALL``
     - Bits 0–3
     - ``15``
     - All tasks complete

**Flow**:

#. ``SELECT ... FOR UPDATE`` locks the episode row.
#. Checks if ``current_status & task_bit`` is non-zero (already done).
#. If not set, applies ``UPDATE episodes SET enrichment_status = enrichment_status | task_bit``.
#. Returns ``True`` (caller should proceed) or ``False`` (already completed or
   being processed by another worker).

Cache Invalidation
==================

.. code-block:: python

   await service.invalidate_user_cache(org_id, user_id)

Uses Redis ``SCAN`` to find all keys matching
``OpenZync:{env}:cache:{org_id}:{user_id}:*`` and deletes them in batches
of 100.

.. _api-layer-dependencies:

--------------------
FastAPI Dependencies
--------------------

Authentication Dependencies
===========================

.. module:: dependencies.auth

:file:`dependencies/auth.py`

Five levels of auth dependency, all relying on ``request.state`` attributes
set by :class:`AuthMiddleware <openzync.middleware.auth.AuthMiddleware>`:

.. list-table::
   :header-rows: 1

   * - Dependency
     - Return Type
     - Description
   * - :func:`get_org_id`
     - ``str | None``
     - Optional auth. Returns org ID if authenticated, ``None`` otherwise.
       Works with both API keys and JWT.
   * - :func:`require_org_id`
     - ``str``
     - Mandatory auth. Returns org ID or raises HTTP 401 with RFC 7807 body.
   * - :func:`require_scope(scope_name)`
     - ``str``
     - Dependency factory. Checks API key has a specific scope. JWT users
       automatically pass (full access). Raises HTTP 403 on missing scope.
   * - :func:`get_dashboard_user`
     - ``str``
     - Requires JWT auth. Returns the ``user_id`` from JWT claims. Raises
       HTTP 401 if API key auth is used instead.
   * - :func:`get_current_user_id`
     - ``UUID``
     - Works with both JWT and API key. Returns the authenticated user's UUID.
       Raises HTTP 401 if not authenticated.

**Usage example**:

.. code-block:: python

   @router.get("/sensitive")
   async def sensitive_endpoint(
       org_id: str = Depends(require_org_id),
       user_id: UUID = Depends(get_current_user_id),
   ):
       ...

   @router.post("/admin/action")
   async def admin_action(
       org_id: str = Depends(require_scope("admin:write")),
   ):
       ...

Project Auth Dependencies
=========================

.. module:: dependencies.project_auth

:file:`dependencies/project_auth.py`

Provides unified authentication + project authorization:

.. list-table::
   :header-rows: 1

   * - Dependency
     - Description
   * - :func:`require_project_membership`
     - Verifies the user is a member of the project. For JWT: checks project
       exists and user is a member. For API keys: checks key is scoped to this
       project. Raises 401/403/404.
   * - :func:`require_project_owner`
     - Like ``require_project_membership`` but additionally checks the
       ``owner`` role. API key auth is NOT supported (raises 403).

DB Session Dependency
=====================

.. module:: dependencies.db

:file:`dependencies/db.py`

.. code-block:: python

   async def get_db(request: Request) -> AsyncGenerator[AsyncSession, None]:

Yields an :class:`AsyncSession` from the application's session factory.

**Key behaviour**:

- Reads the session factory from ``request.app.state.db_session_factory``.
- Applies PostgreSQL RLS context: sets ``app.org_id`` and ``app.bypass_rls``
  session-level config parameters via ``SELECT set_config(...)``.
- On yield, commits the transaction. On exception, rolls back.

Service Dependencies
====================

.. module:: dependencies.services

:file:`dependencies/services.py`

Factory functions that construct domain services with their required
dependencies (DB session, Redis, repositories):

.. list-table::
   :header-rows: 1

   * - Factory
     - Service
     - Dependencies Injected
   * - :func:`get_webhook_service`
     - :class:`WebhookService`
     - ``WebhookRepository``
   * - :func:`get_user_service`
     - :class:`UserService`
     - ``UserRepository``, ``WebhookService``
   * - :func:`get_session_service`
     - :class:`SessionService`
     - ``SessionRepository``, ``WebhookService``
   * - :func:`get_auth_service`
     - :class:`AuthService`
     - ``AuthRepository``, ``EmailService``, ``OtpService``, Redis
   * - :func:`get_fact_service`
     - :class:`FactService`
     - ``FactRepository``, ``SessionRepository``, Redis, ``WebhookService``
   * - :func:`get_memory_service`
     - :class:`MemoryService`
     - ``EpisodeRepository``, ``SessionRepository``, ``UserRepository``,
       ``FactRepository``, ``OrganizationRepository``, Redis, ``WebhookService``
   * - :func:`get_graph_service`
     - :class:`GraphService`
     - ``GraphBackendDispatcher`` (resolved per-org), ``UserRepository``,
       ``FactRepository``, ``WebhookService``
   * - :func:`get_auth_throttle`
     - :class:`AuthThrottle`
     - Redis, rate-limit settings

Org Config Dependency
=====================

.. module:: dependencies.org_config

:file:`dependencies/org_config.py`

.. code-block:: python

   async def get_org_config(
       request: Request,
       org_id: str = Depends(require_org_id),
   ) -> OrgConfigBase:

Fetches per-org configuration from Redis cache (fast path) or OpenBao KV
(authoritative slow path). Every field may be ``None`` — there is no env-var
fallback.

Email Dependency
================

.. module:: dependencies.email

:file:`dependencies/email.py`

.. list-table::
   :header-rows: 1

   * - Factory
     - Service
     - Description
   * - :func:`get_email_service`
     - :class:`EmailService`
     - Stateless, reads SMTP config from ``Settings`` singleton.
   * - :func:`get_otp_service`
     - :class:`OtpService`
     - Redis-backed OTP generation, ``EmailService`` for delivery.

.. _api-layer-schemas:

-------
Schemas
-------

.. module:: schemas

All Pydantic schemas live in :file:`schemas/` and follow strict separation:
no ORM imports, no business logic, no service imports.

.. list-table:: Schema Files
   :header-rows: 1

   * - File
     - Key Models
   * - :file:`auth.py`
     - ``SignupRequest``, ``LoginRequest``, ``TokenResponse``, ``LoginResponse``,
       ``DashboardUserResponse``, ``RefreshRequest``, ``MfaVerifyRequest``,
       ``MfaEnableRequest``, ``MfaDisableRequest``, ``UpdateProfileRequest``,
       ``VerifyEmailRequest``
   * - :file:`users.py`
     - ``UserResponse``, ``UserResponseWithStats``, ``UserListResponse``,
       ``CreateUserRequest``, ``UpdateUserRequest``
   * - :file:`sessions.py`
     - ``SessionResponse``, ``SessionListResponse``, ``CreateSessionRequest``,
       ``MessageResponse``
   * - :file:`memory.py`
     - ``IngestMemoryRequest``, ``IngestMemoryResponse``, ``Message``,
       ``DeleteMemoryResponse``
   * - :file:`context.py`
     - ``ContextResponse``
   * - :file:`facts.py`
     - ``FactResponse``, ``FactBatchRequest``, ``FactBatchResponse``
   * - :file:`graph.py`
     - ``GraphNode``, ``GraphNodeDetail``, ``GraphEdge``, ``GraphCommunity``,
       ``GraphNodesListResponse``, ``GraphEdgesListResponse``,
       ``GraphCommunitiesListResponse``, ``PaginatedGraphNodes``,
       ``PaginatedGraphEdges``
   * - :file:`projects.py`
     - ``ProjectResponse``, ``CreateProjectRequest``, ``UpdateProjectRequest``,
       ``ProjectMemberResponse``, ``AddMemberRequest``
   * - :file:`api_keys.py`
     - ``ApiKeyResponse``, ``ApiKeyCreatedResponse``, ``ApiKeyListResponse``,
       ``CreateApiKeyRequest``
   * - :file:`common.py`
     - ``PaginatedResponse`` (generic cursor pagination wrapper)
   * - :file:`organizations.py`
     - ``CreateOrgRequest``, ``CreateOrgResponse``
   * - :file:`organization_config.py`
     - ``OrgConfigBase``, ``OrgConfigResponse``, ``UpdateOrgConfigRequest``
   * - :file:`auth.py`` (schemas)
     - ``SignupRequest``, ``LoginRequest``, ``TokenResponse``, ``LoginResponse``
   * - :file:`audit_log.py`
     - ``AuditLogResponse``, ``AuditLogListResponse``, ``AuditLogFilter``
   * - :file:`webhook.py`
     - ``CreateWebhookRequest``, ``UpdateWebhookRequest``, ``WebhookSecretResponse``
   * - :file:`extraction_schemas.py`
     - ``ExtractionSchemaResponse``, ``ExtractionSchemaListResponse``,
       ``CreateExtractionSchemaRequest``, ``UpdateExtractionSchemaRequest``
   * - :file:`classifications.py`
     - ``ClassificationResponse``, ``ClassificationListResponse``
   * - :file:`structured_extractions.py`
     - ``StructuredExtractionResponse``, ``StructuredExtractionListResponse``
   * - :file:`custom_instructions.py`
     - ``CustomInstructionSchema``, ``CustomInstructionsResponse``,
       ``SetCustomInstructionsRequest``
   * - :file:`prompt_templates.py`
     - ``PromptTemplateDetail``, ``PromptTemplateSummary``,
       ``PromptTemplateListResponse``, ``PromptTemplateVersionsResponse``,
       ``SystemPromptGroupsResponse``, ``SetPromptTemplateRequest``,
       ``ImportPromptRequest``
   * - :file:`admin_metrics.py`
     - ``MetricsSummaryResponse``, ``EpisodeStats``, ``GraphStats``
   * - :file:`admin_stats.py`
     - ``OrgStatsResponse``, ``UsageStatsResponse``
   * - :file:`health.py`
     - (none — returns raw dict)
   * - :file:`llm_outputs.py`
     - LLM parsing schemas
   * - :file:`mappers.py`
     - Schema mapping utilities
   * - :file:`email.py`
     - ``OtpResponse``, ``SendOtpRequest``, ``VerifyOtpRequest``,
       ``ResetPasswordRequest``
   * - :file:`pii.py`
     - PII detection schemas
   * - :file:`observation.py`
     - Observation schemas
   * - :file:`user_summary.py`
     - ``UserSummaryResponse``, ``UserSummaryTriggerResponse``

.. _api-layer-repositories:

------------
Repositories
------------

.. module:: repositories

All repositories live in :file:`repositories/` and follow a consistent
interface pattern. Each accepts an :class:`AsyncSession` and implements one
query per method — no conditional branching that changes the query shape.

.. list-table:: Repository Files
   :header-rows: 1

   * - File
     - Key Methods
   * - :file:`user_repository.py`
     - ``get_by_id``, ``get_by_org_external_id``, ``list_by_org``, ``create``,
       ``update``, ``soft_delete``
   * - :file:`session_repository.py`
     - ``get_by_id``, ``get_by_external_id``, ``list_by_project``, ``create``,
       ``soft_delete``, ``get_messages``
   * - :file:`episode_repository.py`
     - ``create``, ``get_by_id_for_update``, ``apply_enrichment_bits``,
       ``list_by_session``, ``delete_by_project``
   * - :file:`fact_repository.py`
     - ``create``, ``list_by_session``, ``list_by_user``, ``soft_delete_by_project``,
       ``batch_insert``
   * - :file:`api_key_repository.py`
     - ``create``, ``get_by_lookup_hash``, ``list_by_org``, ``list_by_project``,
       ``revoke``, ``update_last_used``
   * - :file:`auth_repository.py`
     - ``create_user``, ``get_user_by_email``, ``verify_email``, ``update_password``,
       ``create_organization``
   * - :file:`project_repository.py`
     - ``create``, ``get_by_id``, ``list_by_org``, ``update``, ``archive``,
       ``add_member``, ``get_member``, ``list_members``, ``remove_member``,
       ``update_member_role``
   * - :file:`organization_repository.py`
     - ``create``, ``get_by_id``
   * - :file:`webhook_repository.py`
     - ``create``, ``get_by_id``, ``list_by_org``, ``update``, ``delete``
   * - :file:`audit_log_repository.py`
     - ``create``, ``query`` (filtered + paginated)
   * - :file:`extraction_schema_repository.py`
     - ``create``, ``get_by_id``, ``list_by_org``, ``update``, ``soft_delete``
   * - :file:`dialog_classification_repository.py`
     - ``create``, ``get_by_episode``, ``list_by_session``
   * - :file:`structured_extraction_repository.py`
     - ``create``, ``get_by_episode``, ``list_by_session``
   * - :file:`entity_repository.py`
     - Graph entity CRUD
   * - :file:`custom_instruction_repository.py`
     - ``get_by_scope``, ``set_by_scope``
   * - :file:`prompt_template_repository.py`
     - ``get_active``, ``list_names``, ``list_versions``, ``set_for_org``,
       ``import_system_template``, ``rollback``, ``delete_for_org``,
       ``set_as_type_default``, ``list_system_grouped``

.. _api-layer-core-infra:

-----------------------
Core Infrastructure
-----------------------

.. module:: core

Configuration
=============

.. module:: core.config

:file:`core/config.py`

Two settings classes:

- **:class:`BootstrapSettings`** — Pydantic ``BaseSettings`` read from
  environment variables only (``OZ_OPENBAO_ADDR``, ``OZ_OPENBAO_ROLE_ID``,
  ``OZ_OPENBAO_SECRET_ID``). No ``.env`` file fallback. Used to reach OpenBao
  for the first time.

- **:class:`Settings`** — Pydantic ``BaseModel`` populated from OpenBao KV
  (``system/`` namespace) at startup. Not available until
  :func:`init_settings` has been called. Singleton accessed via
  :func:`get_settings`.

Settings include: ``DATABASE_URL``, ``REDIS_URL``, ``SECRET_KEY``,
``WEBHOOK_SIGNING_SECRET``, JWT TTLs, CORS origins, FalkorDB config,
prompt caching toggles, SMTP config, rate-limit defaults, and environment
metadata.

Database
========

.. module:: core.db

:file:`core/db.py`

- :func:`init_db_engine` — creates ``AsyncEngine`` with ``postgresql+asyncpg://``
  validation, pool-pre-ping, pool-size=20, max-overflow=10, pool-recycle=3600s.
- :func:`get_async_session` — returns ``async_sessionmaker`` with
  ``expire_on_commit=False``.
- :func:`get_db` — FastAPI dependency yielding ``AsyncSession``.
- :func:`check_db_health` — ``SELECT 1`` ping.

Redis
=====

.. module:: core.redis

:file:`core/redis.py`

- :func:`init_redis` — creates ``redis.asyncio.Redis`` with connection pooling,
  ``decode_responses=True``, health-check interval 30s, max connections 50.
- :func:`get_redis` — FastAPI dependency.
- :func:`check_redis_health` — ``PING`` check.

Events (Webhook Registry)
=========================

.. module:: core.events

:file:`core/events.py`

Registry of all webhook event types, using :class:`EventType` string constants:

.. code-block:: python

   EventType.SESSION_CREATED      = "session.created"
   EventType.SESSION_CLOSED       = "session.closed"
   EventType.MESSAGE_ADDED        = "message.added"
   EventType.EPISODE_PROCESSED    = "episode.processed"
   EventType.INGEST_BATCH_COMPLETED  = "ingest.batch.completed"
   EventType.INGEST_EPISODE_COMPLETED = "ingest.episode.completed"
   EventType.GRAPH_ENTITY_CREATED = "graph.entity.created"
   EventType.GRAPH_ENTITY_UPDATED = "graph.entity.updated"
   EventType.GRAPH_EDGE_CREATED   = "graph.edge.created"
   EventType.FACT_EXTRACTED       = "fact.extracted"
   EventType.FACT_DELETED         = "fact.deleted"
   EventType.CLASSIFICATION_CREATED = "classification.created"
   EventType.EXTRACTION_CREATED   = "extraction.created"
   EventType.USER_CREATED         = "user.created"

Grouped into categories: ``Session``, ``Message``, ``Graph``, ``Fact``,
``Classification``, ``Extraction``, ``User``.

Middleware Order Recap
=====================

.. list-table:: Middleware Execution Order (Outermost → Innermost)
   :header-rows: 1

   * - # (Runtime)
     - Middleware
     - Responsibility
     - Registered (reverse)
   * - 0
     - :class:`MetricsMiddleware`
     - RED metrics (rate, errors, duration)
     - Last
   * - 1
     - :class:`CORSMiddleware`
     - CORS preflight
     - 2nd last
   * - 2
     - :class:`LoggingMiddleware`
     - Request/response structured logging
     - ...
   * - 3
     - :class:`TracingMiddleware`
     - OpenTelemetry spans
     - ...
   * - 4
     - :class:`RateLimitMiddleware`
     - Sliding-window rate limiting
     - ...
   * - 5
     - :class:`AuthMiddleware`
     - JWT / API key authentication
     - ...
   * - 6
     - :class:`AuditMiddleware`
     - Post-response audit log
     - ...
   * - 7
     - :class:`GZipMiddleware`
     - Response compression (>= 1 KB)
     - ...
   * - 8
     - :class:`TrustedHostMiddleware`
     - Host-header attack prevention
     - ...
   * - 9
     - :class:`RequestIDMiddleware`
     - ``X-Request-ID`` propagation
     - First

.. _api-layer-error-handling:

--------------
Error Handling
--------------

All API errors conform to `RFC 7807 <https://www.rfc-editor.org/rfc/rfc7807>`_
(Problem Details for HTTP APIs). The response body has this structure:

.. code-block:: json

   {
     "type": "https://errors.openzync.tech/{error_code}",
     "title": "Human Readable Title",
     "status": 422,
     "detail": "Detailed explanation of the error.",
     "instance": "/v1/projects/abc/sessions"
   }

Common error status codes:

- **400** — Bad request (Pydantic validation failure).
- **401** — Missing or invalid authentication.
- **403** — Authenticated but insufficient permissions.
- **404** — Resource not found.
- **409** — Resource conflict (e.g. duplicate ``external_id``).
- **413** — Payload too large (single message > 64KB).
- **422** — Business-rule validation failure.
- **429** — Rate limit exceeded.
- **502** — External service error (LLM, database, etc.).
- **503** — Infrastructure component unavailable.
- **504** — Graph database timeout.

Error handling follows the **zero-fallback discipline**: infrastructure
failures (cache, database, graph backend) always propagate as exceptions.
They are never silently swallowed.

.. code-block:: python

   # ✅ Correct — let it raise
   if redis is None:
       raise RuntimeError("Redis not configured")

   # ❌ Wrong — silent fallback hides a production incident
   if redis is None:
       logger.warning("Redis not configured, skipping cache")
       return result  # stale data, no alert raised

.. _api-layer-idempotency-design:

-----------------------
Idempotency Design Notes
-----------------------

The three-layer idempotency model was designed to prevent duplicate work at
every boundary: the HTTP boundary (client retries), the ingestion boundary
(content dedup), and the worker boundary (task isolation).

**Why three layers?**

.. list-table::
   :header-rows: 1

   * - Layer
     - Granularity
     - Scope
     - TTL
   * - HTTP (``Idempotency-Key``)
     - Per-request
     - A single HTTP request (identified by key)
     - 48h
   * - Content hash
     - Per-content
     - ``(org, user, session, messages)`` tuple
     - 48h
   * - Worker bitmask
     - Per-task
     - ``(episode, enrichment_task)`` tuple
     - Permanent (persisted in DB)

**Worker bitmask rationale**: Using an integer column with bitwise operations
is more efficient than a separate ``enrichment_tasks`` table with rows —
episodes are created at high volume and the bitmask avoids a join for every
worker check.

.. seealso::

   * :ref:`api-layer`
   * :doc:`/domains/memory`
   * :doc:`/domains/knowledge_graph`
   * :doc:`/domains/authentication`
   * :doc:`/architecture/index`
