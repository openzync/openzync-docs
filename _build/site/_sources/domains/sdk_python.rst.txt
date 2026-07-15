Python SDK (openzync)
=====================

.. note::

   This document covers the **OpenZync Python SDK** at
   ``/home/rohan-linkai/code/personal/openzync/openzync-sdk-python/`` — the
   official Python client library for the OpenZync API.  The SDK is an
   installable ``pip`` package (``openzync``) that wraps all OpenZync REST
   endpoints behind typed, async-first clients.

   The SDK is the primary way to interact with OpenZync from Python
   applications, LangChain agents, and MCP servers.  Every backend domain
   (memory, facts, graph, users, sessions, projects) has a corresponding
   client class with full type hints and Pydantic response models.

   **This document covers**: installation, authentication, the sync and async
   client APIs, every domain client with all methods, the Pydantic model
   reference, the exception hierarchy, pagination patterns, error handling,
   and the optional LangChain integration.

   **What this document does not cover**: The OpenZync REST API contract
   (see :doc:`api_layer`), the MCP server (see :doc:`mcp_server`), or the
   backend domain internals (see :doc:`memory_context`, :doc:`auth`,
   :doc:`graph_backends`).

.. contents:: Sections
   :local:
   :depth: 3
   :class: this-will-duplicate-information-and-it-is-still-useful-here

.. _sdk-installation:

Installation
------------

Stable release
~~~~~~~~~~~~~~

.. code-block:: bash

   pip install openzync

The SDK requires Python ``>=3.11`` and pulls in two runtime dependencies:

* ``httpx >= 0.27`` — async HTTP transport with connection pooling and
  retry support.
* ``pydantic >= 2.7`` — request/response model validation.

LangChain integration (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   pip install openzync[langchain]

Installs the extra dependency ``langchain-core >= 0.3,<0.4`` and
enables the :ref:`sdk-langchain-integration` sub-package.

Development extras
~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   pip install openzync[dev]

Installs ``pytest``, ``pytest-asyncio``, and ``respx`` for running the test
suite.

Version identification
~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   import openzync
   print(openzync.__version__)  # e.g. "0.3.0"

The version is dynamically resolved from the installed package metadata via
:func:`importlib.metadata.version`.  If the package is not installed, it falls
back to ``"0.0.0"``.

.. _sdk-authentication:

Authentication
--------------

Getting an API key
~~~~~~~~~~~~~~~~~~

1. Deploy the OpenZync backend (see :doc:`deployment`) or connect to a hosted
   instance.
2. Create an API key scoped to a project.  This key is sent as a
   ``Bearer`` token on every request.
3. The key is used to resolve the ``project_id`` automatically — you never
   need to provide it explicitly for project-scoped operations.

Initialising the client
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   from openzync import OpenZync  # sync wrapper
   from openzync import AsyncOpenZync  # native async

   # Synchronous client (uses asyncio.run() internally)
   client = OpenZync(
       api_key="oz_live_xxxxxxxxxxxx",
       base_url="http://localhost:8000",   # default
       timeout=30.0,                        # default
   )

   # Async client (preferred for async applications)
   async with AsyncOpenZync(
       api_key="oz_live_xxxxxxxxxxxx",
       base_url="http://localhost:8000",
       timeout=30.0,
   ) as client:
       ...

.. _sdk-quickstart:

Quickstart
----------

Ingest conversation messages and retrieve context in a few lines:

.. code-block:: python

   from openzync import AsyncOpenZync
   import asyncio

   async def main() -> None:
       async with AsyncOpenZync(api_key="oz_live_...") as client:
           # 1. Ingest messages
           resp = await client.memory.ingest([
               {"role": "user", "content": "My name is Alice and I work at Acme Corp."},
               {"role": "assistant", "content": "Nice to meet you, Alice!"},
           ])
           print(f"Ingested {resp.episode_count} episode(s)  job={resp.job_id}")

           # 2. Retrieve context for an LLM
           ctx = await client.memory.get_context(
               query="What does Alice do?",
               limit=5,
           )
           print(ctx.context)  # formatted context block ready for prompt injection

           # 3. Create a user
           user = await client.users.create(
               external_id="alice@example.com",
               name="Alice",
           )
           print(f"Created user {user.id}")

           # 4. Create a session
           session = await client.sessions.create(
               external_id="conv-001",
               metadata={"channel": "web"},
           )
           print(f"Created session {session.id}")

   asyncio.run(main())

Client Architecture
-------------------

The SDK has two client entry points:

.. list-table::
   :header-rows: 1

   * - Class
     - When to use
     - Notes
   * - :class:`AsyncOpenZync`
     - Async applications (FastAPI, asyncio, Jupyter with ``await``)
     - Native async/await.  Use as a context manager.  **Preferred**
       client.
   * - :class:`OpenZync`
     - Simple scripts, REPL, synchronous code
     - Wraps ``AsyncOpenZync`` via ``asyncio.run()``.  **Not safe** inside a
       running event loop (Jupyter notebooks, async frameworks).

Both expose the same set of domain client attributes:

.. code-block:: python

   client.memory      # AsyncMemoryClient
   client.facts       # AsyncFactsClient
   client.graph       # AsyncGraphClient
   client.users       # AsyncUsersClient
   client.sessions    # AsyncSessionsClient
   client.projects    # AsyncProjectsClient

Each domain client is a lightweight object that receives the shared
:class:`AsyncHTTPTransport` instance from the parent client.  The transport
handles:

* Bearer token authentication
* Exponential backoff retry for ``429``, ``502``, ``503``, ``504`` responses
  (up to 3 attempts)
* Structured error mapping via :mod:`openzync._errors`
* Automatic ``project_id`` resolution from the API key
* Connection pooling with configurable timeout

Resource: Memory
----------------

.. module:: openzync.memory

The memory client ingests conversation messages into a project's memory store
and retrieves context for LLM injection.

.. class:: AsyncMemoryClient(http)

   Async client for memory operations.

   .. method:: ingest(messages, session_id=None, idempotency_key=None)

      Ingest conversation messages into a project's memory.

      :param messages: List of message objects.  Each can be a
          :class:`openzync.models.memory.Message` instance or a plain ``dict``
          with ``role`` and ``content`` keys.  Max 1000 messages.
      :type messages: list[Message | dict]
      :param session_id: Optional session external ID.  When provided,
          messages are associated with the given session for later retrieval.
      :type session_id: str | None
      :param idempotency_key: Optional idempotency key sent as the
          ``Idempotency-Key`` header.  Replays of the same key within the
          deduplication window are silently ignored.
      :type idempotency_key: str | None
      :returns: :class:`IngestMemoryResponse` with ``job_id``, ``episode_count``,
          ``status``, and ``message`` fields.
      :rtype: IngestMemoryResponse
      :raises AuthenticationError: Invalid or missing API key.
      :raises ValidationError: Request payload failed validation.
      :raises PayloadTooLargeError: Message batch exceeds size limit.

      .. code-block:: python

         # Ingest with dict messages
         resp = await client.memory.ingest([
             {"role": "system", "content": "You are a helpful assistant."},
             {"role": "user", "content": "What is the capital of France?"},
             {"role": "assistant", "content": "The capital of France is Paris."},
         ])

         # Ingest with Message objects
         from openzync.models import Message

         resp = await client.memory.ingest([
             Message(role="user", content="Hello"),
             Message(role="assistant", content="Hi there!"),
         ])

         # Ingest with session association and idempotency key
         resp = await client.memory.ingest(
             messages=[{"role": "user", "content": "Remember this."}],
             session_id="conv-001",
             idempotency_key="req-abc-123",
         )

   .. method:: get_context(query, limit=20)

      Assemble a formatted context block for LLM injection.  The server
      performs hybrid retrieval (semantic + keyword + graph traversal) across
      the project's memory and returns a single context string.

      :param query: Natural-language query describing the context needed.
      :type query: str
      :param limit: Maximum results per source type (episodes, facts,
          entities).
      :type limit: int
      :returns: :class:`ContextResponse` with a ``context`` string suitable
          for use as a system-prompt prefix, and ``metadata`` dict with
          assembly info.
      :rtype: ContextResponse

      .. code-block:: python

         ctx = await client.memory.get_context(
             query="What does Alice know about project Orion?",
             limit=10,
         )
         print(ctx.context)
         # "Alice is the project lead for Orion...
         #  The Orion project has 3 active milestones...
         #  Alice reported a delay in milestone 2..."

   .. method:: delete()

      Delete **all** memory for the project (soft-delete).  This
      operation is irreversible — use with care.

      :returns: ``None``
      :rtype: None

      .. code-block:: python

         await client.memory.delete()

Resource: Facts
---------------

.. module:: openzync.facts

The facts client ingests structured fact triples (subject–predicate–object)
into a project's knowledge graph.

.. class:: AsyncFactsClient(http)

   Async client for business fact operations.

   .. method:: add(facts, session_id=None)

      Ingest a batch of fact triples into the knowledge graph.

      :param facts: List of fact triples.  Each can be a
          :class:`openzync.models.facts.FactTriple` instance or a ``dict`` with
          ``subject``, ``predicate``, ``object`` keys (``content`` and
          ``confidence`` are optional).  Max 500 facts.
      :type facts: list[FactTriple | dict]
      :param session_id: Optional session external ID to associate the facts
          with a conversation session.
      :type session_id: str | None
      :returns: :class:`FactBatchResponse` with ``job_id``,
          ``accepted_count``, ``status``, and ``message``.
      :rtype: FactBatchResponse

      .. code-block:: python

         # Ingest with dicts
         resp = await client.facts.add([
             {"subject": "Alice", "predicate": "works_for", "object": "Acme Corp"},
             {"subject": "Acme Corp", "predicate": "headquartered_in", "object": "San Francisco"},
             {
                 "subject": "Alice",
                 "predicate": "has_role",
                 "object": "Project Lead",
                 "content": "Alice is the project lead for Project Orion",
                 "confidence": 0.95,
             },
         ])
         print(f"Accepted {resp.accepted_count} facts  job={resp.job_id}")

         # Ingest with FactTriple objects
         from openzync.models import FactTriple

         resp = await client.facts.add([
             FactTriple(subject="Bob", predicate="reports_to", object="Alice"),
         ])

      .. note::

         Facts are processed asynchronously.  The ``job_id`` can be used to
         track enrichment progress.  ``accepted_count`` reflects how many
         facts passed basic validation.

Resource: Graph
---------------

.. module:: openzync.graph

The graph client provides read access to the knowledge graph — entity nodes,
relationship edges, community summaries, and hybrid search.

.. class:: AsyncGraphClient(http)

   Async client for knowledge graph operations.

   .. method:: nodes(entity_type=None, limit=50)

      List entity nodes with optional type filter.  Returns a paginated async
      iterator that auto-fetches subsequent pages as items are consumed.

      :param entity_type: Optional entity type label to filter by (e.g.
          ``"person"``, ``"organization"``, ``"project"``).
      :type entity_type: str | None
      :param limit: Maximum items fetched per page.
      :type limit: int
      :returns: An :class:`AsyncPaginatedIterator` yielding
          :class:`openzync.models.graph.GraphNode` objects.
      :rtype: AsyncPaginatedIterator

      .. code-block:: python

         # List all nodes
         async for node in await client.graph.nodes():
             print(f"{node.name} ({node.type})")

         # Filter by entity type
         async for node in await client.graph.nodes(entity_type="person"):
             print(node.name)

   .. method:: node_detail(node_id)

      Get a single entity node with all its incident edges (relationships
      to other nodes).

      :param node_id: The UUID of the entity node.
      :type node_id: str
      :returns: :class:`GraphNodeDetail` containing the node information and
          a list of incident :class:`GraphEdge` objects.
      :rtype: GraphNodeDetail
      :raises EntityNotFoundError: If the node does not exist.
      :raises GraphTimeoutError: If the graph query times out.

      .. code-block:: python

         detail = await client.graph.node_detail(node_id="abc-123")
         print(f"Node: {detail.node.name} ({detail.node.type})")
         print(f"Summary: {detail.node.summary}")
         for edge in detail.edges:
             print(f"  {edge.source_id} --[{edge.type}]--> {edge.target_id}")

   .. method:: delete_node(node_id)

      Delete an entity node and all its incident edges from the knowledge
      graph.

      :param node_id: The UUID of the node to delete.
      :type node_id: str
      :returns: ``None``
      :rtype: None

      .. code-block:: python

         await client.graph.delete_node(node_id="abc-123")

   .. method:: edges(subject_id, predicate=None, limit=50)

      List relationship edges for a specific entity node.

      :param subject_id: The UUID of the source entity whose edges to list.
      :type subject_id: str
      :param predicate: Optional relationship type filter (e.g.
          ``"works_for"``, ``"reports_to"``).
      :type predicate: str | None
      :param limit: Maximum items fetched per page.
      :type limit: int
      :returns: An :class:`AsyncPaginatedIterator` yielding edge dicts.
      :rtype: AsyncPaginatedIterator

      .. code-block:: python

         async for edge in await client.graph.edges(
             subject_id="abc-123",
             predicate="works_for",
         ):
             print(f"{edge['source_id']} -> {edge['target_id']}")

   .. method:: communities()

      List community summary nodes.  Communities are clusters of related
      entities discovered by the graph analysis pipeline.

      :returns: List of :class:`GraphCommunity` objects with ``id``,
          ``name``, ``summary``, and ``member_count``.
      :rtype: list[GraphCommunity]

      .. code-block:: python

         communities = await client.graph.communities()
         for c in communities:
             print(f"{c.name}: {c.member_count} members — {c.summary[:80]}...")

   .. method:: search(query, types="episodes,facts", limit=20)

      Hybrid search across the project's memory.  Combines semantic
      (embedding), keyword (BM25), and graph traversal signals to surface
      relevant results.

      :param query: The search query string.
      :type query: str
      :param types: Comma-separated result type filter.  Supported values:
          ``"episodes"``, ``"facts"``, ``"entities"``.
      :type types: str
      :param limit: Maximum results per type.
      :type limit: int
      :returns: List of result dicts.  Each result contains ``content``,
          ``score``, ``type``, ``node_id``, ``node_name``, and source-specific
          fields.
      :rtype: list[dict]

      .. code-block:: python

         results = await client.graph.search(
             query="What does Alice know about Acme Corp?",
             types="episodes,facts,entities",
             limit=10,
         )
         for r in results:
             print(f"[{r['type']}] (score={r['score']:.3f}) {r.get('content', '')}")

Resource: Users
---------------

.. module:: openzync.users

The users client provides CRUD operations for user entities.  Users are
caller-defined entities with an ``external_id`` that maps to your identity
system.

.. class:: AsyncUsersClient(http)

   Async client for user operations.

   .. method:: create(external_id, name=None, email=None, metadata=None)

      Create a new user.

      :param external_id: Caller-defined user identifier (e.g. email, UUID
          from your auth system).  Must be unique per organization.
      :type external_id: str
      :param name: Optional display name.
      :type name: str | None
      :param email: Optional email address.
      :type email: str | None
      :param metadata: Optional arbitrary metadata dict.
      :type metadata: dict | None
      :returns: :class:`UserResponse` with internal ``id``, ``external_id``,
          and usage counters (``message_count``, ``fact_count``,
          ``session_count``).
      :rtype: UserResponse
      :raises ConflictError: If a user with the same ``external_id`` already
          exists.

      .. code-block:: python

         user = await client.users.create(
             external_id="alice@acme.com",
             name="Alice",
             email="alice@acme.com",
             metadata={"department": "engineering", "timezone": "US/Pacific"},
         )
         print(f"Created user {user.id}")

   .. method:: get(user_id)

      Get user details by internal UUID.

      :param user_id: The internal UUID of the user.
      :type user_id: str
      :returns: :class:`UserResponse` with all user fields and usage counters.
      :rtype: UserResponse
      :raises NotFoundError: If no user with the given ID exists.

      .. code-block:: python

         user = await client.users.get(user_id="abc-123")
         print(f"{user.name}: {user.message_count} messages, {user.session_count} sessions")

   .. method:: update(user_id, name=None, email=None, metadata=None)

      Update user fields.  Only provided fields are changed.

      :param user_id: The internal UUID of the user.
      :type user_id: str
      :param name: Optional new display name.
      :type name: str | None
      :param email: Optional new email address.
      :type email: str | None
      :param metadata: Optional new metadata dict.  Replaces existing metadata
          entirely when provided.
      :type metadata: dict | None
      :returns: :class:`UserResponse` with updated fields.
      :rtype: UserResponse
      :raises NotFoundError: If no user with the given ID exists.

      .. code-block:: python

         user = await client.users.update(
             user_id="abc-123",
             name="Alice B.",
             metadata={"department": "product"},
         )

   .. method:: delete(user_id)

      Soft-delete a user.  The user is marked ``is_deleted=True`` but data
      is retained.

      :param user_id: The internal UUID of the user.
      :type user_id: str
      :returns: ``None``
      :rtype: None

      .. code-block:: python

         await client.users.delete(user_id="abc-123")

   .. method:: list(limit=50, cursor=None)

      List users with cursor-based pagination.

      :param limit: Maximum results per page.
      :type limit: int
      :param cursor: Opaque cursor from a previous ``list()`` response.
      :type cursor: str | None
      :returns: Dict with ``data`` (list of :class:`UserResponse`),
          ``next_cursor``, and ``has_more`` keys.
      :rtype: dict

      .. code-block:: python

         page = await client.users.list(limit=20)
         for user in page["data"]:
             print(f"{user.name} ({user.email})")
         if page["has_more"]:
             next_page = await client.users.list(cursor=page["next_cursor"])

   .. method:: list_iter(limit=50)

      Iterate over **all** users with automatic pagination.  Fetches
      subsequent pages transparently as the iterator is consumed.

      :param limit: Maximum items per page.
      :type limit: int
      :returns: An :class:`AsyncPaginatedIterator` yielding user dicts.
      :rtype: AsyncPaginatedIterator

      .. code-block:: python

         async for user in client.users.list_iter(limit=100):
             print(f"{user['name']} - {user['email']}")

Resource: Sessions
------------------

.. module:: openzync.sessions

The sessions client manages conversation sessions within a project.  Sessions
group messages and facts into logical conversations.

.. class:: AsyncSessionsClient(http)

   Async client for session operations.

   .. method:: create(external_id, metadata=None)

      Create a new session within a project.

      :param external_id: Caller-defined session identifier (e.g. your
          internal conversation ID).  Must be unique per project.
      :type external_id: str
      :param metadata: Optional metadata dict (e.g. ``{"channel": "web"}``).
      :type metadata: dict | None
      :returns: :class:`SessionResponse` with internal ``id``,
          ``external_id``, ``project_id``, ``created_by``, and counters.
      :rtype: SessionResponse
      :raises ConflictError: If a session with the same ``external_id``
          already exists in this project.

      .. code-block:: python

         session = await client.sessions.create(
             external_id="conv-001",
             metadata={"channel": "api", "user_agent": "Mozilla/5.0"},
         )
         print(f"Session {session.id} created")

   .. method:: get(session_id)

      Get session details by internal UUID.

      :param session_id: The internal UUID of the session.
      :type session_id: str
      :returns: :class:`SessionResponse` with session fields and counters.
      :rtype: SessionResponse
      :raises NotFoundError: If no session with the given ID exists.

      .. code-block:: python

         session = await client.sessions.get(session_id="abc-123")
         print(f"{session.external_id}: {session.message_count} messages")

   .. method:: delete(session_id)

      Close and soft-delete a session.  The session is marked
      ``is_active=False``.

      :param session_id: The internal UUID of the session.
      :type session_id: str
      :returns: ``None``
      :rtype: None

      .. code-block:: python

         await client.sessions.delete(session_id="abc-123")

   .. method:: list(limit=50, cursor=None)

      List sessions for a project with cursor-based pagination.

      :param limit: Maximum results per page.
      :type limit: int
      :param cursor: Opaque cursor from a previous response.
      :type cursor: str | None
      :returns: Dict with ``data`` (list of :class:`SessionResponse`),
          ``next_cursor``, and ``has_more`` keys.
      :rtype: dict

      .. code-block:: python

         page = await client.sessions.list(limit=10)
         for session in page["data"]:
             print(f"{session.external_id}: active={session.is_active}")

   .. method:: messages(session_id, limit=50, cursor=None)

      Get messages for a session.

      :param session_id: The internal UUID of the session.
      :type session_id: str
      :param limit: Maximum results per page.
      :type limit: int
      :param cursor: Opaque cursor from a previous response.
      :type cursor: str | None
      :returns: :class:`SessionMessagesResponse` with ``data`` (list of
          :class:`SessionMessagesResponse.MessageItem`), ``next_cursor``,
          and ``has_more``.
      :rtype: SessionMessagesResponse

      .. code-block:: python

         msgs = await client.sessions.messages(session_id="abc-123", limit=100)
         for msg in msgs.data:
             print(f"[{msg.role}] {msg.content[:80]}...")

Resource: Projects
------------------

.. module:: openzync.projects

The projects client manages projects and their members.  Most SDK operations
are scoped to a project via the API key — you rarely need to provide a
``project_id`` explicitly.

.. class:: AsyncProjectsClient(http)

   Async client for project operations.

   .. method:: create(name, description=None, metadata=None)

      Create a new project.  The authenticated user is automatically added
      as an ``owner``.

      :param name: Display name for the project (1–255 characters).
      :type name: str
      :param description: Optional description (max 2000 characters).
      :type description: str | None
      :param metadata: Optional arbitrary metadata dict.
      :type metadata: dict | None
      :returns: :class:`ProjectResponse` with all project fields.
      :rtype: ProjectResponse

      .. code-block:: python

         project = await client.projects.create(
             name="My Project",
             description="Knowledge base for the AI assistant",
             metadata={"environment": "staging"},
         )
         print(f"Created project {project.id}")

   .. method:: get()

      Get project details for the project associated with the current API
      key.

      :returns: :class:`ProjectResponse` with project fields, including
          ``member_count`` and timestamps.
      :rtype: ProjectResponse

      .. code-block:: python

         project = await client.projects.get()
         print(f"{project.name}: {project.member_count} members")

   .. method:: update(name=None, description=None, metadata=None, is_archived=None)

      Update project fields.  Only provided fields are changed.

      :param name: Optional new display name.
      :type name: str | None
      :param description: Optional new description.
      :type description: str | None
      :param metadata: Optional new metadata dict.
      :type metadata: dict | None
      :param is_archived: Optional archive flag.
      :type is_archived: bool | None
      :returns: :class:`ProjectResponse` with updated fields.
      :rtype: ProjectResponse

      .. code-block:: python

         project = await client.projects.update(
             description="Updated description",
             is_archived=False,
         )

   .. method:: list(limit=50, cursor=None)

      List projects with offset-based pagination.

      :param limit: Maximum results per page.
      :type limit: int
      :param cursor: Opaque cursor (offset value) from a previous response.
      :type cursor: str | None
      :returns: Dict with ``data``, ``next_cursor``, and ``has_more`` keys.
      :rtype: dict

      .. code-block:: python

         page = await client.projects.list(limit=10)
         for project in page["data"]:
             print(f"{project['name']} - archived={project.get('is_archived', False)}")

   .. method:: list_iter(limit=50)

      Iterate over all projects with automatic pagination.

      :param limit: Maximum items per page.
      :type limit: int
      :returns: An :class:`AsyncPaginatedIterator` yielding project dicts.
      :rtype: AsyncPaginatedIterator

      .. code-block:: python

         async for project in client.projects.list_iter():
             print(project["name"])

   .. method:: archive()

      Archive (soft-delete) the project associated with the current API key.
      This operation is irreversible.

      :returns: ``None``
      :rtype: None

      .. code-block:: python

         await client.projects.archive()

   .. method:: add_member(user_id, role="member")

      Add a user as a member of the project.

      :param user_id: The UUID of the user to add.
      :type user_id: str
      :param role: Project role.  Must be ``"owner"`` or ``"member"``.
      :type role: str
      :returns: :class:`ProjectMemberResponse` with membership details.
      :rtype: ProjectMemberResponse

      .. code-block:: python

         member = await client.projects.add_member(
             user_id="user-abc-123",
             role="member",
         )
         print(f"Added member {member.user_id} with role {member.role}")

   .. method:: remove_member(user_id)

      Remove a member from the project.

      :param user_id: The UUID of the user to remove.
      :type user_id: str
      :returns: ``None``
      :rtype: None

      .. code-block:: python

         await client.projects.remove_member(user_id="user-abc-123")

   .. method:: list_members(limit=50, cursor=None)

      List members of the project.

      :param limit: Maximum results per page.
      :type limit: int
      :param cursor: Opaque cursor from a previous response.
      :type cursor: str | None
      :returns: Dict with ``data``, ``next_cursor``, and ``has_more`` keys.
      :rtype: dict

      .. code-block:: python

         members = await client.projects.list_members(limit=20)
         for m in members.get("data", []):
             print(f"User {m.get('user_id')} - role {m.get('role')}")

Pagination
----------

.. module:: openzync._pagination

List endpoints that return multiple resources use **cursor-based pagination**.
The SDK provides two iterator classes that handle page fetching transparently.

.. class:: AsyncPaginatedIterator(fetch_page, limit=50)

   Async iterator that auto-fetches subsequent pages as items are consumed.

   :param fetch_page: An async callable that accepts an optional cursor
       string and returns a paginated response dict with ``items`` (or
       ``data``), ``next_cursor``, and ``has_more`` keys.
   :param limit: Maximum items fetched per page.

   .. method:: __aiter__()

      Returns the iterator itself.

   .. method:: __anext__()

      Returns the next item or raises ``StopAsyncIteration``.  Fetches a
      new page transparently when the current page is exhausted.

   .. code-block:: python

      # Manual pagination with cursor
      page = await client.users.list(limit=50)
      for user in page["data"]:
          print(user["name"])
      while page["has_more"]:
          page = await client.users.list(cursor=page["next_cursor"])
          for user in page["data"]:
              print(user["name"])

      # Auto-pagination with AsyncPaginatedIterator
      async for user in client.users.list_iter(limit=50):
          print(user["name"])

.. class:: SyncPaginatedIterator(fetch_page, limit=50)

   Synchronous wrapper around :class:`AsyncPaginatedIterator`.  Each
   ``__next__`` call runs the underlying async fetch via
   :func:`asyncio.run`.

   .. warning::

      Not safe inside an existing event loop.  Use the async iterator in
      async contexts.

   .. code-block:: python

      for user in client.users.list_iter(limit=50):
          print(user["name"])

Model Reference
---------------

.. module:: openzync.models

All Pydantic models are re-exported from :mod:`openzync.models` for
convenience:

.. code-block:: python

   from openzync.models import (
       Message,
       IngestMemoryRequest,
       IngestMemoryResponse,
       ContextResponse,
       FactTriple,
       FactBatchRequest,
       FactBatchResponse,
       GraphNode,
       GraphEdge,
       GraphNodeDetail,
       GraphCommunity,
       PaginatedGraphNodes,
       PaginatedGraphEdges,
       UserCreateRequest,
       UserUpdateRequest,
       UserResponse,
       UserListResponse,
       SessionCreateRequest,
       SessionResponse,
       SessionListResponse,
       SessionMessagesResponse,
   )

Memory models
~~~~~~~~~~~~~

.. class:: Message

   A single conversation turn.

   .. attribute:: role

      :type: str
      :required:

      Message sender role: ``"user"``, ``"assistant"``, ``"system"``, or
      ``"tool"``.

   .. attribute:: content

      :type: str
      :required:
      :max_length: 65536

      Message body text.

   .. attribute:: created_at

      :type: datetime | None

      ISO-8601 timestamp.  Server-assigned if omitted.

   .. attribute:: metadata

      :type: dict[str, Any]

      Caller-defined metadata.  Defaults to ``{}``.

.. class:: IngestMemoryRequest

   Request body for ``POST /v1/projects/{project_id}/memory``.

   .. attribute:: session_id

      :type: str | None

   .. attribute:: messages

      :type: list[Message]
      :min_length: 1
      :max_length: 1000

.. class:: IngestMemoryResponse

   Response returned after successful ingestion.

   .. attribute:: job_id

      :type: str | None

      UUID of the async enrichment job.

   .. attribute:: episode_count

      :type: int

      Number of episodes ingested.

   .. attribute:: status

      :type: str

      Always ``"accepted"``.

   .. attribute:: message

      :type: str

      Human-readable status message.

.. class:: ContextResponse

   Response from the context assembly endpoint.

   .. attribute:: context

      :type: str

      Formatted context block for LLM injection.

   .. attribute:: metadata

      :type: dict[str, Any]

      Assembly metadata (source counts, scores, etc.).

Facts models
~~~~~~~~~~~~

.. class:: FactTriple

   A single fact triple for batch ingestion.

   .. attribute:: subject

      :type: str
      :min_length: 1
      :max_length: 500

      Subject entity name.

   .. attribute:: predicate

      :type: str
      :min_length: 1
      :max_length: 200

      Relationship verb (e.g. ``"works_for"``, ``"located_in"``).

   .. attribute:: object

      :type: str
      :min_length: 1
      :max_length: 500

      Object entity name.

   .. attribute:: content

      :type: str | None

      Human-readable fact statement.

   .. attribute:: confidence

      :type: float
      :default: 1.0
      :range: [0.0, 1.0]

      Confidence score.

.. class:: FactBatchResponse

   Response after successful fact batch ingestion.

   .. attribute:: job_id

      :type: str

      UUID of the async enrichment job.

   .. attribute:: accepted_count

      :type: int
      :ge: 0

      Number of facts accepted for processing.

   .. attribute:: status

      :type: str

      Always ``"accepted"``.

   .. attribute:: message

      :type: str

      Human-readable status message.

Graph models
~~~~~~~~~~~~

.. class:: GraphNode

   A single entity node in the knowledge graph.

   .. attribute:: id

      :type: str

      UUID of the entity node.

   .. attribute:: name

      :type: str

      Human-readable display name.

   .. attribute:: type

      :type: str

      Entity type label (e.g. ``"person"``, ``"organization"``).

   .. attribute:: summary

      :type: str

      Text summary or description.

   .. attribute:: created_at

      :type: str | None

      ISO-8601 creation timestamp.

   .. attribute:: metadata

      :type: dict[str, Any]

.. class:: GraphEdge

   A single relationship edge in the knowledge graph.

   .. attribute:: id

      :type: str

      UUID of the edge.

   .. attribute:: source_id

      :type: str

      UUID of the source entity.

   .. attribute:: target_id

      :type: str

      UUID of the target entity.

   .. attribute:: type

      :type: str

      Relationship label (e.g. ``"works_for"``).

   .. attribute:: properties

      :type: dict[str, Any]

      Edge-specific properties.

   .. attribute:: created_at

      :type: str | None

.. class:: GraphNodeDetail

   A single entity node with all its incident edges.

   .. attribute:: node

      :type: GraphNode

      The entity node.

   .. attribute:: edges

      :type: list[GraphEdge]

      All incident edges (both incoming and outgoing).

.. class:: GraphCommunity

   A community summary node.

   .. attribute:: id

      :type: str

      UUID of the community node.

   .. attribute:: name

      :type: str

      Community cluster name.

   .. attribute:: summary

      :type: str

      Community summary description.

   .. attribute:: member_count

      :type: int
      :ge: 0

   .. attribute:: created_at

      :type: str | None

.. class:: PaginatedGraphNodes

   Cursor-paginated response for entity node listing.

   .. attribute:: items

      :type: list[GraphNode]

   .. attribute:: next_cursor

      :type: str | None

   .. attribute:: has_more

      :type: bool

.. class:: PaginatedGraphEdges

   Cursor-paginated response for edge listing.

   .. attribute:: items

      :type: list[GraphEdge]

   .. attribute:: next_cursor

      :type: str | None

   .. attribute:: has_more

      :type: bool

User models
~~~~~~~~~~~

.. class:: UserCreateRequest

   Request body for ``POST /v1/users``.

   .. attribute:: external_id

      :type: str
      :min_length: 1
      :max_length: 255

      Caller-defined user identifier.

   .. attribute:: name

      :type: str | None
      :max_length: 255

   .. attribute:: email

      :type: str | None
      :max_length: 255

   .. attribute:: metadata

      :type: dict[str, Any]

.. class:: UserUpdateRequest

   Request body for ``PATCH /v1/users/{user_id}``.

   .. attribute:: name

      :type: str | None

   .. attribute:: email

      :type: str | None

   .. attribute:: metadata

      :type: dict[str, Any] | None

.. class:: UserResponse

   Response from user CRUD endpoints.

   .. attribute:: id

      :type: str

      Internal UUID.

   .. attribute:: external_id

      :type: str

      Caller-defined identifier.

   .. attribute:: name

      :type: str | None

   .. attribute:: email

      :type: str | None

   .. attribute:: metadata

      :type: dict[str, Any]

   .. attribute:: organization_id

      :type: str

      Owning organization UUID.

   .. attribute:: created_at

      :type: str

      ISO-8601 creation timestamp.

   .. attribute:: updated_at

      :type: str

      ISO-8601 update timestamp.

   .. attribute:: is_deleted

      :type: bool

   .. attribute:: message_count

      :type: int

   .. attribute:: fact_count

      :type: int

   .. attribute:: session_count

      :type: int

.. class:: UserListResponse

   Response from ``GET /v1/users``.

   .. attribute:: data

      :type: list[UserResponse]

   .. attribute:: next_cursor

      :type: str | None

   .. attribute:: has_more

      :type: bool

Session models
~~~~~~~~~~~~~~

.. class:: SessionCreateRequest

   Request body for ``POST /v1/projects/{project_id}/sessions``.

   .. attribute:: external_id

      :type: str
      :min_length: 1
      :max_length: 255

      Caller-defined session identifier.

   .. attribute:: metadata

      :type: dict[str, Any]

.. class:: SessionResponse

   Response from session CRUD endpoints.

   .. attribute:: id

      :type: str

      Internal UUID.

   .. attribute:: project_id

      :type: str

      Owning project UUID.

   .. attribute:: created_by

      :type: str

      User UUID who created the session.

   .. attribute:: external_id

      :type: str

      Caller-defined identifier.

   .. attribute:: metadata

      :type: dict[str, Any]

   .. attribute:: is_active

      :type: bool

   .. attribute:: message_count

      :type: int

   .. attribute:: fact_count

      :type: int

   .. attribute:: created_at

      :type: str

      ISO-8601 creation timestamp.

.. class:: SessionListResponse

   Response from ``GET /v1/projects/{project_id}/sessions``.

   .. attribute:: data

      :type: list[SessionResponse]

   .. attribute:: next_cursor

      :type: str | None

   .. attribute:: has_more

      :type: bool

.. class:: SessionMessagesResponse

   Response from ``GET /v1/projects/{project_id}/sessions/{session_id}/messages``.

   .. attribute:: data

      :type: list[SessionMessagesResponse.MessageItem]

      List of messages in the session.

   .. attribute:: next_cursor

      :type: str | None

   .. attribute:: has_more

      :type: bool

   .. class:: MessageItem

      A single message within a session.

      .. attribute:: id

         :type: str

         Episode UUID.

      .. attribute:: role

         :type: str

         Message role (``"user"``, ``"assistant"``, ``"system"``).

      .. attribute:: content

         :type: str

         Message content.

      .. attribute:: metadata

         :type: dict[str, Any]

      .. attribute:: token_count

         :type: int

      .. attribute:: sequence_number

         :type: int

      .. attribute:: created_at

         :type: str

         ISO-8601 timestamp.

Project models
~~~~~~~~~~~~~~

.. class:: CreateProjectRequest

   Request body for ``POST /v1/projects``.

   .. attribute:: name

      :type: str
      :min_length: 1
      :max_length: 255

      Project display name.

   .. attribute:: description

      :type: str | None
      :max_length: 2000

   .. attribute:: metadata

      :type: dict[str, Any]

.. class:: UpdateProjectRequest

   Request body for ``PUT /v1/projects/{project_id}``.

   .. attribute:: name

      :type: str | None

   .. attribute:: description

      :type: str | None

   .. attribute:: metadata

      :type: dict[str, Any] | None

   .. attribute:: is_archived

      :type: bool | None

.. class:: ProjectResponse

   Response from project CRUD endpoints.

   .. attribute:: id

      :type: str

      Internal UUID.

   .. attribute:: name

      :type: str

   .. attribute:: description

      :type: str | None

   .. attribute:: metadata

      :type: dict[str, Any]

   .. attribute:: is_archived

      :type: bool

   .. attribute:: member_count

      :type: int

   .. attribute:: created_by

      :type: str | None

      User UUID who created the project.

   .. attribute:: created_at

      :type: str

      ISO-8601 creation timestamp.

   .. attribute:: updated_at

      :type: str

      ISO-8601 update timestamp.

.. class:: AddMemberRequest

   Request body for ``POST /v1/projects/{project_id}/members``.

   .. attribute:: user_id

      :type: str

      UUID of the user to add.

   .. attribute:: role

      :type: str
      :pattern: ``^(owner|member)$``

      Project role.  Defaults to ``"member"``.

.. class:: ProjectMemberResponse

   Response for project member endpoints.

   .. attribute:: id

      :type: str

      Internal UUID.

   .. attribute:: project_id

      :type: str

   .. attribute:: user_id

      :type: str

   .. attribute:: role

      :type: str

      ``"owner"`` or ``"member"``.

   .. attribute:: created_at

      :type: str

      ISO-8601 timestamp.

Exception Hierarchy
-------------------

.. module:: openzync._errors

Every HTTP error response is mapped to a typed exception.  The SDK raises
these automatically — you never need to inspect raw response bodies.

.. class:: OpenZyncError

   Base exception for all SDK errors.

   .. attribute:: status_code

      :type: int
      :default: 500

   .. attribute:: code

      :type: str
      :default: ``"internal_error"``

   .. attribute:: message

      :type: str
      :default: ``"An unexpected error occurred."``

   .. attribute:: detail

      :type: dict | None

``OpenZyncError`` subclasses:

.. list-table::
   :header-rows: 1
   :widths: 30 10 60

   * - Exception
     - Status
     - When raised
   * - :class:`AuthenticationError`
     - ``401``
     - Missing, expired, or invalid API key.
   * - :class:`AuthorizationError`
     - ``403``
     - Authenticated but insufficient permissions.
   * - :class:`NotFoundError`
     - ``404``
     - Requested resource (user, session, project) does not exist.
   * - :class:`EntityNotFoundError`
     - ``404``
     - Requested graph entity node was not found.
   * - :class:`ConflictError`
     - ``409``
     - Resource already exists (e.g. duplicate ``external_id``) or is in a
       conflicting state.
   * - :class:`PayloadTooLargeError`
     - ``413``
     - Request body exceeds the maximum allowed size.
   * - :class:`ValidationError`
     - ``422``
     - Request payload failed server-side validation.
   * - :class:`RateLimitError`
     - ``429``
     - Client exceeded rate-limit allowance.
   * - :class:`ExternalServiceError`
     - ``502``
     - External dependency (LLM, database, etc.) returned an error.
   * - :class:`GraphTimeoutError`
     - ``504``
     - Graph database operation exceeded the configured timeout.

Error handling patterns
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   from openzync import AsyncOpenZync
   from openzync._errors import (
       AuthenticationError,
       NotFoundError,
       ConflictError,
       RateLimitError,
       ValidationError,
   )

   async with AsyncOpenZync(api_key="...") as client:
       try:
           resp = await client.memory.ingest([
               {"role": "user", "content": "Hello"},
           ])
       except AuthenticationError:
           print("Check your API key")
       except ValidationError as err:
           print(f"Invalid request: {err.message}")
           if err.detail:
               print("Details:", err.detail)
       except RateLimitError:
           print("Slow down — retry after backoff")
       except ConflictError as err:
           print(f"Resource conflict: {err.message}")

.. _sdk-langchain-integration:

LangChain Integration
---------------------

.. module:: openzync.integrations.langchain

The SDK provides optional LangChain integration classes that wrap OpenZync
as memory, chat history, retriever, and tools for LangChain agents.

Install the extra dependency:

.. code-block:: bash

   pip install openzync[langchain]

OZChatMessageHistory
~~~~~~~~~~~~~~~~~~~~

.. class:: OZChatMessageHistory(session_id, project_id, client, *, max_messages=1000)

   Implements :class:`langchain_core.chat_history.BaseChatMessageHistory` with
   OpenZync as the backing store.

   :param session_id: LangChain conversation / session identifier.
   :type session_id: str
   :param project_id: OpenZync project UUID.
   :type project_id: str
   :param client: An :class:`AsyncOpenZync` client instance.
   :param max_messages: Maximum number of messages to fetch from the server.
   :type max_messages: int

   Provides both sync (``messages``, ``add_message``, ``add_messages``,
   ``clear``) and async (``aget_messages``, ``aadd_messages``, ``aclear``)
   interfaces.

   Sync methods use :func:`asyncio.run` internally and are **not safe**
   inside a running event loop.  Use async methods in async environments.

   .. code-block:: python

      from openzync import AsyncOpenZync
      from openzync.integrations.langchain import OZChatMessageHistory

      client = AsyncOpenZync(api_key="...")
      history = OZChatMessageHistory(
          session_id="session-123",
          project_id="project-abc",
          client=client,
      )

      # Sync interface
      history.add_user_message("Hi!")
      history.add_ai_message("Hello! How can I help?")
      for msg in history.messages:
          print(f"{msg.type}: {msg.content}")

      # Async interface
      await history.aadd_messages([
          HumanMessage(content="What's the weather?"),
          AIMessage(content="Sunny!"),
      ])
      messages = await history.aget_messages()

OZMemory
~~~~~~~~

.. class:: OZMemory(session_id, project_id, client, *, memory_key="chat_history", return_messages=True, input_key=None, output_key=None, max_messages=1000)

   Implements :class:`langchain_core.memory.BaseMemory` backed by OpenZync.
   Designed as a drop-in replacement for
   ``ConversationBufferMemory`` in LangChain chains.

   :param session_id: LangChain conversation / session identifier.
   :type session_id: str
   :param project_id: OpenZync project UUID.
   :type project_id: str
   :param client: An :class:`AsyncOpenZync` client instance.
   :param memory_key: Key under which memory variables are stored.
   :type memory_key: str
   :param return_messages: If ``True``, returns a list of
       :class:`BaseMessage`; if ``False``, returns a concatenated string.
   :type return_messages: bool
   :param input_key: Optional key for the input variable (auto-detected if
       not provided).
   :type input_key: str | None
   :param output_key: Optional key for the output variable (auto-detected if
       not provided).
   :type output_key: str | None
   :param max_messages: Maximum number of messages to fetch.
   :type max_messages: int

   .. code-block:: python

      from langchain.chains import ConversationChain
      from langchain.llms import OpenAI
      from openzync import AsyncOpenZync
      from openzync.integrations.langchain import OZMemory

      client = AsyncOpenZync(api_key="...")
      memory = OZMemory(
          session_id="session-123",
          project_id="project-abc",
          client=client,
          memory_key="chat_history",
          return_messages=True,
      )
      chain = ConversationChain(llm=OpenAI(), memory=memory)

   .. method:: get_context(query, limit=10)

      Retrieve relevant context from memory for LLM injection.

      :param query: Natural-language query describing the context needed.
      :type query: str
      :param limit: Maximum results per source type.
      :type limit: int
      :returns: :class:`ContextResponse` with a ``context`` string suitable
          for use as a system-prompt prefix.
      :rtype: ContextResponse

OZGraphRetriever
~~~~~~~~~~~~~~~~

.. class:: OZGraphRetriever(client, project_id, *, types="episodes,facts", k=5, score_threshold=None)

   Implements :class:`langchain_core.retrievers.BaseRetriever` that uses
   OpenZync's hybrid graph search to surface relevant context.

   :param client: An :class:`AsyncOpenZync` client instance.
   :param project_id: OpenZync project UUID to search within.
   :type project_id: str
   :param types: Comma-separated result types to include
       (``"episodes"``, ``"facts"``, ``"entities"``).
   :type types: str
   :param k: Maximum number of results to return.
   :type k: int
   :param score_threshold: Minimum relevance score (0–1) for results.
       ``None`` means no threshold.
   :type score_threshold: float | None

   .. code-block:: python

      from openzync import AsyncOpenZync
      from openzync.integrations.langchain import OZGraphRetriever

      client = AsyncOpenZync(api_key="...")
      retriever = OZGraphRetriever(
          client=client,
          project_id="project-abc",
          types="episodes,facts",
          k=10,
          score_threshold=0.5,
      )

      # Sync invoke
      docs = retriever.invoke("What does Alice know about Acme Corp?")
      for doc in docs:
          print(f"[{doc.metadata['type']}] (score={doc.metadata['score']:.3f}) {doc.page_content[:100]}")

      # Async invoke
      docs = await retriever.ainvoke("Project Orion status")

LangChain Tools
~~~~~~~~~~~~~~~

The SDK provides four LangChain :class:`BaseTool` implementations that give
LLM agents access to the OpenZync knowledge graph.

.. class:: AddFactsTool(client)

   Tool that adds structured fact triples to the knowledge graph.

   .. code-block:: python

      from openzync.integrations.langchain.tools import AddFactsTool

      tool = AddFactsTool(client=client)
      # The agent invokes this with AddFactsInput schema:
      # {
      #   "project_id": "...",
      #   "facts": [
      #     {"subject": "Alice", "predicate": "works_for", "object": "Acme Corp"}
      #   ]
      # }

.. class:: GraphSearchTool(client)

   Tool that searches the knowledge graph for relevant episodes, facts, or
   entities.

   Input schema: :class:`GraphSearchInput` (``query``, ``project_id``,
   optional ``types``, optional ``limit``).

.. class:: GraphNodeDetailTool(client)

   Tool that retrieves detailed information about a specific graph node,
   including all incident relationships.

   Input schema: :class:`GraphNodeDetailInput` (``project_id``, ``node_id``).

.. class:: ListGraphNodesTool(client)

   Tool that lists entity nodes in the knowledge graph, optionally filtered
   by entity type.

   Input schema: :class:`ListGraphNodesInput` (``project_id``, optional
   ``entity_type``, optional ``limit``).

Advanced Usage
--------------

Custom base URL
~~~~~~~~~~~~~~~

Point the SDK at a self-hosted or staging instance:

.. code-block:: python

   client = AsyncOpenZync(
       api_key="oz_live_...",
       base_url="https://api.staging.openzync.example.com",
   )

Custom timeout
~~~~~~~~~~~~~~

Adjust the per-request timeout for long-running operations:

.. code-block:: python

   client = AsyncOpenZync(
       api_key="oz_live_...",
       timeout=60.0,  # default is 30 seconds
   )

Resource cleanup
~~~~~~~~~~~~~~~~

Always close the client to release connection pool resources:

.. code-block:: python

   # Async — context manager (preferred)
   async with AsyncOpenZync(api_key="...") as client:
       ...

   # Async — manual
   client = AsyncOpenZync(api_key="...")
   try:
       ...
   finally:
       await client.close()

   # Sync
   client = OpenZync(api_key="...")
   try:
       ...
   finally:
       client.close()

Combining operations
~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   async def track_conversation(
       client: AsyncOpenZync,
       user_ext_id: str,
       session_ext_id: str,
       messages: list[dict],
   ) -> None:
       """Ingest messages with user and session context."""
       # Ensure user and session exist
       try:
           user = await client.users.create(external_id=user_ext_id)
       except ConflictError:
           # User already exists — fetch by listing (no get-by-external-id)
           # TODO: server-side lookup by external_id when available
           user = None

       try:
           session = await client.sessions.create(external_id=session_ext_id)
       except ConflictError:
           session = None

       # Ingest with session association
       resp = await client.memory.ingest(
           messages=messages,
           session_id=session_ext_id,
       )
       print(f"Ingested {resp.episode_count} episodes")

.. _sdk-repository-layout:

Repository Layout
-----------------

The SDK source tree at ``openzync-sdk-python/src/openzync/``::

   openzync-sdk-python/
   ├── pyproject.toml              # Package metadata, dependencies, build config
   └── src/openzync/
       ├── __init__.py             # Exports OpenZync, AsyncOpenZync, __version__
       ├── py.typed                # PEP 561 marker
       ├── _version.py             # Version from importlib.metadata
       ├── _errors.py              # Exception hierarchy (RFC 7807 mapping)
       ├── _http.py                # Async HTTP transport (httpx, retry, auth)
       ├── _pagination.py          # Cursor-based async/sync paginated iterators
       ├── client.py               # AsyncOpenZync + sync OpenZync wrapper
       ├── memory.py               # Memory domain client
       ├── facts.py                # Facts domain client
       ├── graph.py                # Graph domain client
       ├── users.py                # Users domain client
       ├── sessions.py             # Sessions domain client
       ├── projects.py             # Projects domain client
       ├── models/
       │   ├── __init__.py         # Re-exports all Pydantic models
       │   ├── memory.py
       │   ├── facts.py
       │   ├── graph.py
       │   ├── user.py
       │   ├── session.py
       │   └── project.py
       └── integrations/
           ├── __init__.py
           └── langchain/
               ├── __init__.py     # Exports OZMemory, OZChatMessageHistory, OZGraphRetriever, tools
               ├── memory.py       # OZMemory (BaseMemory implementation)
               ├── message_history.py  # OZChatMessageHistory (BaseChatMessageHistory)
               ├── retriever.py    # OZGraphRetriever (BaseRetriever)
               └── tools/
                   ├── __init__.py
                   ├── facts.py    # AddFactsTool
                   └── graph.py    # GraphSearchTool, GraphNodeDetailTool, ListGraphNodesTool
