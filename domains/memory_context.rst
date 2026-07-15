Memory & Context Domain
=======================

.. note::

   This document covers the Memory & Context domain within the
   ``openzync-core`` monolith at ``/home/rohan-linkai/code/personal/openzync/openzync-core/``.
   Every module, class, method, and endpoint described here has been verified
   against the actual source code at commit time.

   The domain is responsible for:

   * **Ingestion** — persisting conversation messages as episodes, extracting
     structured knowledge (facts, entities, classifications), and enqueuing
     async enrichment jobs.
   * **Retrieval** — assembling LLM-ready context blocks via hybrid search
     (vector + BM25 + graph BFS) with Reciprocal Rank Fusion and optional
     cross-encoder re-ranking.
   * **Graph services** — entity/relationship CRUD, community detection, graph
     topology observations, and temporal consistency validation.
   * **Session management** — creating, listing, and soft-deleting conversation
     sessions with paginated message and fact retrieval.

   **Design principles**: idempotency at every boundary (Idempotency-Key header,
   content-hash dedup), enrichment-bitmask for idempotent background processing,
   no silent degradation (every leg of hybrid search raises on failure),
   and warn-only temporal checks (no auto-mutation without a feature flag).

.. contents:: Sections
   :local:
   :depth: 3
   :class: this-will-duplicate-information-and-it-is-still-useful-here


End-to-End Pipeline Overview
----------------------------

::

    ┌─────────────────────────────────────────────────────────────────────┐
    │                      INGESTION PIPELINE                            │
    │                                                                     │
    │  POST /memory                                                      │
    │    │                                                               │
    │    ├─ 1. Idempotency check (Redis key, 48h TTL)                    │
    │    ├─ 2. Resolve or auto-create session                            │
    │    ├─ 3. Content-hash dedup (SHA-256 of payload)                    │
    │    ├─ 4. Batch-insert episodes (Episodes table)                    │
    │    ├─ 5. [Optional] PII redaction via PIIService                   │
    │    ├─ 6. Commit to PostgreSQL ───────────────┐                     │
    │    └─ 7. Enqueue ARQ enrichment tasks         │                     │
    │         ├─ enrich_episode (combined)          │                     │
    │         ├─ embed_episode                      │                     │
    │         └─ link_entities_to_episode           │                     │
    │              │                                │                     │
    │    Events:    │          Episodes & Facts      │                     │
    │    ┌──────────┘          visible to workers   │                     │
    │    ▼                                         ▼                     │
    │  Invalidate context cache (SCAN + DEL)                            │
    │  Emit webhook events (INGEST_BATCH_COMPLETED, MESSAGE_ADDED)      │
    │                                                                     │
    │                      ENRICHMENT PIPELINE  (ARQ workers)             │
    │                                                                     │
    │  enrich_episode:                                                    │
    │    ├─ Entity & relationship extraction (LLM) → graph_entities      │
    │    ├─ Zero-shot fact extraction (LLM) → facts table                │
    │    ├─ Dialog classification (LLM) → dialog_classifications         │
    │    └─ Sets enrichment_status bits on episode                       │
    │                                                                     │
    │  embed_episode:                                                     │
    │    └─ Generate pgvector embedding from episode content             │
    │                                                                     │
    │  embed_fact:                                                        │
    │    └─ Generate pgvector embedding from fact content                │
    │                                                                     │
    │  link_entities_to_episode:                                          │
    │    └─ Link extracted entities to source episode (low priority)     │
    │                                                                     │
    │  compute_observations (scheduled):                                  │
    │    └─ Run graph-topology observation detection                     │
    │       ├─ Co-occurrence frequency                                   │
    │       ├─ Temporal gap analysis                                     │
    │       └─ Behavioral pattern detection                              │
    └─────────────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────────────┐
    │                    CONTEXT RETRIEVAL PIPELINE                       │
    │                                                                     │
    │  GET /context?query="..."                                           │
    │    │                                                               │
    │    ├─ 1. Check Redis cache (key = ctx:{org}:{project}:{query})    │
    │    │     └── Cache hit → return immediately                         │
    │    │                                                               │
    │    ├─ 2. Hybrid search (5 concurrent legs)                        │
    │    │     ├─ Episode vector search  (pgvector <=>)                  │
    │    │     ├─ Episode BM25 search    (PostgreSQL ts_rank)            │
    │    │     ├─ Fact vector search     (pgvector <=>)                  │
    │    │     ├─ Fact BM25 search       (PostgreSQL ts_rank)            │
    │    │     └─ Graph BFS search       (entity traversal)              │
    │    │                                                               │
    │    ├─ 3. RRF merge per type                                         │
    │    │     score(d) = Σ 1 / (60 + rank_s(d))                         │
    │    │                                                               │
    │    ├─ 4. [Optional] Cross-encoder re-ranking                      │
    │    │     └── Reranker.rerank(query, top-K) → top-N                │
    │    │                                                               │
    │    ├─ 5. Format context (text or JSON)                             │
    │    ├─ 6. Cache result (Redis, TTL=30s)                             │
    │    └─ 7. Return context block with metadata                        │
    └─────────────────────────────────────────────────────────────────────┘


Data Model
----------

Episode
~~~~~~~

.. module:: models.episode

.. class:: Episode

   A single message turn within a conversation session.  Episodes are
   ordered by ``sequence_number`` within a session.  Each episode captures
   the role, content, optional pgvector embedding, and enrichment-status
   bitmask.

   .. attribute:: id

      :type: uuid.UUID
      :server_default: ``gen_random_uuid()``
      :primary_key: True

   .. attribute:: organization_id

      :type: uuid.UUID
      :ForeignKey: ``organizations.id`` ON DELETE CASCADE
      :nullable: False

   .. attribute:: project_id

      :type: uuid.UUID
      :ForeignKey: ``projects.id`` ON DELETE CASCADE
      :nullable: False
      :index: True

      Denormalized for efficient project-scoped queries without joining
      through the session.

   .. attribute:: session_id

      :type: uuid.UUID
      :ForeignKey: ``sessions.id`` ON DELETE CASCADE
      :nullable: False

   .. attribute:: user_id

      :type: uuid.UUID
      :ForeignKey: ``users.id`` ON DELETE CASCADE
      :nullable: False

   .. attribute:: role

      :type: str
      :nullable: False

      One of ``user``, ``assistant``, ``system``, ``tool`` (enforced by a
      ``CheckConstraint``).

   .. attribute:: content

      :type: str
      :max_length: 65536 characters (UTF-8 bytes)

      Message body text.  The ``ck_episode_content_length`` check constraint
      enforces the char-level limit; the schema-level validator in
      :class:`schemas.memory.Message` enforces the 64KB byte-level limit
      (multi-byte characters).

   .. attribute:: metadata_

      :type: JSONB
      :nullable: False
      :default: ``{}``

      Note the trailing underscore — ``metadata`` is reserved by SQLAlchemy.
      The DB column is named ``metadata``.

   .. attribute:: embedding

      :type: str | None (stand-in), ``vector(1536)`` in production
      :nullable: True

      pgvector embedding.  The ORM uses ``Text`` as a stand-in because
      pgvector may not be installed in dev/test environments.  The Alembic
      migration alters the column to ``vector(1536)`` in production.

   .. attribute:: token_count

      :type: int
      :default: 0

      Approximate token count for the message.

   .. attribute:: sequence_number

      :type: int
      :default: 0

      Zero-based ordering within the session.  Deterministic ordering is
      guaranteed by this field (unlike ``created_at`` which can have ties).

   .. attribute:: enrichment_status

      :type: int
      :default: 0

      Bitmask tracking which enrichment passes have completed:

      * Bit 0 — entity extraction
      * Bit 1 — fact extraction
      * Bit 2 — dialog classification
      * Bit 3 — embedding
      * Bit 6 — observations (``compute_observations`` worker)

      Each worker checks its bit before processing and sets it after
      completion, making enrichment idempotent: if a worker crashes after
      committing results but before the final update, the next retry skips
      the already-completed step.

   .. attribute:: is_deleted

      :type: bool
      :default: False

      Soft-delete flag for the GDPR memory-wipe operation.

   .. attribute:: created_at

      :type: datetime
      :server_default: ``now()``

      Inherited from :class:`models.base.TimestampMixin`.

   .. attribute:: updated_at

      :type: datetime
      :server_default: ``now()``
      :onupdate: ``now()``

      Inherited from :class:`models.base.TimestampMixin`.

   .. rubric:: Table Constraints

   * ``ck_episode_role`` — ``role IN ('user', 'assistant', 'system', 'tool')``
   * ``ck_episode_content_length`` — ``char_length(content) <= 65536``
   * ``ix_episode_session_sequence`` — index on ``(session_id, sequence_number)``
   * ``ix_episode_user_id`` — index on ``user_id``


Session
~~~~~~~

.. module:: models.session

.. class:: Session

   A conversation session scoped to a project.  Sessions group related episodes
   into a single conversational context.

   .. attribute:: id

      :type: uuid.UUID
      :server_default: ``gen_random_uuid()``
      :primary_key: True

   .. attribute:: organization_id

      :type: uuid.UUID
      :ForeignKey: ``organizations.id`` ON DELETE CASCADE
      :nullable: False
      :index: True

   .. attribute:: project_id

      :type: uuid.UUID
      :ForeignKey: ``projects.id`` ON DELETE CASCADE
      :nullable: False
      :index: True

   .. attribute:: user_id

      :type: uuid.UUID
      :ForeignKey: ``users.id`` ON DELETE CASCADE
      :nullable: False

      Attribution only — ownership is at the project level.

   .. attribute:: external_id

      :type: str
      :nullable: False

      Caller-defined session identifier.  Uniqueness is enforced by a
      ``UniqueConstraint`` on ``(project_id, external_id)``.

   .. attribute:: metadata_

      :type: JSONB
      :default: ``{}``

   .. attribute:: is_active

      :type: bool
      :default: True

      ``False`` when the session is closed (no longer accepting new messages).

   .. attribute:: is_deleted

      :type: bool
      :default: False

      Soft-delete flag.

   .. attribute:: closed_at

      :type: datetime | None
      :nullable: True

      Timestamp of session closure.

   .. rubric:: Table Constraints

   * ``uq_session_project_external`` — ``UNIQUE (project_id, external_id)``
   * ``ix_session_user_id`` — index on ``user_id``


Fact
~~~~

.. module:: models.fact

.. class:: Fact

   An extracted knowledge fact in subject-predicate-object form, optionally
   linked back to its source episode and resolved graph entities.

   .. attribute:: id

      :type: uuid.UUID
      :server_default: ``gen_random_uuid()``

   .. attribute:: project_id

      :type: uuid.UUID
      :ForeignKey: ``projects.id`` ON DELETE CASCADE
      :index: True

      Denormalized for project-scoped queries.

   .. attribute:: user_id

      :type: uuid.UUID
      :ForeignKey: ``users.id`` ON DELETE CASCADE

   .. attribute:: organization_id

      :type: uuid.UUID
      :nullable: False

      .. caution::

         No FK constraint — this is denormalized for RLS performance and
         application-enforced.  Must be kept in sync at write time.

   .. attribute:: content

      :type: str
      :nullable: False

      Human-readable fact statement (e.g. ``"Alice likes hiking"``).

   .. attribute:: subject

      :type: str | None
      :nullable: True

   .. attribute:: predicate

      :type: str | None
      :nullable: True

   .. attribute:: object

      :type: str | None
      :nullable: True

   .. attribute:: subject_type

      :type: str
      :default: ``"literal"``

      Set to ``"entity"`` when resolved to a ``graph_entities`` row.

   .. attribute:: object_type

      :type: str
      :default: ``"literal"``

      Set to ``"entity"`` when resolved to a ``graph_entities`` row.

   .. attribute:: confidence

      :type: float
      :default: 1.0

      Extraction confidence score (0.0–1.0).

   .. attribute:: source_episode_id

      :type: uuid.UUID | None
      :ForeignKey: ``episodes.id`` ON DELETE SET NULL
      :nullable: True

   .. attribute:: subject_entity_id

      :type: uuid.UUID | None
      :ForeignKey: ``graph_entities.id`` ON DELETE SET NULL
      :index: True

      FK to ``graph_entities`` — resolved entity for the subject.

   .. attribute:: object_entity_id

      :type: uuid.UUID | None
      :ForeignKey: ``graph_entities.id`` ON DELETE SET NULL
      :index: True

      FK to ``graph_entities`` — resolved entity for the object.

   .. attribute:: valid_from

      :type: datetime | None
      :nullable: True

      Start of temporal validity (for time-bound facts).

   .. attribute:: valid_to

      :type: datetime | None
      :nullable: True

      End of temporal validity.  ``NULL`` = open-ended.

   .. attribute:: invalid_at

      :type: datetime | None
      :nullable: True

      Timestamp when this fact was invalidated/retracted.

   .. attribute:: embedding

      :type: list[float] | None
      :nullable: True

      pgvector embedding.  Uses ``ARRAY(Float)`` as the ORM stand-in type;
      the production column is ``vector(1536)`` via Alembic migration.

   .. rubric:: Table Constraints

   * ``ix_fact_user_id`` — index on ``user_id``
   * ``ix_fact_user_valid_range`` — index on ``(user_id, valid_from, valid_to)``


GraphEntity
~~~~~~~~~~~

.. module:: models.graph_entity

.. class:: GraphEntity

   .. caution::

      This is a **read-only stub model**.  All writes to the ``graph_entities``
      table go through the Graphiti backend client or raw SQL in worker tasks.
      No ORM-based repository or service layer should mutate this table.

   The ``graph_entities`` table is created via raw SQL in migration 0006 and
   managed primarily through the Graphiti backend client.  This model exists
   only so that SQLAlchemy can resolve the ``ForeignKey("graph_entities.id")``
   references in :class:`models.fact.Fact` and other models.

   .. attribute:: id

      :type: uuid.UUID
      :server_default: ``gen_random_uuid()``

   .. attribute:: organization_id

      :type: uuid.UUID
      :nullable: False

   .. attribute:: project_id

      :type: uuid.UUID
      :nullable: False
      :index: True

   .. attribute:: name

      :type: str
      :nullable: False

   .. attribute:: entity_type

      :type: str
      :server_default: ``"custom"``

   .. attribute:: summary

      :type: str | None
      :nullable: True

   .. attribute:: attributes

      :type: JSONB
      :server_default: ``{}``

   .. attribute:: created_at

      :type: datetime
      :server_default: ``now()``

   .. attribute:: updated_at

      :type: datetime
      :server_default: ``now()``


GraphObservation
~~~~~~~~~~~~~~~~

.. module:: models.graph_observation

.. class:: GraphObservation

   A single observation about an entity, surfaced from graph-topology
   analysis (co-occurrence, temporal patterns, behavioral patterns).
   Created via migration 0025.

   .. attribute:: id

      :type: uuid.UUID
      :server_default: ``gen_random_uuid()``

   .. attribute:: organization_id

      :type: uuid.UUID
      :nullable: False

   .. attribute:: project_id

      :type: uuid.UUID
      :ForeignKey: ``projects.id`` ON DELETE CASCADE
      :nullable: False

   .. attribute:: subject_entity_id

      :type: uuid.UUID
      :ForeignKey: ``graph_entities.id`` ON DELETE CASCADE
      :nullable: False

      The entity this observation is about.

   .. attribute:: related_entity_id

      :type: uuid.UUID | None
      :ForeignKey: ``graph_entities.id`` ON DELETE SET NULL
      :nullable: True

      For pair-level observations (e.g., co-occurrence): the other entity.
      ``NULL`` for entity-level observations (temporal patterns, behavioral
      patterns).

   .. attribute:: observation_type

      :type: str
      :nullable: False

      One of the :class:`ObservationType` enum values.

   .. attribute:: content

      :type: str
      :nullable: False

      Natural-language description of the observation, generated by LLM
      with template-based fallback.

   .. attribute:: supporting_fact_ids

      :type: list[uuid.UUID] | None
      :nullable: True

   .. attribute:: supporting_relationship_ids

      :type: list[uuid.UUID] | None
      :nullable: True

   .. attribute:: confidence

      :type: float
      :server_default: ``0.0``

   .. attribute:: valid_from

      :type: datetime | None
      :nullable: True

   .. attribute:: valid_to

      :type: datetime | None
      :nullable: True

   .. attribute:: observation_metadata

      :type: JSONB | None
      :nullable: True

   .. attribute:: created_at

      :type: datetime
      :server_default: ``now()``

      Inherited from :class:`TimestampMixin`.

   .. attribute:: updated_at

      :type: datetime
      :server_default: ``now()``

      Inherited from :class:`TimestampMixin`.


.. class:: ObservationType

   .. attribute:: CO_OCCURRENCE

      ``"co_occurrence"`` — Two entities co-appear in episodes above a
      frequency threshold.

   .. attribute:: TEMPORAL_PATTERN

      ``"temporal_pattern"`` — An entity's appearances follow a notable
      temporal cadence (periodic, widening, narrowing, burst, irregular).

   .. attribute:: BEHAVIORAL_PATTERN

      ``"behavioral_pattern"`` — An entity's facts or relationships show
      a consistent behavioral pattern.


DialogClassification
~~~~~~~~~~~~~~~~~~~~

.. module:: models.dialog_classification

.. class:: DialogClassification

   Classification labels for a single episode — intent, emotion, valence,
   arousal — produced by the enrichment pipeline.

   .. attribute:: id

      :type: uuid.UUID
      :server_default: ``gen_random_uuid()``

   .. attribute:: project_id

      :type: uuid.UUID
      :ForeignKey: ``projects.id`` ON DELETE CASCADE
      :index: True

      Denormalized for project-scoped queries.

   .. attribute:: organization_id

      :type: uuid.UUID
      :ForeignKey: ``organizations.id`` ON DELETE CASCADE

   .. attribute:: episode_id

      :type: uuid.UUID
      :ForeignKey: ``episodes.id`` ON DELETE CASCADE
      :nullable: False

   .. attribute:: intent

      :type: str | None
      :nullable: True

      Predicted intent label (e.g. ``"greeting"``, ``"question"``,
      ``"command"``).

   .. attribute:: emotion

      :type: str | None
      :nullable: True

      Predicted emotion label (e.g. ``"joy"``, ``"frustration"``).

   .. attribute:: valence

      :type: str | None
      :nullable: True

      Sentiment valence (e.g. ``"positive"``, ``"negative"``, ``"neutral"``).

   .. attribute:: arousal

      :type: str | None
      :nullable: True

      Emotional arousal level (e.g. ``"low"``, ``"medium"``, ``"high"``).

   .. attribute:: confidence

      :type: float
      :default: 0.0

   .. attribute:: raw

      :type: dict | None
      :nullable: True

      Raw LLM classifier output (full JSON response).  This field is
      excluded from the query endpoint response — available via direct DB
      access only.


CustomInstruction
~~~~~~~~~~~~~~~~~

.. module:: models.custom_instruction

.. class:: CustomInstruction

   A named instruction snippet guiding LLM extraction behavior.  Scoped to an
   organization, extraction domain, and optional target entity.

   .. attribute:: id

      :type: uuid.UUID
      :server_default: ``gen_random_uuid()``

   .. attribute:: organization_id

      :type: uuid.UUID
      :ForeignKey: ``organizations.id`` ON DELETE CASCADE
      :nullable: False

   .. attribute:: scope

      :type: str

      Instruction scope — ``"extraction"`` or ``"user_summary"``.

   .. attribute:: target_id

      :type: uuid.UUID | None
      :nullable: True

      Optional target UUID (e.g. user UUID for ``"user_summary"`` scope).
      ``NULL`` represents org-level instructions.

   .. attribute:: name

      :type: str

      Human-readable label (e.g. ``"legal_domain"``, ``"healthcare"``).

   .. attribute:: text

      :type: str

      The instruction text content injected into extraction prompts.


PromptTemplate
~~~~~~~~~~~~~~

.. module:: models.prompt_template

.. class:: PromptTemplate

   A versioned prompt template belonging to an organization.  Templates are
   org-scoped (all rows have a non-null ``organization_id``).  The source of
   truth for defaults is ``services/worker/prompts/manifest.yaml`` plus
   ``.jinja2`` files on disk, seeded at signup.

   Only one template per ``(organization_id, template_name)`` can be active
   at a time.

   .. attribute:: id

      :type: uuid.UUID
      :server_default: ``gen_random_uuid()``

   .. attribute:: organization_id

      :type: uuid.UUID | None
      :ForeignKey: ``organizations.id`` ON DELETE CASCADE
      :nullable: True

      All current rows have a non-null ``organization_id`` (Option A
      migration).  The property ``is_system_default`` always returns
      ``False``.

   .. attribute:: template_name

      :type: str

      Unique logical name within the scope (e.g. ``"memory_summary"``).

   .. attribute:: template_text

      :type: str

      The actual prompt text with ``{placeholder}`` variables.

   .. attribute:: version

      :type: int
      :default: 1

      Monotonically increasing version number per ``(org, name)``.

   .. attribute:: description

      :type: str | None
      :nullable: True

   .. attribute:: type

      :type: str | None
      :nullable: True

   .. attribute:: is_default_for_type

      :type: bool
      :default: False

   .. attribute:: is_active

      :type: bool
      :default: True


Ingestion Pipeline
------------------

Memory Ingestion
~~~~~~~~~~~~~~~~

.. module:: services.memory_service

.. class:: MemoryService

   The primary entry point for persisting agent memory.  This service
   orchestrates message ingestion, session resolution, PII redaction, batch
   episode creation, ARQ background task enqueueing, and context cache
   invalidation.

   .. rubric:: Constructor

   .. method:: __init__(db, redis_client, episode_repo=None, session_repo=None, user_repo=None, fact_repo=None, webhook_service=None, org_repo=None)

      :param db: Request-scoped async SQLAlchemy session.
      :param redis_client: Async Redis client for caching and idempotency.
      :param episode_repo: (optional) EpisodeRepository — auto-created from
         ``db`` if omitted.
      :param session_repo: (optional) SessionRepository.
      :param user_repo: (optional) UserRepository.
      :param fact_repo: (optional) FactRepository.
      :param webhook_service: (optional) WebhookService for event emission.
      :param org_repo: (optional) OrganizationRepository.

   .. rubric:: Public Methods

   .. method:: async ingest(org_id, project_id, created_by, session_external_id, messages, idempotency_key=None) -> IngestMemoryResponse

      The core ingestion flow with 11 steps:

      #. **Idempotency check** — if ``idempotency_key`` is provided, Redis
         is checked for a cached response (48h TTL).  A hit returns the
         previous response immediately.

      #. **Resolve or create session** — if ``session_external_id`` is
         provided, the session is looked up by external ID (with UUID
         fallback).  If ``None``, a ``__default__`` session is auto-created
         using ``INSERT ... ON CONFLICT DO NOTHING`` for race safety.

      #. **Content hash computation** — SHA-256 of
         ``(project_id, session_id, sorted messages)`` computed via
         ``_compute_content_hash``.

      #. **Content dedup check** — Redis is checked for the content hash.
         A hit returns the existing ``job_id``.

      #. **Sequence number allocation** — ``get_next_sequence(session_id)``
         for ordered insertion ordering.

      #. **Episode dict building** — transforms validated ``Message``
         objects into episode dicts with assigned sequence numbers.

      #. **PII detection & redaction** — if the org's PII config has
         ``mode != "off"``, a ``PIIService`` instance processes each
         message for sensitive content.  Only redacted content is stored.

      #. **Batch insertion** — ``episode_repo.batch_create()`` inserts all
         episodes in a single round-trip.

      #. **Commit** — the database transaction is committed so that
         enrichment workers can see the episodes immediately.

      #. **Enqueue ARQ tasks** — three task types per episode:

         * ``enrich_episode`` on ``high`` queue — combined entity
           extraction, fact extraction, and classification (replaces four
           old LLM workers).
         * ``embed_episode`` on ``high`` queue — pgvector embedding
           generation.
         * ``link_entities_to_episode`` on ``low`` queue — links
           extracted entities back to the episode.

      #. **Cache & invalidate** — stores the idempotency key and content
         hash in Redis, invalidates the project's context cache
         (``SCAN ctx:{org}:{project}:*`` + ``DEL``), and emits webhook
         events.

      :param org_id: The authenticated organization UUID.
      :param project_id: The project UUID.
      :param created_by: The authenticated user UUID for attribution.
      :param session_external_id: Optional session external ID.
         ``None`` → auto-create ``__default__`` session.
      :param messages: List of validated :class:`~schemas.memory.Message`
         objects.  1–1000 messages per request.
      :param idempotency_key: Optional ``Idempotency-Key`` header value.

      :returns: An :class:`~schemas.memory.IngestMemoryResponse` with
         ``job_id``, ``episode_count``, ``status``, and ``message``.

      :raises NotFoundError: If a specific ``session_external_id`` is
         provided but no matching session exists.
      :raises ValidationError: If messages fail byte-size or role
         validation (propagated from Pydantic).

      HTTP adapter: ``routers.memory.ingest_messages`` — ``POST /v1/projects/{project_id}/memory``
      Returns ``202 Accepted``.

   .. method:: async delete_project_memory(org_id, project_id) -> tuple[int, int]

      Soft-delete all memory for a project.  Calls
      ``episode_repo.soft_delete_by_project()`` and
      ``fact_repo.soft_delete_by_project()``.  Sessions are **not** deleted.

      :param org_id: The authenticated organization UUID.
      :param project_id: The project UUID.
      :returns: ``(episodes_deleted, facts_deleted)`` counts.

      HTTP adapter: ``routers.memory.delete_project_memory`` — ``DELETE /v1/projects/{project_id}/memory``
      Returns ``204 No Content``.

   .. rubric:: Idempotency Layer

   The system enforces two levels of deduplication:

   1. **Request-level idempotency** (``Idempotency-Key`` header)

      Cached in Redis at ``idempotency:<key>`` with a 48-hour TTL.
      The entire response is serialized via ``model_dump_json()``.  A
      duplicate key returns the cached response without any processing.

   2. **Content-level dedup** (SHA-256 content hash)

      Cached at ``contenthash:<sha256>`` with a 48-hour TTL.  Two
      identical payloads from different clients (or the same client
      without an idempotency key) produce the same hash and return the
      same ``job_id``.  The hash covers ``(project_id, session_id,
      sorted messages)`` with ``orjson.OPT_SORT_KEYS`` for canonical
      serialization.

   .. rubric:: Context Cache Invalidation

   After every successful ingestion, the service invalidates the project's
   context cache by scanning Redis for keys matching ``ctx:{org_id}:{project_id}:*``
   and deleting them in batches of 100.  This ensures subsequent context
   assembly queries fetch fresh data from the database.

   .. rubric:: PII Redaction

   When the organization's quotas JSONB contains a ``pii`` configuration
   with ``mode != "off"``, a ``PIIService`` instance processes each
   message synchronously before persistence.  The PII config is fetched
   from ``organizations.quotas -> 'pii'`` via the ``OrganizationRepository``.

   .. rubric:: ARQ Task Enqueueing

   .. method:: _enqueue_arq_tasks(job_id, org_id, project_id, session_id, episodes)

      Enqueues enrichment tasks for each episode on the appropriate ARQ
      queues:

      * ``enrich_episode`` → ``OpenZync:{env}:queue:high``
      * ``embed_episode`` → ``OpenZync:{env}:queue:high``
      * ``link_entities_to_episode`` → ``OpenZync:{env}:queue:low``

      If the ARQ pool is unavailable (Redis down), the method logs a
      critical error and re-raises — episodes are safe in PostgreSQL and
      must be picked up by a reconciliation worker.

      Each task receives ``episode_id``, ``content``, ``org_id``,
      ``project_id``, ``trace_id``, and ``metadata`` for correlation.

   .. rubric:: Session Resolution

   .. method:: _resolve_session(organization_id, project_id, created_by, session_external_id) -> Session

      Resolution rules:

      * **With** ``session_external_id``: looks up by external ID first,
        then falls back to parsing as a raw UUID (for backward compatibility
        with callers that pass the session's internal UUID).  Raises
        ``NotFoundError`` if no match.

      * **Without** ``session_external_id``: gets or creates the
        ``__default__`` session.  Uses ``INSERT ... ON CONFLICT DO NOTHING``
        for race safety across concurrent requests.

   .. rubric:: API Schema

   .. module:: schemas.memory

   .. class:: Message

      A single conversation turn.

      :param role: ``"user"`` | ``"assistant"`` | ``"system"`` | ``"tool"``
      :param content: Body text, max 64KB UTF-8 (enforced by
         ``field_validator`` checking byte length, not just char count).
      :param created_at: Optional ISO-8601 timestamp (server-assigned if
         omitted).
      :param metadata: Optional caller-defined key-value pairs.

   .. class:: IngestMemoryRequest

      Request body for ``POST /v1/projects/{project_id}/memory``.

      :param session_id: Optional session external ID.  ``None`` →
         auto-create ``__default__``.
      :param messages: List of :class:`Message`, 1–1000 items.

   .. class:: IngestMemoryResponse

      :param job_id: UUID string for tracking the enrichment job.
      :param episode_count: Number of episodes ingested.
      :param status: Always ``"accepted"``.
      :param message: Status message.

   .. class:: DeleteMemoryResponse

      :param status: ``"deleted"``
      :param episodes_deleted: Count of soft-deleted episodes.
      :param facts_deleted: Count of soft-deleted facts.


Fact Ingestion
~~~~~~~~~~~~~~

.. module:: services.fact_service

.. class:: FactService

   Service layer for batch fact ingestion — validates fact triples,
   deduplicates via content hash, persists to the ``facts`` table, and
   enqueues embedding tasks.

   .. method:: async ingest_facts(org_id, project_id, created_by, facts, session_external_id=None) -> FactBatchResponse

      Flow:

      #. Compute batch content hash (SHA-256 of ``(project_id, sorted facts)``).
      #. Check Redis for dedup — hit returns existing ``job_id``.
      #. Resolve session if ``session_external_id`` provided (raises
         ``NotFoundError`` if not found).
      #. Early return for empty fact lists.
      #. Bulk-insert facts into PostgreSQL via ``fact_repo.batch_create()``.
      #. Enqueue ``embed_fact`` ARQ tasks for each fact (one task per fact).
      #. Cache content hash in Redis (48h TTL).
      #. Emit ``FACT_EXTRACTED`` webhook event.

      :param org_id: The authenticated organization UUID.
      :param project_id: The project UUID.
      :param created_by: The authenticated user UUID.
      :param facts: List of :class:`~schemas.facts.FactTriple` objects.
      :param session_external_id: Optional session external ID.

      :returns: A :class:`~schemas.facts.FactBatchResponse` with
         ``job_id``, ``accepted_count``, ``status``, and ``message``.

      :raises NotFoundError: If ``session_external_id`` is provided but
         no matching session exists.

      HTTP adapter: ``routers.facts.ingest_facts`` — ``POST /v1/projects/{project_id}/facts``
      Returns ``202 Accepted``.

   .. method:: async list_facts_by_session(organization_id, session_id, limit=50, cursor=None) -> tuple[list[dict], str | None]

      List non-invalidated facts for a session with cursor-based pagination.

      :param organization_id: Tenant scope.
      :param session_id: The session UUID.
      :param limit: Max results per page (1–200).
      :param cursor: Opaque base64 cursor from a previous page.

      :returns: Tuple of (list of fact dicts, next_cursor or ``None``).

   .. rubric:: API Schema

   .. module:: schemas.facts

   .. class:: FactTriple

      :param subject: Subject entity name (1–500 chars).
      :param predicate: Relationship verb (1–200 chars).
      :param object: Object entity name (1–500 chars).
      :param content: Optional human-readable statement.  Auto-generated
         from ``"{subject} {predicate} {object}"`` if omitted.
      :param confidence: Extraction confidence (0.0–1.0, default 1.0).

   .. class:: FactBatchRequest

      :param session_id: Optional session external ID.
      :param facts: List of :class:`FactTriple`, 1–500 items.

   .. class:: FactBatchResponse

      :param job_id: UUID string for tracking.
      :param accepted_count: Number of facts accepted.
      :param status: Always ``"accepted"``.
      :param message: Human-readable message.

   .. class:: FactResponse

      Full fact representation returned from list endpoints.

      :param id: Internal fact UUID.
      :param content: Human-readable fact statement.
      :param subject/predicate/object: Triple components.
      :param confidence: Extraction confidence.
      :param source_episode_id: Optional FK to source episode.
      :param subject_type/object_type: ``"literal"`` or ``"entity"``.
      :param subject_entity_id/object_entity_id: Resolved entity UUIDs.
      :param created_at: Fact creation timestamp.


Session Management
~~~~~~~~~~~~~~~~~~

.. module:: services.session_service

.. class:: SessionService

   Provides CRUD operations for conversation sessions within a project.
   All DB access is delegated to ``SessionRepository``.

   .. method:: async create_session(organization_id, project_id, created_by, external_id, metadata=None) -> SessionResponse

      Creates a new session.  Checks for duplicates by external_id first.

      :param external_id: Caller-defined session identifier (unique per
         project).
      :param metadata: Optional metadata dict.

      :returns: A :class:`~schemas.sessions.SessionResponse`.

      :raises ConflictError: A session with this ``external_id`` already
         exists in the project.

      Emits ``SESSION_CREATED`` webhook event.

      HTTP adapter: ``routers.sessions.create_session`` — ``POST /v1/projects/{project_id}/sessions``
      Returns ``201 Created``.

   .. method:: async get_session(org_id, session_id, project_id=None) -> SessionResponse

      Get a session by UUID with aggregate statistics (message count,
      fact count, pending enrichment count).  Loads stats via
      ``repo.get_stats()``.

      :raises NotFoundError: Session not found or soft-deleted.

   .. method:: async get_session_by_external_id(org_id, project_id, external_id) -> SessionResponse

      Get a session within a project by its ``external_id``.

   .. method:: async get_session_by_uuid(org_id, session_id, project_id=None) -> SessionResponse

      Alias for ``get_session()`` — provided for callers that already
      have the UUID.

   .. method:: async list_sessions(org_id, project_id, limit=50, cursor=None, include_closed=False) -> PaginatedResponse[SessionListResponse]

      List sessions with cursor-based pagination.  Uses
      ``repo.batch_get_stats()`` for N+1 prevention.

      :param include_closed: If ``True``, include closed sessions.
         By default returns only open, non-deleted sessions (excluding
         ``__default__``).

      :raises ValidationError: If ``limit`` is outside 1–200.

   .. method:: async get_messages(org_id, session_id, limit=100, cursor=None, project_id=None) -> PaginatedResponse[MessageResponse]

      Get paginated messages for a session, ordered by ``sequence_number``
      for deterministic, tie-free ordering.

      :raises NotFoundError: If the session does not exist.
      :raises ValidationError: If ``limit`` is outside 1–500.

   .. method:: async delete_session(org_id, session_id, project_id=None) -> None

      Soft-delete a session.

      :raises NotFoundError: Session not found or already deleted.

      Emits ``SESSION_CLOSED`` webhook event.

      HTTP adapter: ``routers.sessions.delete_session`` — ``DELETE /v1/projects/{project_id}/sessions/{session_id}``
      Returns ``204 No Content``.

   .. rubric:: API Schema

   .. module:: schemas.sessions

   .. class:: CreateSessionRequest

      :param external_id: Caller-defined session ID (1–255 chars, unique
         per project).
      :param metadata: Optional metadata key-value pairs.

   .. class:: SessionResponse

      :param id: Internal session UUID.
      :param project_id: Project UUID.
      :param created_by: Creator UUID.
      :param external_id: Caller-defined identifier.
      :param metadata: Session metadata.
      :param is_active: Whether the session is accepting new messages.
      :param message_count: Total messages (episodes) in the session.
      :param fact_count: Total extracted facts.
      :param pending_enrichment_count: Messages pending enrichment.
      :param closed_at: Closure timestamp (``None`` if open).
      :param created_at/updated_at: Timestamps.

   .. class:: SessionListResponse

      Lightweight representation for list endpoints — excludes
      ``metadata`` and ``updated_at``.

   .. class:: MessageResponse

      :param id: Episode UUID.
      :param role: Message role.
      :param content: Message body.
      :param metadata: Per-message metadata.
      :param token_count: Approximate token count.
      :param sequence_number: Zero-based position within the session.
      :param created_at: Timestamp.


Enrichment Pipeline
-------------------

Enrichment is performed by ARQ background workers that process episodes
after ingestion.  The pipeline is idempotent — each step is guarded by
the :attr:`Episode.enrichment_status` bitmask.

ARQ Tasks
~~~~~~~~~

Each ingestion enqueues the following tasks per episode:

======================= ============= ====================================
Task                    Queue         Purpose
======================= ============= ====================================
``enrich_episode``      high          Combined entity extraction, fact
                                      extraction, and dialog classification
                                      (single LLM pass replacing four
                                      old workers).
``embed_episode``       high          Generate pgvector embedding from
                                      episode content.
``link_entities_to_episode`` low      Link extracted entities back to the
                                      source episode.
======================= ============= ====================================

For fact ingestion, one ``embed_fact`` task is enqueued per fact on the
``high`` queue.

The scheduled ``compute_observations`` worker (bit 6) runs graph-topology
pattern detection via :class:`~services.observation_service.ObservationService`.

Enrichment Status Bitmask
~~~~~~~~~~~~~~~~~~~~~~~~~

.. attribute:: Episode.enrichment_status

   :type: int

   The bitmask uses the following positions:

   ===== ========================== ==============================
   Bit   Meaning                    Set by
   ===== ========================== ==============================
   0     Entity extraction          ``enrich_episode`` worker
   1     Fact extraction            ``enrich_episode`` worker
   2     Dialog classification      ``enrich_episode`` worker
   3     Episode embedding          ``embed_episode`` worker
   6     Graph observations         ``compute_observations`` worker
   ===== ========================== ==============================

   Each worker checks its bit before proceeding.  If the bit is already
   set, the worker skips the operation — allowing safe retries after
   partial failures.

   Bit positions 4, 5, and 7+ are reserved for future enrichment passes.


Graph Services
--------------

GraphService
~~~~~~~~~~~~

.. module:: services.graph_service

.. class:: GraphService

   Wraps a :class:`~packages.graph_backend.interface.GraphBackend`
   implementation (typically :class:`~PostgresGraphBackend`) to provide a
   clean service-layer interface for graph query endpoints.  Every method
   enforces ``org_id`` isolation.

   All methods raise :class:`~core.exceptions.GraphBackendUnavailableError`
   when no backend is configured.

   .. method:: async ensure_user_exists(org_id, user_id) -> None

      :raises NotFoundError: If the user repository is not configured or
         the user does not exist in the organization.

   .. method:: async get_entities(org_id, project_id, *, entity_type=None, limit=50, cursor=None, session_id=None) -> dict

      List entity nodes with optional type filtering and cursor pagination.

      When ``session_id`` is provided, entities are scoped to those linked
      to episodes in the session.  Cursor pagination is not supported for
      session-scoped queries — all matching entities are returned in a
      single page.

      :returns: A dict with ``items``, ``next_cursor``, and ``has_more``.

   .. method:: async get_entity(org_id, project_id, entity_id) -> dict

      Get a single entity node with all incident edges.

      :returns: A dict with ``node`` and ``edges`` keys.

      :raises EntityNotFoundError: If the entity does not exist.

   .. method:: async delete_entity(org_id, project_id, entity_id) -> bool

      Delete an entity node from the knowledge graph.

      :returns: ``True`` if deleted, ``False`` if not found.

   .. method:: async get_edges(org_id, project_id, *, subject_id=None, subject_ids=None, predicate=None, limit=50, cursor=None) -> dict

      List relationship edges with optional filters.  Supports both
      single-entity and batch (``subject_ids``) queries.  Batch queries
      fetch edges in parallel via ``asyncio.gather`` and deduplicate by
      edge ID.

      :returns: A dict with ``items``, ``next_cursor``, and ``has_more``.

   .. method:: async get_communities(org_id, project_id) -> list[dict]

      List community summary nodes.  Communities are created by the
      scheduled ``summarise_community`` ARQ worker, which runs Label
      Propagation on the entity graph and stores community entities with
      ``entity_type='community'``.

      :returns: A list of community dicts with ``id``, ``name``,
         ``summary``, ``member_count``, and ``created_at``.

   .. rubric:: HTTP Adapters

   .. module:: routers.graph

   ========================================= ======== ==================================
   Endpoint                                  Method   Description
   ========================================= ======== ==================================
   ``/v1/projects/{id}/graph/nodes``         GET      List entity nodes (paginated)
   ``/v1/projects/{id}/graph/nodes/{nid}``   GET      Get single node with edges
   ``/v1/projects/{id}/graph/nodes/{nid}``   DELETE   Delete entity node
   ``/v1/projects/{id}/graph/edges``         GET      List edges (by subject_id or
                                                      subject_ids)
   ``/v1/projects/{id}/graph/communities``   GET      List community summaries
   ========================================= ======== ==================================

   .. rubric:: Graph Endpoint Details

   ``GET /v1/projects/{project_id}/graph/nodes``

   Query parameters:

   * ``entity_type`` — optional filter by type (e.g. ``"Person"``).
   * ``session_id`` — optional scope to a specific session.
   * ``limit`` — 1–200 (default 50).
   * ``cursor`` — opaque pagination cursor.

   ``GET /v1/projects/{project_id}/graph/edges``

   Query parameters:

   * ``subject_id`` — UUID of the source entity (single-entity mode).
   * ``subject_ids`` — comma-separated UUIDs (batch mode).
   * ``predicate`` — optional edge label filter.
   * ``limit`` — 1–200 (default 50).
   * ``cursor`` — opaque pagination cursor.

   Either ``subject_id`` or ``subject_ids`` is required (422 if both
   omitted).


ObservationService
~~~~~~~~~~~~~~~~~

.. module:: services.observation_service

.. class:: ObservationService

   Graph-topology pattern detection and observation persistence.  This
   service implements the **second-pass inference** over graph topology
   that surfaces high-level observations not visible from individual
   messages or triples.

   Uses the :class:`~packages.graph_backend.interface.GraphBackend` ABC
   for all graph operations.  Non-graph queries (episode counts, fact
   predicate counts) use direct SQL via the injected ``AsyncSession``.

   The LLM is used only for generating the ``content`` (natural-language
   description) field of observations.  When unavailable, a template-based
   description is used instead.

   .. note::

      All detection is **warn-only** — the service logs contradictions but
      **never mutates** existing facts, relationships, or observations.
      Auto-expiry or corrective mutation must go behind a feature flag.

   .. rubric:: Pattern Dataclasses

   .. class:: CoOccurrencePattern

      :param entity_a_id/entity_a_name: First entity
      :param entity_b_id/entity_b_name: Second entity
      :param co_count: Episodes where both appear
      :param total_episodes: Total project episodes
      :param relationship_ids: Supporting ``graph_relationships`` UUIDs

   .. class:: TemporalGapPattern

      :param entity_id/entity_name: The entity
      :param appearance_count: Number of episode appearances
      :param pattern_type: ``"periodic"`` | ``"widening"`` | ``"narrowing"``
         | ``"burst"`` | ``"irregular"``
      :param mean_gap_hours/stddev_gap_hours/min_gap_hours/max_gap_hours:
         Gap statistics in hours
      :param span_days: Total time span in days

   .. class:: BehavioralPattern

      :param entity_id/entity_name/entity_type: Entity identifiers
      :param frequent_predicates: Dict of ``{predicate: count}``
      :param total_facts: Total facts about this entity
      :param description_hint: Structured hint for LLM description
         generation

   .. rubric:: Public Methods

   .. method:: async run_full_project_scan(project_id, organization_id, llm_backend=None) -> dict[str, int]

      Orchestrates the full detection pipeline:

      #. Validate backend availability.
      #. Detect co-occurrences via
         :meth:`detect_co_occurrences`.
      #. Detect temporal gaps via
         :meth:`detect_temporal_gaps`.
      #. Detect behavioral patterns via
         :meth:`detect_behavioral_patterns`.
      #. Generate descriptions (LLM or template-based).
      #. Persist via ``backend.upsert_observation()``.

      :param llm_backend: Optional LLM backend for content generation.
         ``None`` = template-based descriptions.
      :returns: Count dict like ``{"co_occurrence": 5, "temporal_pattern": 3, ...}``

   .. method:: async detect_co_occurrences(project_id, organization_id=None) -> list[CoOccurrencePattern]

      Finds entity pairs that co-appear in episodes above
      ``min_co_count`` (default 3).  Delegates to
      ``backend.get_co_occurring_entity_pairs()``.

      Returns pairs ordered by co-occurrence count descending.

   .. method:: async detect_temporal_gaps(project_id, organization_id=None) -> list[TemporalGapPattern]

      For each entity with sufficient appearances (``min_appearances_for_temporal``,
      default 3), fetches timestamps and classifies the gap pattern:

      * **Periodic** — coefficient of variation (CV) < 0.25
      * **Widening** — gaps consistently increase (60% threshold)
      * **Narrowing** — gaps consistently decrease (60% threshold)
      * **Burst** — tight clusters with long inter-cluster gaps
      * **Irregular** — none of the above

      Results sorted by regularity (periodic first, then by mean gap).

   .. method:: async detect_behavioral_patterns(project_id, organization_id=None) -> list[BehavioralPattern]

      Queries the ``facts`` table directly for predicate frequency per
      entity (as subject).  Only predicates appearing 2+ times are
      included.  Entity names are resolved via ``backend.resolve_entity_names()``.

      Results sorted by total facts descending.

   .. rubric:: Description Generation

   Three description methods provide template-based fallback when no LLM
   backend is available:

   * :meth:`build_co_occurrence_description` — percentage-based::
         "Entity X appears alongside Entity Y in 12 out of 100 episodes
         (12% co-occurrence rate)."

   * :meth:`build_temporal_description` — pattern-labeled::
         "Entity X appears at regular intervals (mean gap: 48.0h, span:
         30.0d, 12 appearances)."

   * :meth:`build_behavioral_description` — predicate-dominant::
         "Entity X most frequently exhibits the predicate 'likes' (5 out
         of 8 facts)."

   Each method accepts an optional ``llm_content`` parameter.  When
   provided (from an LLM call), it is used verbatim.

   .. rubric:: Persistence

   * Co-occurrences produce **two** observations per pair — one for each
     direction (A→B and B→A), each with a swapped description.
   * Temporal and behavioral patterns produce one observation per entity
     (``related_entity_id`` is ``NULL``).
   * Confidence scores are computed per observation type:

     * Co-occurrence: ``min(co_count / co_confidence_cap, 1.0)``
     * Temporal: mapped from pattern type (periodic=0.85, irregular=0.40)
     * Behavioral: ``min(top_count / total_facts * 1.5, 0.95)``


TemporalValidationService
~~~~~~~~~~~~~~~~~~~~~~~~~

.. module:: services.temporal_service

.. class:: TemporalValidationService

   **Warn-only** temporal consistency checks for extracted facts.
   Gathers data on temporal anomalies without mutating any data.
   Auto-mutation requires a feature flag (future phase).

   .. class:: TemporalWarning

      :param code: Machine-readable code (``"overlap"``, ``"invalid_range"``,
         ``"future_date"``, ``"batch_overlap"``).
      :param message: Human-readable description.
      :param detail: Structured context for log aggregation.

   .. method:: async check_project_temporal_consistency(project_id, *, organization_id=None) -> list[dict]

      Scans all non-invalidated facts for overlapping triples with the
      same ``(subject, predicate, object)`` but different
      ``source_episode_id``.  The DB exclusion constraint only prevents
      overlaps within the same episode, so cross-episode duplicates are
      detected here.

   .. method:: async check_fact_ranges(project_id, *, organization_id=None) -> list[dict]

      Scans for:

      * ``valid_to < valid_from`` — logically impossible ranges.
      * ``valid_from > now() + 24h`` — future-dated facts, indicating
        possible data-pipeline issues.

   .. method:: async validate_batch(facts) -> list[dict]

      Pre-insert validation for incoming batches.  Detects
      self-overlapping triples within the batch before they reach
      PostgreSQL.

   .. method:: _ranges_overlap(from_a, to_a, from_b, to_b) -> bool

      Checks whether two half-open ``[from, to)`` ranges overlap.
      ``None`` is treated as unbounded (``-infinity`` / ``infinity``),
      mirroring the exclusion constraint's ``COALESCE`` logic.


Context Retrieval Pipeline
--------------------------

HybridRetriever
~~~~~~~~~~~~~~~

.. module:: services.hybrid_retriever

.. class:: HybridRetriever

   Combines three retrieval strategies and merges results using Reciprocal
   Rank Fusion (RRF) for robust context retrieval:

   1. **Vector search** (pgvector ``<=>`` cosine similarity) — semantic
      matching on episode and fact embeddings.
   2. **BM25 search** (PostgreSQL ``ts_rank``) — keyword/lexical matching
      on episode and fact content.
   3. **Graph BFS** — entity-relationship traversal via configured
      ``GraphBackend`` backends (PostgreSQL recursive CTE, SurrealQL graph
      traversal, or FalkorDB Cypher).

   .. rubric:: Constructor

   .. method:: __init__(db, org_id, redis=None, graph_backends=None, org_config=None, reranker=None)

      :param db: Async SQLAlchemy session (request-scoped).
      :param org_id: Authenticated organization UUID for tenant isolation.
      :param redis: Optional async Redis client for result caching.
      :param graph_backends: List of ``GraphBackend`` instances.
      :param org_config: Org configuration (embedding model, dimensions,
         reranker settings).
      :param reranker: Optional ``CrossEncoderReranker`` for second-pass
         re-ranking.

   .. rubric:: Public Methods

   .. method:: async hybrid_search(query, project_id, limit=20) -> dict

      Runs **five concurrent retrieval legs**:

      * Episode vector search (``_vector_search_episodes``)
      * Episode BM25 search (``_bm25_search_episodes``)
      * Fact vector search (``_vector_search_facts``)
      * Fact BM25 search (``_bm25_search_facts``)
      * Graph BFS (``_graph_bfs_search`` — entity traversal)

      **No silent partial results**: if any single leg fails, the entire
      search fails with :class:`~core.exceptions.SearchLegFailedError`.
      The DB session is rolled back on failure.

      After retrieval, results are merged per type via RRF, then
      optionally re-ranked by the ``reranker``.

      :param query: Natural-language search query.
      :param project_id: Project UUID for scoping.
      :param limit: Max items per source type before RRF merge.

      :returns: A dict with:

      * ``episodes`` — RRF-merged + optionally re-ranked episodes.
      * ``facts`` — RRF-merged + optionally re-ranked facts.
      * ``entities`` — Graph BFS traversal results.
      * ``communities`` — Currently an empty list (placeholder for
        future community detection integration).
      * ``source_counts`` — Item count per source type and leg.
      * ``total_items`` — Sum of all items across sources.
      * ``query_embedding_dim`` — Dimension of the generated embedding.

   .. rubric:: Vector Search

   .. method:: _embed_query(query) -> list[float]

      Generates an embedding vector via the configured LLM backend.
      Uses :func:`core.llm.resolve_backend` with the org's
      ``embedding_backend`` and ``embedding_model`` configuration.

   .. method:: _vector_search_episodes(query, project_id, limit=20) -> list[dict]

      Searches ``episodes.embedding`` (cast to ``vector(dim)`` at query
      time) using the ``<=>`` cosine distance operator.  Filters to
      non-deleted episodes with non-null, non-empty embeddings.

      Embedding dimension defaults to 1536 (``text-embedding-3-small``)
      and can be overridden by ``org_config.embedding_dim``.

   .. method:: _vector_search_facts(query, project_id, limit=20) -> list[dict]

      Same pattern as episodes but operates on ``facts.embedding``.
      Filters to non-invalidated facts.

   .. rubric:: BM25 Search

   .. method:: _bm25_search_episodes(query, project_id, limit=20) -> list[dict]

      Uses ``plainto_tsquery('english', query)`` for user-friendly query
      parsing and ``ts_rank`` for relevance scoring.  Filters via
      ``to_tsvector('english', content) @@ tsquery``.

   .. method:: _bm25_search_facts(query, project_id, limit=20) -> list[dict]

      Same BM25 pattern on the ``facts`` table.

   .. rubric:: Graph BFS Search

   .. method:: _graph_bfs_search(query, project_id) -> list[dict]

      Runs ``backend.retrieve_graph()`` on every registered
      ``GraphBackend`` in parallel.  Results are deduplicated by entity
      ``id`` (first occurrence wins — lower distance), sorted by
      ``distance`` ascending, and capped at ``MAX_BFS_RESULTS`` (50).

      When no backends are configured, returns an empty list with a
      debug log.

   .. rubric:: RRF Merge

   .. method:: _rrf_merge(ranked_lists, top_n=20) -> list[dict]

      Reciprocal Rank Fusion::

          score(d) = Σ 1 / (RRF_K + rank_s(d))

      where ``RRF_K = 60`` and ``rank_s(d)`` is the 1-based rank of
      document ``d`` in source ``s``.  Results are deduplicated by ``id``.
      The fused score is stored in the ``rrf_score`` key.

   .. rubric:: Re-ranking

   When a ``reranker`` is configured:

   1. The candidate pool size is widened from ``limit`` to
      ``reranker_top_k`` (default 50) before RRF.
   2. Top RRF-scored candidates are sent to
      ``reranker.rerank(query, top_k)`` which returns ``top_n``
      (default 10) re-scored results.
   3. Re-ranker scores are accessible via the ``reranker_score`` key.

   Re-ranking latency is recorded in the ``reranker_latency_seconds``
   Prometheus histogram (labeled by backend name).

   .. rubric:: Observability

   All retrieval legs log structured debug messages with result counts.
   Graph search total latency is recorded in
   ``graph_search_latency_seconds``.  Every failure is logged with
   ``exc_info=True`` and the DB session is rolled back before
   re-raising :class:`~core.exceptions.SearchLegFailedError`.


ContextService
~~~~~~~~~~~~~~

.. module:: services.context_service

.. class:: ContextService

   Orchestrates the retrieval → format → cache pipeline for LLM context
   assembly.  Every public method is idempotent — identical inputs
   produce identical outputs (modulo cache staleness).

   .. method:: async assemble(project_id, query, limit=20, format="text") -> dict

      Full pipeline:

      #. Build a cache key from ``(org_id, project_id, query)`` and check
         Redis.  Cache hit → return immediately with ``cache_hit=True``.
      #. On miss, run :meth:`HybridRetriever.hybrid_search` with the
         query.
      #. Format results via :func:`~services.context_formatter.format_text`
         or :func:`~services.context_formatter.format_json`.
      #. Store the formatted string in Redis (TTL = 30 seconds).
      #. Return the context string with assembly metadata.

      :param project_id: Project UUID.
      :param query: Natural-language query (1–2000 chars).
      :param limit: Max items per source type (1–100, default 20).
      :param format: ``"text"`` (default) or ``"json"``.

      :returns: A dict with:

      * ``context`` — The assembled context string (plain text or
        serialized JSON).
      * ``metadata`` — Dict with ``cache_hit``, ``assembly_time_ms``,
        ``source_counts``, and ``total_items``.

   .. rubric:: Observability

   * Latency is recorded in the ``context_latency_seconds`` Prometheus
     histogram (labeled ``"warm"`` for cache hits, ``"cold"`` for misses).
   * Structured logs include the top episode/fact preview (first 500 chars
     with scores), query embedding dimension, configured embedding
     dimension, and full source counts.
   * The ``X-Cache`` HTTP header is set in the response (``"HIT"`` or
     ``"MISS"``).

   .. rubric:: HTTP Adapter

   ``GET /v1/projects/{project_id}/context``

   Query parameters:

   * ``query`` (required, 1–2000 chars) — natural-language query.
   * ``limit`` (optional, 1–100, default 20) — items per source type.
   * ``format`` (optional, ``"text"`` | ``"json"``, default ``"text"``).

   ``router.context.get_context`` — Returns :class:`~schemas.context.ContextResponse`.

   The router handles SurrealDB connection pooling when
   ``org_config.graph_backend == "surrealdb"``.  SurrealDB connections
   are lazily acquired from the connection pool, so SurrealDB is only
   contacted for orgs that explicitly configure it.

   .. rubric:: API Schema

   .. module:: schemas.context

   .. class:: ContextRequest

      :param query: Natural-language query (1–2000 chars).
      :param limit: Max items per source type (1–100, default 20).
      :param format: Output format — ``"text"`` or ``"json"``.

   .. class:: ContextResponse

      :param context: The assembled context block as a string.
      :param metadata: :class:`ContextMetadata` with cache/timing/counts.

   .. class:: ContextMetadata

      :param cache_hit: Whether served from cache.
      :param assembly_time_ms: Wall-clock time in milliseconds.
      :param source_counts: Per-source item breakdown.
      :param total_items: Total items in context.


ContextFormatter
~~~~~~~~~~~~~~~~

.. module:: services.context_formatter

.. function:: format_text(episodes, facts, entities, communities) -> str

   Formats retrieval results as a plain-text context block suitable for
   LLM prompt injection.  Sections are ordered by likely relevance:

   #. **Recent Episodes** — with score (re-reranker > RRF > raw), role,
      and truncated content (capped at 2000 characters).  Multi-line
      content is indented for readability.
   #. **Facts** — with confidence and score.
   #. **Entities** — with name, type, distance, and summary.
   #. **Community Summaries** — with name and summary.

   Each section is visually separated by a horizontal rule (``─`` × 72).
   Returns ``"No context found."`` if all inputs are empty.

.. function:: format_json(episodes, facts, entities, communities) -> dict

   Formats retrieval results as a structured JSON object with typed
   arrays per source category.  Internal ranking fields (``score``,
   ``rrf_score``, ``reranker_score``, ``distance``) are stripped from
   each item before return so that the JSON is clean for LLM consumption.

   :returns: A dict with ``episodes``, ``facts``, ``entities``, and
      ``communities`` keys, each containing a cleaned list of item dicts.

   Internal cleanup functions:

   * ``_clean_episodes`` — keeps ``id``, ``role``, ``content``,
     ``created_at``.
   * ``_clean_facts`` — keeps ``id``, ``content``, ``subject``,
     ``predicate``, ``object``, ``confidence``, ``created_at``.
   * ``_clean_entities`` — keeps ``id``, ``name``, ``type``, ``summary``.
   * ``_clean_communities`` — keeps ``id``, ``name``, ``summary``.


Search Endpoint
~~~~~~~~~~~~~~~

.. module:: routers.search

``GET /v1/projects/{project_id}/search`` — Hybrid search across project
memory.

Query parameters:

* ``query`` (required, 1–2000 chars) — search query.
* ``limit`` (optional, 1–100, default 20) — results per source type.
* ``types`` (optional, default ``"episodes,facts"``) — comma-separated
  result type filter.  Valid values: ``episodes``, ``facts``,
  ``entities``, ``communities``.

The endpoint creates a :class:`HybridRetriever` with the org's configured
graph backends (including optional SurrealDB lazy connection), runs hybrid
search, filters by the requested types, and returns a flat result list::

    {
      "query": "original query",
      "results": [...],
      "total": 42
    }


Auxiliary Services
------------------

ClassificationService
~~~~~~~~~~~~~~~~~~~~~

.. module:: services.classification_service

.. class:: ClassificationService

   Read-only service for querying dialog classification results produced
   by the enrichment pipeline.

   .. method:: async get_classifications_for_session(org_id, session_id, project_id=None) -> list[ClassificationResponse]

      Returns all classifications for episodes in a session, ordered by
      episode sequence number.

      :raises NotFoundError: If the session does not exist.

   .. method:: async get_classification_for_episode(org_id, episode_id) -> ClassificationResponse | None

      Returns the classification for a specific episode, or ``None`` if
      not yet classified.

   .. method:: async count_classifications_for_session(org_id, session_id, project_id=None) -> int

      Counts classified episodes in a session.

   .. rubric:: HTTP Adapters

   .. module:: routers.classifications

   * ``GET /v1/projects/{project_id}/sessions/{session_id}/classifications``
     — list all classifications in a session.
   * ``GET /v1/projects/{project_id}/sessions/{session_id}/classifications/{episode_id}``
     — get classification for a specific episode (returns 404 if
     unclassified).

   .. rubric:: API Schema

   .. module:: schemas.classifications

   .. class:: ClassificationResponse

      :param id: Classification UUID.
      :param episode_id: Source episode UUID.
      :param intent: Predicted intent label.
      :param emotion: Predicted emotion label.
      :param valence: Sentiment valence.
      :param arousal: Emotional arousal level.
      :param confidence: Classifier confidence (0.0–1.0).
      :param created_at: Classification timestamp.
      
      The ``raw`` LLM output field from the model is excluded from this
      response — available via direct DB access only.

   .. class:: ClassificationListResponse

      :param data: List of :class:`ClassificationResponse`.
      :param total: Total count.


CustomInstructionService
~~~~~~~~~~~~~~~~~~~~~~~~

.. module:: services.custom_instruction_service

.. function:: format_custom_instructions(instructions) -> str

   A pure formatting utility that turns a list of ``{name, text}`` dicts
   into a prompt-ready text block.  Each instruction produces a Markdown
   section with the name as a level-3 heading followed by the text body.
   Sections are separated by a blank line.

   Returns an empty string if ``instructions`` is empty.

   Example::

      >>> format_custom_instructions([
      ...     {"name": "legal", "text": "Use legal terminology."},
      ... ])
      '### legal\nUse legal terminology.'

   This is deliberately **not** a class — it is a module-level pure
   function with no dependencies, no database access, and no async
   operations.  It is safe to import and use anywhere in the codebase
   without DI wiring.


UserSummaryService
~~~~~~~~~~~~~~~~~~

.. module:: services.user_summary_service

.. class:: UserSummaryService

   Coordinates user summary generation (trigger + background ARQ task),
   summary retrieval, and custom-instruction management for the
   ``user_summary`` scope.

   .. method:: async trigger_generation(org_id, user_id) -> UserSummaryTriggerResponse

      Enqueues a background ``generate_user_summary`` ARQ job.  Rate-limited
      to one generation per 5 minutes per user via Redis ``SET NX EX``.

      :raises CacheUnavailableError: If Redis is not configured.
      :raises RateLimitError: If within the 5-minute cooldown window.

   .. method:: async get_summary(org_id, user_id) -> UserSummaryResponse | None

      Fetches the currently stored summary for a user.  Returns ``None``
      if no summary has been generated yet.

      .. caution::

         ``org_id`` is accepted for future tenant-isolation enforcement,
         but the underlying ``UserRepository.get_summary`` currently
         queries by ``user_id`` only.  If users span organisations, a
         WHERE clause on ``organization_id`` must be added.

   .. rubric:: Custom Instructions CRUD (user_summary scope)

   .. method:: async get_instructions(org_id, user_id) -> list[dict]

      Fetches custom instructions scoped to ``"user_summary"`` for the
      user.  Returns ``[{name, text}, ...]`` dicts ordered alphabetically
      by name.

   .. method:: async set_instructions(org_id, user_id, instructions) -> list[dict]

      Atomically replaces all custom instructions for this scope + target.
      Deletes existing rows and bulk-inserts new ones in a single
      transaction.

   .. method:: async delete_instructions(org_id, user_id) -> None

      Deletes all custom instructions for this scope + target.

   .. rubric:: Rate Limiting

   .. method:: _check_rate_limit(org_id, user_id) -> bool

      Uses Redis ``SET key "1" NX EX 300`` (5-minute TTL) on key
      ``ratelimit:summary:{org_id}:{user_id}``.


Configuration
-------------

The Memory & Context domain is configured through the organization's
``OrgConfig`` (resolved at request time via :func:`dependencies.org_config.get_org_config`).
Key configuration settings that affect behavior:

====================== ======================================================
Setting                Effect
====================== ======================================================
``embedding_backend``  LLM backend used for embedding generation
                       (vector search query embedding).
``embedding_model``    Model name passed to the embedding backend.
``embedding_dim``      Vector dimension for ``pgvector`` cast and dimension
                       mismatch detection in logs (default 1536).
``reranker_top_k``     Candidate pool size for re-ranking (default 50).
``reranker_top_n``     Final results returned by the re-ranker (default 10).
``graph_backend``      Graph backend type — ``"postgres"``, ``"surrealdb"``,
                       or ``"none"``.
``context_cache_ttl``  TTL for context assembly cache entries in seconds
                       (default: 30).
====================== ======================================================


Error Handling & Performance
----------------------------

All domain services use the shared exception hierarchy from
:mod:`core.exceptions`:

.. list-table::
   :header-rows: 1

   * - Exception
     - HTTP Status
     - When Raised
   * - ``NotFoundError``
     - 404
     - Session, entity, or project not found.
   * - ``ConflictError``
     - 409
     - Duplicate session external_id.
   * - ``ValidationError``
     - 422
     - Invalid limit, cursor, or input shape.
   * - ``SearchLegFailedError``
     - 500
     - Any leg of hybrid search fails (no silent partial results).
   * - ``GraphBackendUnavailableError``
     - 503
     - Graph backend not configured or connection failed.
   * - ``RateLimitError``
     - 429
     - User summary generation rate limited.
   * - ``CacheUnavailableError``
     - 503
     - Redis required but not configured.
   * - ``EntityNotFoundError``
     - 404
     - Graph entity not found.

Performance considerations:

* **N+1 prevention**: Session list endpoint uses ``batch_get_stats()`` to
  load message and fact counts for all returned sessions in a single query.
* **Connection pooling**: SurrealDB connections are lazily acquired from a
  connection pool — SurrealDB is only contacted for orgs that explicitly
  configure it.  Avoids unnecessary connections and prevents 503 when
  SurrealDB is not running.
* **Cache invalidation**: Context cache is invalidated after every
  successful ingestion via Redis ``SCAN`` + ``DEL``, ensuring context
  assembly always reflects the latest data.
* **Idempotency**: Every write endpoint supports either
  ``Idempotency-Key`` header (48h TTL) or content-hash dedup (48h TTL)
  to prevent duplicate processing on retry.
