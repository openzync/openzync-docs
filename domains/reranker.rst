Re-ranking & Community Detection
================================

.. note::

   This document covers two distinct packages that sit **after** the initial
   retrieval stage in the OpenZync pipeline:

   * ``packages/reranker/`` — Pluggable cross-encoder re-ranking backends
     that re-score the candidate pool produced by hybrid (semantic + graph)
     retrieval.

   * ``packages/community/`` — Graph-based community detection that clusters
     related entities via Label Propagation, then summarises each cluster
     through an LLM.

   Both packages live in the ``openzync-core`` monolith but are independently
   importable.  Code examples assume the package is on the Python path as
   ``from packages.reranker import ...``.

   **Design principle**: the reranker is a *pure per-query transform* (no
   persistent state), while community detection is a *background materialised
   view* (stored as community entity nodes in the graph backend).

.. contents:: Sections
   :local:
   :depth: 2
   :class: this-will-duplicate-information-and-it-is-still-useful-here


Reranker Abstraction
--------------------

Module: ``packages.reranker``

The reranker package provides an abstract interface for cross-encoder
re-ranking together with two concrete backends and a config-driven factory.

Position in the retrieval pipeline::

    User Query
        │
        ▼
    ┌─────────────────────────────────────────────┐
    │  HybridRetriever (services/hybrid_retriever)│
    │                                             │
    │  1. Semantic search (episodes + facts)      │
    │  2. Graph BFS traversal                     │
    │  3. Reciprocal Rank Fusion (RRF) merge      │
    │  4. ──► Reranker.rerank(query, top-K) ◄──  │
    │  5. Return top-N to context assembler       │
    └─────────────────────────────────────────────┘
                              │
                              ▼
    ContextFormatter consumes ``reranker_score``
    alongside ``score`` and ``rrf_score`` for
    final ranking.

When a reranker is configured the ``HybridRetriever`` widens the candidate
pool it passes to RRF from ``limit`` to ``reranker_top_k`` (default 50),
then sends the top RRF-scored candidates to the reranker, which returns the
final ``top_n`` (default 10) results.


Optional Dependencies
~~~~~~~~~~~~~~~~~~~~~

.. code-block:: text

    # Local cross-encoder model (sentence-transformers)
    pip install openzync[reranker]

    # Cohere Rerank API
    pip install openzync[cohere]

Both extras are declared in ``pyproject.toml``:

.. code-block:: toml

    [project.optional-dependencies]
    reranker = ["sentence-transformers>=3.0.0"]
    cohere = ["cohere>=5.0.0"]

The factory silently returns ``None`` when a backend's dependencies are not
installed, allowing graceful fallback to RRF-only scoring.


Module-Level Constants
~~~~~~~~~~~~~~~~~~~~~~

.. py:data:: RRF_K
   :type: int
   :value: 60

   RRF (Reciprocal Rank Fusion) constant that controls how quickly rank
   contribution decays.  Used by the ``HybridRetriever`` when merging
   semantic and graph result sets **before** the reranker is invoked.
   Higher values equalise rank contributions; lower values favour top-ranked
   results.

.. py:data:: DEFAULT_RERANK_TOP_K
   :type: int
   :value: 50

   Default number of RRF-merged candidates to pass to the reranker.
   Overridable per-organisation via ``OrgConfig.reranker_top_k``.

.. py:data:: DEFAULT_RERANK_TOP_N
   :type: int
   :value: 10

   Default number of results to return after re-ranking.
   Overridable per-organisation via ``OrgConfig.reranker_top_n``.


Abstract Interface — ``packages.reranker.interface``
----------------------------------------------------

.. automodule:: packages.reranker.interface
   :members: CrossEncoderReranker
   :show-inheritance:
   :exclude-members: __init__

All reranker backends inherit from :class:`~packages.reranker.interface.CrossEncoderReranker`
and must implement two abstract members:

.. py:class:: CrossEncoderReranker

   .. py:property:: backend_name
      :type: str
      :abstractmethod:

      Human-readable backend identifier for Prometheus metric labels and
      structured log events.  Concrete implementations return values like
      ``"sentence_transformers"`` or ``"cohere"``.

   .. py:method:: rerank(query, candidates, top_n=10)

      :param str query: The search query string.
      :param list[dict] candidates: Candidate dicts.  Each must have at
          minimum an ``id`` and ``content`` key.  Items without ``content``
          receive a score of ``0.0``.
      :param int top_n: Maximum number of results to return (default 10).
      :returns: Same dicts with ``reranker_score: float`` added, sorted
                descending, truncated to ``top_n``.
      :rtype: list[dict]

      The contract is identical across all backends — a caller can swap
      ``SentenceTransformersReranker`` for ``CohereReranker`` without any
      other code change.

Input/output contract
~~~~~~~~~~~~~~~~~~~~~

**Input** — each candidate dict::

    {
        "id": "uuid-str",          # required
        "content": "text...",      # required (scored); absent → 0.0
        "rrf_score": 0.85,         # optional, preserved
        ...                        # any other keys preserved
    }

**Output** — same dicts with ``reranker_score`` added::

    {
        "id": "uuid-str",
        "content": "text...",
        "rrf_score": 0.85,
        "reranker_score": 0.9234,  # added by reranker, 0-1 range
        ...
    }

Results are sorted by ``reranker_score`` descending and truncated to
``top_n``.  The original ``rrf_score`` is preserved so downstream
consumers (e.g. :class:`~services.context_formatter.ContextFormatter`)
can combine scores.


Implementation — SentenceTransformersReranker
---------------------------------------------

Module: ``packages.reranker.sentence_transformers``

.. automodule:: packages.reranker.sentence_transformers
   :members: SentenceTransformersReranker
   :show-inheritance:
   :exclude-members: __init__

A local cross-encoder model powered by the ``sentence-transformers``
library.  Default model: ``cross-encoder/ms-marco-MiniLM-L-6-v2``.

Key behavioural details:

* **Lazy loading**: the model is not loaded in ``__init__``; it is loaded
  on the first :meth:`~packages.reranker.interface.CrossEncoderReranker.rerank`
  call via :meth:`SentenceTransformersReranker._ensure_model`.

* **Module-level cache with double-checked locking**: loaded models are
  stored in ``_MODEL_CACHE`` (a ``dict[str, CrossEncoder]``) with
  per-model-name ``asyncio.Lock`` instances (``_MODEL_LOCKS``).  Multiple
  reranker instances sharing the same model name share a single loaded
  model.  The lock prevents concurrent loading of the same model when
  multiple requests arrive simultaneously for an uncached model.

* **Non-blocking inference**: ``model.predict(pairs)`` runs inside
  ``loop.run_in_executor()`` to avoid blocking the asyncio event loop.

* **Graceful import failure**: if ``sentence-transformers`` is not
  installed, both ``_ensure_model`` and the factory
  (:class:`~packages.reranker.factory.RerankerFactory`) raise
  ``ImportError`` with a hint to ``pip install openzync[reranker]``.

Usage example::

    from packages.reranker import SentenceTransformersReranker

    reranker = SentenceTransformersReranker(
        model_name="cross-encoder/ms-marco-MiniLM-L-6-v2",
    )

    candidates = [
        {"id": "1", "content": "The quick brown fox jumps over the lazy dog."},
        {"id": "2", "content": "Python is a high-level programming language."},
        {"id": "3"},  # no content → score 0.0
    ]

    reranked = await reranker.rerank(
        query="programming languages",
        candidates=candidates,
        top_n=2,
    )
    # reranked[0]["id"] == "2", reranked[0]["reranker_score"] > 0
    # reranked[1]["id"] == "1", reranked[1]["reranker_score"] > 0 (lower)
    # "3" is excluded (score 0.0, pushed below top_n)



Implementation — CohereReranker
-------------------------------

Module: ``packages.reranker.cohere``

.. automodule:: packages.reranker.cohere
   :members: CohereReranker
   :show-inheritance:
   :exclude-members: __init__

A remote API reranker using the Cohere Rerank endpoint.
Default model: ``rerank-english-v3.0``.

Key behavioural details:

* **Lazy client**: the ``cohere.Client`` is created on first use via
  :meth:`CohereReranker._ensure_client`, not in ``__init__``.

* **Synchronous SDK off the event loop**: the Cohere Python SDK is
  synchronous; every API call is wrapped in ``loop.run_in_executor()``
  to avoid blocking the event loop.

* **Result mapping**: Cohere returns results by index; the impl maps
  ``response.results[i].relevance_score`` back to the original candidate
  by position.  Candidates without content are never sent to the API and
  receive a score of ``0.0``.

* **Client configuration**: timeout is 30 s, max retries is 2 (configured
  at ``cohere.Client`` construction).

* **Graceful import failure**: if ``cohere`` is not installed,
  ``_ensure_client`` raises ``ImportError`` with a hint to
  ``pip install openzync[cohere]``.

Usage example::

    from packages.reranker import CohereReranker

    reranker = CohereReranker(
        api_key="your-cohere-api-key",
        model_name="rerank-english-v3.0",
    )

    reranked = await reranker.rerank(
        query="machine learning frameworks",
        candidates=[
            {"id": "a1", "content": "PyTorch is an open-source ML framework."},
            {"id": "b2", "content": "Paris is the capital of France."},
        ],
        top_n=1,
    )
    # reranked = [{"id": "a1", "content": "...", "reranker_score": 0.98}]


Config-Driven Factory — ``packages.reranker.factory``
------------------------------------------------------

.. automodule:: packages.reranker.factory
   :members: RerankerFactory
   :show-inheritance:
   :exclude-members: __init__

:class:`~packages.reranker.factory.RerankerFactory.create` is the single
entry point for constructing a reranker from org-level configuration.
It is called by :class:`~services.context_service.ContextService` every
time a context is assembled::

    # services/context_service.py (simplified)
    reranker = RerankerFactory.create(org_config) if org_config else None
    retriever = HybridRetriever(
        ...,
        reranker=reranker,
    )

The factory inspects these fields from
:class:`~schemas.organization_config.OrgConfigBase`:

.. list-table:: Config fields consumed
   :header-rows: 1

   * - Field
     - Type
     - Purpose
   * - ``reranker_backend``
     - ``str | None``
     - Selector: ``"sentence_transformers"``, ``"cohere"``, or ``None``
       (disabled).
   * - ``reranker_model``
     - ``str | None``
     - Model override; defaults to each backend's ``DEFAULT_MODEL`` when
       unset.
   * - ``reranker_top_k``
     - ``int | None``
     - RRF candidate pool size passed to the reranker (default 50).
   * - ``reranker_top_n``
     - ``int | None``
     - Final result count after re-ranking (default 10).
   * - ``cohere_api_key``
     - ``str | None``
     - Required when ``reranker_backend == "cohere"``.

If the backend is ``None``, empty, unknown, or its dependencies are
missing, the factory returns ``None`` — the caller falls back to
RRF-only scoring.

.. rubric:: Prometheus metrics

When the reranker is active, the
:class:`~services.hybrid_retriever.HybridRetriever` records latency via
``reranker_latency_seconds`` with a ``backend`` label set to the
reranker's ``backend_name`` (``"sentence_transformers"`` or
``"cohere"``).  Failures propagate as a
:class:`~core.exceptions.SearchLegFailedError` with ``leg_name="reranker"``.

.. rubric:: Score precedence in context assembly

The :class:`~services.context_formatter.ContextFormatter` uses the
following precedence when rendering scored results (highest priority
first):

#. ``reranker_score`` — cross-encoder relevance score.
#. ``rrf_score`` — Reciprocal Rank Fusion score (used when no reranker is
   active).
#. ``score`` — raw semantic similarity score.

Only the selected score is included in the final prompt context.
Internal score fields are stripped from the rendered output.


Community Detection
===================

Module: ``packages.community``

The community detection package provides graph-based entity clustering
using the **Label Propagation** algorithm.  It is a pure topological
analysis pass: the input is an entity-relationship graph, the output is
a list of entity sets (communities).  A downstream ARQ worker consumes
these sets to materialise community entities with LLM-generated summaries
in the graph backend.

Architecture overview::

    ┌──────────────────────────────────────────────────┐
    │  ARQ Worker (workers/tasks/summarise_community)  │
    │                                                   │
    │  1. Resolve per-org GraphBackend                  │
    │  2. For each project with ≥ 5 entities:           │
    │     a. Fetch all entities + relationships         │
    │     b. build_entity_graph() → nx.Graph            │
    │     c. detect_communities_label_propagation()     │
    │     d. For each community (≥ 2 members):          │
    │        i.   Generate LLM summary                  │
    │        ii.  backend.create_entity(entity_type=    │
    │             "community")                          │
    │        iii. backend.create_relationship_bulk(     │
    │             member_of edges)                      │
    └──────────────────────────────────────────────────┘
                              │
                              ▼
    ┌──────────────────────────────────────────────────┐
    │  Query path                                       │
    │                                                   │
    │  GET /v1/projects/{id}/graph/communities          │
    │    → GraphService.get_communities()               │
    │    → backend.list_entities(entity_type="community")│
    │    → GraphCommunitiesListResponse                 │
    └──────────────────────────────────────────────────┘


Algorithms — ``packages.community.algorithms``
----------------------------------------------

.. automodule:: packages.community.algorithms
   :members:
   :show-inheritance:
   :exclude-members: __init__

The module exposes two pure functions.  There is no class, no state, and
no dependency on any graph backend — all data is passed explicitly.


``build_entity_graph``
~~~~~~~~~~~~~~~~~~~~~~

.. py:function:: build_entity_graph(entities, relationships)

   Construct a :class:`networkx.Graph` from entity and relationship dicts.

   :param list[dict] entities: Each dict must have at minimum ``id``,
       ``name``, and ``type`` keys.  ``id`` is used as the graph node
       identifier.
   :param list[dict] relationships: Each dict must have ``source_id``,
       ``target_id``, and ``relationship_type`` keys.  ``relationship_type``
       is stored as a ``type`` edge attribute.
   :returns: A NetworkX undirected graph.
   :rtype: :class:`networkx.Graph`

   **Safety**: edges whose ``source_id`` or ``target_id`` does not
   correspond to a known entity node are silently skipped.  This guards
   against stale or orphaned relationship records.

   **Complexity**: :math:`O(V + E)` — one pass over entities (node
   insertion) and one over relationships (edge insertion with existence
   check).


``detect_communities_label_propagation``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. py:function:: detect_communities_label_propagation(graph)

   Run Label Propagation community detection on an entity graph.

   :param networkx.Graph graph: A graph constructed by
       :func:`build_entity_graph`.
   :returns: List of node sets, where each set is a community of at least
       two members.
   :rtype: list[set[str]]

   **Algorithm**: wraps
   :func:`networkx.algorithms.community.label_propagation_communities`,
   which implements **asynchronous Label Propagation**:

   #. Every node is initialised with a unique label (its own ID).
   #. Nodes are visited in random order; each node adopts the most frequent
      label among its neighbours (ties broken arbitrarily).
   #. Repeat until convergence (no label changes) or a maximum iteration
      limit is reached.

   **Post-filter**: communities with fewer than 2 members (isolated
   singleton nodes) are removed — a single node with no relationships
   cannot form a meaningful community.

   **Complexity**: near-linear :math:`O(V + E)` per iteration.  The
   algorithm typically converges in 5–10 iterations on sparse
   entity-relationship graphs.

   **Determinism**: results are not strictly deterministic due to the
   random node visitation order in NetworkX's implementation.  For most
   use cases the cluster assignments are stable at the community level,
   but individual node assignments near community boundaries may vary
   between runs.

Usage example::

    from packages.community.algorithms import (
        build_entity_graph,
        detect_communities_label_propagation,
    )

    entities = [
        {"id": "e1", "name": "Alice", "type": "Person"},
        {"id": "e2", "name": "Bob", "type": "Person"},
        {"id": "e3", "name": "Acme Corp", "type": "Organization"},
        {"id": "e4", "name": "Charlie", "type": "Person"},
        {"id": "e5", "name": "Globex Inc", "type": "Organization"},
        {"id": "e6", "name": "Diana", "type": "Person"},
    ]

    relationships = [
        {"source_id": "e1", "target_id": "e3", "relationship_type": "works_at"},
        {"source_id": "e2", "target_id": "e3", "relationship_type": "works_at"},
        {"source_id": "e4", "target_id": "e5", "relationship_type": "works_at"},
        {"source_id": "e6", "target_id": "e5", "relationship_type": "works_at"},
    ]

    graph = build_entity_graph(entities, relationships)
    communities = detect_communities_label_propagation(graph)

    for community in communities:
        print(f"Community: {community}")
        # Community: {'e1', 'e2', 'e3'}   (Alice, Bob, Acme Corp)
        # Community: {'e4', 'e5', 'e6'}   (Charlie, Globex Inc, Diana)


Dependency
~~~~~~~~~~

The package requires ``networkx >= 3.2.0`` (declared as a direct
dependency in ``pyproject.toml``).  No optional extras are needed.


ARQ Worker — ``workers.tasks.summarise_community``
---------------------------------------------------

.. automodule:: workers.tasks.summarise_community
   :members: summarise_community
   :show-inheritance:
   :exclude-members: __init__

The :func:`~workers.tasks.summarise_community.summarise_community` task
is the production consumer of the community detection algorithms.  It is
an ARQ background job that:

#. Discovers eligible organisations (all organisations with a reachable
   graph backend).
#. For each organisation, discovers non-archived projects.
#. For each project with at least ``COMMUNITY_MIN_ENTITY_COUNT`` (5) entities:

   a. Fetches all entities via :meth:`GraphBackend.get_all_entities`.
   b. Fetches all relationships via :meth:`GraphBackend.get_all_relationships`.
   c. Runs :func:`~packages.community.algorithms.build_entity_graph` and
      :func:`~packages.community.algorithms.detect_communities_label_propagation`.
   d. For each detected community with ≥ 2 members, calls
      :func:`_create_community` which:

      * Builds a structured prompt from community entities and
        relationships.
      * Generates a 2–3 sentence summary via the configured LLM.
      * Persists the community as a ``graph_entities`` row with
        ``entity_type='community'``.
      * Creates ``member_of`` relationships from each entity to the
        community node.

   e. Falls back to a descriptive name
      (``"Community of N entities: A, B, C..."``) if the LLM call fails.

The worker returns a dict with ``status``, ``orgs_processed``,
``orgs_failed``, and ``communities_created`` keys.  If all orgs fail
the task raises ``RuntimeError`` so ARQ can retry.


Trigger Mechanisms
------------------

Community detection runs in one of two modes, controlled by the
:attr:`OZ_AUTO_RUN_COMMUNITY_DETECTION` setting.

.. list-table:: Trigger modes
   :header-rows: 1

   * - Mode
     - Config value
     - Behaviour
     - Deduplication
   * - Nightly cron (default)
     - ``OZ_AUTO_RUN_COMMUNITY_DETECTION=false``
     - ARQ cron job at ``02:00 UTC`` daily.
       Job ID: ``nightly_community_detection``.
     - ``unique=True`` (ARQ built-in, prevents overlapping runs).
   * - Event-driven
     - ``OZ_AUTO_RUN_COMMUNITY_DETECTION=true``
     - Enqueued after each
       :func:`~workers.tasks.link_entities_to_episode.link_entities_to_episode`
       completion (graph sync step).  Runs on the ``low_queue_full``
       worker queue.
     - Per-organisation Redis key
       ``community:recently_enqueued:{org_id}`` with 1-hour TTL.
       Prevents re-enqueueing the same org within 60 minutes.

The event-driven mode is designed for organisations that ingest entities
frequently and want near-real-time community updates.  The nightly cron
mode is appropriate for batch-oriented ingestion patterns.

In both modes, the worker respects the per-project entity count threshold
(``COMMUNITY_MIN_ENTITY_COUNT = 5``) — projects with fewer entities are
skipped with an info log.


Result Storage & Querying
-------------------------

Communities are stored as **first-class entity nodes** in the graph
backend, using the same schema as regular entities.

**Storage** (performed by :func:`_create_community`):

.. code-block:: python

    # Community entity
    backend.create_entity(
        org_id=org_id,
        project_id=project_id,
        name="Community: Alice, Bob, Acme Corp...",
        entity_type="community",
        summary="<LLM-generated 2-3 sentence summary>",
    )

    # Member edges
    backend.create_relationship_bulk(
        org_id=org_id,
        project_id=project_id,
        relationships=[
            {"source_id": entity_uuid,
             "target_id": community_uuid,
             "relationship_type": "member_of"},
            # ... one per member
        ],
    )

**Querying** — via :class:`~services.graph_service.GraphService`::

    # services/graph_service.py
    communities = await service.get_communities(
        org_id=org_id,
        project_id=project_id,
    )

``get_communities`` delegates to
:meth:`~packages.graph_backend.interface.GraphBackend.list_entities`
with a filter for ``entity_type='community'`` (limit 200).  Each result
dict is augmented with a ``member_count`` key extracted from the stored
entity attributes.

**API endpoint**:

.. code-block:: http

    GET /v1/projects/{project_id}/graph/communities
    Authorization: Bearer <token>

Returns :class:`~schemas.graph.GraphCommunitiesListResponse` — a list of
:class:`~schemas.graph.GraphCommunity` objects:

.. code-block:: json

    {
      "data": [
        {
          "id": "uuid-of-community-node",
          "name": "Community: Alice, Bob, Acme Corp...",
          "summary": "This group represents the engineering team...",
          "member_count": 3,
          "created_at": "2026-07-15T02:00:00Z"
        }
      ]
    }

The endpoint requires project membership (``Depends(require_project_membership)``).


Pydantic Schemas — ``schemas.graph``
-------------------------------------

.. automodule:: schemas.graph
   :members: GraphCommunity, GraphCommunitiesListResponse
   :show-inheritance:
   :exclude-members: __init__


Constants & Thresholds
-----------------------

.. list-table::
   :header-rows: 1

   * - Constant
     - Value
     - Location
     - Purpose
   * - ``COMMUNITY_MIN_ENTITY_COUNT``
     - 5
     - ``workers/tasks/summarise_community.py``
     - Minimum entities in a project for detection to run.
   * - Singleton filter
     - ``len(c) >= 2``
     - ``packages/community/algorithms.py``
     - Communities must have at least 2 members; isolated nodes are
       excluded.
   * - Community naming
     - First 3 names + ``"..."``
     - ``workers/tasks/summarise_community.py``
     - Auto-generated name: ``"Community: Alice, Bob, Charlie..."``.
   * - Query limit
     - 200
     - ``services/graph_service.py``
     - Maximum communities returned by ``get_communities``.
   * - Event-driven cooldown
     - 3600 s (1 hour)
     - ``workers/tasks/link_entities_to_episode.py``
     - Redis TTL on ``community:recently_enqueued:{org_id}`` key.


Frontend Hull Visualization
---------------------------

The OpenZync frontend (``openzync-frontend``, not part of
``openzync-core``) provides a force-directed graph view of entities and
communities.  Community clusters are rendered as **convex hulls** around
their member nodes using D3's ``d3.polygonHull``:

.. code-block:: typescript

    // frontend/src/components/force-graph.tsx
    const hull = d3.polygonHull(points);
    // Expanded by 12 px for visual breathing room

The frontend reads community membership from the ``member_of`` edges in
the graph data returned by the API.  Hulls are rendered as semi-transparent
filled polygons (12 % opacity at rest, 22 % on hover) beneath nodes and
edges.  This is a **purely visual client-side feature** — the backend does
not compute or store hull geometries.

When a project has no community detection results, the graph view renders
entities without hull grouping.  Community detection must be enabled and
have completed at least one pass (nightly or event-driven) for hulls to
appear.


.. rubric:: TODO: needs author clarification

#. The ``PROMPT_NAME = "summarise_community_v1"`` constant is defined in
   ``workers/tasks/summarise_community.py`` but the actual prompt is
   constructed inline in ``_build_community_prompt()``, not loaded as a
   Jinja2 template.  This appears to be a legacy reference — confirm
   whether the Jinja2 loading path should be restored.

#. The ``list_communities`` router endpoint returns
   ``GraphCommunitiesListResponse`` but the underlying
   :meth:`GraphService.get_communities` returns bare dicts.  The schema
   conversion happens in the router via ``GraphCommunity(**c)`` — no
   ``model_validate`` is used.  This may be a minor inconsistency with
   the project convention of using ``from_attributes=True``.

#. ``member_count`` is extracted from entity attributes via
   ``attrs.get("member_count", 0)`` in ``get_communities``, but
   ``_create_community`` does not appear to write ``member_count`` into
   the entity attributes — only ``name``, ``entity_type``, and
   ``summary`` are set on ``create_entity``.  This means
   ``member_count`` will always fall back to ``0`` in the current code.
   Confirm whether ``member_count`` should be explicitly stored as an
   attribute during community creation.
