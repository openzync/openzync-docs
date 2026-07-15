Graph Backend Abstraction
=========================

.. module:: packages.graph_backend

The graph backend abstraction provides a **pluggable interface** for graph
database operations — entity CRUD, relationship management, BFS traversal,
full-text search, observation storage, entity merging, and batch analysis.
An application-level dispatcher resolves a per-organisation backend at
request time so that different orgs can use different graph engines without
caller code changes.

.. rubric:: Architecture overview

::

    +-- core/graph_backend.py ------------+
    |  GraphBackendDispatcher             |
    |    - register(name, cls)            |
    |    - resolve_and_create(org, ...)   |
    |    - create_all_backends(...)       |
    |  init_dispatcher()                  |
    +-------------------------------------+
                   |
                   v
    +-- packages/graph_backend/interface.py --+
    |  GraphBackend (ABC)                     |
    |    - create_entity()                    |
    |    - get_entity()                       |
    |    - traverse()                         |
    |    - search_entities()                  |
    |    - ... (24 abstract methods total)    |
    +-----------------------------------------+
                   |
          +--------+--------+
          |        |         |
          v        v         v
    +--------+ +--------+ +--------+
    |Postgres| |FalkorDB| |Surreal |
    |Backend | |Backend | |DB      |
    |(SQL    | |(Cypher)| |Backend |
    | CTE)   | |        | |(SurQL) |
    +--------+ +--------+ +--------+

.. contents:: Sections
   :local:
   :depth: 2
   :class: this-will-duplicate-information-and-it-is-still-useful-here


Backend Selection
-----------------

Module: :mod:`core.graph_backend`

The active graph backend is determined by the :attr:`graph_backend` field in
:class:`~schemas.organization_config.OrgConfigBase` (stored in the
``organizations.config`` JSONB column per org).  The default value is
``"surrealdb"``.

.. code-block:: python

   from schemas.organization_config import OrgConfigBase

   config = OrgConfigBase(graph_backend="postgres")

To disable graph features for an org, set ``graph_backend`` to ``"none"``.

There is **no** ``OZ_GRAPH_BACKEND`` environment variable — the setting is
exclusively per-org.  Connection credentials for SurrealDB (URL, user,
password, namespace, database) are also stored in the per-org config.  For
FalkorDB, a single app-level connection pool is created from the system-wide
:class:`~core.config.Settings` fields (``FALKORDB_URL``,
``FALKORDB_MAX_CONNECTIONS``, ``FALKORDB_SOCKET_TIMEOUT``).

Implementation status
~~~~~~~~~~~~~~~~~~~~~

+-----------------+----------------------------------+-------------------------+
| Backend name    | Engine                           | Status                  |
+=================+==================================+=========================+
| ``"postgres"``  | PostgreSQL recursive CTEs,       | Fully implemented       |
|                 | ``pg_trgm``, ``pgvector``        |                         |
+-----------------+----------------------------------+-------------------------+
| ``"surrealdb"`` | SurrealDB native graph relations | Fully implemented       |
|                 | (``RELATE`` / arrow syntax),     |                         |
|                 | BM25 full-text search            |                         |
+-----------------+----------------------------------+-------------------------+
| ``"falkordb"``  | FalkorDB (RedisGraph module),    | Fully implemented       |
|                 | ``algo.bfs()``, RediSearch       |                         |
|                 | full-text                        |                         |
+-----------------+----------------------------------+-------------------------+


GraphBackendDispatcher
----------------------

Module: :mod:`core.graph_backend`

.. class:: GraphBackendDispatcher()

   Registry of backend **classes** (not instances) with per-org resolution.
   This is an app-level singleton.  It does **not** hold backend instances
   because backends need request-scoped connections (an ``AsyncSession``
   for Postgres, an ``AsyncSurreal`` for SurrealDB, or a FalkorDB client).
   Instead, it holds the **classes** and creates a fresh instance on every
   call to :meth:`resolve_and_create`.

   **Lifecycle**::

       # App startup (called once):
       from core.graph_backend import init_dispatcher
       app.state.graph_backend_dispatcher = init_dispatcher()

       # Per-request resolution:
       dispatcher = request.app.state.graph_backend_dispatcher
       backend = dispatcher.resolve_and_create(org_config, db)
       entity = await backend.create_entity(org_id=..., name="Acme", ...)

   .. method:: register(name, backend_cls)

      :param str name: Short identifier (e.g. ``"postgres"``, ``"neo4j"``).
      :param type[GraphBackend] backend_cls: A class that implements the
          :class:`GraphBackend` ABC.

      Register a backend class under a short name.  If the name is already
      registered, the new class overwrites the previous one — this allows
      test suites to replace backends without reference counting.

      Example::

          dispatcher = GraphBackendDispatcher()
          dispatcher.register("postgres", PostgresGraphBackend)
          dispatcher.register("surrealdb", SurrealGraphBackend)

   .. method:: resolve_and_create(org_config, db, surreal=None, falkordb_client=None)

      :param OrgConfigBase | None org_config: The resolved per-org
          configuration.  ``None`` is treated as graph disabled.
      :param AsyncSession db: Request-scoped SQLAlchemy session.  Passed
          only to the PostgreSQL backend.
      :param surreal: An optional ``AsyncSurreal`` instance from the per-org
          connection pool.  Passed only to the SurrealDB backend.
      :type surreal: Any | None
      :param falkordb_client: An optional ``FalkorDB`` async client instance
          from the app-level connection pool.  Passed only to the FalkorDB
          backend.
      :type falkordb_client: Any | None
      :returns: An initialised :class:`GraphBackend` instance, or ``None``
          if graph features are disabled for this org.
      :rtype: GraphBackend | None
      :raises ValueError: If the backend name from ``org_config`` is not
          registered and is not ``"none"``.
      :raises GraphBackendUnavailableError: When the required backend
          client is not available (e.g., ``surreal`` is ``None`` but
          ``"surrealdb"`` is configured).

      Resolution steps:

      1. If ``org_config`` is ``None`` or ``org_config.graph_backend`` is
         not set or equals ``"none"`` → returns ``None`` (graph disabled).
      2. Looks up the backend name in the registry.
      3. Creates a new instance with backend-specific kwargs.

      Backend-specific kwargs:

      * **``"postgres"``**: ``db`` and ``max_traversal_depth``.
      * **``"surrealdb"``**: ``surreal`` and ``max_traversal_depth``.
      * **``"falkordb"``**: ``client`` and ``max_traversal_depth``.

   .. method:: create_all_backends(db, org_config=None, surreal=None, falkordb_client=None)

      :param AsyncSession db: SQLAlchemy session for the Postgres backend.
      :param OrgConfigBase | None org_config: Optional per-org config for
          ``graph_max_traversal_depth``.
      :param surreal: Optional ``AsyncSurreal`` instance.
      :type surreal: Any | None
      :param falkordb_client: Optional ``FalkorDB`` client.
      :type falkordb_client: Any | None
      :returns: A list of initialised :class:`GraphBackend` instances (one
          per registered backend).
      :rtype: list[GraphBackend]

      Multi-backend equivalent of :meth:`resolve_and_create`.  Creates **all**
      registered backends instead of picking one from the org config.  Each
      backend receives backend-specific kwargs.  Used by
      ``HybridRetriever`` to run multiple backends in parallel and merge
      results.

   .. method:: resolve_backend_name(org_config)

      :param OrgConfigBase | None org_config: The resolved per-org config.
      :returns: The backend name string (e.g. ``"postgres"``), or ``None``
          if graph features are disabled.
      :rtype: str | None

      Pure lookup — determines which backend an org should use without
      instantiating it.  Useful when the caller only needs to know *which*
      backend is configured.

.. function:: init_dispatcher()

   :returns: A populated :class:`GraphBackendDispatcher` with all registered
       backends.
   :rtype: GraphBackendDispatcher

   Call once during the application lifespan and store the result on
   ``app.state``::

       from core.graph_backend import init_dispatcher

       app.state.graph_backend_dispatcher = init_dispatcher()

   Registers all three backends in default-order priority::

       dispatcher.register("surrealdb", SurrealGraphBackend)
       dispatcher.register("postgres", PostgresGraphBackend)
       dispatcher.register("falkordb", FalkorGraphBackend)

   To add a new backend, import its class and register it inside this
   function before returning the dispatcher.


GraphBackend Interface
----------------------

Module: :mod:`packages.graph_backend.interface`

.. class:: GraphBackend

   Abstract base class for all graph database backends.  Every method
   requires ``org_id`` and ``project_id`` — OpenZync enforces strict
   organisational and project-level isolation.  No cross-project graph
   traversal is possible.

   All methods are :func:`abc.abstractmethod` and must be implemented by
   concrete subclasses.

   The interface is organised into six logical groups:

   * **Entity CRUD** — :meth:`create_entity`, :meth:`get_entity`,
     :meth:`update_entity`, :meth:`delete_entity`.
   * **Relationship CRUD** — :meth:`create_relationship`,
     :meth:`expire_relationship`.
   * **Traversal & Search** — :meth:`traverse`, :meth:`search_entities`,
     :meth:`retrieve_graph`.
   * **Entity Listing** — :meth:`list_entities`,
     :meth:`list_entity_edges`, :meth:`get_entity_with_edges`.
   * **Entity-Episode Linking** — :meth:`link_entity_to_episode`,
     :meth:`get_entities_for_session`,
     :meth:`get_co_occurring_entity_pairs`.
   * **Bulk / Merge Operations** — :meth:`get_all_entities`,
     :meth:`get_all_relationships`, :meth:`bulk_search_entities`,
     :meth:`merge_entities`, :meth:`create_relationship_bulk`.
   * **Observations** — :meth:`upsert_observation`,
     :meth:`get_observations`,
     :meth:`get_entity_appearance_timestamps`,
     :meth:`get_relationship_ids_between`.
   * **Aggregate Queries** — :meth:`get_total_entity_linked_episode_count`,
     :meth:`resolve_entity_names`.
   * **Soft-Delete / Expiry** — :meth:`expire_relationship`.
   * **Observability** — :meth:`health_check`.

   Entity CRUD
   ~~~~~~~~~~~

   .. method:: create_entity(org_id, project_id, name, entity_type, summary=None)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param str name: Human-readable name for the entity.
      :param str entity_type: Type label (e.g. ``"person"``,
          ``"document"``, ``"topic"``).
      :param summary: Optional text summary or description.
      :type summary: str | None
      :returns: A dict representing the created entity with at minimum
          ``id``, ``name``, ``type``, and ``created_at`` keys.
      :rtype: dict[str, Any]
      :raises ExternalServiceError: If the insert fails.

      Create a new entity node in the graph.  All backends implement this
      as an **upsert** (``ON CONFLICT`` / ``MERGE`` / ``IF/THEN/ELSE``) by
      entity name within the org scope so that duplicate extractions update
      rather than silently drop information.

   .. method:: get_entity(org_id, project_id, entity_id)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID entity_id: The UUID of the entity to fetch.
      :returns: The entity dict, or ``None`` if no entity with that ID
          exists within the given org and project.
      :rtype: dict[str, Any] | None

   .. method:: delete_entity(org_id, project_id, entity_id)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID entity_id: The UUID of the entity to delete.
      :returns: ``True`` if the entity was deleted, ``False`` if it did
          not exist.
      :rtype: bool

      Removes the entity and cascades to all incident edges.

   .. method:: update_entity(org_id, project_id, entity_id, *, name=None, entity_type=None, summary=None)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID entity_id: UUID of the entity to update.
      :param name: New name, or ``None`` to leave unchanged.
      :type name: str | None
      :param entity_type: New type label, or ``None`` to leave unchanged.
      :type entity_type: str | None
      :param summary: New summary text, or ``None`` to leave unchanged.
      :type summary: str | None
      :returns: The updated entity dict with at minimum ``id``, ``name``,
          ``entity_type``, ``summary``, and ``updated_at`` keys.
      :rtype: dict[str, Any]
      :raises NotFoundError: If no entity with the given ID exists.

      Only the provided fields are changed; ``None`` fields are left
      untouched.  Dynamically builds the ``SET`` clause from non-None
      parameters.

   Relationship CRUD
   ~~~~~~~~~~~~~~~~~

   .. method:: create_relationship(org_id, project_id, source_id, target_id, relationship_type, properties=None, confidence=None, valid_from=None, valid_to=None)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID source_id: UUID of the source entity.
      :param UUID target_id: UUID of the target entity.
      :param str relationship_type: Label for the edge (e.g.
          ``"mentions"``, ``"authored_by"``).
      :param properties: Optional key-value metadata attached to the edge.
      :type properties: dict[str, Any] | None
      :param confidence: Optional confidence score (default 1.0).
      :type confidence: float | None
      :param valid_from: Optional temporal validity start (ISO-8601).
      :type valid_from: datetime | None
      :param valid_to: Optional temporal validity end (ISO-8601).
      :type valid_to: datetime | None
      :returns: A dict with at minimum ``id``, ``source_id``,
          ``target_id``, ``type``, and ``created_at`` keys.
      :rtype: dict[str, Any]
      :raises ValueError: If the relationship type contains unsafe
          characters (FalkorDB, SurrealDB).
      :raises ExternalServiceError: On FK violations or unexpected DB
          errors.

      Creates a directed edge between two entity nodes.  All backends
      implement this as an **upsert** (``ON CONFLICT DO UPDATE`` /
      ``MERGE ON MATCH`` / ``IF/THEN/ELSE``) using the
      ``(source_id, target_id, relationship_type)`` tuple as the dedup
      key.  Confidence is upserted as ``MAX(existing, new)``, and temporal
      fields (``valid_from``, ``valid_to``) use ``LEAST``/``GREATEST``
      merging semantics.

      Edge type names are **sanitised** to allow only
      ``[a-zA-Z0-9_]`` characters to prevent Cypher/SurrealQL injection.

   .. method:: expire_relationship(org_id, project_id, relationship_id)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID relationship_id: UUID of the relationship to expire.
      :returns: ``True`` if the relationship was expired, ``False`` if it
          did not exist or was already expired.
      :rtype: bool

      Soft-delete a relationship by setting ``invalid_at`` to the current
      timestamp.  Expired relationships are excluded from all traversal
      and list queries.

   Traversal & Search
   ~~~~~~~~~~~~~~~~~~

   .. method:: traverse(org_id, project_id, start_node_id, max_depth=2, edge_types=None)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID start_node_id: UUID of the node to begin traversal from.
      :param int max_depth: Maximum number of edge hops (default 2,
          capped at 5 across all backends).
      :param edge_types: Optional filter — only follow edges with these
          labels.  ``None`` means all edge types are followed.  ``[]``
          (empty list) returns just the start node.
      :type edge_types: list[str] | None
      :returns: A list of node dicts reachable within the depth limit,
          including the start node at depth 0.  Each dict includes a
          ``depth`` key indicating the number of hops from the start node.
      :rtype: list[dict[str, Any]]

      Traversal strategy differs by backend:

      * **Postgres**: Recursive CTE (``WITH RECURSIVE bfs AS ...``) with a
        ``SET LOCAL statement_timeout = '5s'`` guard.  Both incoming and
        outgoing edges are followed.  An :meth:`iterative fallback
        <PostgresGraphBackend.traverse_iterative>` is available for graphs
        > 100K nodes.
      * **FalkorDB**: Iterative BFS in Python.  At each hop, uses
        ``algo.bfs()`` (GraphBLAS) for single-type traversal or Cypher
        variable-length paths for multi-type traversal.
      * **SurrealDB**: Iterative BFS in Python.  At each hop, uses native
        SurrealQL arrow syntax (``->``, ``->?``) for neighbour discovery.

   .. method:: search_entities(org_id, project_id, query, types=None, limit=50, offset=0)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param str query: Free-text search string.
      :param types: Optional filter — only return entities matching these
          type labels.
      :type types: list[str] | None
      :param int limit: Maximum number of results to return.
      :param int offset: Number of results to skip (for pagination).
      :returns: A list of matching entity dicts ordered by relevance
          (descending).  Each dict includes a ``score`` key.
      :rtype: list[dict[str, Any]]

      Search engine differs by backend:

      * **Postgres**: Combines trigram similarity (``pg_trgm``, weighted
        60%) with full-text search (``tsvector``/``plainto_tsquery``,
        weighted 40%).
      * **FalkorDB**: Uses ``CALL db.idx.fulltext.queryNodes()`` for BM25
        / RediSearch full-text search.
      * **SurrealDB**: Uses the ``@@`` operator with the
        ``openzync_entity`` analyzer and ``search::score(0)`` for BM25
        ranking.

   .. method:: retrieve_graph(org_id, project_id, query, *, match_limit=5, max_depth=2, max_results=50)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param str query: Free-text search string.
      :param int match_limit: Max entities to match before traversal.
      :param int max_depth: Max BFS depth from each matched entity.
      :param int max_results: Max total results to return.
      :returns: Entity dicts with ``id``, ``name``, ``type``,
          ``summary``, and ``distance`` keys.  Distance 0 = directly
          matched, 1+ = reached via traversal.  Sorted by distance
          ascending.
      :rtype: list[dict[str, Any]]

      Combines entity text search with BFS graph traversal into a single
      call.  Each backend uses its native search strength, then BFS from
      each matched entity, deduplicates by entity id, and sorts by
      distance.  Designed for the ``HybridRetriever`` which runs multiple
      backends in parallel and merges results.

   Entity Listing
   ~~~~~~~~~~~~~~

   .. method:: list_entities(org_id, project_id, *, entity_type=None, limit=50, cursor=None)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param entity_type: Optional filter by entity type (e.g.
          ``"Person"``).
      :type entity_type: str | None
      :param int limit: Maximum results per page (max 200).
      :param cursor: Opaque cursor for cursor-based pagination.
      :type cursor: str | None
      :returns: A dict with ``items`` (list of entity dicts),
          ``next_cursor`` (str or None), and ``has_more`` (bool).
      :rtype: dict[str, Any]

      Cursor format (Postgres): base64-encoded JSON
      ``{"c": "<created_at>", "i": "<id>"}`` for keyset pagination.

      Cursor format (FalkorDB, SurrealDB): base64-encoded JSON
      ``{"o": <offset>}`` for offset-based pagination.

   .. method:: list_entity_edges(org_id, project_id, entity_id, *, predicate=None, limit=50, cursor=None)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID entity_id: UUID of the entity node whose edges to list.
      :param predicate: Optional filter by edge label.
      :type predicate: str | None
      :param int limit: Maximum results per page (max 200).
      :param cursor: Opaque cursor for cursor-based pagination.
      :type cursor: str | None
      :returns: A dict with ``items`` (list of edge dicts),
          ``next_cursor`` (str or None), and ``has_more`` (bool).
      :rtype: dict[str, Any]

   .. method:: get_entity_with_edges(org_id, project_id, entity_id)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID entity_id: UUID of the entity to fetch.
      :returns: A dict with ``node`` (entity dict) and ``edges`` (list of
          edge dicts), or ``None`` if the entity does not exist.
      :rtype: dict[str, Any] | None

      Convenience method that calls :meth:`get_entity` and
      :meth:`list_entity_edges` (with no limit) and combines the results.

   Entity-Episode Linking
   ~~~~~~~~~~~~~~~~~~~~~~

   .. method:: link_entity_to_episode(org_id, project_id, episode_id, entity_id)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID episode_id: UUID of the episode the entity appears in.
      :param UUID entity_id: UUID of the entity appearing in the episode.
      :raises NotFoundError: If either the episode or entity does not
          exist.
      :raises ExternalServiceError: If the query fails.

      Record that an entity was extracted from (appears in) a specific
      episode.  **Must be idempotent** (``ON CONFLICT DO NOTHING`` /
      ``MERGE`` / ``IF/THEN/ELSE``).

      * **Postgres**: Inserts into ``graph_episode_entities`` with
        ``ON CONFLICT (episode_id, entity_id) DO NOTHING``.
      * **FalkorDB**: Creates a lightweight ``:Episode`` stub node via
        ``MERGE``, then ``MERGE (ep)-[:MENTIONS]->(en)``.
      * **SurrealDB**: Uses the ``has_entity`` edge table via
        ``RELATE`` with an ``IF/THEN/ELSE`` idempotency guard.

   .. method:: get_entities_for_session(org_id, project_id, session_id)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID session_id: UUID of the processing session.
      :returns: List of entity dicts with ``id``, ``name``,
          ``entity_type``, ``summary`` keys.
      :rtype: list[dict[str, Any]]

      Return all distinct graph entities linked to episodes in a session.
      Traverses session → episodes → episode_entity_links → entities.

   .. method:: get_co_occurring_entity_pairs(org_id, project_id, min_co_count=2)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param int min_co_count: Minimum number of co-occurring episodes
          required (default 2).
      :returns: List of dicts with ``entity_a_id``, ``entity_a_name``,
          ``entity_b_id``, ``entity_b_name``, ``co_count``, sorted by
          ``co_count`` descending.
      :rtype: list[dict[str, Any]]

      Find entity pairs that co-appear in episodes above a threshold.

      * **Postgres**: Single SQL self-join on
        ``graph_episode_entities`` with ``GROUP BY / HAVING``.
      * **FalkorDB**: Single Cypher match pattern
        ``(a)<-[:MENTIONS]-(ep)-[:MENTIONS]->(b)``.
      * **SurrealDB**: Two-step approach — fetch all distinct episode
        RecordIDs, then for each episode fetch its entities and build a
        co-occurrence frequency map in Python (O(N\\ :sub:`episodes`)
        queries).

   Bulk / Merge Operations
   ~~~~~~~~~~~~~~~~~~~~~~~

   .. method:: get_all_entities(org_id, project_id, *, include_merged=False)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param bool include_merged: If ``True``, include entities that have
          been soft-deleted via merge.  Defaults to ``False``.
      :returns: A complete list of entity dicts for the project.  Each
          dict includes ``id``, ``name``, ``entity_type``, ``summary``,
          ``is_merged``, and ``created_at``.
      :rtype: list[dict[str, Any]]
      :raises GraphBackendUnavailableError: If the backend is unreachable.

      .. warning::

          BATCH WORKER USE ONLY — no pagination, no limit.  Potentially
          millions of rows.  Do **not** expose via API.

   .. method:: get_all_relationships(org_id, project_id)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :returns: A complete list of relationship dicts.  Each dict includes
          ``id``, ``source_id``, ``target_id``, ``relationship_type``,
          ``confidence``, and ``created_at``.  Only non-expired
          (``invalid_at IS NULL``) relationships are returned.
      :rtype: list[dict[str, Any]]

      Same warning as :meth:`get_all_entities` — batch use only.

   .. method:: bulk_search_entities(org_id, project_id, query, *, fuzzy_threshold=0.3, limit=50)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param str query: Search string to match against entity names.
      :param float fuzzy_threshold: Minimum similarity score (0.0–1.0)
          for a result to be included.
      :param int limit: Maximum number of results to return.
      :returns: List of entity dicts that exceed the similarity threshold,
          sorted by descending score.  Each dict includes a ``score`` key
          (float, 0.0–1.0).
      :rtype: list[dict[str, Any]]

      Used by the ``merge_duplicate_entities`` worker.  Backend-specific
      behaviour:

      * **Postgres**: Uses native ``pg_trgm`` trigram similarity
        (``similarity()`` — character-level fuzzy matching).
      * **FalkorDB**: Uses RediSearch-backed BM25 full-text search on
        entity names.
      * **SurrealDB**: Uses BM25 full-text search via the ``@@``
        operator (word-level, not character-level fuzzy).  Raw BM25 scores
        are normalised via ``1 - 1/(1 + raw_score)`` to 0–1 range.

   .. method:: merge_entities(org_id, project_id, canonical_id, merged_ids)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID canonical_id: UUID of the entity that survives the merge.
      :param list[UUID] merged_ids: UUIDs of entities being absorbed into
          the canonical entity.
      :returns: A dict with keys ``rewired_count`` (int),
          ``deleted_count`` (int), ``merged_count`` (int).
      :rtype: dict[str, Any]
      :raises NotFoundError: If ``canonical_id`` or any ``merged_id`` does
          not exist.
      :raises NotImplementedError: If the backend cannot provide atomicity
          (SurrealDB uses a multi-step approach that is not fully atomic).

      Merge duplicate entities by rewiring all edges to the canonical
      entity and soft-deleting (``is_merged = true``) the merged entities.

      **Atomicity contract**:

      * **Postgres**: Fully atomic via ``begin_nested()`` savepoint with
        ``SET LOCAL statement_timeout = '10s'``.
      * **FalkorDB**: Per-step atomic (one ``graph.query()`` per edge
        type + final soft-delete).  Partial failure may leave visible
        state — operations are idempotent.
      * **SurrealDB**: Multi-step Python approach (discover → rewire →
        delete → soft-delete).  **Not fully atomic** — use a saga pattern
        for strict guarantees.

   .. method:: create_relationship_bulk(org_id, project_id, relationships)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param list[dict[str, Any]] relationships: List of relationship
          descriptor dicts.  Each must have ``source_id``, ``target_id``,
          ``relationship_type``.  Optional keys: ``confidence``,
          ``properties``, ``valid_from``, ``valid_to``.
      :returns: List of created relationship dicts (one per input, in the
          same order).
      :rtype: list[dict[str, Any]]
      :raises ValueError: If any input dict is missing required keys.

   Observations
   ~~~~~~~~~~~~

   .. method:: upsert_observation(org_id, project_id, subject_entity_id, observation_type, content, confidence, *, related_entity_id=None, supporting_fact_ids=None, supporting_relationship_ids=None, valid_from=None, valid_to=None, observation_metadata=None)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID subject_entity_id: The entity this observation is about.
      :param str observation_type: Semantic type label (e.g.
          ``"co_occurrence"``, ``"temporal_gap"``).
      :param str content: Human-readable description of the observation.
      :param float confidence: Confidence score 0.0–1.0.
      :param related_entity_id: Optional secondary entity involved in the
          observation.
      :type related_entity_id: UUID | None
      :param supporting_fact_ids: Optional list of fact UUIDs that support
          this observation.
      :type supporting_fact_ids: list[UUID] | None
      :param supporting_relationship_ids: Optional list of relationship
          UUIDs that support this observation.
      :type supporting_relationship_ids: list[UUID] | None
      :param valid_from: Optional temporal validity start.
      :type valid_from: datetime | None
      :param valid_to: Optional temporal validity end.
      :type valid_to: datetime | None
      :param observation_metadata: Optional arbitrary key-value metadata.
      :type observation_metadata: dict[str, Any] | None
      :returns: The created or updated observation dict with at minimum
          ``id``, ``subject_entity_id``, ``observation_type``,
          ``content``, ``confidence``, and ``created_at`` keys.
      :rtype: dict[str, Any]

      Create or update a graph-topology observation.  Observations are
      second-pass inferences (co-occurrence, temporal patterns) computed
      by the observation service after initial graph construction.

      Upsert uniqueness: ``(subject_entity_id, observation_type,
      COALESCE(related_entity_id, sentinel))`` where the sentinel for
      ``None`` is ``00000000-0000-0000-0000-000000000000``.

   .. method:: get_observations(org_id, project_id, *, subject_entity_id=None, observation_type=None, limit=50, cursor=None)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param subject_entity_id: Optional filter — only observations about
          this entity.
      :type subject_entity_id: UUID | None
      :param observation_type: Optional filter — only observations of
          this type.
      :type observation_type: str | None
      :param int limit: Maximum results per page (max 200).
      :param cursor: Opaque cursor for pagination.
      :type cursor: str | None
      :returns: A dict with ``items``, ``next_cursor``, and
          ``has_more`` — same pattern as :meth:`list_entities`.
      :rtype: dict[str, Any]

   .. method:: get_entity_appearance_timestamps(org_id, project_id, entity_id)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID entity_id: UUID of the entity to query.
      :returns: Sorted list of episode timestamps (oldest first) when the
          entity appeared.  Empty list if the entity has no linked
          episodes.
      :rtype: list[datetime]

      Used by temporal gap analysis in the observation service.

   .. method:: get_relationship_ids_between(org_id, project_id, entity_a_id, entity_b_id)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param UUID entity_a_id: UUID of the first entity.
      :param UUID entity_b_id: UUID of the second entity.
      :returns: List of relationship UUIDs connecting the two entities
          (both directions).  Empty list if no direct relationship exists.
      :rtype: list[UUID]

      Used by the observation service to provide supporting evidence for
      co-occurrence observations.

   Aggregate Queries
   ~~~~~~~~~~~~~~~~~

   .. method:: get_total_entity_linked_episode_count(org_id, project_id)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :returns: Total number of distinct episodes in the project that have
          at least one entity linked to them.
      :rtype: int
      :raises GraphBackendUnavailableError: If the backend is unreachable.

   .. method:: resolve_entity_names(org_id, project_id, entity_ids)

      :param UUID org_id: Organisational scope.
      :param UUID project_id: Project scope.
      :param list[UUID] entity_ids: List of entity UUIDs to resolve.
      :returns: Dict keyed by entity ID string, where each value is
          ``{"name": str, "entity_type": str}``.  Entity IDs not found in
          the graph are omitted from the result.
      :rtype: dict[str, dict]

   Observability
   ~~~~~~~~~~~~~

   .. method:: health_check()

      :returns: ``True`` if the backend is healthy and reachable,
          ``False`` otherwise.
      :rtype: bool

      * **Postgres**: Runs ``SELECT 1``.
      * **FalkorDB**: Runs ``RETURN 1`` on a ``_health_check`` graph.
      * **SurrealDB**: Runs ``SELECT 1``.


Error Handling
--------------

All backend methods raise exceptions from the :mod:`core.exceptions`
hierarchy rather than returning error tuples.  The exception handler
registered in ``main.py`` maps these to RFC 7807 Problem Detail responses.

.. class:: core.exceptions.GraphBackendUnavailableError

   :status_code: 503
   :code: ``"graph_backend_unavailable"``

   Raised when the graph backend is unreachable or the required client is
   not connected (``client is None`` / ``surreal is None``).  Never
   silently swallowed — propagates as HTTP 503 so load-balancers and
   orchestrators react appropriately.

.. class:: core.exceptions.ExternalServiceError

   :status_code: 502
   :code: ``"external_service_error"``

   Raised when a query against the graph backend fails unexpectedly (DB
   constraint violation, network timeout, malformed response).  Every
   backend wraps its raw exceptions with this type, preserving the
   original error via ``raise ... from exc``.

.. class:: core.exceptions.NotFoundError

   :status_code: 404
   :code: ``"not_found"``

   Raised by :meth:`update_entity`, :meth:`merge_entities`, and
   :meth:`link_entity_to_episode` when the target entity does not exist.

Every backend also applies structured logging via ``structlog`` at key
points — entity/relationship upsert, traversal, search, merge — with
``org_id``, ``project_id``, and operation-specific identifiers in the
``extra`` dict for traceability.

.. important:: **Zero fallback policy**

   Backend failures always propagate as exceptions.  There is no silent
   degradation, no cached response fallback, and no "best effort" return.
   If the graph backend is down, the caller gets an exception — never
   stale or partial data.


PostgreSQL Backend
------------------

Module: :mod:`packages.graph_backend.postgres`

.. class:: PostgresGraphBackend(db, max_traversal_depth=2)

   :param AsyncSession db: An async SQLAlchemy session.  Must be
       request-scoped — the caller (usually a FastAPI dependency) is
       responsible for session lifecycle.
   :param int max_traversal_depth: Maximum BFS depth (default 2, max 5).
       Hard-capped at :const:`MAX_TRAVERSAL_DEPTH` (5).

   PostgreSQL-native graph backend.  Requires no external graph database —
   entities and relationships are stored in dedicated PostgreSQL tables.

   Schema
   ~~~~~~

   Three core tables are used:

   ``graph_entities``
       Entity nodes.  Key columns:
       ``id`` (UUID PK), ``organization_id``, ``project_id``, ``name``,
       ``entity_type``, ``summary``, ``attributes`` (JSONB),
       ``is_merged`` (boolean), ``created_at``, ``updated_at``.

       Unique constraint on ``(organization_id, name)`` — entity names
       are unique within an org.  This enables the upsert behaviour where
       duplicate extractions update rather than silently dropping.

   ``graph_relationships``
       Directed edges.  Key columns:
       ``id`` (UUID PK), ``organization_id``, ``project_id``,
       ``source_id`` (UUID FK → entities), ``target_id`` (UUID FK →
       entities), ``relationship_type``, ``properties`` (JSONB),
       ``confidence``, ``fact``, ``valid_from``, ``valid_to``,
       ``invalid_at`` (soft-delete), ``created_at``, ``updated_at``.

       Partial unique index on
       ``(source_id, target_id, relationship_type) WHERE invalid_at IS NULL``
       — only one active edge per type between any two entities.

   ``graph_episode_entities``
       Many-to-many join table linking entities to episodes.  Key columns:
       ``episode_id`` (UUID FK → episodes), ``entity_id`` (UUID FK →
       entities), ``project_id``, ``created_at``.

       Unique constraint on ``(episode_id, entity_id)`` for idempotency.

   ``graph_observations``
       Second-pass observations.  Key columns:
       ``id`` (UUID PK), ``organization_id``, ``project_id``,
       ``subject_entity_id``, ``related_entity_id``, ``observation_type``,
       ``content``, ``confidence``, ``supporting_fact_ids`` (UUID[]),
       ``supporting_relationship_ids`` (UUID[]), ``valid_from``,
       ``valid_to``, ``observation_metadata`` (JSONB), ``created_at``,
       ``updated_at``.

       Functional unique index on
       ``(project_id, subject_entity_id, observation_type,
        COALESCE(related_entity_id, sentinel))``.

   BFS Traversal
   ~~~~~~~~~~~~~

   Uses a **recursive CTE** (``WITH RECURSIVE bfs AS ...``) that follows
   both incoming and outgoing active edges.  A ``SET LOCAL
   statement_timeout = '5s'`` guard prevents runaway queries:

   .. code-block:: sql

      WITH RECURSIVE bfs AS (
          SELECT ge.id, ge.name, ge.entity_type, ge.summary,
                 ge.attributes, ge.created_at, 0 AS depth
          FROM graph_entities ge
          WHERE ge.id = :start_id AND ge.organization_id = :org_id
          AND ge.project_id = :project_id
          UNION
          SELECT DISTINCT e.id, e.name, e.entity_type, e.summary,
                 e.attributes, e.created_at, bfs.depth + 1
          FROM bfs
          JOIN graph_relationships r
              ON (r.source_id = bfs.id OR r.target_id = bfs.id)
              AND r.invalid_at IS NULL
          JOIN graph_entities e
              ON (e.id = CASE WHEN r.source_id = bfs.id
                  THEN r.target_id ELSE r.source_id END)
          WHERE bfs.depth < :max_depth
      )
      SELECT DISTINCT ON (bfs.id) ... FROM bfs ORDER BY bfs.id, bfs.depth

   An alternative :meth:`traverse_iterative` method uses a Python
   ``deque`` with per-hop neighbour queries for graphs exceeding ~100K
   nodes where the recursive CTE may hit the statement timeout.

   Search
   ~~~~~~

   Entity search combines two PostgreSQL text-search features:

   * **Trigram similarity** (``pg_trgm``): ``similarity(ge.name, :query)``
     for fuzzy name matching (weighted 60%).
   * **Full-text search** (``tsvector`` / ``plainto_tsquery``):
     ``ts_rank(to_tsvector('english', ge.summary), plainto_tsquery(...))``
     for summary search (weighted 40%).

   The combined score is::

       COALESCE(similarity(ge.name, :query), 0) * 0.6
       + COALESCE(ts_rank(to_tsvector('english', ge.summary),
                          plainto_tsquery('english', :query)), 0) * 0.4

   Fuzzy dedup search (``bulk_search_entities``) uses
   ``similarity(LOWER(name), LOWER(:query))`` directly.

   Merge Atomicity
   ~~~~~~~~~~~~~~~

   Entity merging is fully atomic via a SQLAlchemy savepoint
   (``begin_nested()``).  The four steps (rewire sources, rewire targets,
   delete duplicates, soft-delete merged) all execute within the same
   subtransaction with a ``SET LOCAL statement_timeout = '10s'`` guard.

   Strengths and limitations
   ~~~~~~~~~~~~~~~~~~~~~~~~~

   * **Zero external dependencies** — runs on the same PostgreSQL
     instance used for application data.
   * **ACID-compliant** — all operations benefit from PostgreSQL
     transaction guarantees.
   * **Cursor pagination** — uses keyset pagination (``(created_at, id)``
     tuples) for consistent pagination under write load.
   * **Limited graph-specific features** — no native graph algorithms
     (shortest path, PageRank, community detection).  All graph
     operations are expressed as SQL/CTEs.
   * **CTE depth limits** — deep traversal (> 5 hops) may exceed
     statement timeout or planner memory limits.

   Usage example::

       from sqlalchemy.ext.asyncio import AsyncSession
       from packages.graph_backend.postgres import PostgresGraphBackend

       async def example(db: AsyncSession) -> None:
           backend = PostgresGraphBackend(db=db, max_traversal_depth=3)

           entity = await backend.create_entity(
               org_id=...,
               project_id=...,
               name="Acme Corp",
               entity_type="company",
               summary="A widget manufacturer",
           )

           await backend.create_relationship(
               org_id=...,
               project_id=...,
               source_id=entity["id"],
               target_id=other_entity["id"],
               relationship_type="supplier",
               confidence=0.95,
           )

           results = await backend.traverse(
               org_id=...,
               project_id=...,
               start_node_id=entity["id"],
               max_depth=2,
           )
           # results[0]["depth"] == 0 for start node


FalkorDB Backend
----------------

Module: :mod:`packages.graph_backend.falkordb`

.. class:: FalkorGraphBackend(client=None, max_traversal_depth=2)

   :param client: An optional connected ``FalkorDB`` async instance.
       When ``None``, all methods raise :class:`ExternalServiceError` with
       message ``"FalkorDB not connected"``.
   :type client: FalkorDB | None
   :param int max_traversal_depth: Maximum BFS depth (default 2, max 5).
       Hard-capped at :const:`MAX_TRAVERSAL_DEPTH` (5).

   FalkorDB-native graph backend using the RedisGraph module.  Each
   org+project pair gets its own isolated FalkorDB graph key
   (``openzync_{org_id}_{project_id}``), guaranteeing tenant isolation at
   the database level.

   Connection
   ~~~~~~~~~~

   Connection and client management happen in the app-layer connection
   pool based on system config:

   .. code-block:: python

       from falkordb.asyncio import FalkorDB
       from core.config import get_settings

       settings = get_settings()
       client = FalkorDB(
           host=settings.FALKORDB_URL,       # "redis://localhost:6379"
           max_connections=settings.FALKORDB_MAX_CONNECTIONS,  # 20
           socket_timeout=settings.FALKORDB_SOCKET_TIMEOUT,    # 30s
       )

   Indexes & Schema
   ~~~~~~~~~~~~~~~~

   FalkorDB uses label-range and full-text indexes rather than a predefined
   schema.  Indexes are idempotently created on the first entity upsert
   per tenant graph:

   .. code-block:: cypher

      CREATE RANGE INDEX FOR (n:Entity) ON (n.name);
      CREATE FULLTEXT INDEX FOR (n:Entity) ON (n.name, n.summary)
          OPTIONS {language: 'english'};
      CREATE RANGE INDEX FOR (n:Episode) ON (n.id);
      CREATE RANGE INDEX FOR (n:Session) ON (n.id);
      CREATE RANGE INDEX FOR (n:Observation)
          ON (n.subject_entity_id, n.observation_type);

   Entity CRUD
   ~~~~~~~~~~~

   Uses ``MERGE`` on the ``name`` property within the per-tenant graph
   (names are unique within a tenant).  ``ON MATCH`` upgrades the entity
   type if the existing type is ``"Custom"`` and updates the summary if a
   non-empty value is provided.

   Relationship CRUD
   ~~~~~~~~~~~~~~~~~

   Uses ``MERGE`` on the pattern ``(s)-[r:TYPE]->(t)``, sanitising the
   edge type name via a regex (only ``[a-zA-Z0-9_]`` characters are
   allowed).  A UUID is generated for each relationship and stored as
   ``r.id``.  ``ON MATCH`` updates properties, confidence (takes the
   max), and temporal validity fields.

   Traversal
   ~~~~~~~~~

   Iterative BFS in Python with three dispatch strategies at each hop:

   * **Single edge type** → ``CALL algo.bfs(n, 1, 'TYPE') YIELD nodes``
     for C-level GraphBLAS speed.
   * **Multiple edge types** → Cypher variable-length path:
     ``MATCH (n)-[r:type1|type2]-(neighbour) RETURN DISTINCT neighbour.id``.
   * **All types** → Wildcard pattern:
     ``MATCH (n)-[r]-(neighbour) RETURN DISTINCT neighbour.id``.

   Search
   ~~~~~~

   Uses ``CALL db.idx.fulltext.queryNodes('Entity', $query)`` for
   RediSearch-backed BM25 full-text search.  Offset/limit slicing is done
   in Python since FalkorDB's ``SKIP``/``LIMIT`` with parameters can be
   unreliable in some versions.

   Episode Linking
   ~~~~~~~~~~~~~~~

   FalkorDB has no built-in episode concept — episodes live in
   PostgreSQL.  Lightweight stub nodes (``:Episode``, ``:Session``) are
   created via ``MERGE`` when linking entities to episodes, providing
   Cypher-traversable paths:

   .. code-block:: cypher

      MATCH (s:Session {id: $session_id})
        -[:HAS_EPISODE]->(ep:Episode)
        -[:MENTIONS]->(en:Entity)
      RETURN DISTINCT en.id, en.name, en.entity_type, en.summary

   Merge
   ~~~~~

   Per-step atomic (one ``graph.query()`` per distinct edge type).
   Steps: collect distinct edge types, rewire incoming per type, rewire
   outgoing per type (``MERGE`` + ``DELETE`` in single Cypher), then
   soft-delete merged entities.  Operations are idempotent — partial
   failures can be retried.

   Observability
   ~~~~~~~~~~~~~

   Episode stubs carry ``created_at`` timestamps for appearance-time
   analysis.  A ``_health_check`` graph is used for health probes.

   Strengths and limitations
   ~~~~~~~~~~~~~~~~~~~~~~~~~

   * **Graph-native** — native Cypher query language, GraphBLAS BFS,
     RediSearch full-text.
   * **Tenant isolation at DB level** — per-tenant graph keys guarantee
     no cross-tenant leaks even without ``WHERE`` filters.
   * **In-memory performance** — FalkorDB keeps the graph in Redis memory
     for sub-millisecond traversal.
   * **Offset pagination** — not keyset-based; may drift under write load.
   * **No built-in co-occurrence queries** — uses Cypher's multi-hop
     pattern matching which is efficient for single queries.
   * **Episodes are stubs** — episode metadata lives in PostgreSQL; only
     IDs are mirrored in FalkorDB for traversal.

   Usage example::

       from falkordb.asyncio import FalkorDB
       from packages.graph_backend.falkordb import FalkorGraphBackend

       client = FalkorDB(host="localhost", port=6379)
       backend = FalkorGraphBackend(client=client)

       entity = await backend.create_entity(
           org_id=...,
           project_id=...,
           name="Acme",
           entity_type="company",
       )
       results = await backend.traverse(
           org_id=...,
           project_id=...,
           start_node_id=entity["id"],
           max_depth=2,
       )


SurrealDB Backend
-----------------

Module: :mod:`packages.graph_backend.surrealdb`

.. class:: SurrealGraphBackend(surreal=None, max_traversal_depth=2)

   :param surreal: An optional connected ``AsyncSurreal`` instance.
       When ``None``, all methods raise :class:`ExternalServiceError` with
       message ``"SurrealDB not connected"``.
   :type surreal: AsyncSurreal | None
   :param int max_traversal_depth: Maximum BFS depth (default 2, max 5).
       Hard-capped at :const:`MAX_TRAVERSAL_DEPTH` (5).

   SurrealDB-native graph backend using SurrealQL's native graph
   relations (``RELATE`` / arrow syntax) for O(1) traversal and ``@@``
   with BM25 for full-text search.

   Schema Bootstrapping
   ~~~~~~~~~~~~~~~~~~~~

   On the first use, the backend runs a set of ``DEFINE`` statements
   to create the schema:

   .. code-block:: surrealql

      -- 1. Analyzer for entity full-text search (BM25 via @@ operator)
      DEFINE ANALYZER openzync_entity
          TOKENIZERS blank, class
          FILTERS lowercase, ascii, snowball(english);

      -- 2. Entity table
      DEFINE TABLE entity SCHEMAFULL;
      DEFINE FIELD name ON entity TYPE string;
      DEFINE FIELD entity_type ON entity TYPE string;
      DEFINE FIELD summary ON entity TYPE string;
      -- ... (organization_id, project_id, attributes, is_merged, etc.)

      -- 3. Full-text BM25 indexes (required for @@ operator)
      DEFINE INDEX entity_name_fts ON entity FIELDS name
          FULLTEXT ANALYZER openzync_entity BM25;
      DEFINE INDEX entity_summary_fts ON entity FIELDS summary
          FULLTEXT ANALYZER openzync_entity BM25;

      -- 4. Unique index for entity upsert by (org_id, project_id, name)
      DEFINE INDEX entity_org_project_name ON entity
          FIELDS organization_id, project_id, name UNIQUE;

      -- 5. Episode & has_entity tables
      DEFINE TABLE episode SCHEMAFULL;
      DEFINE TABLE has_entity SCHEMAFULL;

      -- 6. Observation table
      DEFINE TABLE observation SCHEMAFULL;

   Schema creation is idempotent — duplicate ``DEFINE`` statements
   return ``InternalError`` with "already exists" in SurrealDB 3.x, which
   is caught and treated as a no-op.

   Connection
   ~~~~~~~~~~

   Per-org SurrealDB connections are managed by the
   ``core.surreal_pool`` integration.  Each org's connection URL, user,
   password, namespace, and database are stored in the per-org config:

   .. code-block:: python

       from surrealdb import AsyncSurreal

       surreal = AsyncSurreal(org_config.surrealdb_url)
       await surreal.connect()
       await surreal.signin({
           "username": org_config.surrealdb_user,
           "password": org_config.surrealdb_pass,
       })
       await surreal.use(
           org_config.surrealdb_namespace,
           org_config.surrealdb_database,
       )

   Entity CRUD
   ~~~~~~~~~~~

   Uses a ``LET + IF/THEN/ELSE`` pattern for atomic upserts:

   .. code-block:: surrealql

      LET $existing = (SELECT id, entity_type, summary FROM entity
          WHERE organization_id = $org_id
            AND project_id = $project_id
            AND name = $name LIMIT 1);
      RETURN IF array::len($existing) > 0 THEN
          (UPDATE entity SET
              entity_type = IF $existing[0].entity_type = 'Custom'
                  AND $type != 'Custom' THEN $type
                  ELSE $existing[0].entity_type END,
              summary = IF $summary != '' THEN $summary
                  ELSE $existing[0].summary END,
              updated_at = time::now()
          WHERE id = $existing[0].id RETURN AFTER)
      ELSE
          (CREATE entity SET organization_id = $org_id, ... RETURN AFTER)
      END;

   The query is executed via :meth:`_query_last` which calls
   ``query_raw()`` to retrieve the **last** statement's result (the SDK's
   ``query()`` only returns the first statement's output).

   Relationship CRUD
   ~~~~~~~~~~~~~~~~~

   Uses the same ``LET + IF/THEN/ELSE`` pattern for atomic upserts on
   per-type edge tables.  Edge tables are created **lazily** on the first
   ``RELATE`` for a given type:

   .. code-block:: surrealql

      RELATE $source_id -> mentions -> $target_id
      CONTENT {
          organization_id: $org_id,
          project_id: $project_id,
          properties: $properties,
          confidence: $confidence,
          valid_from: $valid_from,
          valid_to: $valid_to,
          created_at: time::now(),
          updated_at: time::now()
      }

   Edge type names are sanitised via the same regex as FalkorDB (only
   ``[a-zA-Z0-9_]``).

   Traversal
   ~~~~~~~~~

   Iterative BFS in Python using native SurrealQL arrow syntax at each
   hop:

   * **Specific types**: ``SELECT VALUE ->{type}->entity.id FROM $current_id``
   * **All types**: ``SELECT VALUE ->?->entity.id FROM $current_id``

   SurrealDB arrow syntax provides O(1) neighbour lookups since edges are
   stored as indexed RecordID references.

   Search
   ~~~~~~

   Uses the ``@@`` operator with the ``openzync_entity`` analyzer and
   ``search::score(0)`` for BM25 ranking:

   .. code-block:: surrealql

      SELECT *, search::score(0) AS score
      FROM entity
      WHERE organization_id = $org_id
        AND project_id = $project_id
        AND (name @@ $query OR summary @@ $query)
      ORDER BY score DESC
      LIMIT $limit START $offset;

   Fuzzy dedup search (``bulk_search_entities``) normalises raw BM25
   scores via ``1 - 1/(1 + raw)`` to 0–1 range for consistent threshold
   filtering.

   Episode Linking
   ~~~~~~~~~~~~~~~

   Uses the ``has_entity`` edge table:

   .. code-block:: surrealql

      SELECT DISTINCT id, name, entity_type, summary
      FROM entity
      WHERE organization_id = $org_id
        AND project_id = $project_id
        AND <-has_entity<-(episode WHERE session_id = $session_id)
      ORDER BY name ASC;

   .. note::

      The ``episode`` record does **not** need to exist in SurrealDB for
      the ``RELATE`` to succeed — SurrealDB stores RecordID references
      without validating the target record exists.  However,
      :meth:`get_entities_for_session` requires episode records to exist
      with ``session_id`` populated.

   Co-occurrence
   ~~~~~~~~~~~~~

   Two-step approach: fetch all distinct episode RecordIDs from the
   ``has_entity`` edge table, then for each episode fetch its entity
   list and build a co-occurrence frequency map in Python.  O(N\\ :sub:`episodes`)
   queries — acceptable for batch observation workers.

   Merge
   ~~~~~

   Multi-step Python approach (discover → rewire → delete → soft-delete).
   **Not fully atomic** — SurrealDB does not support multi-statement
   transactions with interleaved Python logic.  Consider wrapping in a
   saga pattern for strict guarantees.

   Expiry
   ~~~~~~

   Since SurrealDB edges are stored in per-type tables (``mentions``,
   ``authored_by``, etc.), :meth:`expire_relationship` first discovers
   the edge RecordID by scanning all project edges using ``meta::id()``,
   then issues an ``UPDATE ... SET invalid_at``.

   Strengths and limitations
   ~~~~~~~~~~~~~~~~~~~~~~~~~

   * **Native graph support** — ``RELATE`` / arrow syntax for O(1)
     traversal, no join tables needed.
   * **BM25 full-text search** — native ``@@`` operator with configurable
     analyzer.
   * **Per-tenant isolation** — each org connects to its own
     namespace/database.
   * **No recursive CTEs** — arrow syntax replaces recursive queries for
     small-depth BFS.
   * **Lazy edge tables** — edge tables are created on first use, no
     migration needed for new relationship types.
   * **Co-occurrence is O(N)** — two-step approach is not efficient for
     large projects (consider Postgres or FalkorDB for heavy co-occurrence
     workloads).
   * **Merge is not atomic** — multi-step approach with possible partial
     state on crash.

   Usage example::

       from surrealdb import AsyncSurreal
       from packages.graph_backend.surrealdb import SurrealGraphBackend

       surreal = AsyncSurreal("ws://localhost:8000/rpc")
       await surreal.connect()
       await surreal.signin({"username": "root", "password": "root"})
       await surreal.use("openzync", "openzync")

       backend = SurrealGraphBackend(surreal=surreal)
       entity = await backend.create_entity(
           org_id=...,
           project_id=...,
           name="Acme",
           entity_type="company",
       )


Adding a New Backend
--------------------

To integrate a new graph database engine, follow these steps:

1. **Create a class** implementing the :class:`GraphBackend` ABC in
   ``packages/graph_backend/<name>.py``.  All 24 abstract methods must be
   implemented.

2. **Register it** in the dispatcher.  Modify
   :func:`~core.graph_backend.init_dispatcher`:

   .. code-block:: python

      def init_dispatcher() -> GraphBackendDispatcher:
          from packages.graph_backend.falkordb import FalkorGraphBackend
          from packages.graph_backend.postgres import PostgresGraphBackend
          from packages.graph_backend.surrealdb import SurrealGraphBackend
          from packages.graph_backend.mybackend import MyBackend  # new

          dispatcher = GraphBackendDispatcher()
          dispatcher.register("surrealdb", SurrealGraphBackend)
          dispatcher.register("postgres", PostgresGraphBackend)
          dispatcher.register("falkordb", FalkorGraphBackend)
          dispatcher.register("mybackend", MyBackend)  # new
          return dispatcher

3. **Update ``resolve_and_create``** in :class:`GraphBackendDispatcher` to
   pass the correct backend-specific kwargs (``db``, ``surreal``,
   ``client``, etc.).

4. **Set the org config**::

       PATCH /admin/org/config
       {"graph_backend": "mybackend"}

   No caller code changes are needed.

   The dispatcher handles resolution and instantiation transparently.

Package contents
~~~~~~~~~~~~~~~~

.. data:: __all__

   The package's ``__init__.py`` currently exports only
   :class:`GraphBackend` and :class:`PostgresGraphBackend`:

   .. code-block:: python

      __all__ = [
          "GraphBackend",
          "PostgresGraphBackend",
      ]

   FalkorDB and SurrealDB backends are imported lazily from their
   respective modules (they have optional third-party dependencies).


Pagination Patterns
-------------------

Two pagination strategies are used across the three backends:

Keyset pagination (Postgres)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Postgres uses keyset (cursor-based) pagination for all list operations.
The cursor is a base64-encoded JSON payload:

.. code-block:: json

   {"c": "2026-07-15T10:30:00Z", "i": "550e8400-e29b-41d4-a716-446655440000"}

* ``c`` — ``created_at`` timestamp of the last item on the current page.
* ``i`` — ``id`` of the last item on the current page (tiebreaker).

The query uses a tuple comparison::

   (ge.created_at, ge.id) > (:cursor_ts, :cursor_id::uuid)

This provides consistent pagination under write load (new rows inserted at
the current cursor position do not cause page drift).

Offset pagination (FalkorDB, SurrealDB)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

FalkorDB and SurrealDB use offset-based pagination.  The cursor is a
base64-encoded JSON payload:

.. code-block:: json

   {"o": 50}

The ``LIMIT + 1`` technique detects whether more results exist (``has_more``
is ``True`` when the result set exceeds the requested ``limit``).  Offset
pagination may drift under write load (new rows inserted near the current
offset can cause items to shift between pages).


Constants
---------

.. data:: packages.graph_backend.postgres.MAX_TRAVERSAL_DEPTH

   :type: int
   :value: ``5``

   Hard cap on BFS depth to prevent unbounded recursive CTE queries.

.. data:: packages.graph_backend.postgres.BFS_CTE

   :type: str

   Recursive CTE SQL template for PostgreSQL BFS traversal.  Follows both
   incoming and outgoing edges with optional edge-type filtering.

.. data:: packages.graph_backend.postgres.SEARCH_ENTITIES_SQL

   :type: str

   SQL template for entity search combining trigram similarity and
   full-text search with relevance scoring.

.. data:: packages.graph_backend.postgres.LIST_ENTITIES_SQL

   :type: str

   SQL template for keyset-paginated entity listing.

.. data:: packages.graph_backend.postgres.LIST_RELATIONSHIPS_SQL

   :type: str

   SQL template for keyset-paginated edge listing.

.. data:: packages.graph_backend.falkordb.MAX_TRAVERSAL_DEPTH

   :type: int
   :value: ``5``

   Hard cap on BFS depth to prevent unbounded traversals.

.. data:: packages.graph_backend.falkordb._DEFINE_QUERIES

   :type: list[str]

   List of Cypher ``CREATE INDEX`` statements run idempotently per tenant
   graph on the first entity upsert.

.. data:: packages.graph_backend.surrealdb.MAX_TRAVERSAL_DEPTH

   :type: int
   :value: ``5``

   Hard cap on BFS depth to prevent unbounded queries.

.. data:: packages.graph_backend.surrealdb._DEFINE_SURQL

   :type: str

   SurrealQL script run once per backend instance to define tables,
   indexes, and the BM25 full-text analyzer.

.. data:: packages.graph_backend.surrealdb._SAFE_EDGE_TYPE_RE

   :type: re.Pattern
   :value: ``re.compile(r"^[a-zA-Z_][a-zA-Z0-9_]*$")``

   Regex that accepts only safe SurrealDB and FalkorDB edge-type /
   table-name characters.
