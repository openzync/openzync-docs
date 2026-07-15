Worker System — ARQ Background Job Processing
===============================================

.. note::

   This document covers the ARQ worker system in ``openzync-core``, including
   worker configuration, the task registry, every registered task, the
   enrichment pipeline, graph backend resolution, scaling model, and container
   entrypoint.

   The worker process is **independently deployable** from the API server —
   it runs as a separate container in production.  See
   :doc:`/guides/deployment` for orchestration details.

   Related API references:

   * :doc:`/api/workers` — ``workers.tasks`` module API
   * :doc:`/api/services.worker` — ``services.worker`` module API
   * :doc:`/api/services.worker.tasks` — ``services.worker.tasks`` module API
   * :doc:`/api/workers.tasks` — ``workers.tasks`` module API

.. contents:: Sections
   :local:
   :depth: 3
   :class: this-will-duplicate-information-and-it-is-still-useful-here


Architecture Overview
---------------------

The worker system is built on **ARQ** (`Async Redis Queue`_), a lightweight
async job queue backed by Redis.  A single worker process runs **two separate
ARQ worker pools** in the same Python process:

.. _Async Redis Queue: https://arq-docs.helpmanual.io/

+-------------------+------------------------------------------------------+----------------------------+----------+
| Queue             | Purpose                                              | Concurrency                | Timeout  |
+===================+======================================================+============================+==========+
| **High-priority** | Real-time episode enrichment (LLM calls, embeddings) | ``min(MAX_WORKERS, 8)``    | 300s     |
+-------------------+------------------------------------------------------+----------------------------+----------+
| **Low-priority**  | Batch / scheduled tasks (community detection,        | ``max(1, MAX_WORKERS // 4)``| 600s     |
|                   | entity dedup, enrichment reconciliation)              |                            |          |
+-------------------+------------------------------------------------------+----------------------------+----------+

Queue names follow the pattern ``OpenZync:{env}:queue:{high|low}``
(e.g. ``OpenZync:prod:queue:high``).  The two-pool design ensures that
real-time ingestion tasks never block on long-running batch jobs.

.. note::

   A single process with two pools, rather than two separate containers,
   was chosen for operational simplicity.  If one pool exhausts its
   concurrency, the other is unaffected — they share the event loop but
   have independent ``max_jobs`` limits.

Prometheus metrics are exposed on port ``9095`` and a health-check HTTP server
on port ``8081`` (``/health``, ``/ready``).  See
:ref:`monitoring-label` below for the full metric reference.


Startup Sequence
----------------

The worker process is started via::

    python -m services.worker.worker

This invokes :func:`services.worker.worker.main`, which runs the following
startup phases:

Phase 1 — OpenBao Bootstrapping
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The worker authenticates to OpenBao using **worker-specific AppRole**
credentials (``OPENBAO_WORKER_ROLE_ID`` / ``OPENBAO_WORKER_SECRET_ID``)
if provided, falling back to the main OpenBao credentials.  This allows
operators to scope worker permissions independently from the API service::

    _role_id = _bootstrap.OPENBAO_WORKER_ROLE_ID or _bootstrap.OPENBAO_ROLE_ID
    _secret_id = _bootstrap.OPENBAO_WORKER_SECRET_ID or _bootstrap.OPENBAO_SECRET_ID

Once authenticated, it calls :func:`init_worker_settings_from_bao` to load
``DATABASE_URL``, ``REDIS_URL``, ``ENV``, ``LOG_LEVEL``, and other worker
settings into the :class:`WorkerSettings` singleton.

A **persistent** :class:`OpenBaoClient` is created and stored in the worker
context for per-task org config resolution.  This avoids re-authenticating
on every task invocation.

Phase 2 — Shared Resource Initialisation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following resources are created **once per worker process** and injected
into every task invocation via the ARQ ``ctx`` dictionary:

==================== ==========================================================
Context Key           Resource
==================== ==========================================================
``db_engine``         Shared SQLAlchemy async engine (pool_size=10, max_overflow=5)
``db_session_factory`` Bound :func:`async_sessionmaker`
``graph_backend_dispatcher`` ``GraphBackendDispatcher`` registry
``surreal_connection_pool`` Lazy per-org SurrealDB connection pool
``falkordb_client``   FalkorDB client (optional — graceful fallback)
``openbao_client``    Persistent authenticated OpenBao client
==================== ==========================================================

The shared DB engine eliminates per-task connection churn that would exhaust
PostgreSQL's ``max_connections`` at scale.

Phase 3 — Worker Pool Creation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Two :class:`arq.worker.Worker` instances are created with their respective
task registries.  Cron jobs are attached to the low-priority pool:

* **Enrichment reconciliation** every 5 minutes (always active).
* **Community detection** nightly at 02:00 UTC (only when
  ``AUTO_RUN_COMMUNITY_DETECTION`` is ``False`` — the default).

Phase 4 — Liftoff
~~~~~~~~~~~~~~~~~

The worker registers SIGTERM/SIGINT handlers for graceful shutdown, starts
the aiohttp health-check server, the Prometheus HTTP server, and a background
queue-depth monitoring task.  Finally, it runs both pools concurrently via
``asyncio.gather(high_worker.async_run(), low_worker.async_run())``.

Shutdown is graceful: the current job completes, no new jobs are accepted,
and a second signal forces an immediate exit.


Container Entrypoint
--------------------

The worker container is started by ``scripts/entrypoint_worker.sh``, which:

1. Waits up to 90 seconds for an OpenBao Agent sidecar to render secrets
   to ``/run/secrets/system.env`` (checked periodically — exits fatally if
   the file is not ready in time).
2. Sources the secrets file as environment variables.
3. Falls back to reading OpenBao AppRole credentials from bootstrap files
   at ``/openbao-bootstrap/worker-role_id`` / ``worker-secret_id`` if not
   already set.
4. Execs ``python -m services.worker.worker`` as PID 1 (no intermediate
   shell wrapper — signals are received directly).

The container entrypoint is independent from the API entrypoint
(``scripts/entrypoint_api.sh``), allowing the worker to be deployed as a
separate Kubernetes Deployment or Docker Compose service.


Configuration — ``WorkerSettings``
-----------------------------------

Module: ``services.worker.worker_settings``

Worker configuration is loaded from OpenBao at startup via
:func:`init_worker_settings_from_bao`.  The settings singleton is accessed as::

    from services.worker.worker_settings import settings

    queue = settings.high_queue_full
    # → "OpenZync:prod:queue:high"

All durations are in seconds unless otherwise noted.

.. class:: WorkerSettings

   Worker-specific configuration loaded from OpenBao or environment variables.

   =================================== ========== =================================
   Field                               Default    Description
   =================================== ========== =================================
   ``DATABASE_URL``                    *(required)* PostgreSQL async connection string
   ``REDIS_URL``                       *(required)* Redis connection string
   ``ENV``                             ``"development"`` Environment name (queue prefix)
   ``MAX_WORKERS``                     ``4``       Max concurrent tasks (1–32)
   ``JOB_TIMEOUT_DEFAULT``             ``300``     Default job timeout (seconds)
   ``JOB_KEEP_RESULT_FOR``             ``3600``    Keep successful job results (s)
   ``JOB_KEEP_RESULT_FOR_FAILURE``     ``86400``   Keep failed job results (s)
   ``HIGH_QUEUE_NAME``                 ``"high"``  Logical name for high queue
   ``LOW_QUEUE_NAME``                  ``"low"``   Logical name for low queue
   ``POLL_DELAY``                      ``0.5``     Seconds between empty-queue polls
   ``HEALTH_PORT``                     ``8081``    Health check HTTP server port
   ``LOG_LEVEL``                       ``"INFO"``  Minimum log level
   ``STRUCTLOG_FORMAT``                ``"json"``  ``"json"`` or ``"console"``
   ``STRUCTURED_EXTRACTION_MAX_TOKENS`` ``2000``   Max tokens for structured extraction
   ``PROMETHEUS_PORT``                 ``9095``    Prometheus metrics port
   ``FALKORDB_URL``                    ``"redis://localhost:6379"``
   ``FALKORDB_MAX_CONNECTIONS``        ``10``      FalkorDB connection pool size
   ``FALKORDB_SOCKET_TIMEOUT``         ``10``      FalkorDB socket timeout
   ``AUTO_RUN_COMMUNITY_DETECTION``    ``False``   Event-driven vs nightly cron
   =================================== ========== =================================

OpenBao key mapping follows ``SYSTEM_KEY_MAPPING`` in ``core/openbao.py``.
Validated at startup against the canonical key set — any drift is caught
immediately, not at runtime.

.. _arq-pool-label:

ARQ Connection Management — ``core.arq``
-----------------------------------------

Module: ``core/arq.py``

The ARQ connection pool is managed as a module-level singleton with explicit
lifecycle hooks for FastAPI integration::

    from core.arq import init_arq, close_arq, get_arq

    # During FastAPI lifespan startup
    await init_arq()

    # Enqueue a job
    job_id = await get_arq().enqueue("embed_episode", episode_id="...", org_id="...")

    # During FastAPI lifespan shutdown
    await close_arq()

.. class:: ARQPool

   Manages the ARQ connection pool lifecycle.

   .. method:: initialize()

      Create the pool.  Raises :class:`ConnectionError` if Redis is unreachable.

   .. method:: close()

      Close the pool gracefully.  Safe to call multiple times.

   .. method:: enqueue(task_name, queue_name=None, **kwargs)

      Enqueue a background job.  Returns the job ID or ``None`` if the pool
      was unavailable.

      :param str task_name: Registered worker function name (e.g. ``"embed_episode"``).
      :param queue_name: Optional queue name (``"high"`` or ``"low"``).  Passed
          as ``_queue_name`` to ARQ's ``enqueue_job``.
      :type queue_name: str | None
      :param \\**kwargs: Keyword arguments forwarded to the task function.
      :returns: The enqueued job ID string, or ``None``.
      :rtype: str | None

   Usage from the API server::

       from core.arq import get_arq

       await get_arq().enqueue(
           "extract_entities",
           queue_name="high",
           episode_id=str(episode.id),
           org_id=str(org.org_id),
           project_id=str(episode.project_id),
           content=episode.content,
       )


.. _task-registry-label:

Task Registry
-------------

Tasks are registered with ARQ by being included in one of two lists defined
in ``services/worker/worker.py``:

.. list-table:: High-Priority Queue Tasks (real-time ingestion)
   :header-rows: 1

   * - Task
     - Source Module
     - Enrichment Bit
     - Description
   * - ``enrich_episode``
     - :mod:`workers.tasks.enrich_episode`
     - Bits 0, 2, 4, 5
     - Combined LLM enrichment (replaces 4 separate calls)
   * - ``classify_dialog``
     - :mod:`workers.tasks.classify_dialog`
     - Bit 4
     - Dialog intent/emotion classification
   * - ``extract_entities``
     - :mod:`workers.tasks.extract_entities`
     - Bit 0
     - Named entity + relationship extraction
   * - ``embed_episode``
     - :mod:`workers.tasks.embed_episode`
     - Bit 1
     - pgvector embedding for episode content
   * - ``extract_facts``
     - :mod:`workers.tasks.extract_facts`
     - Bit 2
     - Zero-shot fact triple extraction
   * - ``embed_fact``
     - :mod:`workers.tasks.embed_fact`
     - *(none)*
     - pgvector embedding for fact content
   * - ``extract_structured``
     - :mod:`workers.tasks.extract_structured`
     - Bit 5
     - JSON Schema–guided structured extraction

.. list-table:: Low-Priority Queue Tasks (batch / scheduled)
   :header-rows: 1

   * - Task
     - Source Module
     - Enrichment Bit
     - Description
   * - ``link_entities_to_episode``
     - :mod:`workers.tasks.link_entities_to_episode`
     - Bit 3
     - Link entities to episode via graph join table
   * - ``compute_observations``
     - :mod:`workers.tasks.compute_observations`
     - Bit 6
     - Deferred graph-topology pattern detection
   * - ``summarise_community``
     - :mod:`workers.tasks.summarise_community`
     - *(none)*
     - Community detection + LLM summarisation
   * - ``merge_duplicate_entities``
     - :mod:`workers.tasks.merge_duplicate_entities`
     - *(none)*
     - Entity dedup via exact + fuzzy matching
   * - ``write_audit_log``
     - :mod:`services.worker.tasks.audit_log`
     - *(none)*
     - Async audit log persistence
   * - ``deliver_webhook``
     - :mod:`services.worker.tasks.deliver_webhook`
     - *(none)*
     - Webhook delivery with retry
   * - ``generate_user_summary``
     - :mod:`workers.tasks.generate_user_summary`
     - *(none)*
     - LLM-generated user profile summary
   * - ``reconcile_enrichment``
     - :mod:`workers.tasks.reconcile_enrichment`
     - *(none)*
     - Cron job: re-enqueue stale enrichment tasks

Cron Jobs
~~~~~~~~~

Attached to the low-priority worker pool:

==================== ============================ ============= ============================================
Cron Job             Function                     Schedule       Purpose
==================== ============================ ============= ============================================
Enrichment           ``reconcile_enrichment``      Every 5        Safety net for dropped enrichment tasks
reconciliation                                     minutes
Community            ``summarise_community``       Nightly        Label-propagation community detection
detection (nightly)                                02:00 UTC      (when not event-driven)
==================== ============================ ============= ============================================


Enrichment Bitmask System
-------------------------

Module: ``workers/tasks/base.py``

Enrichment progress is tracked per-episode via an integer bitmask column
(``episodes.enrichment_status``).  Each enrichment step occupies one bit.
Workers check their bit before running (idempotency) and set it after
completion.  The bit allocation is:

+--------------------------------------+-----+----------------------------------------------------------+--------------------------------------+
| Constant                              | Bit | Worker                                                   | Allowed values                       |
+======================================+=====+==========================================================+======================================+
| ``ENRICHMENT_ENTITIES``               | 0   | ``extract_entities`` / ``enrich_episode``                 | ``1 << 0``                           |
+--------------------------------------+-----+----------------------------------------------------------+--------------------------------------+
| ``ENRICHMENT_EMBEDDING``              | 1   | ``embed_episode``                                         | ``1 << 1``                           |
+--------------------------------------+-----+----------------------------------------------------------+--------------------------------------+
| ``ENRICHMENT_FACTS``                  | 2   | ``extract_facts`` / ``enrich_episode``                    | ``1 << 2``                           |
+--------------------------------------+-----+----------------------------------------------------------+--------------------------------------+
| ``ENRICHMENT_ENTITY_LINKS``           | 3   | ``link_entities_to_episode``                              | ``1 << 3``                           |
+--------------------------------------+-----+----------------------------------------------------------+--------------------------------------+
| ``ENRICHMENT_CLASSIFICATION``         | 4   | ``classify_dialog`` / ``enrich_episode``                  | ``1 << 4``                           |
+--------------------------------------+-----+----------------------------------------------------------+--------------------------------------+
| ``ENRICHMENT_STRUCTURED_EXTRACTION``  | 5   | ``extract_structured`` / ``enrich_episode``               | ``1 << 5``                           |
+--------------------------------------+-----+----------------------------------------------------------+--------------------------------------+
| ``ENRICHMENT_OBSERVATIONS``           | 6   | ``compute_observations``                                  | ``1 << 6`` (reserved — not in ALL)   |
+--------------------------------------+-----+----------------------------------------------------------+--------------------------------------+

``ENRICHMENT_ALL`` is the bitmask of all active steps (bits 0–5, excluding
bit 6).  An episode is considered "fully enriched" when
``enrichment_status & ENRICHMENT_ALL == ENRICHMENT_ALL``.

Bit 6 (``ENRICHMENT_OBSERVATIONS``) is intentionally excluded from
``ENRICHMENT_ALL`` — the observations pass is non-blocking and deferred,
and should not gate "fully enriched" status.

.. warning::

   Bit positions are shared across the team and must not be reassigned
   without updating all workers.  Current allocation allows for bits 7+
   as future expansion.


Retry Decorator — ``with_retry``
---------------------------------

.. function:: with_retry(max_retries=3, base_delay_s=1.0, max_delay_s=30.0, *, on_exhaustion="raise", is_retryable=None)

   Decorator that retries an async function with exponential backoff.
   Used by every task in ``workers/tasks/``.

   :param int max_retries: Maximum retry attempts (default ``3``).
   :param float base_delay_s: Initial delay in seconds.
   :param float max_delay_s: Maximum delay cap.
   :param str on_exhaustion: ``"raise"`` (re-raises last exception) or
       ``"log"`` (logs and returns ``None``).
   :param is_retryable: Optional predicate ``Callable[[Exception], bool]``.
       When ``None`` (default), all exceptions are retried.

   Retry schedule: ``base_delay_s × 2²⁰ × attempt``, capped at ``max_delay_s``.
   Example with defaults: 1s, 2s, 4s.

   Usage::

       from workers.tasks.base import with_retry

       @with_retry(max_retries=3, base_delay_s=2.0)
       async def my_task(ctx, episode_id, org_id):
           ...


Graph Backend Resolution — ``workers.backend``
-----------------------------------------------

Module: ``workers/backend.py``

Workers resolve the per-organization graph backend at runtime via
:func:`resolve_graph_backend`.  This function reads the org's configured
backends from the ``organizations.config`` JSONB column (cache-first via
:func:`core.org_config.get_org_config`, DB-authoritative) and instantiates
the appropriate :class:`GraphBackend` implementation.

.. function:: resolve_graph_backend(ctx, org_id, db, *, fallback_to_postgres=True)

   Resolve the per-organization graph backend inside an ARQ worker.

   :param ctx: ARQ worker context dict — must contain
       ``graph_backend_dispatcher`` and may contain
       ``surreal_connection_pool`` and ``falkordb_client``.
   :param org_id: Organization UUID.
   :param db: Async SQLAlchemy session.
   :param bool fallback_to_postgres: If ``True`` (default), returns a
       :class:`PostgresGraphBackend` when no backend is configured or
       resolution fails.  If ``False``, returns ``None``.
   :returns: An initialised :class:`GraphBackend` instance, or ``None``
       if graph is disabled and no fallback.
   :raises GraphBackendUnavailableError: If resolution fails and fallback is
       disabled.

   Resolution order:

   1. Read ``org_config.graph_backend`` (cache-first via OpenBao,
      DB-authoritative).
   2. If the backend name is ``"none"`` or resolution fails: fall back to
      :class:`PostgresGraphBackend` (default) or return ``None``.
   3. If SurrealDB is configured, acquire a connection from the shared
      ``surreal_connection_pool``.
   4. If FalkorDB is configured, use the shared ``falkordb_client``.
   5. Delegate to ``dispatcher.resolve_and_create()``.

   The SurrealDB connection is only acquired when the org explicitly
   configures ``"surrealdb"`` — avoiding unnecessary network round-trips
   for orgs using Postgres or no graph backend.  If SurrealDB is configured
   but unreachable, the error propagates loudly — no silent fallback.

   .. code-block:: python

       from workers.backend import resolve_graph_backend

       async def my_worker(ctx, org_id):
           async with db_session_factory() as db:
               backend = await resolve_graph_backend(ctx, org_id, db)
               if backend is None:
                   # Graph disabled — use Postgres fallback
                   ...
               await backend.link_entity_to_episode(
                   org_id=org_id,
                   project_id=project_id,
                   episode_id=episode_id,
                   entity_id=entity_id,
               )


Task Reference — ``workers/tasks/``
------------------------------------

Every task follows the ARQ contract:

* The first parameter is ``ctx: object`` — the ARQ worker context dict that
  provides shared resources (``db_engine``, ``db_session_factory``, ``redis``,
  ``openbao_client``, etc.).
* Remaining parameters are the task-specific arguments received at enqueue
  time.
* Tasks return ``None`` (void tasks) or a ``dict`` (for tasks whose callers
  read the result).
* Every task is decorated with ``@with_retry(...)`` for resilience.
* DB engine bootstrap follows a standard pattern: use the shared engine from
  ``ctx["db_engine"]`` if available, otherwise create a short-lived engine
  as a fallback (for testability).


``enrich_episode`` — Combined LLM Enrichment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: enrich_episode(ctx, episode_id, org_id, project_id, content, session_id=None, trace_id="", metadata=None, role="user")

   **Queue**: High

   **Enrichment bits**: Sets bits 0 (entities), 2 (facts), 4 (classification),
   5 (structured extraction) on success.

   **Idempotency**: Checks the logical OR of all 4 LLM bits at the start.
   Skips entirely if all are already set.  Each section is processed in an
   independent savepoint (``begin_nested()``), so failures in one section
   do not roll back completed sections.  On partial failure, raises
   :class:`PartialEnrichmentError` carrying the bitmask of successfully
   completed sections — the ARQ retry picks up where the previous attempt
   left off.

   **Pipeline**:

   1. Open session, set RLS context via ``app.org_id``.
   2. Check idempotency — skip if all 4 LLM bits already set.
   3. Resolve ``user_id`` from the episode record.
   4. Render ``enrich_episode_v1.jinja2`` prompt with auto-injected context
      (entities, facts, schemas, conversation history).
   5. Single LLM call with :class:`CombinedLLMOutput` as ``response_model``
      (temperature 0.0, max_tokens 8192).
   6. Process each enrichment section in an independent savepoint:
      classification (bit 4), entities (bit 0), facts (bit 2), structured
      (bit 5).
   7. Commit all successful savepoints atomically.
   8. Raise ``PartialEnrichmentError`` if any section failed (triggers retry).

   **Architecture note**: This task replaces the 4 individual LLM tasks
   (``classify_dialog``, ``extract_entities``, ``extract_facts``,
   ``extract_structured``) with a single LLM call.  The individual tasks
   still exist for backward compatibility and for callers that already know
   which enrichment step they need.

   .. code-block:: python

       from core.arq import get_arq

       await get_arq().enqueue(
           "enrich_episode",
           queue_name="high",
           episode_id=str(episode.id),
           org_id=str(episode.organization_id),
           project_id=str(episode.project_id),
           content=episode.content,
           session_id=str(episode.session_id),
           trace_id=trace_id,
       )


``classify_dialog`` — Dialog Classification
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: classify_dialog(ctx, episode_id, org_id, project_id, content, trace_id="", session_id=None, user_id=None, metadata=None)

   **Queue**: High

   **Enrichment bit**: Bit 4 (``ENRICHMENT_CLASSIFICATION``)

   **Idempotency**: Checks bit 4.  Skips if already set.

   **Pipeline**:

   1. Create or borrow a DB session, set RLS context.
   2. Check bit 4 — skip if already classified.
   3. Fetch organisation's classification schemas (``type='classification'``).
   4. Extract label definitions for intent and emotion.
   5. Render ``classify_dialog_v1.jinja2`` prompt with DB template + custom
      instructions.
   6. Call LLM (temperature 0.0, max_tokens 4096) with
      :class:`ClassificationOutput` as ``response_model``.
   7. Validate labels against allowed sets (intent, emotion, valence, arousal).
   8. Insert ``DialogClassification`` row.
   9. Set bit 4.

   **Label validation**: ``intent`` and ``emotion`` are validated against the
   org's configured ``extraction_schemas``.  Invalid values are logged as
   warnings and stored as ``None``.  ``valence`` is restricted to
   ``{"positive", "negative", "neutral"}``.  ``arousal`` to
   ``{"low", "medium", "high"}``.  ``confidence`` is clamped to ``[0.0, 1.0]``.

   The post-processing logic is exported as :func:`process_classification_output`
   for reuse by the combined ``enrich_episode`` worker.


``extract_entities`` — Entity Extraction
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: extract_entities(ctx, episode_id, org_id, project_id, content, session_id=None, trace_id="", metadata=None)

   **Queue**: High

   **Enrichment bit**: Bit 0 (``ENRICHMENT_ENTITIES``)

   **Idempotency**: Checks bit 0.  Skips if already set.

   **Pipeline**:

   1. Resolve ``user_id`` from episode record.
   2. Resolve graph backend for this org.
   3. Fetch entity type ontology (``extraction_schemas type='entity_type'``).
   4. Fetch known entities from previous turns if ``session_id`` is provided
      (for delta extraction).
   5. Render prompt with context:
      ``extract_entities_v4.jinja2`` (delta) or ``extract_entities_v3.jinja2``
      (first extraction).
   6. Call LLM (temperature 0.1, max_tokens 4096) with
      :class:`EntityExtractionOutput` as ``response_model``.
   7. Filter pronouns and common filler words (see pronoun skip list in source).
   8. Validate entity types against allowed ontology — invalid types are
      reassigned to ``"Custom"``.
   9. Upsert entities to graph backend via ``EntityRepository``.
   10. Upsert relationships as graph edges.
   11. Link entities to episode via ``graph_backend.link_entity_to_episode()``.
   12. Set bit 0.
   13. **Chain** ``extract_facts`` on the high-priority queue.

   **Pronoun filter**: 40+ first/second/third-person pronouns and common
   misspellings are filtered from entity creation.  Pronouns are resolved
   during fact extraction via ``_match_entity()``.

   **Entity recovery pass**: If the LLM mentions an entity in a relationship
   without including it in the ``entities`` array, it is auto-created as
   ``"Custom"`` type so the graph edge is not lost.

   **Relationship to fact extraction**: After bit 0 is set and entities are
   committed, the task enqueues ``extract_facts``.  This chaining ensures
   entity IDs are available in the graph for fact relationship materialisation.

   The post-processing logic is exported as :func:`process_entities_output`
   for reuse by the combined ``enrich_episode`` worker.


``extract_facts`` — Fact Extraction
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: extract_facts(ctx, episode_id, org_id, project_id, content, session_id=None, trace_id="", metadata=None)

   **Queue**: High

   **Enrichment bit**: Bit 2 (``ENRICHMENT_FACTS``)

   **Idempotency**: Checks bit 2.  Skips if already set.

   **Pipeline**:

   1. Resolve ``user_id`` and graph backend from episode.
   2. Fetch known entities and recent conversation history from session
      (for pronoun resolution).
   3. Render prompt: ``extract_facts_v4.jinja2`` (delta) or
      ``extract_facts_v3.jinja2`` (first extraction).
   4. Call LLM (temperature 0.1) with :class:`FactExtractionOutput` as
      ``response_model``.
   5. Filter triples by confidence (>= 0.3) and quality heuristics via
      :func:`_filter_facts`.
   6. Resolve subject/object to canonical entity names and UUIDs via
      :func:`_resolve_fact_entities` (pronoun resolution, substring matching,
      aggressive normalisation fallback).
   7. Deduplicate against existing session facts via :func:`_deduplicate_facts`
      (with predicate synonym mapping — ``"works_at"`` matches ``"employed_by"``).
   8. Batch-persist via ``FactRepository.batch_create_or_skip``.
   9. Attempt live entity lookup in graph if entity IDs were not resolved
      from the prompt context.
   10. Upsert relationships to graph backend (non-fatal on failure).
   11. Set bit 2.
   12. **Chain** ``embed_fact`` per persisted fact (non-blocking — failures
       logged, not propagated).

   .. note::

      Fact extraction must run *after* entity extraction so that entity
      IDs are available in the graph for relationship materialisation.
      The ``extract_entities`` task enqueues ``extract_facts`` explicitly
      after committing entities.

   The post-processing logic is exported as :func:`process_facts_output`
   for reuse by the combined ``enrich_episode`` worker.


``embed_episode`` — Episode Embedding
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: embed_episode(ctx, episode_id, org_id, project_id, content, trace_id="", metadata=None)

   **Queue**: High

   **Enrichment bit**: Bit 1 (``ENRICHMENT_EMBEDDING``)

   **Idempotency**: Checks bit 1.  Skips if already set.

   **Pipeline**:

   1. Check idempotency (bit 1).
   2. Fetch per-org config from OpenBao — resolves embedding backend,
      model, and dimension from ``org_config.embedding_backend``,
      ``embedding_model``, ``embedding_dim``.  **There is no env-var
      fallback** — if any required field is ``None``, the task raises
      :class:`SearchLegFailedError` so ARQ retries.
   3. Resolve the embedding backend via :func:`core.llm.resolve_backend`.
   4. Generate embedding vector for the episode content.
   5. Validate that the returned dimension matches ``embedding_dim``.
   6. Store the vector in ``episodes.embedding`` (pgvector).
   7. Set bit 1.

   :raises SearchLegFailedError: If org config fetch fails, org is not found,
       or no embedding backend is configured.
   :raises ValueError: If the returned embedding dimension does not match
       the configured dimension.

   .. code-block:: python

       await get_arq().enqueue(
           "embed_episode",
           queue_name="high",
           episode_id=str(episode.id),
           org_id=str(org_id),
           project_id=str(project_id),
           content=episode.content,
           trace_id=trace_id,
       )


``embed_fact`` — Fact Embedding
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: embed_fact(ctx, fact_id, content=None, trace_id="", **kwargs)

   **Queue**: High

   **Enrichment bit**: None (facts do not have enrichment status bits).

   **Idempotency**: The task itself is not idempotent (the facts table has
   no enrichment bit), but content-hash dedup at the ``FactRepository`` level
   prevents duplicate embeddings.  In practice, repeated invocations with the
   same ``fact_id`` and content produce the same embedding, making it safe.

   **Pipeline**:

   1. Fetch fact content from DB if not provided as parameter.
   2. Fetch per-org config from OpenBao.
   3. Resolve embedding backend, generate embedding, validate dimension.
   4. Store vector in ``facts.embedding``.

   This task is usually chained by ``extract_facts`` or ``ingest_business_data``,
   not enqueued directly.


``extract_structured`` — Structured Extraction
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: extract_structured(ctx, episode_id, org_id, project_id, session_id, content, trace_id="", metadata=None)

   **Queue**: High

   **Enrichment bit**: Bit 5 (``ENRICHMENT_STRUCTURED_EXTRACTION``)

   **Idempotency**: Checks bit 5.  Skips if already set.

   **Pipeline**:

   1. Set RLS context, check idempotency (bit 5).
   2. Fetch organisation's structured schemas (``type='structured'``).
   3. If no schemas configured, set bit 5 and return immediately
      (nothing to extract).
   4. Render ``extract_structured_v1.jinja2`` prompt.
   5. Call LLM (temperature 0.0, max_tokens configurable via
      ``STRUCTURED_EXTRACTION_MAX_TOKENS``) with
      :class:`StructuredExtractionOutput` as ``response_model``.
   6. For each matched schema, validate output against the JSON Schema
      via ``jsonschema.validate()``.
   7. Insert one ``StructuredExtraction`` row per valid schema with
      ``ON CONFLICT (episode_id, schema_id) DO UPDATE``.
   8. Set bit 5.

   **Schema validation**: Each output key is validated against the
   corresponding JSON Schema.  Invalid outputs are logged and skipped
   (per-schema granularity).  Required fields that are missing from the
   LLM output are filled with type-appropriate defaults.

   The post-processing logic is exported as :func:`process_structured_output`
   for reuse by the combined ``enrich_episode`` worker.


``link_entities_to_episode`` — Entity–Episode Linking
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: link_entities_to_episode(ctx, episode_id, org_id, project_id, content, role, trace_id="", metadata=None)

   **Queue**: Low

   **Enrichment bit**: Bit 3 (``ENRICHMENT_ENTITY_LINKS``)

   **Idempotency**: Checks bit 3.  Skips if already set.

   **Pipeline**:

   1. Fetch the episode.
   2. Resolve graph backend.
   3. Extract potential entity names from content (capitalised words of
      length > 2, simple heuristic).
   4. Search matching entities via ``backend.bulk_search_entities()``.
   5. Link each match via ``backend.link_entity_to_episode()``.
   6. Set bit 3.
   7. **Chain** ``compute_observations`` on the low-priority queue
      (with per-project 30-second dedup in Redis).
   8. Optionally **chain** ``summarise_community`` (when
      ``AUTO_RUN_COMMUNITY_DETECTION`` is ``True``, with per-org 1-hour
      dedup in Redis).

   If the graph backend is unavailable, the task logs a warning and
   skips entity linking (non-fatal — can be re-linked later).


``compute_observations`` — Graph-Topology Observations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: compute_observations(ctx, episode_id, org_id, project_id, trace_id="")

   **Queue**: Low

   **Enrichment bit**: Bit 6 (``ENRICHMENT_OBSERVATIONS``)

   **Idempotency**: Two layers:

   * Per-episode bit 6 check (inside this worker).
   * Per-project dedup at enqueue time (30-second window in
     ``link_entities_to_episode``).

   **Pipeline**:

   1. Check bit 6 — skip if already set.
   2. Resolve graph backend and instantiate :class:`ObservationService`.
   3. Run a full project scan of graph topology data via
      ``service.run_full_project_scan()`` (co-occurrence, temporal gaps,
      behavioural patterns).
   4. Optionally call LLM to generate natural-language ``content`` field
      (falls back to template-based descriptions if LLM is unavailable).
   5. Persist observations via ``backend.upsert_observation()``.
   6. Set bit 6.

   **Pattern detection is SQL-first** — all detection algorithms query
   PostgreSQL directly.  The LLM is used only to generate the descriptive
   ``content`` field; when unavailable, template-based descriptions are used.

   .. note::

      Bit 6 is intentionally excluded from ``ENRICHMENT_ALL`` — the
      observations pass is non-blocking and deferred.  An episode can be
      "fully enriched" without observations.


``summarise_community`` — Community Detection
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: summarise_community(ctx, org_id=None)

   **Queue**: Low

   **Enrichment bit**: None (data maintenance task).

   **Idempotency**: Task-level idempotency is via upsert semantics in the
   graph backend — duplicate communities are not created for the same set
   of entities within a single run.  The nightly cron schedule provides
   natural dedup (runs once per 24h).

   **Pipeline**:

   1. Determine orgs to process: all eligible orgs (when ``org_id`` is
      ``None``) or a single org.
   2. For each org, resolve graph backend and find projects with graph data.
   3. Per project:
      a. Fetch all entities (min 5 required) and relationships via backend.
      b. Build a NetworkX graph (:func:`build_entity_graph`).
      c. Run Label Propagation community detection
         (:func:`detect_communities_label_propagation`).
      d. For each community (≥ 2 entities), generate an LLM summary.
      e. Store community entity + ``MEMBER_OF`` edges via the backend.
   4. Commit per-org transaction.

   **Can be invoked in two modes**:

   * **Scheduled nightly** (default, 02:00 UTC) — configured as a cron job
     on the low-priority pool when ``AUTO_RUN_COMMUNITY_DETECTION=False``.
   * **Event-driven** — chained after ``link_entities_to_episode`` when
     ``AUTO_RUN_COMMUNITY_DETECTION=True`` (with per-org 1-hour Redis dedup).

   :returns: A dict with ``status``, ``orgs_processed``, ``orgs_failed``,
       ``communities_created``.


``merge_duplicate_entities`` — Entity Dedup
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: merge_duplicate_entities(ctx, org_id=None)

   **Queue**: Low

   **Enrichment bit**: None (data maintenance task).

   **Idempotency**: Once merged (``is_merged`` flag set), entities are not
   eligible for future dedup runs.  The task skips merged entities via
   ``backend.get_all_entities(include_merged=False)``.

   **Pipeline**:

   1. Determine orgs: all eligible (when ``org_id`` is ``None``) or single.
   2. For each org, resolve graph backend and find active projects.
   3. Per project:
      a. Fetch all non-merged entities.
      b. Phase 1: Exact match — group by ``LOWER(name)``.
      c. Phase 2: Fuzzy match — ``backend.bulk_search_entities()`` with
         ``fuzzy_threshold=0.85``.
      d. For each duplicate cluster:
         - Select canonical entity (most recent ``created_at``).
         - ``backend.merge_entities()`` — atomic rewire + dedup + soft-delete.
         - Write ``audit_log`` entry with before/after snapshot.
      e. 7-day recovery window via ``is_merged`` soft-delete flag.
   4. Per-org commit (project-level rollback on failure).

   **Batch size**: 100 entity clusters per transaction.  If all clusters
   in a project fail, the project is marked as failed (still continues to
   next project).

   :returns: A dict with ``status``, ``orgs_processed``, ``orgs_failed``,
       ``clusters_merged``, ``entities_merged``, ``relationships_rewired``.


``ingest_business_data`` — Batch Fact Ingestion
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: ingest_business_data(ctx, org_id, project_id, user_id, facts, job_id=None, trace_id="")

   **Queue**: Low

   **Enrichment bit**: None (data ingestion task).

   **Idempotency**: Content-hash dedup is checked at the API layer
   (``FactService``).  The worker relies on the fact that identical payloads
   produce identical facts — the repository's ``batch_create`` handles
   duplicate rows gracefully.  In practice, the ARQ retry is safe because
   ``FactRepository.batch_create`` returns only newly inserted rows.

   **Pipeline**:

   1. Validate each triple (subject, predicate, object required).
   2. Bulk-insert valid facts via ``FactRepository.batch_create``.
   3. Enqueue ``embed_fact`` per inserted fact (non-blocking).
   4. Return result dict with ``status``, ``accepted`` count, and
      ``errors`` list.

   :returns: A dict::

       {
           "status": "completed" | "completed_with_errors",
           "accepted": 42,
           "errors": [
               {"index": 3, "error": "Missing required field: ...", "fact": {...}}
           ],
           "detail": "42 facts ingested, 1 errors",
       }


``generate_user_summary`` — User Profiling
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: generate_user_summary(ctx, org_id, user_id, project_id=None, trace_id="")

   **Queue**: Low

   **Enrichment bit**: None (user profiling task).

   **Idempotency**: Replaces the previous summary on each successful run.
   Repeated invocations produce an updated profile — there is no "already
   done" check because new conversation data may have accumulated since
   the last run.  Callers are responsible for throttling (typically
   triggered after every N new episodes).

   **Pipeline**:

   1. Fetch the user's last 100 conversation episodes (chronological).
   2. Fetch extracted facts (subject-predicate-object triples).
   3. Fetch graph entities linked to the user's sessions.
   4. Fetch aggregate dialog classifications (top intents/emotions).
   5. Render ``summarise_user_v1`` Jinja2 prompt.
   6. Call LLM (temperature 0.3).
   7. Persist the summary on the ``User`` model via
      ``UserRepository.update_summary``.


``reconcile_enrichment`` — Enrichment Reconciliation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. function:: reconcile_enrichment(ctx)

   **Queue**: Low (runs as a cron job every 5 minutes).

   **Enrichment bit**: None (supervisory task).

   **Idempotency**: Designed to be safe for overlapping runs.  The cron
   job uses ``unique=True`` in ARQ, preventing concurrent invocations.
   The backlog guard (high-priority queue depth > 1000) prevents adding
   more jobs faster than workers can drain them.

   **Pipeline**:

   1. Derive dependencies from ARQ context (session factory, Redis).
   2. Backlog guard: skip if high-priority queue has > 1000 pending jobs.
   3. Query episodes where ``enrichment_status != ENRICHMENT_ALL`` AND
      ``updated_at < NOW() - 30 minutes`` (batch size: 100).
   4. For each stale episode, compute missing enrichment bits.
   5. Enqueue missing tasks on the appropriate queue:

      * If any of the 4 LLM bits (0, 2, 4, 5) are missing: enqueue
        ``enrich_episode`` once (high queue).
      * Missing bit 1: enqueue ``embed_episode`` (high queue).
      * Missing bit 3: enqueue ``link_entities_to_episode`` (low queue).

   :returns: A summary string for the cron log, e.g.
       ``"Re-enqueued 12 enrichment tasks across 5 episodes"``.

   This is the **safety net** for worker crashes, job timeouts, or any
   scenario where enrichment tasks are dropped without completion.
   Without this cron job, a worker crash would leave episodes
   un-enriched until an operator manually intervenes.


``deliver_webhook`` — Webhook Delivery
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Defined in :mod:`services.worker.tasks.deliver_webhook`.

**Queue**: Low

**Pipeline**: Sends an HTTP POST to a configured webhook URL with the
event payload.  Includes retry logic for transient HTTP failures.
See :doc:`/api/services.worker.tasks` for details.


``write_audit_log`` — Audit Log Writethrough
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Defined in :mod:`services.worker.tasks.audit_log`.

**Queue**: Low

**Pipeline**: Persists an audit log entry asynchronously.  Used by
the API server to avoid blocking the response on audit writes.
See :doc:`/api/services.worker.tasks` for details.


.. _idempotency-label:

Idempotency Semantics
---------------------

Every task in the enrichment pipeline follows the **same idempotency pattern**:

1. **Bitmask check**: Before doing any work, check the episode's
   ``enrichment_status`` bitmask.  If the relevant bit is already set,
   return immediately.  This is the primary idempotency mechanism.
2. **Exponential backoff retry**: All tasks are decorated with
   ``@with_retry(...)``.  On transient failure (LLM timeout, DB deadlock),
   the task retries with backoff.  The bitmask check at the entry ensures
   that partial progress from a previous attempt (e.g., bit 4 set but bit 0
   not) is not redone.
3. **Savepoint isolation** (combined worker): The ``enrich_episode`` task
   processes each section in an independent ``begin_nested()`` savepoint.
   A failure in facts does not roll back entities.

The ``reconcile_enrichment`` cron job provides a **secondary safety net**
for dropped tasks — it does not rely on the primary idempotency mechanism
and instead re-enqueues missing tasks for any episode stale for > 30 minutes.

Non-enrichment tasks (``summarise_community``, ``merge_duplicate_entities``,
``generate_user_summary``) use upsert semantics or skip-already-processed
checks specific to their domain.

.. note::

   There is **no at-least-once delivery guarantee** in ARQ.  If the worker
   crashes mid-job, the job is lost.  The reconciliation cron job is the
   recovery mechanism for enrichment tasks.  For business-critical tasks,
   consider adding a separate dead-letter queue or using a more durable
   queue backend.


Enrichment Pipeline Flow
------------------------

The episode enrichment pipeline follows this sequence:

1. **Episode committed** — The API server creates an ``Episode`` row in
   PostgreSQL (``enrichment_status = 0``).

2. **Combined enrichment enqueued** — The API enqueues ``enrich_episode``
   on the **high-priority queue** (or, for backward compatibility, individual
   tasks like ``extract_entities``).

3. **LLM enrichment** (bits 0, 2, 4, 5) — ``enrich_episode`` makes a single
   LLM call with ``CombinedLLMOutput`` schema, then processes classification,
   entities, facts, and structured extraction in independent savepoints.

4. **Episode embedding** (bit 1) — ``embed_episode`` generates a pgvector
   embedding for the episode content.  This runs **after** LLM enrichment
   so the embedding can be used for semantic search over enriched episodes.

5. **Entity–episode linking** (bit 3) — ``link_entities_to_episode`` links
   extracted entities to the episode in the graph backend.  This is a
   **low-priority** task because the PostgreSQL data is already authoritative
   — the graph is a secondary index for traversal.

6. **Deferred observations** (bit 6) — ``compute_observations`` runs
   project-wide pattern detection.  This is **non-blocking** — the episode
   is already queryable by enrichment at this point.

The pipeline is **not strictly linear** — individual tasks can be enqueued
independently, and the reconciliation cron handles any gaps.

.. code-block:: text

   Episode Committed
         │
         ▼
   ┌───────────────────────────────┐
   │   enrich_episode              │  ← Single LLM call, savepoint isolation
   │   (bits 0, 2, 4, 5)           │     or 4 individual tasks (legacy)
   └───────────┬───────────────────┘
               │
               ▼
   ┌───────────────────────────────┐
   │   embed_episode               │  ← pgvector, per-org embedding config
   │   (bit 1)                     │
   └───────────┬───────────────────┘
               │
               ▼
   ┌───────────────────────────────┐
   │   link_entities_to_episode    │  ← Low-priority, graph index
   │   (bit 3)                     │
   └───────────┬───────────────────┘
               │
               ▼
   ┌───────────────────────────────┐
   │   compute_observations        │  ← Deferred, non-blocking
   │   (bit 6, not in ALL)         │
   └───────────────────────────────┘


Scaling Model
-------------

Concurrency
~~~~~~~~~~~

Concurrency is controlled by ``MAX_WORKERS`` (default ``4``, range ``1–32``):

* **High-priority pool**: ``min(MAX_WORKERS, 8)`` — bounded at 8 to prevent
  overloading the LLM provider with concurrent requests.
* **Low-priority pool**: ``max(1, MAX_WORKERS // 4)`` — one low-priority slot
  per four high-priority slots.

For I/O-bound tasks (LLM calls, DB queries, embedding), concurrency can be
increased to 8–16 without significant CPU pressure.  For CPU-bound work,
set ``MAX_WORKERS`` to the number of available CPU cores.

The two pools run in the same event loop — they share the asyncio event loop
but have independent ``max_jobs`` limits.  If the high-priority pool is at
capacity, the low-priority pool still accepts jobs (and vice versa).


Resource Injection
~~~~~~~~~~~~~~~~~~

The following shared resources are created once per worker process and
injected into every task invocation:

* **DB engine**: ``core.db`` async engine with ``pool_size=10``,
  ``max_overflow=5``.  This means up to 15 concurrent PostgreSQL connections
  per worker process.  Scale the number of worker replicas and PostgreSQL's
  ``max_connections`` accordingly.
* **Redis**: ARQ manages its own Redis connection per pool.  Each worker pool
  opens one persistent Redis connection for listening, plus one for job
  result writes.  Total: ~4 Redis connections per worker process.
* **FalkorDB**: Optional.  One connection pool (``max_connections=10``,
  default) shared across all tasks.
* **SurrealDB**: Lazy per-org connection pool.  Connections are created on
  demand and cached within the worker process.
* **OpenBao client**: One persistent authenticated client shared across all
  tasks for org config resolution.


Deployment Sizing
~~~~~~~~~~~~~~~~~

============ ========== =========== =========== ===========
Environment  MAX_WORKERS PostgreSQL  Redis       Notes
             (per pod)  connections connections
============ ========== =========== =========== ===========
Development  2          30 (shared)  4 (shared)  Single pod
Staging      4          60           8           1 pod
Production   8–16       120–240      12–24       2+ pods
============ ========== =========== =========== ===========

Each additional worker pod adds 15 DB connections and ~4 Redis connections.
PostgreSQL's ``max_connections`` should be set to
``(pod_count × 15) + API_server_connections + 20% headroom``.


Monitoring
----------

Prometheus Metrics
~~~~~~~~~~~~~~~~~~

The worker exposes the following metrics on ``PROMETHEUS_PORT`` (default 9095):

=========================================== ========== ================================================
Metric                                     Type       Labels
=========================================== ========== ================================================
``openzync_worker_tasks_total``             Counter    ``task_type``, ``status`` (``"success"`` / ``"failure"``)
``openzync_worker_task_duration_seconds``   Histogram  ``task_type`` (buckets: 1, 2.5, 5, 10, 15, 30, 60, 120, 300, 600)
``openzync_worker_queue_depth``             Gauge      ``queue_name``
``openzync_worker_tasks_per_org_total``     Counter    ``org_id``, ``task_type``, ``status``
=========================================== ========== ================================================

Health Check
~~~~~~~~~~~~

Available on ``HEALTH_PORT`` (default 8081):

* ``GET /health`` — Returns ``{"status": "ok", "redis_connected": true}`` or
  ``{"status": "unhealthy", ...}`` with HTTP 503.
* ``GET /ready`` — Same as ``/health`` (for Kubernetes readiness probe).

Both endpoints verify Redis connectivity via ``PING``.


Structured Logging
~~~~~~~~~~~~~~~~~~

The worker uses ``structlog`` with a JSON renderer in production.  Every log
entry is automatically enriched with ``timestamp``, ``level``, ``logger``
(``OpenZync.worker``), and per-task context (``trace_id``, ``org_id``,
``task_type``, ``job_id``) bound via ``structlog.contextvars.bind_contextvars``.

In development, ``STRUCTLOG_FORMAT=console`` produces human-readable output.
In production, JSON output is ingested by Loki for centralised log
aggregation.


Job Lifecycle Callbacks
~~~~~~~~~~~~~~~~~~~~~~~

The worker registers two ARQ callbacks:

* ``on_job_end`` — Logs completion with duration, emits Prometheus metrics.
* ``on_shutdown`` — Closes the persistent OpenBao client, logs shutdown
  completion.

See :func:`services.worker.worker.on_job_end` and
:func:`services.worker.worker.on_shutdown`.

Queue depth is sampled every 15 seconds in a background :class:`asyncio.Task`
and exposed via the ``openzync_worker_queue_depth`` gauge.


Adding a New Task
-----------------

To add a new worker task:

1. **Create the task module** in ``workers/tasks/{task_name}.py``::

       @with_retry(max_retries=3, base_delay_s=2.0)
       async def my_new_task(ctx, episode_id, org_id, **kwargs):
           """Docstring following Google style."""
           ...

   Follow the standard DB engine bootstrap pattern (prefer shared engine
   from ``ctx["db_engine"]``, fall back to short-lived engine for testability).

2. **Export from the tasks package** — add the new function to
   ``workers/tasks/__init__.py``::

       from workers.tasks.my_new_task import my_new_task

       __all__ = [
           ...,
           "my_new_task",
       ]

3. **Register with the worker** — import in ``services/worker/worker.py``
   and add to the appropriate queue list::

       from workers.tasks.my_new_task import my_new_task

       HIGH_QUEUE_TASKS = [
           ...,
           my_new_task,
       ]

4. **Assign an enrichment bit** (if applicable) — add a new constant to
   ``workers/tasks/base.py`` and include it in ``ENRICHMENT_ALL`` if the
   step is required for "fully enriched" status.

5. **Enqueue from the API or service layer**::

       from core.arq import get_arq

       job_id = await get_arq().enqueue(
           "my_new_task",
           queue_name="high" if is_realtime else "low",
           episode_id=str(episode.id),
           org_id=str(org.org_id),
           ...
       )

6. **Write tests** — see :doc:`/guides/contributing` for testing conventions.
   Mock the LLM backend, use a test PostgreSQL via ``asyncpg``, and test both
   the success path and the idempotency skip path.
