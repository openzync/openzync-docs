Core Infrastructure
===================

.. note::

   This document covers the internal ``openzync-core/core/`` package **within
   the OpenZync monolith** — not a separately distributed ``openzync-core``
   library.  Code examples assume the package is importable as ``from core
   import ...`` from the monolith's Python path.

   The ``core/`` package provides shared infrastructure: configuration loading
   (exclusively from OpenBao), async database sessions, Redis connections, ARQ
   background workers, a domain exception hierarchy, structured logging, secrets
   management, cursor-based pagination, prompt template manifests, and per-org
   configuration resolution.

   **Design principle**: zero env-var fallback for runtime secrets.  If OpenBao
   is unreachable at startup, the process fails fast with a clear error.
   Infrastructure failures always propagate as HTTP 503 — no silent degradation.

.. contents:: Sections
   :local:
   :depth: 2
   :class: this-will-duplicate-information-and-it-is-still-useful-here


Configuration System
--------------------

Module: ``core.config``

Two configuration models live in this module.  :class:`BootstrapSettings` is a
Pydantic ``BaseSettings`` that reads *only* from real environment variables
(no ``.env`` file) to reach OpenBao for the first time.
:class:`Settings` is a Pydantic ``BaseModel`` loaded from OpenBao KV at
startup via :func:`init_settings` (re-exported from
:mod:`core.openbao_settings`).  There is **no** module-level instantiation at
import time — the singleton is populated by :func:`init_settings` and accessed
via :func:`get_settings` or the backward-compatible ``from core.config import
settings`` pattern (resolved via ``__getattr__``).


BootstrapSettings
~~~~~~~~~~~~~~~~~

.. class:: BootstrapSettings()

   Minimal settings needed to reach OpenBao for the first time.  These values
   are read from **actual environment variables only** — there is no ``.env``
   file fallback.  They are never stored in OpenBao itself; they bootstrap the
   connection.

   In production, inject these via Docker environment variables, Kubernetes
   Secrets, or your infrastructure's secrets manager.

   The model is frozen (``frozen=True``) and ignores extra fields.

   .. attribute:: OPENBAO_ADDR

      :type: str
      :default: ``"http://localhost:8200"``

      OpenBao server URL.  Read from ``OZ_OPENBAO_ADDR``
      (``validation_alias``).

   .. attribute:: OPENBAO_ROLE_ID

      :type: str

      AppRole RoleID for OpenBao authentication.  Read from
      ``OZ_OPENBAO_ROLE_ID``.

   .. attribute:: OPENBAO_SECRET_ID

      :type: str

      AppRole SecretID for OpenBao authentication.  Read from
      ``OZ_OPENBAO_SECRET_ID``.

   .. attribute:: OPENBAO_WORKER_ROLE_ID

      :type: str | None
      :default: ``None``

      Optional worker-specific AppRole RoleID.  Read from
      ``OZ_OPENBAO_WORKER_ROLE_ID``.

   .. attribute:: OPENBAO_WORKER_SECRET_ID

      :type: str | None
      :default: ``None``

      Optional worker-specific AppRole SecretID.  Read from
      ``OZ_OPENBAO_WORKER_SECRET_ID``.

   Usage::

       from core.config import BootstrapSettings

       bootstrap = BootstrapSettings()
       # bootstrap.OPENBAO_ADDR == "http://vault.example.com:8200"
       # (set via env var OZ_OPENBAO_ADDR)


Settings
~~~~~~~~

.. class:: Settings()

   Single source of truth for all OpenZync system configuration.  Values are
   loaded from OpenBao KV (``system/`` namespace) at startup via
   :func:`init_settings`.  An instance is created **once** and exposed through
   :func:`get_settings`.  Do **not** instantiate ``Settings`` manually.

   Secrets (``DATABASE_URL``, ``REDIS_URL``, ``SECRET_KEY``,
   ``WEBHOOK_SIGNING_SECRET``) have **no defaults** — they must be present in
   OpenBao.  Non-sensitive tunables have sensible defaults that can be
   overridden in OpenBao.

   Per-org configuration (LLM, embeddings, graph, behaviour) is **not** stored
   here — use :func:`core.org_config.get_org_config` for those values.

   .. rubric:: Database

   .. attribute:: DATABASE_URL

      :type: str
      :required: True

      PostgreSQL connection string used by SQLAlchemy async engine.
      Must use the ``postgresql+asyncpg://`` scheme.

   .. rubric:: Redis / Caching

   .. attribute:: REDIS_URL

      :type: str
      :required: True

      Redis connection string for caching, pub/sub, and RQ/ARQ.

   .. rubric:: Secrets

   .. attribute:: SECRET_KEY

      :type: str
      :required: True
      :min_length: 32

      Secret key used for signing JWTs and other cryptographic operations.
      Must be at least 32 characters in production.

   .. attribute:: WEBHOOK_SIGNING_SECRET

      :type: str
      :required: True
      :min_length: 32

      Secret key for HMAC-SHA256 webhook signing.  Consumers use this to
      verify webhook authenticity.

   .. rubric:: Metrics / Observability

   .. attribute:: PROMETHEUS_URL

      :type: str
      :default: ``"http://localhost:9090"``

      Prometheus server URL.  Used by the admin ``/metrics/summary``
      endpoint.

   .. rubric:: HTTP / Server

   .. attribute:: CORS_ORIGINS

      :type: str
      :default: ``"http://localhost:3000"``

      Comma-separated list of allowed CORS origins.

   .. attribute:: HOSTS_ALLOWED

      :type: str
      :default: ``"localhost:8000"``

      Comma-separated list of allowed Host header values for
      ``TrustedHostMiddleware`` in production
      (e.g. ``"api.openzync.tech,localhost:3000"``).  Accepts ``"*"`` in
      development.

   .. rubric:: Environment & Observability

   .. attribute:: ENVIRONMENT

      :type: str
      :default: ``"development"``

      Deployment environment.  Controls logging format, etc.

   .. attribute:: LOG_LEVEL

      :type: str
      :default: ``"INFO"``

      Minimum log level (``DEBUG``, ``INFO``, ``WARNING``, ``ERROR``,
      ``CRITICAL``).

   .. rubric:: Concurrency

   .. attribute:: MAX_WORKERS

      :type: int
      :default: ``4``
      :constraints: ``1 <= value <= 64``

      Maximum number of worker threads/processes.

   .. rubric:: JWT

   .. attribute:: JWT_ACCESS_TOKEN_TTL_MINUTES

      :type: int
      :default: ``30``
      :constraints: ``1 <= value <= 1440``

      Access token TTL in minutes.

   .. attribute:: JWT_REFRESH_TOKEN_TTL_DAYS

      :type: int
      :default: ``7``
      :constraints: ``1 <= value <= 90``

      Refresh token TTL in days.

   .. rubric:: FalkorDB (graph backend)

   .. attribute:: FALKORDB_URL

      :type: str
      :default: ``"redis://localhost:6379"``

      FalkorDB connection URL (Redis RESP protocol).

   .. attribute:: FALKORDB_MAX_CONNECTIONS

      :type: int
      :default: ``20``
      :constraints: ``1 <= value <= 100``

      Max connections in the FalkorDB connection pool.

   .. attribute:: FALKORDB_SOCKET_TIMEOUT

      :type: int
      :default: ``30``
      :constraints: ``>= 1``

      Socket timeout in seconds for FalkorDB connections.

   .. rubric:: Prompt Caching

   .. attribute:: PROMPT_CACHING_ENABLED

      :type: bool
      :default: ``True``

      Master switch for provider-side prompt caching.  When ``True``,
      Anthropic gets ``cache_control`` markers on system prompts,
      OpenAI/Azure get automatic prefix caching, and OpenRouter gets
      ``session_id`` for sticky routing.  Read from ``OZ_PROMPT_CACHING_ENABLED``.

   .. attribute:: PROMPT_CACHING_ANTHROPIC_MIN_TOKENS

      :type: int
      :default: ``1024``
      :constraints: ``>= 512``

      Minimum estimated token count for the system prompt before
      ``cache_control`` is applied to Anthropic calls.  Approximated as
      ``len(text) // 4``.  Read from ``OZ_PROMPT_CACHING_ANTHROPIC_MIN_TOKENS``.

   .. attribute:: PROMPT_CACHING_ANTHROPIC_TTL

      :type: str
      :default: ``"5m"``

      Anthropic cache TTL: ``"5m"`` (1.25x write cost) or ``"1h"`` (2x write
      cost).  Cache reads are always 0.1x.  Invalid values fall back to
      ``"5m"`` via a ``@field_validator``.  Read from
      ``OZ_PROMPT_CACHING_ANTHROPIC_TTL``.

   .. rubric:: Email / SMTP

   .. attribute:: SMTP_HOST

      :type: str
      :default: ``"localhost"``

      SMTP server hostname for sending transactional emails.

   .. attribute:: SMTP_PORT

      :type: int
      :default: ``587``
      :constraints: ``1 <= value <= 65535``

      SMTP server port.

   .. attribute:: SMTP_USERNAME

      :type: str
      :default: ``""``

      SMTP username (empty string = no auth).

   .. attribute:: SMTP_PASSWORD

      :type: str
      :default: ``""``

      SMTP password (empty string = no auth).

   .. attribute:: SMTP_FROM_ADDR

      :type: str
      :default: ``"noreply@openzync.tech"``

      ``From:`` address for outgoing emails.

   .. attribute:: SMTP_USE_TLS

      :type: bool
      :default: ``True``

      Use implicit TLS (SMTPS) on connect.

   .. attribute:: SMTP_START_TLS

      :type: bool
      :default: ``True``

      Use STARTTLS to upgrade to TLS after connect.

   .. rubric:: Rate Limiting

   .. attribute:: RATE_LIMIT_IP_MAX

      :type: int
      :default: ``10``
      :constraints: ``>= 1``

      Max requests per IP within the rate-limit window.

   .. attribute:: RATE_LIMIT_WINDOW_SEC

      :type: int
      :default: ``60``
      :constraints: ``>= 1``

      Rate-limit window in seconds.

   .. rubric:: Validators

   .. method:: _validate_cache_ttl(v)

      :classmethod: ``@field_validator("PROMPT_CACHING_ANTHROPIC_TTL", mode="before")``
      :type v: str
      :rtype: str

      Validate cache TTL, falling back to ``"5m"`` on invalid input.  OpenBao
      KV has no schema enforcement — a manual ``bao kv put`` could write any
      value.  Rather than crashing on a Pydantic ``ValidationError``, we
      silently fall back.


Singleton accessors
~~~~~~~~~~~~~~~~~~~

.. function:: set_settings(settings)

   :param settings: A fully-populated :class:`Settings` instance from OpenBao.
   :type settings: Settings

   Store the :class:`Settings` singleton (called once by
   :func:`init_settings`).

.. function:: get_settings()

   :rtype: Settings
   :raises RuntimeError: If :func:`init_settings` has not been called yet.

   Return the initialised :class:`Settings` singleton.

.. function:: __getattr__(name)

   :param name: The attribute name being looked up.
   :type name: str
   :rtype: Any
   :raises AttributeError: If the name is not recognised.

   Resolve ``settings`` lazily through the singleton accessor.  Enables the
   backward-compatible ``from core.config import settings`` pattern without
   instantiation at import time.

.. function:: init_settings(client)

   (Re-exported from :mod:`core.openbao_settings` — see
   :ref:`openbao-settings-label` for full documentation.)


Real-world usage example::

    from contextlib import asynccontextmanager
    from fastapi import FastAPI
    from core.openbao import OpenBaoClient
    from core.config import BootstrapSettings, get_settings, init_settings

    @asynccontextmanager
    async def lifespan(app: FastAPI):
        bootstrap = BootstrapSettings()                        # env vars only
        async with OpenBaoClient(
            bootstrap.OPENBAO_ADDR,
            bootstrap.OPENBAO_ROLE_ID,
            bootstrap.OPENBAO_SECRET_ID,
        ) as bao:
            await init_settings(bao)                           # populates singleton

        settings = get_settings()
        app.state.db_url = settings.DATABASE_URL               # "postgresql+asyncpg://..."
        app.state.secret_key = settings.SECRET_KEY             # min 32 chars
        yield

    app = FastAPI(lifespan=lifespan)


Database Layer
--------------

Module: ``core.db``

Provides async SQLAlchemy engine initialisation, session factory creation, a
FastAPI dependency for request-scoped sessions, and a health-check helper.

.. function:: init_db_engine(database_url, **kwargs)

   :param str database_url: PostgreSQL connection string.  **Must** use the
       ``postgresql+asyncpg://`` scheme.
   :param \\**kwargs: Additional engine arguments (override the defaults below).
   :type kwargs: Any
   :rtype: AsyncEngine
   :raises ValueError: If ``database_url`` does not use the asyncpg driver.

   Default engine arguments::

       pool_pre_ping=True
       pool_size=20
       max_overflow=10
       pool_recycle=3600
       echo=False
       connect_args={"statement_cache_size": 0}

   Usage::

       engine = init_db_engine(str(settings.DATABASE_URL))

.. function:: close_db_engine(engine)

   :param AsyncEngine engine: The async engine to shut down.
   :rtype: None

   Dispose of the engine and all connections in its pool::

       await close_db_engine(engine)

.. function:: get_async_session(engine)

   :param AsyncEngine engine: An initialised engine.
   :rtype: async_sessionmaker[AsyncSession]

   Create a session factory bound to the given engine with
   ``expire_on_commit=False`` (required to avoid lazy-load errors in async
   context)::

       session_factory = get_async_session(engine)
       app.state.db_session_factory = session_factory

.. function:: get_db(request)

   :param Request request: FastAPI request.
   :type request: fastapi.Request
   :yields: :class:`AsyncSession` from the application's engine.
   :raises RuntimeError: If ``db_session_factory`` was not initialised on
       ``app.state``.

   FastAPI dependency that yields an :class:`AsyncSession`.  The session is
   read from the session factory attached to
   ``request.app.state.db_session_factory`` during the application lifespan::

       @router.get("/items")
       async def list_items(db: AsyncSession = Depends(get_db)):
           ...

.. function:: check_db_health(engine)

   :param AsyncEngine engine: The application's engine.
   :rtype: bool

   Check database connectivity by running ``SELECT 1``.  Returns ``True`` if
   the database is reachable, ``False`` otherwise::

       healthy = await check_db_health(engine)

Real-world usage — FastAPI lifespan::

    from contextlib import asynccontextmanager
    from fastapi import FastAPI
    from core.config import get_settings
    from core.db import init_db_engine, close_db_engine, get_async_session

    @asynccontextmanager
    async def lifespan(app: FastAPI):
        engine = init_db_engine(str(get_settings().DATABASE_URL))
        app.state.db_engine = engine
        app.state.db_session_factory = get_async_session(engine)
        yield
        await close_db_engine(engine)

    app = FastAPI(lifespan=lifespan)

Router dependency::

    from fastapi import Depends
    from sqlalchemy.ext.asyncio import AsyncSession
    from core.db import get_db

    @router.get("/items")
    async def list_items(db: AsyncSession = Depends(get_db)):
        result = await db.execute(select(Item))
        return result.scalars().all()


Redis Connection
----------------

Module: ``core.redis``

Provides async Redis client initialisation with connection pooling, a FastAPI
dependency, and a health-check helper.

.. function:: init_redis(redis_url)

   :param str redis_url: Redis connection string (e.g. ``redis://localhost:6379/0``).
   :rtype: redis.asyncio.Redis
   :raises ValueError: If the URL scheme is not supported.

   Create and return an async Redis client with connection pooling.
   Default pool settings::

       encoding="utf-8"
       decode_responses=True
       socket_connect_timeout=5
       socket_timeout=10
       retry_on_timeout=True
       health_check_interval=30
       max_connections=50

   Usage::

       redis_client = init_redis(str(settings.REDIS_URL))

.. function:: close_redis(client)

   :param redis.asyncio.Redis client: The async Redis client to shut down.
   :rtype: None

   Gracefully close the Redis connection and its pool::

       await close_redis(redis_client)

.. function:: get_redis(request)

   :param Request request: FastAPI request.
   :type request: fastapi.Request
   :rtype: redis.asyncio.Redis
   :raises RuntimeError: If the client is not available on ``app.state``.

   FastAPI dependency that returns an async Redis client.  The client is read
    from ``request.app.state.redis``::

        import redis.asyncio as aioredis

        @router.get("/cache")
        async def get_cache(redis: aioredis.Redis = Depends(get_redis)):
           value = await redis.get("my-key")

.. function:: check_redis_health(client)

   :param redis.asyncio.Redis client: An async Redis client.
   :rtype: bool

   Check whether the Redis server is reachable via ``PING``.  Returns
   ``True`` if ``PONG`` was received, ``False`` otherwise::

       healthy = await check_redis_health(redis_client)

Real-world usage — FastAPI lifespan::

    from contextlib import asynccontextmanager
    from fastapi import FastAPI
    from core.config import get_settings
    from core.redis import init_redis, close_redis

    @asynccontextmanager
    async def lifespan(app: FastAPI):
        redis_client = init_redis(str(get_settings().REDIS_URL))
        app.state.redis = redis_client
        yield
        await close_redis(redis_client)

    app = FastAPI(lifespan=lifespan)


ARQ Worker Integration
----------------------

Module: ``core.arq``

Provides a module-level :class:`ARQPool` singleton for Async Redis Queue
backround job management, compatible with FastAPI's lifespan.

.. class:: ARQPool(redis_url=None)

   :param redis_url: Redis connection string.  Falls back to
       ``settings.REDIS_URL``.
   :type redis_url: str | None

   Manages the ARQ worker connection pool lifecycle.  Wraps
   ``arq.create_pool`` and provides a convenience :meth:`enqueue` method.

   .. note::

      ``ARQPool.__init__`` does **not** connect.  Call :meth:`initialize`
      explicitly (typically from FastAPI's lifespan startup).

   .. method:: initialize()

      :rtype: None
      :raises ConnectionError: If the Redis server is unreachable.

      Create the ARQ connection pool.  Intended to be called once during
      application startup::

          await pool.initialize()

   .. method:: close()

      :rtype: None

      Close the connection pool gracefully.  Safe to call multiple times::

          await pool.close()

   .. attribute:: pool

      :type: Any (``arq.connections.ArqRedis``)

      Access the underlying ARQ pool.  Raises :class:`RuntimeError` if
      :meth:`initialize` has not been called::

          raw_pool = arq_pool.pool

   .. method:: enqueue(task_name, queue_name=None, **kwargs)

      :param str task_name: Name of the registered worker function.
      :param queue_name: Optional queue name (e.g. ``"high"`` or ``"low"``).
          Passed as ``_queue`` to ARQ's ``enqueue_job``.
      :type queue_name: str | None
      :param \\**kwargs: Keyword arguments forwarded to the worker function.
      :rtype: str | None

      Enqueue a background job.  Returns the enqueued job ID, or ``None`` if
      the pool is not available::

          job_id = await arq_pool.enqueue("send_notification", user_id=42)

Module-level singleton helpers:

.. function:: init_arq(redis_url=None)

   :param redis_url: Redis connection string (defaults to ``settings.REDIS_URL``).
   :type redis_url: str | None
   :rtype: ARQPool
   :raises ConnectionError: If Redis is unreachable.

   Initialise the global ARQ pool singleton.  Intended to be called from
   FastAPI's ``lifespan`` startup context.  If a pool already exists, it is
   closed before re-initialisation::

       arq_pool = await init_arq()

.. function:: close_arq()

   :rtype: None

   Shut down the global ARQ pool singleton.  Safe to call multiple times::

       await close_arq()

.. function:: get_arq()

   :rtype: ARQPool
   :raises RuntimeError: If :func:`init_arq()` has not been called.

   Retrieve the global ARQ pool singleton::

       pool = get_arq()
       job_id = await pool.enqueue("process_file", file_id=42)

.. data:: Job

   Re-exported from ``arq.jobs`` for convenience::

       from core.arq import Job

   This is the ARQ :class:`arq.jobs.Job` class, useful for status queries::

       job = await Job(job_id, _pool)
       status = await job.status()

Real-world usage — FastAPI lifespan::

    from contextlib import asynccontextmanager
    from fastapi import FastAPI
    from core.arq import init_arq, close_arq

    @asynccontextmanager
    async def lifespan(app: FastAPI):
        app.state.arq_pool = await init_arq()
        yield
        await close_arq()

    app = FastAPI(lifespan=lifespan)

Inside a router::

    from core.arq import get_arq

    @router.post("/notify")
    async def send_notification(user_id: int):
        job_id = await get_arq().enqueue("send_email", user_id=user_id)
        return {"job_id": job_id}


Exception Hierarchy
-------------------

Module: ``core.exceptions``

Every custom exception inherits from :class:`AppError` and carries:

* ``status_code`` — HTTP status code.
* ``code`` — machine-readable error-code string.
* ``message`` — human-readable description.
* ``detail`` — optional dict for additional context.

Design philosophy: **zero fallback** — infrastructure errors always propagate
as 503.  Never silently swallow an exception to serve stale or degraded data.

Exception classes
~~~~~~~~~~~~~~~~~

.. class:: AppError(message="An unexpected error occurred.", detail=None)

   Base exception for all OpenZync application errors.  Subclass this to
   create domain-specific errors; every subclass **must** set ``status_code``
   and ``code``.

   :param str message: Human-readable error description.
   :param detail: Optional dict for additional context.
   :type detail: dict[str, Any] | None

   .. attribute:: status_code

      :type: int
      :default: ``500``

      HTTP status code returned to the client.

   .. attribute:: code

      :type: str
      :default: ``"internal_error"``

      Machine-readable error-code string.

.. class:: NotFoundError(message="The requested resource was not found.", detail=None)

   :status_code: 404
   :code: ``"not_found"``

   Requested resource does not exist.

.. class:: ValidationError(message="The request payload is invalid.", detail=None)

   :status_code: 422
   :code: ``"validation_error"``

   Request payload failed business-rule validation.

.. class:: AuthenticationError(message="Authentication is required.", detail=None)

   :status_code: 401
   :code: ``"authentication_error"``

   Missing or invalid authentication credentials.

.. class:: AuthorizationError(message="You do not have permission to perform this action.", detail=None)

   :status_code: 403
   :code: ``"authorization_error"``

   Authenticated but insufficient permissions.

.. class:: ConflictError(message="The request conflicts with the current state of the resource.", detail=None)

   :status_code: 409
   :code: ``"conflict"``

   Resource already exists or is in a conflicting state.

.. class:: RateLimitError(message="Too many requests.  Please slow down.", detail=None)

   :status_code: 429
   :code: ``"rate_limit_exceeded"``

   Client exceeded rate-limit allowance.

.. class:: InsufficientCreditsError(message="Insufficient credits to complete this request.", detail=None)

   :status_code: 402
   :code: ``"insufficient_credits"``

   Account balance too low to perform the requested operation.

.. class:: ExternalServiceError(message="An external service error occurred.", detail=None)

   :status_code: 502
   :code: ``"external_service_error"``

   External dependency (LLM, DB, S3, etc.) returned an error or timed out.

.. class:: LLMConfigurationError(message="No LLM backend configured.", detail=None)

   :status_code: 502
   :code: ``"llm_configuration_error"``

   LLM backend cannot be resolved due to missing or invalid configuration.

.. class:: PayloadTooLargeError(message="The request body is too large.", detail=None)

   :status_code: 413
   :code: ``"payload_too_large"``

   Request body exceeds the maximum allowed size.

.. class:: EntityNotFoundError(message="The requested entity was not found in the knowledge graph.", detail=None)

   :status_code: 404
   :code: ``"entity_not_found"``

   Requested graph entity node does not exist.

.. class:: EdgeNotFoundError(message="The requested edge was not found in the knowledge graph.", detail=None)

   :status_code: 404
   :code: ``"edge_not_found"``

   Requested graph edge does not exist.

.. class:: EpisodeNotFoundError(message="The requested episode was not found.", detail=None)

   :status_code: 404
   :code: ``"episode_not_found"``

   Requested episode does not exist.  Raised by ARQ workers when an episode is
   not yet visible due to transaction visibility races; the ``@with_retry``
   decorator re-raises this so the worker can retry.

.. class:: GraphTimeoutError(message="The graph database operation timed out.", detail=None)

   :status_code: 504
   :code: ``"graph_timeout"``

   Graph database operation exceeded the configured timeout.

.. class:: LLMStructuredOutputError(message="LLM output failed to match the expected schema.", \*, model_name="", content_preview="", validation_error="")

   Extends :class:`ExternalServiceError`.

   :status_code: 502 (inherited)
   :code: ``"llm_structured_output_error"``

   LLM output failed validation against the expected Pydantic model.  Raised
   when ``LLMBackend.chat()`` is called with a ``response_model`` and the
   response cannot be parsed after exhausting all validation retries.

Infrastructure failures (all inherit from :class:`ServiceUnavailableError`,
status 503):

.. class:: ServiceUnavailableError(message="A service dependency is unavailable.", detail=None)

   :status_code: 503
   :code: ``"service_unavailable"``

   A shared infrastructure component is unavailable.  Never silently
   swallowed — propagates as HTTP 503 so load-balancers and orchestrators
   react appropriately.

.. class:: CacheUnavailableError

   :code: ``"cache_unavailable"``

   Cache service (Redis/Memcached) cannot be reached.

.. class:: GraphBackendUnavailableError

   :code: ``"graph_backend_unavailable"``

   Graph database backend cannot be reached.

.. class:: RateLimitUnavailableError

   :code: ``"rate_limit_unavailable"``

   Rate-limiting infrastructure cannot be reached.

.. class:: MetricsUnavailableError

   :code: ``"metrics_unavailable"``

   Metrics collection backend cannot be reached.

.. class:: DatabaseUnavailableError

   :code: ``"database_unavailable"``

   Primary or replica database cannot be reached.

.. class:: SearchLegFailedError(leg_name, message=None, original_error="")

   :code: ``"search_leg_failed"``

   A single search retrieval leg (vector, keyword, graph, etc.) failed.
   Carries the leg name and the original error detail so callers can decide
   whether to fail the entire multi-leg search or proceed with degraded
   results (the default is to fail — **zero fallback**).


Exception handler registration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: register_exception_handlers(app)

   :param FastAPI app: The FastAPI application instance.

   Register exception handlers for the complete :class:`AppError` hierarchy.
   Every mapped exception returns a response body conforming to **RFC 7807**
   (Problem Details for HTTP APIs)::

       {
         "type": "https://errors.openzync.tech/not_found",
         "title": "Not Found",
         "status": 404,
         "detail": "The requested resource was not found.",
         "instance": "/api/v1/sessions/abc-123"
       }

   Usage — called once during app creation::

       from core.exceptions import register_exception_handlers

       app = FastAPI()
       register_exception_handlers(app)

   .. note::

      The registration uses closures to correctly dispatch subclass
      exceptions.  A catch-all :class:`AppError` handler is also registered
      so that any future subclasses are handled without code changes.


Real-world usage — raising in service code::

    from core.exceptions import (
        NotFoundError,
        InsufficientCreditsError,
        ExternalServiceError,
    )

    async def process_session(session_id: str, user_id: str) -> Session:
        session = await repo.get_by_id(session_id)
        if session is None:
            raise NotFoundError(f"Session {session_id} not found")
        if session.user_credits <= 0:
            raise InsufficientCreditsError(
                detail={"session_id": session_id, "credits": session.user_credits},
            )
        try:
            result = await llm_client.chat(...)
        except ConnectionError as e:
            raise ExternalServiceError(
                "LLM service unreachable",
                detail={"original_error": str(e)},
            ) from e
        return result


Structured Logging
------------------

Module: ``core.logging``

Structured logging configuration built on ``structlog``.  Provides PII
redaction, request-context binding via ``contextvars``, and automatic
standard-library logging integration.

.. function:: setup_logging(environment, log_level)

   :param str environment: One of ``"development"``, ``"staging"``,
       ``"production"``.  Controls the output renderer.
   :param str log_level: Minimum log level string (e.g. ``"INFO"``,
       ``"DEBUG"``).

   Configure structlog and standard-library logging once at startup::

       from core.logging import setup_logging
       from core.config import get_settings

       setup_logging(
           environment=get_settings().ENVIRONMENT,
           log_level=get_settings().LOG_LEVEL,
       )

   * In **production/staging**: structured JSON output via ``JSONRenderer``.
   * In **development**: coloured human-readable output via ``ConsoleRenderer``.

   Standard-library logging is routed through structlog via
   ``structlog.stdlib.LoggerFactory`` and ``logging.basicConfig`` so that
   third-party libraries (uvicorn, SQLAlchemy, httpx, etc.) also produce
   structured output.  ``logging.captureWarnings(True)`` is called to
   capture Python warnings.

.. function:: add_pii_redaction(logger, method_name, event_dict)

   :param structlog.types.WrappedLogger logger: The wrapped logger instance
       (unused).
   :param str method_name: The log method called (unused).
   :param structlog.types.EventDict event_dict: The mutable event dictionary.
   :rtype: structlog.types.EventDict

   Redact sensitive fields from log events before rendering.  Any key whose
   name contains a recognised sensitive fragment (``"key"``, ``"secret"``,
   ``"password"``, ``"token"``, ``"auth"``, ``"authorization"``,
   ``"api_key"``, ``"api_key_name"``, ``"access_token"``,
   ``"refresh_token"``, ``"client_secret"``, ``"private_key"``) will have
   its value replaced with ``"***REDACTED***"``.

   Registered as a processor — callers do not invoke this directly.

.. function:: bind_request_context(request_id, org_id=None, user_id=None)

   :param str request_id: Unique identifier for the current request.
   :param org_id: Organisation identifier (optional).
   :type org_id: str | None
   :param user_id: Authenticated user identifier (optional).
   :type user_id: str | None

   Bind global request-scoped context variables.  These values are
   automatically included in every log entry emitted during the request::

       from core.logging import bind_request_context

       bind_request_context(
           request_id="abc-123",
           org_id="org-42",
           user_id="usr-7",
       )

   Subsequent log entries will include ``request_id``, ``org_id``, and
   ``user_id`` automatically.

Processors applied in order:

1. ``structlog.processors.filter_by_level`` — drops log entries below the configured minimum level before any processing.
2. ``structlog.contextvars.merge_contextvars``
3. :func:`add_pii_redaction`
4. :func:`_add_context_from_vars` (internal — injects contextvars)
5. ``structlog.stdlib.add_log_level``
6. ``structlog.stdlib.add_logger_name``
7. ``structlog.processors.TimeStamper(fmt="iso", utc=True)``
8. ``structlog.processors.StackInfoRenderer``
9. ``structlog.dev.set_exc_info``
10. ``structlog.processors.format_exc_info``
11. ``structlog.processors.UnicodeDecoder``
12. ``structlog.stdlib.PositionalArgumentsFormatter`` — formats positional ``%s``-style arguments passed to log calls.
13. ``JSONRenderer`` or ``ConsoleRenderer`` (based on environment)

Real-world usage — middleware::

    from starlette.middleware.base import BaseHTTPMiddleware
    from core.logging import bind_request_context
    import uuid

    class RequestLoggingMiddleware(BaseHTTPMiddleware):
        async def dispatch(self, request, call_next):
            request_id = str(uuid.uuid4())
            bind_request_context(
                request_id=request_id,
                user_id=getattr(request.state, "user_id", None),
                org_id=getattr(request.state, "org_id", None),
            )
            response = await call_next(request)
            response.headers["X-Request-ID"] = request_id
            return response

    import structlog
    logger = structlog.get_logger(__name__)
    logger.info("request.started", path="/api/v1/sessions")
    # → {"event": "request.started", "path": "/api/v1/sessions",
    #    "request_id": "abc-123", "org_id": "org-42", "user_id": "usr-7",
    #    "level": "info", "timestamp": "2026-07-15T10:30:00Z"}


OpenBao Secrets Integration
---------------------------

OpenBao (a Vault-compatible secrets manager) is the **sole source of truth**
for all runtime configuration and secrets.  There is no env-var fallback, no
``.env`` file, and no local defaults for sensitive values.  The integration
spans four modules:

* :mod:`core.openbao_exceptions` — typed exception hierarchy.
* :mod:`core.openbao` — async HTTP client with AppRole auth, KV v2, namespace
  management, and Transit encryption.
* :mod:`core.openbao_settings` — loads ``Settings`` from OpenBao system
  namespace.
* :mod:`core.transit` — high-level encryption manager wrapping the Transit
  engine.

.. note::

   The directory ``core/secret_store/`` is **planned** but not yet implemented.
   It will hold a future abstraction layer that caches frequently
   accessed secrets locally.  At present all secrets are fetched directly
   from OpenBao on every read.

.. _openbao-exceptions-label:

OpenBao Exception Hierarchy
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Module: ``core.openbao_exceptions``

.. class:: OpenBaoError(message="", \*, status_code=None)

   :param str message: Human-readable error description.
   :param status_code: Optional HTTP status code from the OpenBao API.
   :type status_code: int | None

   Base exception for all OpenBao-related errors.

.. class:: OpenBaoConnectionError

   OpenBao is unreachable, or a network-level error occurred.  Raised when:

   * The OpenBao server is down or unreachable.
   * The REST API returns a 5xx status.
   * A request times out.

.. class:: OpenBaoAuthError

   Authentication or authorization failure.  Raised when:

   * AppRole login fails (wrong ``role_id`` / ``secret_id``).
   * The client token is expired or invalid.
   * The token lacks sufficient ACL permissions (HTTP 403).

.. class:: OpenBaoSecretNotFoundError

   The requested secret path does not exist.  Raised when:

   * A KV read targets a key that does not exist (HTTP 404).
   * A KV list targets an empty or non-existent prefix.

.. class:: OpenBaoNamespaceError

   Namespace operation failed.  Raised when:

   * Namespace creation fails due to naming conflicts.
   * Operations within a namespace fail (HTTP 412).

.. class:: OpenBaoRateLimitError

   OpenBao returned HTTP 429 — too many requests.  Callers should retry with
   exponential backoff.

.. _openbao-client-label:

OpenBao Client
~~~~~~~~~~~~~~

Module: ``core.openbao``

.. class:: OpenBaoClient(addr, role_id, secret_id, \*, timeout=10.0)

   :param str addr: OpenBao server URL (e.g. ``http://localhost:8200``).
   :param str role_id: AppRole RoleID for authentication.
   :param str secret_id: AppRole SecretID for authentication.
   :param timeout: HTTP request timeout in seconds (default 10).
   :type timeout: float

   Async HTTP client for the OpenBao secrets-management API.  Authenticates
   via AppRole and exposes methods for KV v2 operations, namespace management,
   system- and org-level config read/write, and Transit engine operations.

   **Must** be used as an async context manager::

       async with OpenBaoClient(addr, role_id, secret_id) as bao:
           data = await bao.read_system_config()

   .. rubric:: Authentication (internal)

   .. method:: _authenticate()

      POST ``/v1/auth/approle/login`` with ``role_id`` and ``secret_id``
      and stores the returned ``client_token`` for all subsequent requests.
      Token expiry is tracked via monotonic time; automatic re-authentication
      occurs at 80% of TTL.

   .. rubric:: KV v2 Operations (internal)

   .. method:: _kv_read(path, namespace=None, \*, include_meta=False)

      :param str path: Full KV path (e.g. ``config/data/database_url``).
      :param namespace: Optional namespace.
      :type namespace: str | None
      :param bool include_meta: If ``True``, also return the version number
          as a second element for CAS-aware writes.
      :rtype: dict[str, Any] | tuple[dict[str, Any], int]

      Read a secret from a KV v2 engine.

   .. method:: _kv_write(path, data, namespace=None, \*, cas_version=None)

      :param str path: Full KV path (e.g. ``config/data/database_url``).
      :param dict[str, Any] data: The secret data to persist.
      :param namespace: Optional namespace.
      :type namespace: str | None
      :param cas_version: If set, the write only succeeds if the current
          version matches (Compare-And-Swap).
      :type cas_version: int | None

      Write a secret to a KV v2 engine.

   .. method:: _kv_list(path, namespace=None)

      :param str path: Full KV metadata path (e.g. ``config/metadata/``).
      :param namespace: Optional namespace.
      :type namespace: str | None
      :rtype: list[str]

      List secret keys at a KV v2 metadata path.

   .. method:: _kv_delete(path, namespace=None)

      :param str path: Full KV path (e.g. ``config/data/database_url``).
      :param namespace: Optional namespace.
      :type namespace: str | None

      Delete a secret from a KV v2 engine.

   .. rubric:: Namespace Management

   .. method:: create_namespace(name)

      :param str name: Namespace name (e.g. ``org_<uuid>``).
      :raises OpenBaoConnectionError: On network errors.
      :raises OpenBaoAuthError: On 401/403.

      Create a new OpenBao namespace.  If the namespace already exists, the
      operation is silently ignored (HTTP 400 with ``"already exists"`` is
      treated as a no-op).

   .. method:: delete_namespace(name)

      :param str name: Namespace name to delete.
      :raises OpenBaoSecretNotFoundError: If the namespace does not exist.

   .. method:: enable_kv_v2(mount_path, namespace=None)

      :param str mount_path: Mount path (e.g. ``config``).
      :param namespace: Optional namespace to operate in.
      :type namespace: str | None
      :raises OpenBaoConnectionError: On network errors.
      :raises OpenBaoAuthError: On 401/403.

      Enable the KV v2 secrets engine at the given mount path.  If the mount
      already exists, the operation is silently ignored.

   .. rubric:: System Configuration

   .. method:: read_system_config()

      :rtype: dict[str, Any]

      Read system config from the combined secret at
      ``config/data/system`` in the ``system/`` namespace.  Returns an
      empty dict if no system secret exists yet::

          raw = await bao.read_system_config()
          # {"database_url": "postgresql+asyncpg://...", ...}

   .. method:: write_system_config(config)

      :param dict[str, Any] config: Flat dict of key/value pairs to merge
          into the system secret.
      :raises OpenBaoError: If CAS version has changed (write conflict).

      Write system config as a single combined secret.  Uses CAS-aware
      read-modify-write so that concurrent writers do not silently clobber
      each other's keys.

   .. rubric:: Organisation Configuration

   .. method:: read_org_config(org_id)

      :param UUID org_id: Organisation UUID.
      :rtype: dict[str, Any]

      Read all configuration keys for an organisation from its own namespace.
      Returns an empty dict if no config exists yet.

   .. method:: write_org_config(org_id, config)

      :param UUID org_id: Organisation UUID.
      :param dict[str, Any] config: Flat dict of key/value pairs.  ``None``
          values trigger deletion of the corresponding secret.

      Write organisation-level configuration key/value pairs.  Ensures the
      org namespace and KV engine exist (idempotent, one-time cost per org).

   .. method:: create_org_namespace(org_id)

      :param UUID org_id: Organisation UUID.

      Bootstrap a namespace for a new organisation.  Creates the
      ``org_<uuid>`` namespace and enables a KV v2 secrets engine at
      ``config/`` inside it.  Both operations are idempotent.

   .. method:: delete_org_namespace(org_id)

      :param UUID org_id: Organisation UUID.

      Tear down the namespace for an organisation, deleting all secrets
      within it.

   .. rubric:: Transit Engine (Encryption-as-a-Service)

   .. method:: enable_transit_engine(mount_path="transit")

      :param str mount_path: Mount path for the transit engine.

      Enable the Transit secrets engine.  Idempotent if already mounted.

   .. method:: create_encryption_key(key_name, key_type="aes256-gcm96", mount_path="transit")

      :param str key_name: Name of the encryption key.
      :param str key_type: Key type (``"aes128-gcm96"``, ``"aes256-gcm96"``,
          ``"chacha20-poly1305"``, ``"ed25519"``, ``"ecdsa-p256"``).
      :param str mount_path: Transit engine mount path.

      Create an encryption key.  Idempotent if the key already exists.

   .. method:: encrypt_data(key_name, plaintext, mount_path="transit", context=None)

      :param str key_name: Name of the encryption key.
      :param str plaintext: Data to encrypt.
      :param str mount_path: Transit engine mount path.
      :param context: Optional additional authenticated data (AAD).  The
          method base64-encodes this before sending to OpenBao.
      :type context: str | None
      :rtype: str

      Encrypt plaintext using the named Transit encryption key.  Returns the
      OpenBao ciphertext string (includes key version info).

   .. method:: decrypt_data(key_name, ciphertext, mount_path="transit", context=None)

      :param str key_name: Name of the encryption key.
      :param str ciphertext: Ciphertext as returned by :meth:`encrypt_data`.
      :param str mount_path: Transit engine mount path.
      :param context: Optional AAD that was used during encryption.
      :type context: str | None
      :rtype: str

      Decrypt ciphertext using the named Transit encryption key.  Returns
      the decrypted plaintext string.

   .. method:: rotate_encryption_key(key_name, mount_path="transit")

      :param str key_name: Name of the encryption key to rotate.
      :param str mount_path: Transit engine mount path.

      Rotate the named encryption key.  New data will be encrypted with the
      new key version; old data can still be decrypted (OpenBao keeps
      previous key versions).

   .. method:: rewrap_data(key_name, ciphertext, mount_path="transit", context=None)

      :param str key_name: Name of the encryption key.
      :param str ciphertext: Ciphertext to rewrap.
      :param str mount_path: Transit engine mount path.
      :param context: Optional AAD that was used during encryption.
      :type context: str | None
      :rtype: str

      Rewrap ciphertext under the latest version of the encryption key.
      Decrypts then re-encrypts the data without revealing the plaintext to
      the caller (server-side rewrap).  Useful for key rotation.

Constants
^^^^^^^^^

.. data:: SYSTEM_NAMESPACE

   :type: str
   :value: ``"system/"``

   Namespace path where system-level configuration is stored.

.. data:: ORG_NAMESPACE_PREFIX

   :type: str
   :value: ``"org_"``

   Prefix applied to organisation namespace names.

.. data:: KV_MOUNT

   :type: str
   :value: ``"config"``

   KV v2 mount path for configuration secrets.

.. data:: SYSTEM_KEY_MAPPING

   :type: dict[str, str]

   Maps OpenBao config key names (snake_case) to ``OZ_`` environment variable
   names.  Used by :func:`init_settings` to translate the flat OpenBao secret
   into :class:`Settings` field names.

.. _openbao-settings-label:

Settings Loader
~~~~~~~~~~~~~~~

Module: ``core.openbao_settings``

.. function:: init_settings(client)

   :param OpenBaoClient client: An authenticated OpenBao client.
   :rtype: Settings

   Read system secrets from OpenBao and populate the ``Settings`` singleton.
   Uses :meth:`OpenBaoClient.read_system_config` to fetch the raw key/value
   pairs, maps OpenBao key names to ``Settings`` field names via
   :data:`core.openbao.SYSTEM_KEY_MAPPING`, and performs type casting for
   integer fields.

   The resulting instance is stored as the module-level singleton and can be
   retrieved with :func:`core.config.get_settings`::

       from core.config import BootstrapSettings
       from core.openbao import OpenBaoClient
       from core.openbao_settings import init_settings

       bootstrap = BootstrapSettings()
       async with OpenBaoClient(
           bootstrap.OPENBAO_ADDR,
           bootstrap.OPENBAO_ROLE_ID,
           bootstrap.OPENBAO_SECRET_ID,
       ) as bao:
           settings = await init_settings(bao)
           # settings.DATABASE_URL, settings.SECRET_KEY, ...


Transit Manager
~~~~~~~~~~~~~~~

Module: ``core.transit``

High-level wrapper around OpenBao Transit encryption with typed
encrypt/decrypt methods for specific data contexts.

.. class:: TransitManager(bao_client)

   :param OpenBaoClient bao_client: An authenticated OpenBao client.

   .. method:: encrypt_org_api_key(org_id, api_key)

      :param UUID org_id: Organisation UUID (used as AAD context).
      :param str api_key: The plaintext API key.
      :rtype: str

   .. method:: decrypt_org_api_key(org_id, ciphertext)

      :param UUID org_id: Organisation UUID (must match the AAD used at
          encryption).
      :param str ciphertext: Ciphertext from :meth:`encrypt_org_api_key`.
      :rtype: str

   .. method:: encrypt_webhook_secret(webhook_id, secret)

      :param UUID webhook_id: Webhook UUID (used as AAD context).
      :param str secret: The plaintext webhook signing secret.
      :rtype: str

   .. method:: decrypt_webhook_secret(webhook_id, ciphertext)

      :param UUID webhook_id: Webhook UUID (must match AAD).
      :param str ciphertext: Ciphertext from :meth:`encrypt_webhook_secret`.
      :rtype: str

   .. method:: encrypt_pii(user_id, plaintext)

      :param UUID user_id: User UUID (used as AAD context).
      :param str plaintext: The PII data to encrypt.
      :rtype: str

   .. method:: decrypt_pii(user_id, ciphertext)

      :param UUID user_id: User UUID (must match AAD).
      :param str ciphertext: Ciphertext from :meth:`encrypt_pii`.
      :rtype: str

   .. method:: rotate_all_keys()

      :rtype: dict[str, Any]

      Rotate all known encryption keys.  New data will use new key versions;
      old data remains decryptable.  Returns a dict mapping key names to
      outcome messages (``"rotated"`` or ``"failed: <error>"``).

Constants:

.. data:: ORG_API_KEY_KEY

   :type: str
   :value: ``"org-api-key"``

.. data:: WEBHOOK_SECRET_KEY

   :type: str
   :value: ``"webhook-secret"``

.. data:: PII_ENCRYPTION_KEY

   :type: str
   :value: ``"pii-encryption"``

Real-world usage — encrypting an org's LLM API key::

    from core.openbao import OpenBaoClient
    from core.transit import TransitManager

    async with OpenBaoClient(addr, role_id, secret_id) as bao:
        transit = TransitManager(bao)

        # Encrypt before storing in DB
        encrypted = await transit.encrypt_org_api_key(org_id, "sk-...")

        # Decrypt when needed at runtime
        plaintext = await transit.decrypt_org_api_key(org_id, encrypted)


Event System
------------

Module: ``core.events``

Webhook event type registry and typed payload definitions.  Every meaningful
action in the system maps to an event type constant.  Services call
``WebhookService.emit()`` with an event type and payload dict; the webhook
service enqueues an ARQ job to deliver the webhook asynchronously.

.. class:: EventType(str)

   A webhook event type constant.  Event type strings follow the pattern
   ``{domain}.{action}``::

       event = EventType.SESSION_CREATED
       assert event == "session.created"

   .. attribute:: SESSION_CREATED

      :type: ClassVar[str]
      :value: ``"session.created"``

   .. attribute:: SESSION_CLOSED

      :type: ClassVar[str]
      :value: ``"session.closed"``

   .. attribute:: MESSAGE_ADDED

      :type: ClassVar[str]
      :value: ``"message.added"``

   .. attribute:: EPISODE_PROCESSED

      :type: ClassVar[str]
      :value: ``"episode.processed"``

   .. attribute:: INGEST_BATCH_COMPLETED

      :type: ClassVar[str]
      :value: ``"ingest.batch.completed"``

   .. attribute:: INGEST_EPISODE_COMPLETED

      :type: ClassVar[str]
      :value: ``"ingest.episode.completed"``

   .. attribute:: GRAPH_ENTITY_CREATED

      :type: ClassVar[str]
      :value: ``"graph.entity.created"``

   .. attribute:: GRAPH_ENTITY_UPDATED

      :type: ClassVar[str]
      :value: ``"graph.entity.updated"``

   .. attribute:: GRAPH_EDGE_CREATED

      :type: ClassVar[str]
      :value: ``"graph.edge.created"``

   .. attribute:: FACT_EXTRACTED

      :type: ClassVar[str]
      :value: ``"fact.extracted"``

   .. attribute:: FACT_DELETED

      :type: ClassVar[str]
      :value: ``"fact.deleted"``

   .. attribute:: CLASSIFICATION_CREATED

      :type: ClassVar[str]
      :value: ``"classification.created"``

   .. attribute:: EXTRACTION_CREATED

      :type: ClassVar[str]
      :value: ``"extraction.created"``

   .. attribute:: USER_CREATED

      :type: ClassVar[str]
      :value: ``"user.created"``

.. class:: EventMeta(type, label, category, description)

   A :class:`NamedTuple` holding metadata about a registered event type.

   :param str type: Event type string (e.g. ``"session.created"``).
   :param str label: Human-readable label.
   :param str category: Category grouping.
   :param str description: Description of when the event fires.

.. data:: EVENT_REGISTRY

   :type: list[EventMeta]

   An ordered registry of all known events, used by the create-webhook UI and
   the event registry.  Contains one :class:`EventMeta` entry per event type.

.. function:: event_type_labels()

   :rtype: Mapping[str, str]

   Return a mapping of event type → human-readable label::

       labels = event_type_labels()
       # {"session.created": "Session Created", ...}

.. function:: event_categories()

   :rtype: Mapping[str, list[EventMeta]]

   Return event registry grouped by category::

       categories = event_categories()
       # {"Session": [EventMeta(...), ...], "Graph": [...], "Fact": [...]}

Real-world usage::

    from core.events import EventType, event_type_labels
    from services.webhook_service import WebhookService

    # Display available events in a dropdown
    for event_type, label in event_type_labels().items():
        print(f"{label} ({event_type})")

    # Emit an event
    await webhook_service.emit(
        org_id=org.id,
        event_type=EventType.SESSION_CREATED,
        payload={"session_id": str(session.id), "user_id": str(user.id)},
    )


Cursor Pagination
-----------------

Module: ``core.cursor``

Base64 encoding/decoding primitives for cursor-based pagination.  Each
repository defines its own cursor payload format; this module provides only
the encode/decode primitives.

.. function:: encode_cursor(value)

   :param str value: The raw cursor string to encode.
   :rtype: str

   Encode a cursor value as a URL-safe base64 string without padding::

       cursor = encode_cursor("2026-07-15T10:30:00Z_session_123")
       # "MjAyNi0wNy0xNVQxMDozMDowMFpfc2Vzc2lvbl8xMjM"

.. function:: decode_cursor(cursor)

   :param str cursor: The base64-encoded cursor string (with or without
       padding).
   :rtype: str
   :raises ValueError: If the cursor is malformed.

   Decode a URL-safe base64 cursor string::

       raw = decode_cursor("MjAyNi0wNy0xNVQxMDozMDowMFpfc2Vzc2lvbl8xMjM")
       # "2026-07-15T10:30:00Z_session_123"

Real-world usage in a repository::

    from core.cursor import encode_cursor, decode_cursor

    class SessionRepository:
        async def list_paginated(
            self, *, limit: int = 20, cursor: str | None = None
        ) -> tuple[list[Session], str | None]:
            query = select(Session).order_by(Session.created_at.desc()).limit(limit + 1)

            if cursor:
                decoded = decode_cursor(cursor)
                # Parse cursor value into a filter condition
                cursor_value = datetime.fromisoformat(decoded)
                query = query.where(Session.created_at < cursor_value)

            result = await self._db.execute(query)
            rows = result.scalars().all()

            next_cursor = None
            if len(rows) > limit:
                rows = rows[:limit]
                last = rows[-1]
                next_cursor = encode_cursor(last.created_at.isoformat())

            return list(rows), next_cursor


Prompt Manifest
---------------

Module: ``core.prompt_manifest``

Loads system-default prompt templates from a ``manifest.yaml`` and ``.jinja2``
files on disk.  Provides a caching loader and lookup helpers so the rest of
the system never needs to know about the file layout.

.. data:: PROMPTS_DIR

   :type: pathlib.Path

   Absolute path to the directory containing ``manifest.yaml`` and ``.jinja2``
   files.  Resolved as ``services/worker/prompts/`` relative to the module
   location::

       PROMPTS_DIR = Path(__file__).resolve().parent.parent / "services" / "worker" / "prompts"

.. data:: MANIFEST_FILENAME

   :type: str
   :value: ``"manifest.yaml"``

.. class:: PromptManifest(data)

   :param dict data: The parsed YAML content.

   Parsed manifest data with efficient lookup helpers.

   .. attribute:: version

      :type: int
      :default: ``1``

      Manifest schema version from the YAML.

   .. attribute:: templates

      :type: list[dict]

      Raw list of template dicts from the manifest.

   .. attribute:: by_name

      :type: dict[str, dict]

      Mapping of ``template_name → entry``.

   .. attribute:: by_type

      :type: dict[str, list[dict]]

      Mapping of ``type → [entries]``.

   .. method:: get_by_name(name)

      :param str name: Template name.
      :rtype: dict | None

      Return the manifest entry for a template name, or ``None``.

   .. method:: get_default_for_type(type)

      :param str type: Template type (e.g. ``"fact_extraction"``).
      :rtype: dict | None

      Return the manifest entry marked as type default, or ``None``.  Only
      one entry per type should have ``is_default_for_type: true``; if
      multiple are accidentally marked, the first match wins.

   .. method:: get_default_names()

      :rtype: list[str]

      Return the template names of all type-default entries.

   .. method:: get_template_text(file_name)

      :param str file_name: Relative filename from the manifest
          (e.g. ``"extract_facts_v4.jinja2"``).
      :rtype: str
      :raises FileNotFoundError: If the file does not exist inside
          :data:`PROMPTS_DIR`.

      Read the actual prompt template text from disk.

.. function:: load_manifest(*, reload=False)

   :param bool reload: If ``True``, bypass the cache and re-read from disk.
   :rtype: PromptManifest
   :raises FileNotFoundError: If ``manifest.yaml`` does not exist.
   :raises yaml.YAMLError: If the manifest file is malformed.

   Load and return the parsed prompt manifest.  Results are cached
   module-globally across calls::

       manifest = load_manifest()
       entry = manifest.get_default_for_type("fact_extraction")
       if entry:
           text = manifest.get_template_text(entry["file"])

.. function:: invalidate_manifest_cache()

   :rtype: None

   Clear the module-level manifest cache.  The next call to
   :func:`load_manifest` will re-read from disk (useful in tests or after
   deploys)::

       invalidate_manifest_cache()
       new_manifest = load_manifest()  # forces re-read


Org Config
----------

Module: ``core.org_config``

Per-organisation configuration resolution — cache-first, OpenBao-authoritative.
Every request path and background worker that needs org-level settings (LLM,
embeddings, graph, behaviour) should resolve them through this module.

**Resolution order:**

1. **Redis cache** (key ``org_config:{org_id}``, TTL 5 min) — performance
   optimisation only.  Cache failures are logged at ERROR but the request
   continues to OpenBao.
2. **OpenBao KV** (per-org namespace ``org_<uuid>/config/``) — the
   authoritative source.  OpenBao failures propagate as hard errors.

There is **no** env-var fallback — if a field is not set in OpenBao it is
returned as ``None`` and the caller decides what to do.

On config update the cache is invalidated inline; invalidation failures are
logged at ERROR but do not fail the operation (stale cache expires via TTL).

.. data:: ORG_CONFIG_CACHE_TTL

   :type: int
   :value: ``300``

   TTL in seconds for cached org config (5 minutes).

.. data:: CACHE_KEY_PREFIX

   :type: str
   :value: ``"org_config"``

   Redis key prefix for cached org config.

.. function:: get_org_config(org_id, redis=None, bao_client=None, \*, skip_cache=False)

   :param UUID org_id: The organisation UUID.
   :param redis: An optional async Redis client.  When ``None``, caching is
       skipped.
   :type redis: redis.asyncio.Redis | None
    :param bao_client: An authenticated :class:`OpenBaoClient`.
       **Required for operation (signature allows None).**
    :type bao_client: OpenBaoClient | None
   :param bool skip_cache: If ``True``, bypass cache and always fetch from
       OpenBao.
   :rtype: OrgConfigBase
   :raises OpenBaoConnectionError: If *bao_client* is ``None``.

   Fetch the stored config for an org: cache → OpenBao.  There is no env-var
   fallback — every field is returned as stored in OpenBao.  Unset fields are
   ``None``.

   When OpenBao has no config at all for the org, every field in the returned
   :class:`OrgConfigBase` is set to ``None`` (Pydantic defaults are **not**
   applied)::

       config = await get_org_config(
           org_id=org.id,
           redis=redis_client,
           bao_client=bao_client,
       )
       api_key = config.llm_api_key  # str | None

.. function:: update_org_config(org_id, update_data, bao_client, redis=None)

   :param UUID org_id: The organisation UUID.
   :param update_data: Fields to update.  Can be an
       :class:`UpdateOrgConfigRequest` or a plain dict.
   :type update_data: UpdateOrgConfigRequest | dict[str, Any]
   :param OpenBaoClient bao_client: An authenticated OpenBao client.
   :param redis: An optional async Redis client (for cache invalidation).
   :type redis: redis.asyncio.Redis | None
   :rtype: OrgConfigBase

   Update stored org config in OpenBao, invalidate cache, return fresh config.
   Performs a deep merge: provided keys replace existing values.  Keys set to
   ``None`` are removed from the stored config (returning ``None`` on next
   read).  OpenBao is the sole authoritative store — there is no database
   dual-write::

       updated = await update_org_config(
           org_id=org.id,
           update_data={"graph_backend": "falkordb", "llm_api_key": None},
           bao_client=bao_client,
           redis=redis_client,
       )
       # Returns the freshly stored config after update

.. function:: build_cache_key(org_id)

   :param UUID org_id: The organisation UUID.
   :rtype: str

   Build the Redis cache key for an org's config, returning a string like
   ``"org_config:<uuid>"``::

       key = build_cache_key(org.id)
       # "org_config:550e8400-e29b-41d4-a716-446655440000"


Additional Core Infrastructure
------------------------------

The following modules are part of ``core/`` but are documented separately due
to their scope and domain-specific nature.


Graph Backend Dispatcher (`core.graph_backend`)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Module: ``core.graph_backend``

Provides :class:`GraphBackendDispatcher` — a registry of backend **classes**
(not instances) and a :meth:`resolve_and_create` factory that selects and
instantiates the per-org graph backend based on ``org_config.graph_backend``.

* Registered backends: ``"surrealdb"``, ``"postgres"``, ``"falkordb"``.
* ``graph_backend="none"`` or ``None`` → graph features disabled, returns
  ``None``.
* :func:`init_dispatcher()` creates and populates the dispatcher — call once
  during the application lifespan.

See ``packages/graph_backend/`` for the ``GraphBackend`` ABC and concrete
implementations.


Email Configuration (`core.email`)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Module: ``core.email``

Typed SMTP configuration extracted from runtime :class:`Settings`.

.. class:: EmailConfig

   :param str host: SMTP server hostname.
   :param int port: SMTP server port.
   :param str username: SMTP username (empty = no auth).
   :param str password: SMTP password (empty = no auth).
   :param str from_addr: ``From:`` address.
   :param bool use_tls: Use implicit TLS on connect.
   :param bool start_tls: Use STARTTLS.

   Has :attr:`__slots__` for memory efficiency.

   .. classmethod:: from_settings(settings)

      :param Settings settings: The initialised ``Settings`` instance.
      :rtype: EmailConfig

   Usage::

       from core.email import EmailConfig
       from core.config import get_settings

       email_cfg = EmailConfig.from_settings(get_settings())
       # email_cfg.HOST == settings.SMTP_HOST

.. function:: build_email_message(to, subject, html_body, text_body=None, from_addr="noreply@openzync.tech")

   :param str to: Recipient email address.
   :param str subject: Email subject line.
   :param str html_body: HTML body content.
   :param text_body: Optional plain-text fallback.  If ``None``, a crude
       HTML-stripped version is used.
   :type text_body: str | None
   :param str from_addr: Sender address.
   :rtype: email.message.EmailMessage

   Build a multi-part email message with HTML and plain-text alternatives::

       msg = build_email_message(
           to="user@example.com",
           subject="Your OTP code",
           html_body="<p>Your code is 123456</p>",
       )


LLM Abstraction (`core.llm` and `core.llm_backends`)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Module: ``core.llm`` defines the :class:`LLMBackend` ABC, the
:class:`LLMBackendRegistry`, the :class:`LLMProvider` enum, and the
:func:`resolve_backend` factory.

Module: ``core.llm_backends`` provides concrete implementations:
:class:`OllamaBackend`, :class:`OpenAIBackend`, :class:`AzureBackend`,
:class:`AnthropicBackend`.

These modules are documented in the ``LLM Infrastructure`` domain document
(:doc:`llm`).


SurrealDB Connection Pool (`core.surreal_pool`)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Module: ``core.surreal_pool``

Per-org SurrealDB connection pool created lazily and cached by ``org_id``.
Each org gets its own :class:`AsyncSurreal` connection using credentials from
:class:`OrgConfigBase`.  Thread safety via per-org ``asyncio.Lock``.

.. class:: SurrealConnectionPool

   .. method:: get_or_create(org_id, org_config)

      :param UUID org_id: Organisation UUID.
      :param OrgConfigBase org_config: Org config with SurrealDB credentials.

   .. method:: close_all()

      Close all pooled connections on shutdown.

See the ``SurrealDB`` domain documentation for full details.


Design Principles
-----------------

The ``core/`` package enforces these non-negotiable design rules:

1. **Zero env-var fallback for secrets.**  If OpenBao is unreachable at
   startup, the process fails fast with :class:`OpenBaoConnectionError`.

2. **No silent degradation.**  Infrastructure errors
   (:class:`CacheUnavailableError`, :class:`DatabaseUnavailableError`, ...)
   always propagate as HTTP 503 — never swallowed, never served stale data.

3. **OpenBao is the sole source of truth** for both system-level and
   per-org configuration.  Redis caching is best-effort; OpenBao failures
   are hard errors.

4. **One-time initialisation.**  :class:`Settings`, :class:`ARQPool`, and
   the :class:`PromptManifest` cache are populated once at startup and
   accessed through singleton accessors thereafter.
