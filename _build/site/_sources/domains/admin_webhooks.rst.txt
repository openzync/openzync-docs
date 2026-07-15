Admin & Webhooks Domain
=======================

.. note::

   This document covers the Admin and Webhooks domain **within the OpenZync
   monolith** (``openzync-core``).  Code examples assume the relevant packages
   are importable from the monolith's Python path.

   The Admin domain spans: organization bootstrap and management, per-org
   configuration (LLM, embeddings, graph, behaviour), prompt template management,
   custom instructions, extraction schema CRUD, structured extraction queries,
   audit logging, Prometheus metrics, webhook endpoint management and event
   delivery, and transactional email.

   **Design principles**:

   * Multi-tenant isolation via ``organization_id`` on every query — all admin
     endpoints are scoped to the authenticated organization.
   * Per-org configuration is stored in OpenBao and cached in Redis — no
     env-var fallback, static defaults shipped via YAML.
   * Webhook delivery is async (ARQ background jobs) with HMAC-SHA256 signing
     and retry semantics.
   * Audit logs are append-only — ``AuditLog`` and ``LLMUsage`` models are
     immutable by design (``updated_at`` intentionally absent).
   * Metrics expose no PII — the Prometheus ``/metrics`` endpoint is
     intentionally unauthenticated.
   * The bootstrap endpoint (``POST /admin/organizations``) has no
     authentication — it is designed for first-use flow before any API keys
     exist.  In production it should be disabled or gated behind a
     deployment-time secret.

.. contents:: Sections
   :local:
   :depth: 3
   :class: this-will-duplicate-information-and-it-is-still-useful-here


Core Concepts
-------------

.. _admin-multi-tenancy:

Multi-Tenancy Model
~~~~~~~~~~~~~~~~~~~

OpenZync uses a three-level hierarchy for tenant isolation:

::

    Organization (tenant)
        └─ Project (logical workspace)
            └─ User (identity, scoped to organization)

**Organization** (``models.organization.Organization``) is the top-level tenant
entity.  Every entity in the system is scoped to an ``organization_id``.
Organizations have a billing plan (``free``, ``pro``, ``enterprise``) enforced
by a ``CheckConstraint`` on the ``plan`` column.

**Projects** are logical workspaces within an organization.  API keys are scoped
to a single project, not to the entire organization.  At bootstrap time a
default project is created automatically.

**Users** are identities scoped to an organization.  Dashboard users authenticate
via JWT; SDK clients authenticate via project-scoped API keys
(``oz_live_`` / ``oz_test_`` prefix).

PostgreSQL Row-Level Security (RLS) enforces tenant isolation at the database
level, and every repository method accepts an ``organization_id`` parameter.

.. _admin-per-org-config:

Per-Organization Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Organizations can configure their own LLM backend, embedding model, graph
backend, and behavioural settings through the ``organizations.config`` JSONB
column and the per-org OpenBao namespace.

**Config resolution order** (defined in ``core.org_config``):

1. **Redis cache** (key ``org_config:{org_id}``, TTL 5 min) — performance
   optimisation.  Cache failures are logged at ERROR but the request
   continues to OpenBao.
2. **OpenBao KV** (per-org namespace ``org_<uuid>/config/``) — authoritative
   source.  OpenBao failures propagate as hard errors.

There is **no** env-var fallback — if a field is not set in OpenBao it is
returned as ``None`` and the caller decides what to do (typically falling back
to built-in defaults in ``core.config.Settings``).

The config schema (``schemas.organization_config.OrgConfigBase``) covers:

* **LLM**: ``llm_backend``, ``llm_model``, ``llm_temperature``,
  ``llm_max_tokens``, and provider API keys (OpenAI, Azure, Anthropic,
  OpenRouter, Ollama).
* **Embeddings**: ``embedding_backend``, ``embedding_model``, ``embedding_dim``.
* **Graph**: ``graph_backend`` (default ``surrealdb``), ``graph_search_type``,
  ``graph_max_traversal_depth``, and SurrealDB connection details.
* **Behaviour**: ``context_cache_ttl``, ``audit_log_response_body``.
* **Re-ranker**: ``reranker_backend``, ``reranker_model``, ``reranker_top_k``,
  ``reranker_top_n``, ``cohere_api_key``.

On config update (``PATCH`` / ``PUT``), the cache is invalidated inline.
Invalidation failures are logged at ERROR but do not fail the operation
(stale cache expires via TTL).

.. _admin-webhook-semantics:

Webhook Delivery & Retry Semantics
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Webhook delivery is fully asynchronous.  When a service calls
``WebhookService.emit()``:

1. The service finds all active endpoints subscribed to the given event type.
2. An HMAC-SHA256 signature is computed over the payload body.
3. An ARQ job (``deliver_webhook``) is enqueued on the **low-priority**
   queue for each matching endpoint.
4. The ARQ worker performs the HTTP ``POST`` with retry logic.

**Signing**: ``sign_payload()`` returns a Svix-compatible signature header::

    t=<unix_timestamp>,v1=<hex_signature>

The signature is HMAC-SHA256 of ``"<timestamp>.<body>"`` using the global
``WEBHOOK_SIGNING_SECRET``.  Consumers verify by recomputing the same HMAC
and comparing it with the ``v1`` value.

**Retry policy**: (implemented in the ARQ worker — the service layer defines
attempt tracking in ``WebhookDeliveryLog``.)  Every delivery attempt creates
a ``WebhookDeliveryLog`` row recording the attempt number, HTTP status code,
success/failure, and error message.

**Idempotency**: Events are not re-emitted by the service layer on failure.
The ARQ worker handles retries.  The ``(episode_id, schema_id)`` unique
constraint on ``StructuredExtraction`` prevents duplicate extraction records.

**Shared secret**: All endpoints share the same global signing secret
(``OZ_WEBHOOK_SIGNING_SECRET`` env var).  Rotating the secret cycles all
consumers.  The secret is returned once at endpoint creation time.

.. _admin-webhook-event-types:

Webhook Event Types
^^^^^^^^^^^^^^^^^^^

Every meaningful action maps to an event type string following the pattern
``{domain}.{action}``:

.. list-table::
   :header-rows: 1

   * - Event Type Constant
     - String
     - Category
     - Description
   * - ``SESSION_CREATED``
     - ``session.created``
     - Session
     - Fired when a new conversation session is created
   * - ``SESSION_CLOSED``
     - ``session.closed``
     - Session
     - Fired when a session is closed
   * - ``MESSAGE_ADDED``
     - ``message.added``
     - Message
     - Fired when a message is added to a session
   * - ``EPISODE_PROCESSED``
     - ``episode.processed``
     - Graph
     - Fired when an episode finishes processing into the graph
   * - ``INGEST_BATCH_COMPLETED``
     - ``ingest.batch.completed``
     - Graph
     - Fired when a batch ingestion operation completes
   * - ``INGEST_EPISODE_COMPLETED``
     - ``ingest.episode.completed``
     - Graph
     - Fired when a single-episode ingestion completes
   * - ``GRAPH_ENTITY_CREATED``
     - ``graph.entity.created``
     - Graph
     - Fired when a new graph entity (node) is created
   * - ``GRAPH_ENTITY_UPDATED``
     - ``graph.entity.updated``
     - Graph
     - Fired when a graph entity is updated
   * - ``GRAPH_EDGE_CREATED``
     - ``graph.edge.created``
     - Graph
     - Fired when a relationship edge is created between entities
   * - ``FACT_EXTRACTED``
     - ``fact.extracted``
     - Fact
     - Fired when a fact (triple) is extracted
   * - ``FACT_DELETED``
     - ``fact.deleted``
     - Fact
     - Fired when a fact is deleted
   * - ``CLASSIFICATION_CREATED``
     - ``classification.created``
     - Classification
     - Fired when a dialog classification is created
   * - ``EXTRACTION_CREATED``
     - ``extraction.created``
     - Extraction
     - Fired when a structured extraction is created
   * - ``USER_CREATED``
     - ``user.created``
     - User
     - Fired when a new user is created

.. _admin-audit-log:

Audit Log Structure
~~~~~~~~~~~~~~~~~~~

The ``AuditLog`` model (``models.audit_log.AuditLog``) is an **immutable,
append-only** record of security-relevant events.  Key characteristics:

* **No ``updated_at``** — the model intentionally inherits ``CreatedAtMixin``
  instead of ``TimestampMixin``, enforcing immutability at the schema level.
* **No UPDATE or DELETE** is permitted at the application layer.
* **Actor types** are constrained to ``user``, ``api_key``, or ``system`` via
  a ``CheckConstraint`` on the ``actor_type`` column.

Each entry captures:

* ``organization_id`` — nullable for unauthenticated actions (e.g. failed login).
* ``actor_id`` / ``actor_type`` — who did it (user UUID, API key prefix, or
  system component name).
* ``action`` — what was done (e.g. ``session.create``, ``api_key.revoke``).
* ``resource_type`` / ``resource_id`` — what was affected.
* ``details`` — arbitrary JSONB payload with action-specific context.
* ``ip_address`` — source IP for the request.

.. _admin-metrics:

Metrics Collection
~~~~~~~~~~~~~~~~~~

Metrics are collected through two independent systems:

**Prometheus** (via the Python ``prometheus_client`` library):

* Exposed at ``GET /metrics`` (unauthenticated — Prometheus scrapers cannot
  carry bearer tokens).  Uses an isolated ``METRICS_REGISTRY`` that excludes
  default process/GC metrics to keep the payload lean.
* Histograms track HTTP request duration (``openzync_http_request_duration_seconds``),
  context pipeline latency (``openzync_context_latency_seconds``), and graph
  search latency (``openzync_graph_search_latency_seconds``).
* Counters track total requests by status class (2xx, 4xx, 5xx) and
  in-flight requests.
* Gauges expose ARQ worker queue depth by priority.

**SQL queries** (scoped to organization):

* Episode stats: total, last-24h, enrichment pipeline status (pending,
  in-progress, fully-enriched, with-embeddings).
* Graph stats: total entities, entities added in last 24h.
* User count, session count, fact count, API key count.

**LLM Usage tracking** (``models.llm_usage.LLMUsage``):

* Every LLM inference call creates an immutable row.
* Tracks ``prompt_tokens``, ``completion_tokens``, ``total_tokens`` (generated
  column: ``prompt_tokens + completion_tokens``), ``cost_estimate`` (``Numeric(12, 8)``
  USD), and ``duration_ms``.
* ``organization_id`` is denormalized for fast aggregation queries.


Data Models
-----------

Organization
~~~~~~~~~~~~

.. module:: models.organization

.. class:: Organization

   The top-level tenant entity.  Each organization owns users, API keys,
   extraction schemas, and billing config.

   .. attribute:: id

      UUID primary key, generated server-side via ``gen_random_uuid()``.
      Type: ``Mapped[uuid.UUID]``.

   .. attribute:: name

      Human-readable organization name.  Type: ``Mapped[str]`` (Text,
      nullable=False).

   .. attribute:: plan

      Billing plan.  One of ``free``, ``pro``, ``enterprise``.  Enforced via
      ``CheckConstraint('plan IN (\'free\', \'pro\', \'enterprise\')')``.
      Type: ``Mapped[str]`` (String(20), default ``"free"``).

   .. attribute:: config

      JSONB blob for all per-org UI-exposed configuration (LLM, embeddings,
      graph, behaviour).  ``None`` fields fall back to env-var defaults from
      ``core.config.Settings``.  Type: ``Mapped[dict]`` (JSONB,
      default ``{}``).

   .. attribute:: llm_config

      **Deprecated** — kept for backward compatibility during migration.
      Reads/writes alias to ``config->'llm'``.  Prefer ``config`` for new
      code.  Type: ``Mapped[dict]`` (JSONB, default ``{}``).

   .. attribute:: quotas

      JSONB blob for usage quotas (max_sessions, max_episodes, etc.).
      Type: ``Mapped[dict]`` (JSONB, default ``{}``).

   .. attribute:: is_active

      Soft toggle for deactivation.  Type: ``Mapped[bool]`` (Boolean,
      default ``True``).

   Inherits ``created_at`` and ``updated_at`` from ``TimestampMixin``.

.. module:: models.audit_log

AuditLog
~~~~~~~~

.. class:: AuditLog

   An immutable audit trail entry.  Inherits ``CreatedAtMixin`` (no
   ``updated_at`` — intentionally append-only).

   .. attribute:: id

      UUID primary key, generated server-side via ``gen_random_uuid()``.

   .. attribute:: organization_id

      Optional — may be ``None`` for unauthenticated actions.
      Type: ``Mapped[uuid.UUID | None]``.

   .. attribute:: actor_id

      Identifier of the acting entity (user ID, API key prefix, or system
      name).  Type: ``Mapped[str | None]`` (Text).

   .. attribute:: actor_type

      Type of actor — one of ``user``, ``api_key``, ``system``.  Enforced
      via ``CheckConstraint('actor_type IN (\'user\', \'api_key\', \'system\')')``.
      Type: ``Mapped[str | None]`` (Text).

   .. attribute:: action

      The action performed (e.g. ``session.create``, ``api_key.revoke``).
      Type: ``Mapped[str]`` (Text, nullable=False).

   .. attribute:: resource_type

      Type of resource affected (e.g. ``session``, ``fact``).
      Type: ``Mapped[str]`` (Text, nullable=False).

   .. attribute:: resource_id

      Identifier of the affected resource (nullable for collection-level
      actions).  Type: ``Mapped[str | None]`` (Text).

   .. attribute:: details

      Arbitrary JSONB payload with action-specific context.
      Type: ``Mapped[dict]`` (JSONB, default ``{}``).

   .. attribute:: ip_address

      Source IP address of the request.
      Type: ``Mapped[str | None]`` (Text).

   .. attribute:: created_at

      Immutable timestamp of the event (inherited from ``CreatedAtMixin``).

.. module:: models.llm_usage

LLMUsage
~~~~~~~~

.. class:: LLMUsage

   A single LLM inference usage record.  Append-only — rows are inserted once
   and never modified.  The ``total_tokens`` column is a **generated column**
   computed by PostgreSQL as ``prompt_tokens + completion_tokens``.

   .. attribute:: id

      UUID primary key, generated server-side via ``gen_random_uuid()``.

   .. attribute:: organization_id

      Owning organization (denormalized for fast aggregation queries).
      Type: ``Mapped[uuid.UUID]`` (nullable=False).

   .. attribute:: model

      Model identifier (e.g. ``gpt-4o``, ``claude-sonnet-4``).
      Type: ``Mapped[str]`` (Text, nullable=False).

   .. attribute:: task_type

      Type of task (e.g. ``chat.completion``, ``embedding``,
      ``classification``).  Type: ``Mapped[str]`` (Text, nullable=False).

   .. attribute:: prompt_tokens

      Number of tokens in the prompt.
      Type: ``Mapped[int]`` (Integer, default ``0``).

   .. attribute:: completion_tokens

      Number of tokens in the completion.
      Type: ``Mapped[int]`` (Integer, default ``0``).

   .. attribute:: total_tokens

      **Generated column** — always equals ``prompt_tokens + completion_tokens``.
      Computed and stored by PostgreSQL; cannot be written directly.
      Type: ``Mapped[int]`` (Integer, ``Computed("prompt_tokens + completion_tokens")``).

   .. attribute:: cost_estimate

      Estimated cost in USD (12 digits, 8 decimal places).
      Type: ``Mapped[Decimal]`` (``Numeric(12, 8)``, default ``0``).

   .. attribute:: duration_ms

      Wall-clock duration of the inference call in milliseconds.
      Type: ``Mapped[int]`` (Integer, default ``0``).

   .. attribute:: created_at

      Immutable timestamp (inherited from ``CreatedAtMixin``).

.. module:: models.webhook

WebhookEndpoint
~~~~~~~~~~~~~~~

.. class:: WebhookEndpoint

   A webhook endpoint configured by an organization.  Each row represents an
   HTTPS endpoint that receives POST requests for subscribed event types.
   Inherits ``TimestampMixin``.

   .. attribute:: id

      UUID primary key, generated server-side via ``gen_random_uuid()``.

   .. attribute:: organization_id

      Foreign key to the owning organization
      (``ForeignKey("organizations.id", ondelete="CASCADE")``).
      Indexed via ``ix_webhook_endpoints_org``.

   .. attribute:: name

      Human-readable label (e.g. "Production Slack").
      Type: ``Mapped[str]`` (Text, nullable=False).

   .. attribute:: url

      HTTPS endpoint URL that receives POST requests.
      Type: ``Mapped[str]`` (Text, nullable=False).

   .. attribute:: events

      JSON array of subscribed event type strings, e.g.
      ``["session.created", "fact.extracted"]``.
      Type: ``Mapped[str]`` (Text, nullable=False).

   .. attribute:: is_active

      Whether this endpoint is currently accepting deliveries.
      Type: ``Mapped[bool]`` (Boolean, default ``True``).

   .. attribute:: last_delivery_at

      Timestamp of the most recent delivery attempt.
      Type: ``Mapped[datetime | None]``.

WebhookDeliveryLog
~~~~~~~~~~~~~~~~~~

.. class:: WebhookDeliveryLog

   Log of a webhook delivery attempt.  Every attempt (including retries)
   creates a row for observability.  Inherits ``TimestampMixin``.

   .. attribute:: id

      UUID primary key.

   .. attribute:: endpoint_id

      Foreign key to ``webhook_endpoints.id``
      (``ForeignKey("webhook_endpoints.id", ondelete="CASCADE")``).
      Indexed.

   .. attribute:: event_type

      The event type string delivered (e.g. ``session.created``).
      Type: ``Mapped[str]`` (Text, nullable=False).

   .. attribute:: attempt

      Attempt number (0-based).  Type: ``Mapped[int]``, default ``0``.

   .. attribute:: status_code

      HTTP status code from the delivery attempt.
      Type: ``Mapped[int | None]``.

   .. attribute:: success

      Whether the delivery was successful.
      Type: ``Mapped[bool]`` (Boolean, default ``False``).

   .. attribute:: error

      Error message if the delivery failed.
      Type: ``Mapped[str | None]`` (Text).

.. module:: models.extraction_schema

ExtractionSchema
~~~~~~~~~~~~~~~~

.. class:: ExtractionSchema

   A named JSON Schema definition for structured extractions or classification
   schemas.  Each organization maintains its own catalog.  Inherits
   ``TimestampMixin``.

   .. attribute:: id

      UUID primary key.

   .. attribute:: organization_id

      Foreign key to the owning organization
      (``ForeignKey("organizations.id", ondelete="CASCADE")``).

   .. attribute:: name

      Human-readable schema name, unique within an organization (enforced via
      ``UniqueConstraint("organization_id", "name")``).
      Type: ``Mapped[str]`` (Text, nullable=False).

   .. attribute:: type

      Schema type — ``'structured'`` (default) or ``'classification'``.
      Type: ``Mapped[str]`` (String(50), default ``"structured"``).

   .. attribute:: json_schema

      The JSON Schema definition that extraction payloads must conform to.
      For ``type='classification'``, this stores the label definitions
      (intent, emotion, valence, arousal options).
      Type: ``Mapped[dict]`` (JSONB, nullable=False).

   .. attribute:: prompt_template

      Optional organization-specific prompt override for guiding the LLM
      extraction.  Type: ``Mapped[str | None]`` (Text).

   .. attribute:: is_active

      Soft toggle — inactive schemas are not available for new extractions
      but existing references are preserved.
      Type: ``Mapped[bool]`` (Boolean, default ``True``).

.. module:: models.structured_extraction

StructuredExtraction
~~~~~~~~~~~~~~~~~~~~

.. class:: StructuredExtraction

   A single structured extraction result scoped to an episode and schema.
   The ``(episode_id, schema_id)`` pair is unique, ensuring idempotent
   re-processing.  Inherits ``TimestampMixin``.

   .. attribute:: id

      UUID primary key.

   .. attribute:: project_id

      Denormalized FK to ``projects.id`` (``ondelete="CASCADE"``) for
      efficient project-scoped queries without joining through episode.
      Indexed.

   .. attribute:: session_id

      Foreign key to the session this extraction belongs to
      (``ForeignKey("sessions.id", ondelete="CASCADE")``).

   .. attribute:: episode_id

      Foreign key to the episode that triggered this extraction
      (``ForeignKey("episodes.id", ondelete="CASCADE")``).  Indexed.

   .. attribute:: schema_id

      Optional FK to ``extraction_schemas.id``
      (``ForeignKey("extraction_schemas.id", ondelete="SET NULL")``).
      Nullable to allow ad-hoc extractions without a schema definition.

   .. attribute:: data

      The extracted JSONB payload, conforming to the schema definition.
      Type: ``Mapped[dict]`` (JSONB, nullable=False).

   Unique constraint: ``("episode_id", "schema_id")`` via
   ``uq_structured_extraction_episode_schema``.


Services
--------

OrganizationService
~~~~~~~~~~~~~~~~~~~

.. module:: services.organization_service

.. class:: OrganizationService

   Business logic for organization bootstrap and management.  Separated from
   main domain services because the bootstrap flow (creating the first
   organization + API key) has no authentication requirement and runs before
   any user exists.

   .. method:: create_organization(payload: CreateOrgRequest) -> CreateOrgResponse

      Create a new organization with a default project and admin API key.
      Performs a single atomic transaction:

      1. Creates an ``Organization`` record.
      2. Creates a default project scoped to the organization.
      3. Generates a ``oz_live_`` API key scoped to the default project.
      4. Seeds default prompt templates for the new org.
      5. Bootstrap OpenBao namespace + default config (non-fatal — failures
         are logged but do not prevent org creation).
      6. Commits everything atomically.

      The raw API key is returned **exactly once** in the response — only the
      salted SHA-256 hash is persisted.

      :param payload: Organization name and optional plan.
      :type payload: :class:`schemas.organizations.CreateOrgRequest`
      :returns: Org details and the raw API key.
      :rtype: :class:`schemas.organizations.CreateOrgResponse`

   .. method:: _load_org_defaults() -> dict[str, Any]

      Load default per-org config values from
      ``config/defaults/org_config.yaml``.  Returns ``{}`` if the file is
      missing or unreadable.  Errors are logged at WARNING level.

      :returns: Flat dict of key/value pairs.
      :rtype: dict

OrgConfigService
~~~~~~~~~~~~~~~~

.. module:: services.org_config_service

.. class:: OrgConfigService

   Business logic for per-organization configuration management.  Orchestrates
   the config update flow: validate → delegate to ``core.org_config`` for the
   OpenBao update + cache invalidation → return stored config.

   .. method:: get_config(org_id: UUID) -> OrgConfigBase

      Return the stored config for an organization.

      :param org_id: The organization UUID.
      :type org_id: :class:`uuid.UUID`
      :returns: An :class:`schemas.organization_config.OrgConfigBase` with
                only explicitly stored fields.  Unset fields are ``None``.

   .. method:: get_config_response(org_id: UUID) -> OrgConfigResponse

      Return the stored config wrapped in an ``OrgConfigResponse``.

      :param org_id: The organization UUID.
      :type org_id: :class:`uuid.UUID`
      :returns: An :class:`schemas.organization_config.OrgConfigResponse`.

   .. method:: update_config(org_id: UUID, payload: UpdateOrgConfigRequest) -> OrgConfigBase

      Partially update an org's configuration.  Only fields explicitly set
      in *payload* are updated.  Fields set to ``None`` are removed from
      the stored config.  The cache is invalidated after the update.

      :param org_id: The organization UUID.
      :param payload: The fields to update.
      :type payload: :class:`schemas.organization_config.UpdateOrgConfigRequest`
      :returns: The freshly stored config after the update.
      :rtype: :class:`schemas.organization_config.OrgConfigBase`

AuditLogService
~~~~~~~~~~~~~~~

.. module:: services.audit_log_service

.. class:: AuditLogService

   Service for recording and querying audit log entries.

   .. method:: log_action(
        *,
        organization_id: UUID | None = None,
        actor_id: str | None = None,
        actor_type: str | None = None,
        action: str,
        resource_type: str,
        resource_id: str | None = None,
        details: dict | None = None,
        ip_address: str | None = None,
      ) -> None

      Record an audit log entry.  Validates that ``actor_type`` is one of
      ``user``, ``api_key``, or ``system``.

      :raises ValueError: If ``actor_type`` is not a valid value.

   .. method:: query_logs(
        organization_id: UUID | None,
        *,
        action: str | None = None,
        actor_id: str | None = None,
        actor_type: str | None = None,
        resource_type: str | None = None,
        resource_id: str | None = None,
        status_code: int | None = None,
        created_after: str | None = None,
        created_before: str | None = None,
        limit: int = 50,
        offset: int = 0,
      ) -> tuple[list, int]

      Query audit log entries with optional filters.  Returns a tuple of
      ``(list of AuditLog ORM objects, total_count)``.

      :param organization_id: Filter by organization (from auth context).
      :param action: Exact-match filter on action.
      :param actor_id: Exact-match filter on actor.
      :param actor_type: Exact-match filter on actor type.
      :param resource_type: Exact-match filter on resource type.
      :param resource_id: Exact-match filter on resource ID.
      :param status_code: Filter by HTTP status code.
      :param created_after: ISO 8601 — include entries after this.
      :param created_before: ISO 8601 — include entries before this.
      :param limit: Max entries per page (default 50).
      :param offset: Pagination offset (default 0).
      :returns: Tuple of (list of ORM objects, total count).
      :rtype: tuple[list, int]

MetricsService
~~~~~~~~~~~~~~

.. module:: services.metrics_service

.. class:: MetricsService

   Aggregate metrics from Prometheus for the admin dashboard.  Runs multiple
   PromQL queries concurrently and returns a frontend-friendly JSON shape.
   If Prometheus is unreachable or any query fails,
   ``MetricsUnavailableError`` is raised.

   **PromQL queries**:

   * **Latency** (histogram_quantile at p50/p95/p99):
     - ``openzync_http_request_duration_seconds`` → ``overall_latency_ms``
     - ``openzync_context_latency_seconds`` → ``context_latency_ms``
     - ``openzync_graph_search_latency_seconds`` → ``graph_search_latency_ms``
   * **Request rate**: 2xx/4xx/5xx per second over 5m window.
   * **Error rate**: Percentage of 5xx errors.
   * **Counters**: Total requests, active requests.
   * **Queue depth**: ARQ worker queues for ``high`` and ``low`` priority.

   .. method:: get_summary() -> MetricsSummaryResponse

      Run all PromQL queries and assemble the response.

      :returns: A fully populated :class:`schemas.admin_metrics.MetricsSummaryResponse`.
      :raises MetricsUnavailableError: If Prometheus is unreachable or any
                                       query fails.

WebhookService
~~~~~~~~~~~~~~

.. module:: services.webhook_service

.. class:: WebhookService

   Manages webhook endpoints and emits events via ARQ background jobs.

   **Endpoint management** (CRUD):

   .. method:: list_endpoints(organization_id: UUID) -> list[dict]

      List all webhook endpoints for an organization.

   .. method:: get_endpoint(endpoint_id: UUID, organization_id: UUID) -> dict | None

      Get a single webhook endpoint by ID, verifying ownership.

   .. method:: create_endpoint(organization_id: UUID, name: str, url: str, events: list[str] | None = None) -> tuple[dict, str]

      Create a webhook endpoint.  Returns a tuple of ``(endpoint_dict,
      global_signing_secret)``.  The global ``WEBHOOK_SIGNING_SECRET`` is
      returned so the consumer can verify HMAC-SHA256 signatures.

   .. method:: update_endpoint(endpoint_id: UUID, organization_id: UUID, updates: Mapping[str, object]) -> dict | None

      Update a webhook endpoint.  Returns updated endpoint or ``None`` if
      not found or not owned.

   .. method:: toggle_endpoint(endpoint_id: UUID, organization_id: UUID, is_active: bool) -> dict | None

      Enable or disable a webhook endpoint.

   .. method:: delete_endpoint(endpoint_id: UUID, organization_id: UUID) -> bool

      Delete a webhook endpoint.  Returns ``True`` if deleted.

   **Event emission**:

   .. method:: emit(organization_id: UUID, event_type: str, payload: dict | None = None) -> None

      Emit an event to all subscribed webhook endpoints via ARQ.  Finds
      active endpoints subscribed to ``event_type`` and enqueues a
      ``deliver_webhook`` job for each.  Delivery is async — errors are
      logged but never propagated to the caller.

      :param organization_id: The organization emitting the event.
      :param event_type: The event type string (e.g. ``session.created``).
      :param payload: Optional event payload dict.

   **Utility functions**:

   .. function:: services.webhook_service.sign_payload(secret: str, payload: bytes) -> str

      Return a Svix-compatible HMAC-SHA256 signature header value in the
      format ``t=<unix_timestamp>,v1=<hex_signature>``.

      :param secret: The shared signing secret (``whsec_``-prefixed).
      :param payload: The raw JSON body to sign.
      :returns: A signature string suitable for the ``X-Webhook-Signature`` header.

EmailService
~~~~~~~~~~~~

.. module:: services.email_service

.. class:: EmailService

   Async SMTP email delivery service.  Uses ``aiosmtplib`` for non-blocking
   SMTP communication, and Jinja2 to render email body templates stored in
   ``prompts/email/``.

   Creates a fresh SMTP connection per message (KISS — transactional email
   volume is low).

   .. method:: send_email(to: str, subject: str, html_body: str, text_body: str | None = None) -> None

      Send an email via SMTP.  Creates a fresh SMTP connection, authenticates
      (if credentials are configured), sends the message, and quits.

      :param to: Recipient email address.
      :param subject: Email subject line.
      :param html_body: Rendered HTML body.
      :param text_body: Optional plain-text fallback.  If ``None``, the
                        ``EmailMessage`` builder will auto-strip the HTML.
      :raises ExternalServiceError: If the SMTP server cannot be reached or
                                    the message cannot be sent.

   **Template rendering functions**:

   .. function:: services.email_service.render_email_template(template_name: str, context: dict | None = None) -> str

      Render a Jinja2 email template with the given context.  Template files
      live in ``prompts/email/`` with the naming pattern
      ``{template_name}.html.jinja2``.

      :param template_name: Template filename without extension (e.g. ``"otp"``).
      :param context: Variables to inject into the template.
      :returns: Rendered HTML string.
      :raises ExternalServiceError: If the template file is missing or invalid.

   .. function:: services.email_service.render_text_template(template_name: str, context: dict | None = None) -> str

      Render the plain-text variant of an email template.  Falls back to
      ``{name}.txt.jinja2`` or, if missing, returns an empty string (the
      ``EmailMessage`` builder will strip HTML).

SchemaService
~~~~~~~~~~~~~

.. module:: services.schema_service

.. class:: SchemaService

   Business logic for managing extraction schemas.  Orgs define extraction
   schemas for two purposes:

   1. **Structured extraction** (``type='structured'``) — JSON Schema
      documents that the LLM must conform to when extracting data.
   2. **Classification labels** (``type='classification'``) — label sets
      that define the intent, emotion, valence, and arousal categories for
      dialog classification.

   .. method:: create_schema(org_id: UUID, payload: CreateExtractionSchemaRequest) -> ExtractionSchemaResponse

      Create a new extraction schema for an organization.  Validates schema
      structure based on type.  For ``type='classification'``, validates the
      expected label structure (intent, emotion, valence, arousal must be
      lists of non-empty strings).  For ``type='structured'``, validates
      against JSON Schema draft-07 using the ``jsonschema`` library.

      :raises ConflictError: If a schema with the same name already exists.
      :raises ValidationError: If the payload fails domain validation.

   .. method:: list_schemas(org_id: UUID, schema_type: str | None = None, is_active: bool | None = None) -> list[ExtractionSchemaResponse]

      List schemas for an organization with optional filters.

   .. method:: get_schema(org_id: UUID, schema_id: UUID) -> ExtractionSchemaResponse

      Get a single schema by ID.

      :raises NotFoundError: If the schema does not exist or belongs to
                             another org.

   .. method:: update_schema(org_id: UUID, schema_id: UUID, payload: UpdateExtractionSchemaRequest) -> ExtractionSchemaResponse

      Update an existing schema.  The ``type`` field is immutable after
      creation.  Name uniqueness is enforced within the organization.

      :raises NotFoundError: If the schema does not exist.
      :raises ConflictError: If the new name conflicts with an existing schema.

   .. method:: delete_schema(org_id: UUID, schema_id: UUID) -> None

      Soft-delete a schema (set ``is_active=false``).  Existing extractions
      referencing this schema are preserved (FK uses ``ON DELETE SET NULL``).

      :raises NotFoundError: If the schema does not exist.

   .. method:: _validate_classification_schema(json_schema: dict) -> None

      Validate that *json_schema* has the expected classification shape.
      Expected structure (all keys optional)::

          {
              "intent": ["greeting", "question", ...],
              "emotion": ["joy", "frustration", ...],
              "valence": ["positive", "negative", "neutral"],
              "arousal": ["low", "medium", "high"]
          }

      :raises ValidationError: If the schema structure is invalid.

   .. method:: _validate_json_schema(json_schema: dict) -> None

      Validate that *json_schema* is a valid JSON Schema draft-07 document
      using the ``jsonschema`` library.  This does **not** validate data
      against the schema — only that the schema itself is well-formed.

      :raises ValidationError: If the schema is not a valid JSON Schema.

StructuredExtractionService
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. module:: services.structured_extraction_service

.. class:: StructuredExtractionService

   Read-only service for querying structured extraction results.
   Structured extractions are produced by the ``extract_structured`` worker
   and inserted directly into the database.

   .. method:: get_session_extractions(org_id: UUID, session_id: UUID, project_id: UUID | None = None) -> StructuredExtractionListResponse

      Return all extractions for episodes in a session.  Verifies session
      ownership before returning data.

      :param org_id: The authenticated organization UUID.
      :param session_id: The session UUID.
      :param project_id: Optional project UUID for intra-org isolation.
      :returns: ``StructuredExtractionListResponse`` with items ordered by
                episode sequence number.  May be empty if no extractions exist.
      :raises NotFoundError: If the session does not exist.

   .. method:: get_episode_extraction(org_id: UUID, session_id: UUID, episode_id: UUID, project_id: UUID | None = None) -> StructuredExtractionResponse | None

      Return the extraction for a specific episode, or ``None``.

      :param org_id: The authenticated organization UUID.
      :param session_id: The session UUID.
      :param episode_id: The episode UUID.
      :param project_id: Optional project UUID for intra-org isolation.
      :returns: A ``StructuredExtractionResponse`` or ``None`` if not yet
                extracted.
      :raises NotFoundError: If the session does not exist.


API Endpoints
-------------

Admin Bootstrap
~~~~~~~~~~~~~~~

.. module:: routers.admin

Router prefix: ``/admin``, tags: ``Admin``

.. function:: routers.admin.create_organization(payload: CreateOrgRequest, db: AsyncSession) -> CreateOrgResponse

   **POST /admin/organizations** (status 201)

   Create a new organization and generate an admin API key.  This is a
   bootstrap endpoint for initial setup.  It performs a single atomic
   transaction:

   1. Creates a new ``Organization`` record.
   2. Generates a ``oz_live_`` API key with ``read``, ``write``, and
      ``admin`` scopes.
   3. Returns the raw API key — this is the **only** time it is visible.

   **Security notes:**

   * This endpoint has **no authentication** — it is designed for the
     first-use flow before any API keys exist.
   * In production, disable this endpoint or gate it behind a
     deployment-time secret environment variable.
   * The raw key is returned exactly once and is **not** persisted.
     Only the salted SHA-256 hash is stored.

Admin — Organization Management (Prompts & Custom Instructions)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. module:: routers.admin_organizations

Router prefix: ``/admin/org``, tags: ``Admin - Organizations``

All endpoints in this router require JWT authentication via
``get_dashboard_user`` and organization ID via ``require_org_id``.

**Prompt Templates**:

.. function:: routers.admin_organizations.list_prompt_templates(db, org_id, _user_id) -> PromptTemplateListResponse

   ``GET /admin/org/prompts``

   List all prompt template names with override status.  Returns one entry
   per template name — includes the current version, whether the org has
   customised it, and its last-updated timestamp.

.. function:: routers.admin_organizations.list_system_prompts(db, org_id, _user_id) -> SystemPromptGroupsResponse

   ``GET /admin/org/prompts/system``

   List all system-default prompt templates grouped by base name.  Returns
   every system-default version (not just active ones) so users can see old
   versions.  Each group is annotated with which template names the
   organisation has already imported.

.. function:: routers.admin_organizations.import_system_prompt(body, db, org_id, _user_id) -> PromptTemplateDetail

   ``POST /admin/org/prompts/import`` (status 201)

   Import a system-default prompt template into the organisation.  Creates
   an org-specific copy at ``version = 1`` with the text from the active
   system default.

   :status 409: If the template is already imported.
   :status 404: If no active system default exists.

.. function:: routers.admin_organizations.set_prompt_type_default(name, db, org_id, _user_id) -> PromptTemplateDetail

   ``POST /admin/org/prompts/{name}/set-default``

   Mark a prompt template as the active default for its type.  Sets
   ``is_default_for_type = True`` for this template and ``False`` for all
   other templates of the same type and scope.

   :status 404: If the template does not exist or has no ``type`` assigned.

.. function:: routers.admin_organizations.get_prompt_template(name, db, org_id, _user_id) -> PromptTemplateDetail

   ``GET /admin/org/prompts/{name}``

   Get the active template for an organization.  Returns the org-specific
   template if it exists.

   :status 404: If not found.

.. function:: routers.admin_organizations.list_prompt_template_versions(name, db, org_id, _user_id) -> PromptTemplateVersionsResponse

   ``GET /admin/org/prompts/{name}/versions``

   List all versions of a named template for this org.  Returns only
   org-scoped versions, ordered by version descending (newest first).

   :status 404: If no template exists with this name.

.. function:: routers.admin_organizations.set_prompt_template(name, body, request, db, org_id, _user_id) -> PromptTemplateDetail

   ``PUT /admin/org/prompts/{name}`` (status 201)

   Create a new org-specific version of a prompt template.  Creates an
   org-scoped copy at ``version = max(existing) + 1``.  Invalidates any
   Redis cache entries for this template after update.

   Note: System-level defaults no longer exist (Option A).  Defaults come
   from ``manifest.yaml`` on disk.  To start from the disk default, import
   it first via ``POST /admin/org/prompts/import``.

.. function:: routers.admin_organizations.rollback_prompt_template(name, version, db, org_id, _user_id) -> PromptTemplateDetail

   ``POST /admin/org/prompts/{name}/rollback/{version}``

   Rollback to a previous version of a prompt template.  Creates a **new**
   version whose ``template_text`` is copied from the target version.  The
   new version is activated and all previously active versions are
   deactivated.

   :status 404: If the target version does not exist.

.. function:: routers.admin_organizations.delete_prompt_template_override(name, db, org_id, _user_id) -> None

   ``DELETE /admin/org/prompts/{name}`` (status 204)

   Delete all org-specific versions of a prompt template.  Re-import from
   the disk manifest via ``POST /admin/org/prompts/import`` if needed.

   :status 404: If no org-specific override exists.
   :status 409: If the template is the active default for its type.

**Custom Instructions**:

.. function:: routers.admin_organizations.list_custom_instructions(db, org_id, _user_id) -> CustomInstructionsResponse

   ``GET /admin/org/custom-instructions``

   List all extraction custom instructions for the organization.

.. function:: routers.admin_organizations.set_custom_instructions(body, db, org_id, _user_id) -> CustomInstructionsResponse

   ``PUT /admin/org/custom-instructions`` (status 201)

   Replace all extraction custom instructions for the organization
   atomically.  The existing instructions in the ``extraction`` scope are
   deleted and replaced with the provided list.

.. function:: routers.admin_organizations.clear_custom_instructions(db, org_id, _user_id) -> Response

   ``DELETE /admin/org/custom-instructions`` (status 204)

   Clear all extraction custom instructions for the organization.  No-op
   if no instructions exist.

Admin — Organization Config
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. module:: routers.admin_org_config

Router prefix: ``/admin/org/config``, tags: ``Admin - Organization Config``

.. function:: routers.admin_org_config.get_org_config_defaults() -> UpdateOrgConfigRequest

   ``GET /admin/org/config/defaults``

   Return seeded onboarding defaults for a new organization.  These are
   **not** the stored config — they are starter values for the onboarding
   form.  No auth required.  Secrets such as ``openai_api_key`` are returned
   as empty strings.

   :status 500: If the defaults configuration file is not found.

.. function:: routers.admin_org_config.get_org_config(_org_id, service) -> OrgConfigResponse

   ``GET /admin/org/config``

   Get the stored configuration for the current organization.  Returns only
   the fields explicitly set in OpenBao.  Unset fields are ``null``.

.. function:: routers.admin_org_config.update_org_config(body, _org_id, service) -> OrgConfigBase

   ``PATCH /admin/org/config``

   Partially update the organization's configuration.  Only fields
   explicitly provided in the request body are updated.  Set a field to
   ``null`` to remove it from the stored config.

   Requires ``admin:write`` scope.

.. function:: routers.admin_org_config.replace_org_config(body, _org_id, service) -> OrgConfigBase

   ``PUT /admin/org/config``

   Replace the entire organization configuration.  Every field is stored as
   provided.  Fields set to ``null`` are stored as ``null``.  Fields not
   included in the request body are **removed** from the stored config.
   Prefer ``PATCH`` for updating individual fields.

   Requires ``admin:write`` scope.

Admin — Webhooks
~~~~~~~~~~~~~~~~

.. module:: routers.admin_webhooks

Router prefix: ``/v1/admin/webhooks``, tags: ``Admin Webhooks``

.. function:: routers.admin_webhooks.list_event_types() -> dict

   ``GET /v1/admin/webhooks/events``

   Return all subscribable webhook event types, grouped by category.  No
   authentication required — this endpoint is public.

   Returns a dict with key ``"data"`` mapping to category objects, each
   containing ``type``, ``label``, ``category``, and ``description`` fields.

.. function:: routers.admin_webhooks.list_webhooks(service, org_id, _user_id) -> dict

   ``GET /v1/admin/webhooks``

   List all webhook endpoints for the authenticated organization.

.. function:: routers.admin_webhooks.get_webhook(endpoint_id, service, org_id, _user_id) -> dict

   ``GET /v1/admin/webhooks/{endpoint_id}``

   Get a single webhook endpoint by ID.

   :status 404: If the endpoint is not found or not owned by the org.

.. function:: routers.admin_webhooks.create_webhook(body, service, org_id, _user_id) -> WebhookSecretResponse

   ``POST /v1/admin/webhooks`` (status 201)

   Create a new webhook endpoint.  Returns the global webhook signing secret
   so the consumer can verify HMAC-SHA256 signatures.

   :returns: :class:`schemas.webhook.WebhookSecretResponse`

.. function:: routers.admin_webhooks.update_webhook(endpoint_id, body, service, org_id, _user_id) -> dict

   ``PATCH /v1/admin/webhooks/{endpoint_id}``

   Update a webhook endpoint's name, URL, events, or active status.

   :status 400: If no fields to update.
   :status 404: If the endpoint is not found or not owned by the org.

.. function:: routers.admin_webhooks.delete_webhook(endpoint_id, service, org_id, _user_id) -> None

   ``DELETE /v1/admin/webhooks/{endpoint_id}`` (status 204)

   Delete a webhook endpoint.

   :status 404: If the endpoint is not found or not owned by the org.

Admin — Metrics
~~~~~~~~~~~~~~~

.. module:: routers.admin_metrics

Router prefix: ``/metrics``, tags: ``Admin - Metrics``

All endpoints in this router require authentication via ``require_org_id``.

.. function:: routers.admin_metrics.get_metrics_summary(db, org_id, prom) -> MetricsSummaryResponse

   ``GET /metrics/summary``

   Get aggregated metrics for the admin dashboard.  Merges DB counts and
   Prometheus metrics into a single response.  DB counts are scoped to the
   authenticated organization.

   The ``status`` field is ``"degraded"`` if Prometheus is unreachable — DB
   counts are still returned.

   **DB counts returned**:

   * **EpisodeStats**: total episodes, episodes in last 24h, in-progress
     enrichment, enrichment pending, fully enriched, with embeddings, and
     percentage fully enriched.
   * **GraphStats**: total entities, entities in last 24h.
   * **User count**: total non-deleted users.

   **Prometheus metrics returned**:

   * Request rate by status class (``2xx``, ``4xx``, ``5xx``) per second.
   * Error rate as percentage of 5xx errors.
   * Latency percentiles (p50, p95, p99) for HTTP requests, context
     pipeline, and graph search (in milliseconds).
   * Total and active request counters.
   * ARQ worker queue depth (high and low priority).

.. function:: routers.admin_metrics.get_promql_query(query: str, _org_id, prom) -> dict

   ``GET /metrics/query?query=<promql>``

   Run an arbitrary PromQL instant query and return the raw result.  Useful
   for the frontend to build custom charts without backend changes.

   :param query: The PromQL expression to evaluate.
   :returns: Raw Prometheus query result with ``status`` and ``data`` fields.
   :status 502: If Prometheus is unreachable.

.. function:: routers.admin_metrics.get_prometheus_targets(_org_id) -> dict

   ``GET /metrics/targets``

   Get Prometheus scrape target health.  Returns a list of active targets
   with their job name, instance, health status, last scrape time, and last
   error.

   :returns: Dict with ``targets`` list and ``status``.
   :status 502: If Prometheus is unreachable.

Admin — Stats
~~~~~~~~~~~~~

.. module:: routers.admin_stats

Router prefix: ``/v1/admin/stats``, tags: ``Admin - Stats``

All endpoints require JWT authentication (dashboard session) via
``get_dashboard_user``.

.. function:: routers.admin_stats.get_org_stats(db, org_id, _user_id) -> OrgStatsResponse

   ``GET /v1/admin/stats/org``

   Get aggregate statistics for the authenticated organization.  Returns
   total users, sessions, episodes, facts, messages, and API keys (all
   scoped to the organization).

.. function:: routers.admin_stats.get_usage_stats(days, db, org_id, _user_id) -> list[UsageStatsResponse]

   ``GET /v1/admin/stats/usage?days=30``

   Get daily usage statistics for the organization.  Returns daily message
   and session counts for the last N days.  Useful for dashboard charts.

   :param days: Look-back window in days (default 30, max 365).
   :returns: List of daily usage data points, newest first.

Admin — Schemas
~~~~~~~~~~~~~~~

.. module:: routers.admin_schemas

Router prefix: ``/v1/admin/schemas``, tags: ``Admin - Schemas``

.. function:: routers.admin_schemas.create_schema(payload, service, org_id) -> ExtractionSchemaResponse

   ``POST /v1/admin/schemas`` (status 201)

   Create a new extraction or classification schema.  Requires ``admin``
   scope.  The schema name must be unique within the organization.  For
   ``type='classification'``, the ``json_schema`` must follow the expected
   classification label structure.

.. function:: routers.admin_schemas.list_schemas(type, is_active, service, org_id) -> ExtractionSchemaListResponse

   ``GET /v1/admin/schemas``

   List all schemas for the authenticated organization.  Supports optional
   filtering by ``type`` (``structured``/``classification``) and
   ``is_active`` status.

.. function:: routers.admin_schemas.get_schema(schema_id, service, org_id) -> ExtractionSchemaResponse

   ``GET /v1/admin/schemas/{schema_id}``

   Get a single schema by ID.  Scoped to the authenticated organization.

.. function:: routers.admin_schemas.update_schema(schema_id, payload, service, org_id) -> ExtractionSchemaResponse

   ``PUT /v1/admin/schemas/{schema_id}``

   Update an existing schema.  Requires ``admin`` scope.  The ``type`` field
   is immutable after creation.  Name uniqueness is enforced within the
   organization.

.. function:: routers.admin_schemas.delete_schema(schema_id, service, org_id) -> None

   ``DELETE /v1/admin/schemas/{schema_id}`` (status 204)

   Soft-delete a schema (set ``is_active`` to ``false``).  Requires
   ``admin`` scope.  Existing extractions referencing this schema are
   preserved (FK uses ``ON DELETE SET NULL``).

Structured Extractions
~~~~~~~~~~~~~~~~~~~~~~

.. module:: routers.structured_extractions

Router prefix: ``/v1/projects/{project_id}/sessions/{session_id}/structured-extractions``,
tags: ``Structured Extraction``

All endpoints guarded by ``require_project_membership``.

.. function:: routers.structured_extractions.list_structured_extractions(request, session_id, service) -> StructuredExtractionListResponse

   ``GET /v1/projects/{project_id}/sessions/{session_id}/structured-extractions``

   List all structured extractions for episodes in a session.  Returns an
   empty list if no episodes have been processed by the ``extract_structured``
   worker yet, or if no structured schemas are configured.

.. function:: routers.structured_extractions.get_episode_extraction(request, session_id, episode_id, service) -> StructuredExtractionResponse

   ``GET /v1/projects/{project_id}/sessions/{session_id}/structured-extractions/{episode_id}``

   Get the structured extraction for a specific episode in a session.

   :status 404: If the episode has not been processed yet or no matching
                extraction exists.

Audit Logs
~~~~~~~~~~

.. module:: routers.audit_log

Router prefix: ``/v1/admin/audit-logs``, tags: ``Admin - Audit Logs``

.. function:: routers.audit_log.list_audit_logs(db, org_id, _user_id, ...) -> AuditLogListResponse

   ``GET /v1/admin/audit-logs``

   Get paginated audit log entries for the admin dashboard.  Supports
   filtering by:

   * ``action`` — exact-match filter on action.
   * ``actor_id`` — exact-match filter on actor.
   * ``actor_type`` — filter by actor type (``user``, ``api_key``, ``system``).
   * ``resource_type`` — filter by resource type.
   * ``resource_id`` — filter by resource ID.
   * ``status_code`` — filter by HTTP status code.
   * ``created_after`` / ``created_before`` — ISO 8601 timestamp range.
   * ``limit`` — page size (default 50, max 500).
   * ``offset`` — pagination offset (default 0).

   Response includes enriched fields: ``status_code``, ``method``, and
   ``path`` extracted from the ``details`` JSONB payload.

   Requires JWT authentication (dashboard session).

Prometheus Metrics (unauthenticated)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. module:: routers.metrics

Router prefix: ``/metrics``, tags: ``Metrics``

This endpoint is intentionally **unauthenticated** — Prometheus scrapers
cannot carry bearer tokens.  It exposes no PII or business data, only
aggregate performance counters.

.. function:: routers.metrics.get_metrics() -> Response

   ``GET /metrics``

   Return Prometheus metrics in text format.  Uses the isolated application
   registry (``METRICS_REGISTRY``) which excludes default process/GC metrics
   to keep the payload lean.

   Exposed histograms:

   * ``openzync_http_request_duration_seconds`` — HTTP request latency.
   * ``openzync_context_latency_seconds`` — context pipeline latency.
   * ``openzync_graph_search_latency_seconds`` — graph search latency.

   Exposed counters:

   * ``openzync_http_requests_total`` — total requests by status class.
   * ``openzync_http_requests_in_progress`` — currently in-flight requests.

   Exposed gauges:

   * ``openzync_worker_queue_depth`` — ARQ queue depth by name.


Key Schemas
-----------

.. module:: schemas.organizations

.. class:: CreateOrgRequest

   Request body for ``POST /admin/organizations``.

   .. attribute:: name

      Human-readable organization name.  ``str``, required, 1–255 chars.

   .. attribute:: plan

      Billing plan.  ``str``, default ``"free"``.  Must match
      ``^(free|pro|enterprise)$``.

.. class:: CreateOrgResponse

   Response body for ``POST /admin/organizations``.

   .. attribute:: organization_id

      UUID of the newly created organization.  ``UUID``.

   .. attribute:: organization_name

      Name of the organization.  ``str``.

   .. attribute:: api_key

      Full API key string (shown once — not persisted).  ``str``.

   .. attribute:: api_key_prefix

      Prefix identifying the key type (``oz_live_``).  ``str``.

   .. attribute:: api_key_name

      Human-readable label for the key (``"default"``).  ``str``.

   .. attribute:: message

      Warning that the key will not be retrievable later.  ``str``.

.. module:: schemas.organization_config

.. class:: OrgConfigBase

   Raw per-org config stored in the ``organizations.config`` JSONB column.
   Every field is **optional**.  When a field is ``None`` (absent from the
   JSONB), the caller must decide what to do — there is no env-var fallback
   at this layer.

   :model_config extra=ignore: Silently drops unknown keys.

   **LLM fields**:

   * ``llm_backend`` (``str | None``) — LLM provider (ollama, openai, azure,
     anthropic, openrouter).
   * ``llm_model`` (``str | None``) — Model name/tag for the LLM backend.
   * ``llm_temperature`` (``float | None``, 0.0–2.0) — LLM sampling temperature.
   * ``llm_max_tokens`` (``int | None``, ge=1) — Maximum tokens in response.
   * ``openai_api_key``, ``openrouter_api_key``, ``azure_openai_endpoint``,
     ``azure_openai_key``, ``anthropic_api_key``, ``ollama_base_url``.

   **Embeddings fields**:

   * ``embedding_backend``, ``embedding_model``, ``embedding_dim`` (64–4096).

   **Graph fields**:

   * ``graph_backend`` (default ``"surrealdb"``), ``graph_search_type``,
     ``graph_max_traversal_depth`` (1–10).
   * ``surrealdb_url``, ``surrealdb_user``, ``surrealdb_pass``,
     ``surrealdb_namespace``, ``surrealdb_database``.

   **Behaviour fields**:

   * ``context_cache_ttl`` (``int | None``, ge=1) — TTL in seconds for
     cached context summaries.
   * ``audit_log_response_body`` (``bool | None``) — Capture response body
     in audit_logs.details (may contain PII).

   **Re-ranker fields**:

   * ``reranker_backend``, ``reranker_model``, ``reranker_top_k`` (10–200),
     ``reranker_top_n`` (1–100), ``cohere_api_key``.

   Helper methods:

   .. method:: to_llm_config_dict() -> dict[str, str | float | int]

      Return config as a dict suitable for ``core.llm.resolve_backend()``.
      Only non-``None`` fields are included.  Maps canonical field names to
      provider-specific keys.

   .. method:: to_embedding_config_dict() -> dict[str, str | int]

      Return embedding config as a flat dict.  Only non-``None`` fields are
      included.  Used by worker tasks that read embedding settings directly.

.. class:: UpdateOrgConfigRequest

   Request body for ``PATCH /admin/org/config`` and ``PUT /admin/org/config``.
   Same shape as ``OrgConfigBase`` — every field is optional, only provided
   fields are updated.  Set a field to ``null`` to remove it.

.. class:: OrgConfigResponse

   Response for config GET endpoints.

   .. attribute:: stored

      Raw config stored in OpenBao — only explicitly set fields.
      ``OrgConfigBase``.

.. module:: schemas.admin_metrics

.. class:: LatencyPercentiles

   Latency distribution at key percentiles (in milliseconds).

   * ``p50``: 50th percentile (``float``)
   * ``p95``: 95th percentile (``float``)
   * ``p99``: 99th percentile (``float``)

.. class:: QueueDepth

   ARQ worker queue depths.

   * ``high``: High-priority queue depth (``int``)
   * ``low``: Low-priority queue depth (``int``)

.. class:: EpisodeStats

   Episode metrics — ingestion pipeline status.

   * ``added_total``, ``added_24h``, ``in_progress``, ``enrichment_pending``,
     ``fully_enriched``, ``with_embeddings``, ``fully_enriched_pct``.

.. class:: GraphStats

   Graph entity metrics.

   * ``entities_total``, ``entities_24h``, ``relationships_total``.

.. class:: MetricsSummaryResponse

   Aggregated metrics for the admin dashboard frontend.  Combines DB counts
   with Prometheus-sourced latency and error metrics.

   * ``episodes`` (:class:`EpisodeStats`)
   * ``graphs`` (:class:`GraphStats`)
   * ``users_total`` (``int``)
   * ``request_rate`` (``dict[str, float]``) — requests per second by status
     class (2xx, 4xx, 5xx).
   * ``error_rate_pct`` (``float``)
   * ``overall_latency_ms``, ``context_latency_ms``, ``graph_search_latency_ms``
     (:class:`LatencyPercentiles`)
   * ``queue_depth`` (:class:`QueueDepth` or ``None``)
   * ``total_requests`` (``int``), ``active_requests`` (``int``)
   * ``status`` (``str``) — ``"ok"`` or ``"degraded"``
   * ``message`` (``str | None``)

.. module:: schemas.webhook

.. class:: CreateWebhookRequest

   Request body for creating a new webhook endpoint.

   * ``name``: Human-readable label (``str``, 1–255 chars, required).
   * ``url``: Endpoint URL (``HttpUrl``, required, HTTPS recommended).
   * ``events``: List of event types to subscribe to (``list[str]``, default
     empty = all events).

.. class:: UpdateWebhookRequest

   Request body for updating a webhook endpoint.  All fields are optional —
   only provided fields are updated.

   * ``name`` (``str | None``)
   * ``url`` (``HttpUrl | None``)
   * ``events`` (``list[str] | None``)
   * ``is_active`` (``bool | None``)

.. class:: WebhookSecretResponse

   Response returned once after creating a webhook endpoint.  The ``secret``
   field is the raw signing secret and is never persisted in plaintext — the
   client must save it immediately.

   * ``id`` (``UUID``), ``name`` (``str``), ``url`` (``str``), ``secret`` (``str``)

.. module:: schemas.audit_log

.. class:: AuditLogResponse

   Single audit log entry returned to the frontend.

   * ``id`` (``UUID``), ``organization_id`` (``UUID | None``),
     ``actor_id`` (``str | None``), ``actor_type`` (``str | None``)
   * ``action`` (``str``), ``resource_type`` (``str``),
     ``resource_id`` (``str | None``)
   * ``details`` (``dict``), ``ip_address`` (``str | None``)
   * ``status_code`` (``int | None``), ``method`` (``str | None``),
     ``path`` (``str | None``)
   * ``created_at`` (``datetime``)

.. class:: AuditLogListResponse

   Paginated list of audit log entries.

   * ``items`` (``list[AuditLogResponse]``), ``total`` (``int``),
     ``limit`` (``int``), ``offset`` (``int``)

.. module:: schemas.extraction_schemas

.. class:: CreateExtractionSchemaRequest

   Request body for creating a new extraction schema.

   * ``name``: ``str``, 1–255 chars, must start with a letter and contain
     only alphanumeric, underscore, hyphen, or space chars.
   * ``json_schema``: ``dict`` — JSON Schema document or classification labels.
   * ``type``: ``"structured"`` (default) or ``"classification"``.
   * ``prompt_template``: ``str | None``, max 10000 chars.

.. class:: ExtractionSchemaResponse

   * ``id``, ``organization_id``, ``name``, ``type``, ``json_schema``,
     ``prompt_template``, ``is_active``, ``created_at``, ``updated_at``.

.. module:: schemas.structured_extractions

.. class:: StructuredExtractionResponse

   A single structured extraction result.

   * ``id``, ``session_id``, ``episode_id``, ``schema_id``, ``data``,
     ``created_at``.

.. module:: schemas.admin_stats

.. class:: OrgStatsResponse

   Aggregate statistics for the dashboard overview.

   * ``organization_id`` (``UUID``), ``total_users``, ``total_sessions``,
     ``total_episodes``, ``total_facts``, ``total_messages``,
     ``total_api_keys`` (all ``int``).

.. class:: UsageStatsResponse

   Daily usage statistics for the dashboard.

   * ``date`` (``str``, YYYY-MM-DD), ``message_count`` (``int``),
     ``session_count`` (``int``).

.. module:: schemas.custom_instructions

.. class:: CustomInstructionSchema

   A single named custom instruction.

   * ``name``: Human-readable label (``str``, 1–255 chars).
   * ``text``: The instruction text content (``str``, min 1 char).


Core Infrastructure
-------------------

.. module:: core.org_config

Per-organization configuration resolution — cache-first, OpenBao-authoritative.

.. function:: core.org_config.get_org_config(
      org_id: UUID,
      redis: redis.asyncio.Redis | None = None,
      bao_client: OpenBaoClient | None = None,
      *,
      skip_cache: bool = False,
   ) -> OrgConfigBase

   Fetch the stored config for an org: cache → OpenBao.  There is no env-var
   fallback — every field is returned as stored in OpenBao.

   :param skip_cache: If ``True``, bypass cache and always fetch from OpenBao.
   :returns: :class:`schemas.organization_config.OrgConfigBase` with only
             explicitly set fields.
   :raises OpenBaoConnectionError: If *bao_client* is ``None``.

.. function:: core.org_config.update_org_config(
      org_id: UUID,
      update_data: UpdateOrgConfigRequest | dict[str, Any],
      bao_client: OpenBaoClient,
      redis: redis.asyncio.Redis | None = None,
   ) -> OrgConfigBase

   Update stored org config in OpenBao, invalidate cache, return fresh
   config.  Performs a deep merge: provided keys replace existing stored
   values.  Keys set to ``None`` are removed from the stored config.

.. module:: core.email

.. class:: core.email.EmailConfig

   Typed SMTP configuration extracted from runtime :class:`core.config.Settings`.

   * ``HOST`` (``str``), ``PORT`` (``int``), ``USERNAME`` (``str``),
     ``PASSWORD`` (``str``), ``FROM_ADDR`` (``str``)
   * ``USE_TLS`` (``bool``), ``START_TLS`` (``bool``)

   .. method:: from_settings(settings: Settings) -> EmailConfig

      Build an ``EmailConfig`` from the runtime ``Settings`` singleton.

.. function:: core.email.build_email_message(
      to: str,
      subject: str,
      html_body: str,
      text_body: str | None = None,
      from_addr: str = "noreply@openzync.tech",
   ) -> EmailMessage

   Build an :class:`email.message.EmailMessage` with both HTML and plain-text
   parts.  If ``text_body`` is ``None``, a crude HTML-stripped version of
   ``html_body`` is used.

.. module:: core.events

.. function:: core.events.event_categories() -> Mapping[str, list[EventMeta]]

   Return event registry grouped by category.  Categories: ``Session``,
   ``Message``, ``Graph``, ``Fact``, ``Classification``, ``Extraction``,
   ``User``.

.. function:: core.events.event_type_labels() -> Mapping[str, str]

   Return a mapping of event type → human-readable label.

.. class:: core.events.EventType

   A webhook event type constant.  Usage::

      event = EventType.SESSION_CREATED
      assert event == "session.created"

   Constants: ``SESSION_CREATED``, ``SESSION_CLOSED``, ``MESSAGE_ADDED``,
   ``EPISODE_PROCESSED``, ``INGEST_BATCH_COMPLETED``, ``INGEST_EPISODE_COMPLETED``,
   ``GRAPH_ENTITY_CREATED``, ``GRAPH_ENTITY_UPDATED``, ``GRAPH_EDGE_CREATED``,
   ``FACT_EXTRACTED``, ``FACT_DELETED``, ``CLASSIFICATION_CREATED``,
   ``EXTRACTION_CREATED``, ``USER_CREATED``.
