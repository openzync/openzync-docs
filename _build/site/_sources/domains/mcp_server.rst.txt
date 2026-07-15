MCP Server (FastMCP)
====================

.. note::

   This document covers the **OpenZync MCP Server** at
   ``/home/rohan-linkai/code/personal/openzync/openzync-mcp/`` — an
   independent package (``openzync_mcp``) that exposes OpenZync's memory
   capabilities as LLM-accessible tools via the `Model Context Protocol
   (MCP) <https://spec.modelcontextprotocol.io>`_.

   The MCP server **replaces** the previous custom JSON-RPC 2.0
   implementation that lived in ``services.mcp`` inside the
   ``openzync-core`` monolith (see :doc:`../api/services.mcp` for the
   legacy API docs).  The new server is built on `FastMCP
   <https://gofastmcp.com>`_, which handles protocol compliance, transport
   negotiation, schema generation, and input validation automatically.

   **What this document does not cover**: The ``openzync-core`` monolith's
   REST API.  See :doc:`memory_context` for the ingestion and retrieval
   pipeline, :doc:`auth` for authentication, and :doc:`api_layer` for the
   full HTTP API reference.

.. contents:: Sections
   :local:
   :depth: 3
   :class: this-will-duplicate-information-and-it-is-still-useful-here


Overview
--------

The OpenZync MCP Server is a lightweight Python process that translates
`MCP tool calls <https://spec.modelcontextprotocol.io/latest/schema/#tool>`_
into OpenZync REST API calls via the OpenZync Python SDK
(``openzync.client.AsyncOpenZync``).  It is designed to be run alongside an
LLM host (Claude Desktop, Cursor, custom agent frameworks) and provides the
agent with persistent, queryable memory.

.. mermaid::

   flowchart LR
       A[LLM Host<br/>e.g. Claude Desktop] -->|MCP stdio/SSE/HTTP| B[OpenZync MCP Server]
       B -->|OpenZync SDK| C[OpenZync API]

       subgraph MCP Server Process
           B --> D[FastMCP Framework]
           D --> E[Tool Handlers<br/>memory/facts/graph/users/sessions]
           E --> F[AsyncOpenZync Client]
       end

       C --> G[(PostgreSQL)]
       C --> H[(Redis)]

**Architecture highlights:**

* **FastMCP framework** — handles JSON-RPC 2.0 protocol, schema generation
  from Python type hints, input validation, and transport negotiation.
* **SDK-backed** — tools call the OpenZync REST API via
  ``AsyncOpenZync``.  No direct database access.
* **Three transports** — stdio (default, for Claude Desktop), SSE (legacy),
  and Streamable HTTP (for remote deployments).
* **Zero silent fallback** — every failure propagates as an exception to
  the MCP client.  No degraded behaviour, no cached results returned on
  failure.

Repository Layout
-----------------

The MCP server lives in its own repository at
``/home/rohan-linkai/code/personal/openzync/openzync-mcp/``::

   openzync-mcp/
   └── openzync_mcp/
       ├── __init__.py
       ├── __main__.py          # Entry point with CLI arg parsing
       ├── server.py            # FastMCP server singleton + lifespan
       ├── Dockerfile           # Production container definition
       └── tools/
           ├── __init__.py
           ├── memory.py        # add_memory, get_context, search_memory, delete_memory
           ├── facts.py         # add_fact, list_facts
           ├── graph.py         # get_user_graph
           ├── sessions.py      # list_sessions
           └── users.py         # create_user

**Files not present** (noted as gaps):

* ``pyproject.toml`` — missing.  The Dockerfile builds from a ``setup.py``
  or a separate project config that is not in this repository.
* ``LICENSE`` — missing from the repository root (may be inherited from the
  parent project).
* ``README`` — missing.
* ``.gitignore`` — missing.

These are packaging/documentation gaps.  See :ref:`mcp-repo-gaps` for
details.


Server Lifecycle
----------------

Module: ``openzync_mcp/server.py``

The server is a singleton :class:`FastMCP` instance created at import time::

   mcp = FastMCP(
       "OpenZync-mcp",
       instructions=(
           "OpenZync agent memory platform — persist, query, and manage "
           "agent memory..."
       ),
       version="0.1.0",
       lifespan=openzync_lifespan,
   )

The lifespan function (``openzync_lifespan``) manages the SDK client:

#. Reads ``OPENZYN_API_KEY`` and ``OPENZYN_BASE_URL`` from environment
   variables (set by ``__main__.py`` before starting the server).
#. Creates an ``AsyncOpenZync`` client if one is not already injected (for
   test support).
#. Yields the client in the lifespan context dict, accessible to tool
   handlers via ``ctx.lifespan_context["client"]``.
#. On shutdown, closes the SDK client.

.. rubric:: Test Injection Hook

For unit tests, pre-set ``server._oz_client`` with a mock client before
creating ``Client(mcp)``.  The lifespan will use the pre-set client as-is
and will **not** close it on shutdown (the test fixture owns the lifecycle).


Entry Point & CLI
-----------------

Module: ``openzync_mcp/__main__.py``

The server is started via ``python -m openzync_mcp`` with the following
arguments:

.. list-table::
   :header-rows: 1

   * - Argument
     - Default
     - Description
   * - ``--transport``
     - ``stdio``
     - Transport protocol.  Choices: ``stdio``, ``sse``, ``http``.
     - ``--host``
     - ``0.0.0.0``
     - Bind address (used for ``sse`` and ``http`` transports only).
   * - ``--port``
     - ``8100``
     - Server port (used for ``sse`` and ``http`` transports only).
   * - ``--api-key``
     - ``OPENZYN_API_KEY`` env var
     - OpenZync API key.  Required — the server will exit with an error
       if neither the argument nor the env var is set.
   * - ``--base-url``
     - ``http://localhost:8000``
     - OpenZync API base URL.  Set to the production URL when deploying.

**Logging**: All logging goes to **stderr** (stdout is reserved for the
stdio transport protocol).

Usage examples::

   # Run locally for Claude Desktop (stdio transport)
   python -m openzync_mcp --api-key oz_live_abc123

   # Run as a remote HTTP server
   python -m openzync_mcp --transport http --port 8100 \
       --api-key oz_live_abc123 \
       --base-url https://api.openzync.com

   # Run with SSE transport (legacy)
   python -m openzync_mcp --transport sse --port 8100


Authentication Model
--------------------

The MCP server itself does **not** authenticate incoming MCP requests —
authentication is handled at the OpenZync API level via the API key.

.. mermaid::

   sequenceDiagram
       participant LLM as LLM Host
       participant MCP as MCP Server
       participant API as OpenZync API

       Note over MCP: Startup: reads OPENZYN_API_KEY
       MCP->>MCP: AsyncOpenZync(api_key=..., base_url=...)

       LLM->>MCP: MCP Tool Call (stdio/SSE/HTTP)
       MCP->>MCP: Validate inputs (FastMCP)
       MCP->>API: HTTP Request + Authorization: Bearer {api_key}
       API->>API: Verify API key (AuthMiddleware)
       API-->>MCP: Response / Error
       MCP-->>LLM: MCP Tool Result / Error

**Key points:**

* The API key must be a valid OpenZync project-scoped key (prefix
  ``oz_live_`` or ``oz_test_``).
* The key is passed to the ``AsyncOpenZync`` client at startup and sent as
  a Bearer token on every API call.
* There is **no per-request authentication** at the MCP level — any MCP
  client that can connect to the server inherits its authority.
* For insecure networks, run the server with ``--transport stdio``
  (process-local) or use a reverse proxy with authentication (e.g. mutual
  TLS) in front of the HTTP transport.

.. seealso::

   :doc:`auth` for the full API key authentication flow in the OpenZync
   backend, including key creation, revocation, and verification.


Available MCP Tools
-------------------

The MCP server exposes **8 tools** across 5 domains:

.. list-table::
   :header-rows: 1

   * - Tool Name
     - Module
     - Category
     - Description
   * - ``add_memory``
     - :mod:`openzync_mcp.tools.memory`
     - Memory
     - Ingest conversation messages into a project.
   * - ``get_context``
     - :mod:`openzync_mcp.tools.memory`
     - Memory
     - Assemble an LLM-ready context block from hybrid search.
   * - ``search_memory``
     - :mod:`openzync_mcp.tools.memory`
     - Memory
     - Search across episodes, facts, and entities.
   * - ``delete_memory``
     - :mod:`openzync_mcp.tools.memory`
     - Memory
     - Soft-delete all project memory (GDPR wipe).
   * - ``add_fact``
     - :mod:`openzync_mcp.tools.facts`
     - Facts
     - Inject fact triples into the knowledge graph.
   * - ``list_facts``
     - :mod:`openzync_mcp.tools.facts`
     - Facts
     - Search facts by keyword query.
   * - ``get_user_graph``
     - :mod:`openzync_mcp.tools.graph`
     - Graph
     - Explore the entity-relationship knowledge graph.
   * - ``list_sessions``
     - :mod:`openzync_mcp.tools.sessions`
     - Sessions
     - List conversation sessions with pagination.
   * - ``create_user``
     - :mod:`openzync_mcp.tools.users`
     - Users
     - Create a new end-user.

All tools follow a consistent pattern:

* Log every invocation, success, and error with structured context
  (duration, IDs, counts).
* Validate all inputs with guard clauses (raise ``ValueError`` for invalid
  arguments).
* Time every operation and include the duration in log lines.
* Never swallow exceptions — errors propagate back to the MCP client.

Below is the detailed specification for each tool.


add_memory
~~~~~~~~~~

**Purpose**: Persist conversation messages as episodes and enqueue async
enrichment (entity extraction, fact extraction, embedding, classification).

.. list-table:: Input Parameters
   :header-rows: 1

   * - Parameter
     - Type
     - Required
     - Description
   * - ``project_id``
     - ``string``
     - Yes
     - The internal UUID of the target project.
   * - ``messages``
     - ``list[dict]``
     - Yes
     - List of message objects.  Each must have ``role``
       (``"user"`` | ``"assistant"`` | ``"system"`` | ``"tool"``)
       and ``content`` (message body).  1–1000 messages per call.
   * - ``session_id``
     - ``string | null``
     - No
     - Optional session external ID.  If omitted, a
       ``__default__`` session is auto-created.

**Output**: A confirmation string with the job ID and episode count::

   "Memory recorded. 5 messages ingested (job: abc-def-ghi)."

**Example call**::

   # In Claude Desktop or any MCP client
   {
     "name": "add_memory",
     "arguments": {
       "project_id": "550e8400-e29b-41d4-a716-446655440000",
       "messages": [
         {"role": "user", "content": "What is machine learning?"},
         {"role": "assistant", "content": "Machine learning is a subset of AI..."}
       ],
       "session_id": "my-session-001"
     }
   }

**Backend mapping**: Calls ``POST /v1/projects/{project_id}/memory`` on the
OpenZync API.

**Errors**:

* ``ValueError`` — empty ``messages`` list or more than 1000 messages.
* All API errors (auth failure, project not found, quota exceeded)
  propagate as-is from the SDK.


get_context
~~~~~~~~~~~

**Purpose**: Assemble a context block for LLM injection from a
natural-language query.  Returns recent episodes, extracted facts, and
graph entities related to the query, formatted as plain text suitable for
inclusion in an LLM prompt.

.. list-table:: Input Parameters
   :header-rows: 1

   * - Parameter
     - Type
     - Required
     - Description
   * - ``project_id``
     - ``string``
     - Yes
     - The internal UUID of the target project.
   * - ``query``
     - ``string``
     - Yes
     - A natural-language query describing the needed context
       (e.g. ``"what does the user know about machine learning"``).
   * - ``limit``
     - ``integer``
     - No
     - Maximum items per source type (1–100, default 20).

**Output**: A plain-text context block with episodes, facts, and entities
formatted for LLM consumption.

**Example call**::

   {
     "name": "get_context",
     "arguments": {
       "project_id": "550e8400-e29b-41d4-a716-446655440000",
       "query": "what have we discussed about Python async patterns",
       "limit": 10
     }
   }

**Backend mapping**: Calls ``GET /v1/projects/{project_id}/context`` on the
OpenZync API.

**Retrieval pipeline** (see :doc:`memory_context` for details):

#. Hybrid search (5 concurrent legs): episode vector, episode BM25, fact
   vector, fact BM25, graph BFS.
#. RRF merge per source type.
#. Optional cross-encoder re-ranking.
#. Format as plain text.
#. Cache result in Redis (TTL 30s).

**Errors**:

* ``ValueError`` — empty query or limit outside 1–100.


search_memory
~~~~~~~~~~~~~

**Purpose**: Search across a project's memory using hybrid retrieval.
Returns results fused via RRF and sorted by relevance score.

.. list-table:: Input Parameters
   :header-rows: 1

   * - Parameter
     - Type
     - Required
     - Description
   * - ``project_id``
     - ``string``
     - Yes
     - The internal UUID of the target project.
   * - ``query``
     - ``string``
     - Yes
     - Search query string.
   * - ``types``
     - ``string``
     - No
     - Comma-separated result types to include:
       ``"episodes"``, ``"facts"``, ``"entities"``
       (default: ``"episodes,facts"``).
   * - ``limit``
     - ``integer``
     - No
     - Maximum results per type (default 20, max 100).

**Output**: A formatted string of search results with relevance scores::

   Found 3 result(s):
     [0.8921] Machine learning is a subset of AI...
     [0.7450] The user asked about supervised learning...
     [0.6233] Alice is proficient in Python and ML frameworks

Returns ``"No results found."`` if no matches.

**Example call**::

   {
     "name": "search_memory",
     "arguments": {
       "project_id": "550e8400-e29b-41d4-a716-446655440000",
       "query": "machine learning topics",
       "types": "episodes,facts,entities",
       "limit": 5
     }
   }

**Backend mapping**: Calls the OpenZync graph search endpoint with the
specified types.

**Errors**:

* ``ValueError`` — empty query, limit outside 1–100, or invalid type
  string (allowed: ``episodes``, ``facts``, ``entities``).


delete_memory
~~~~~~~~~~~~~

**Purpose**: Delete all memory for a project (soft-delete).  This is the
GDPR memory-wipe operation and is **not** reversible — deleted data is
marked inactive but preserved for a 30-day grace period before hard-purge.

.. list-table:: Input Parameters
   :header-rows: 1

   * - Parameter
     - Type
     - Required
     - Description
   * - ``project_id``
     - ``string``
     - Yes
     - The internal UUID of the target project.

**Output**: ``"Memory deleted successfully."``

**Example call**::

   {
     "name": "delete_memory",
     "arguments": {
       "project_id": "550e8400-e29b-41d4-a716-446655440000"
     }
   }

**Backend mapping**: Calls ``DELETE /v1/projects/{project_id}/memory`` on
the OpenZync API.

**Warning**: This operation:

* Soft-deletes all episodes and facts for the project.
* Does **not** delete sessions.
* Is irreversible via the API (data can be recovered by support within
  30 days via DB restore).


add_fact
~~~~~~~~

**Purpose**: Inject business fact triples into a project's knowledge graph.
Facts are persisted as (subject, predicate, object) triples and queued for
async embedding.

.. list-table:: Input Parameters
   :header-rows: 1

   * - Parameter
     - Type
     - Required
     - Description
   * - ``project_id``
     - ``string``
     - Yes
     - The internal UUID of the target project.
   * - ``facts``
     - ``list[dict]``
     - Yes
     - List of fact triples.  Each dict must have ``subject``,
       ``predicate``, and ``object`` keys.  Optional
       ``confidence`` (float, default 1.0).  Max 500 per call.
   * - ``session_id``
     - ``string | null``
     - No
     - Optional session external ID for attribution.

**Output**: A confirmation string with accepted count and job ID::

   "3 fact(s) accepted for processing (job: abc-def-ghi)."

**Example call**::

   {
     "name": "add_fact",
     "arguments": {
       "project_id": "550e8400-e29b-41d4-a716-446655440000",
       "facts": [
         {
           "subject": "Alice",
           "predicate": "expert_in",
           "object": "Machine Learning",
           "confidence": 0.95
         },
         {
           "subject": "Alice",
           "predicate": "works_at",
           "object": "Acme Corp"
         }
       ],
       "session_id": "my-session-001"
     }
   }

**Backend mapping**: Calls ``POST /v1/projects/{project_id}/facts`` on the
OpenZync API.

**Errors**:

* ``ValueError`` — empty ``facts`` list, more than 500 facts, or any fact
  dict missing ``subject``, ``predicate``, or ``object`` keys.


list_facts
~~~~~~~~~~

**Purpose**: Search facts (knowledge triples) by keyword query.  Returns
matching facts sorted by relevance.

.. list-table:: Input Parameters
   :header-rows: 1

   * - Parameter
     - Type
     - Required
     - Description
   * - ``project_id``
     - ``string``
     - Yes
     - The internal UUID of the target project.
   * - ``query``
     - ``string``
     - Yes
     - Keyword search query.
   * - ``limit``
     - ``integer``
     - No
     - Maximum results (default 20, max 100).

**Output**: A formatted string of matching facts with confidence scores::

   Found 2 fact(s):
     [0.95] Alice expert_in Machine Learning
     [0.85] Alice works_at Acme Corp

Returns ``"No facts found."`` if no matches.

**Example call**::

   {
     "name": "list_facts",
     "arguments": {
       "project_id": "550e8400-e29b-41d4-a716-446655440000",
       "query": "Alice",
       "limit": 20
     }
   }

**Backend mapping**: Calls the OpenZync graph search endpoint scoped to
``types="facts"``.

**Errors**:

* ``ValueError`` — empty query or limit outside 1–100.


get_user_graph
~~~~~~~~~~~~~~

**Purpose**: Get the entity graph for a project — nodes (entities) and
edges (relationships).  Optionally filter by entity type.  Edges are
fetched in parallel for up to 20 entities; for larger graphs, edges are
shown only for the first 20 entities.

.. list-table:: Input Parameters
   :header-rows: 1

   * - Parameter
     - Type
     - Required
     - Description
   * - ``project_id``
     - ``string``
     - Yes
     - The internal UUID of the target project.
   * - ``entity_type``
     - ``string | null``
     - No
     - Optional entity type filter (e.g. ``"Person"``,
       ``"Organization"``, ``"Topic"``).  Omit to return all types.
   * - ``limit``
     - ``integer``
     - No
     - Maximum entities and edges to return (default 50, max 200).

**Output**: A formatted string listing entities and their relationships::

   Found 3 entity(ies):
     [Person] Alice (abc12345...)
     [Topic] Machine Learning (def67890...)
     [Organization] Acme Corp (ghi11121...)

   2 edge(s) (from 3 entity sources):
     [expert_in] abc12345... → def67890...
     [works_at] abc12345... → ghi11121...

Returns ``"No entities found in the graph."`` if the graph is empty.

**Example call**::

   {
     "name": "get_user_graph",
     "arguments": {
       "project_id": "550e8400-e29b-41d4-a716-446655440000",
       "entity_type": "Person",
       "limit": 30
     }
   }

**Backend mapping**: Calls the OpenZync graph nodes and edges endpoints.

**Edge-fetch behaviour**:

* Edges are fetched for up to 20 entities in parallel
  (``asyncio.gather`` with ``return_exceptions=True``).
* Partial failures are handled gracefully — failed entity edge fetches
  are skipped with a warning logged.
* Edges are deduplicated by ``(source_id, target_id, type)``.

**Errors**:

* ``ValueError`` — empty ``project_id`` or limit outside 1–200.


list_sessions
~~~~~~~~~~~~~

**Purpose**: List conversation sessions for a project with cursor-based
pagination.  Each session groups a sequence of conversation messages
(episodes).

.. list-table:: Input Parameters
   :header-rows: 1

   * - Parameter
     - Type
     - Required
     - Description
   * - ``project_id``
     - ``string``
     - Yes
     - The internal UUID of the target project.
   * - ``limit``
     - ``integer``
     - No
     - Maximum sessions per page (default 50, max 200).
   * - ``cursor``
     - ``string | null``
     - No
     - Opaque pagination cursor from a previous response.
       Omit to fetch the first page.

**Output**: A formatted string listing sessions with IDs and message
counts, plus a pagination hint if more pages are available::

   Found 3 session(s):
     [abc12345] my-session-001 (15 messages)
     [def67890] my-session-002 (8 messages)
     [ghi11121] __default__ (42 messages)

   More sessions available. Use cursor="eyJsYXN0X2lkI..." for the next page.

Returns ``"No sessions found."`` if the project has no sessions.

**Example call**::

   {
     "name": "list_sessions",
     "arguments": {
       "project_id": "550e8400-e29b-41d4-a716-446655440000",
       "limit": 20
     }
   }

   # Second page:
   {
     "name": "list_sessions",
     "arguments": {
       "project_id": "550e8400-e29b-41d4-a716-446655440000",
       "cursor": "eyJsYXN0X2lkIjoiYWJjMTIzIn0=",
       "limit": 20
     }
   }

**Backend mapping**: Calls the OpenZync sessions list endpoint.

**Errors**:

* ``ValueError`` — empty ``project_id`` or limit outside 1–200.


create_user
~~~~~~~~~~~

**Purpose**: Create a new end-user within an organization.  Users represent
end-users within an organization and are identified by a caller-chosen
``external_id``.

.. list-table:: Input Parameters
   :header-rows: 1

   * - Parameter
     - Type
     - Required
     - Description
   * - ``external_id``
     - ``string``
     - Yes
     - Caller-defined user identifier
       (e.g. ``"customer-abc-123"``).  Must be non-empty and
       unique per organization.
   * - ``name``
     - ``string | null``
     - No
     - Optional display name for the user.

**Output**: A confirmation string with the created user's ID and details::

   User created successfully.
     ID: 550e8400-e29b-41d4-a716-446655440000
     External ID: customer-abc-123
     Name: Alice Smith

**Example call**::

   {
     "name": "create_user",
     "arguments": {
       "external_id": "customer-abc-123",
       "name": "Alice Smith"
     }
   }

**Backend mapping**: Calls ``POST /v1/users`` on the OpenZync API (or the
equivalent SDK method).

**Errors**:

* ``ValueError`` — empty ``external_id``.
* ``ConflictError`` from the API if ``external_id`` already exists in the
  organization (propagates from the SDK).


Container Setup
---------------

The Docker image uses a multi-stage build for a minimal production image:

.. code-block:: dockerfile

   FROM python:3.12-slim AS builder
   WORKDIR /app
   COPY pyproject.toml .
   RUN pip install --no-cache-dir --user -e .

   FROM python:3.12-slim AS runtime
   WORKDIR /app
   COPY --from=builder /root/.local /root/.local
   COPY . .
   ENV PATH=/root/.local/bin:$PATH \
       PYTHONUNBUFFERED=1 \
       PYTHONDONTWRITEBYTECODE=1
   EXPOSE 8100
   CMD ["python", "-m", "openzync_mcp", "--transport", "http", \
        "--host", "0.0.0.0", "--port", "8100"]

**Build & run**::

   docker build -t openzync-mcp:latest -f openzync_mcp/Dockerfile .
   docker run -p 8100:8100 --env-file .env openzync-mcp:latest

The container expects:

* ``OPENZYN_API_KEY`` — required at runtime.
* ``OPENZYN_BASE_URL`` — defaults to ``http://localhost:8000`` (set to
  the actual API URL in production).


Setup & Connection
------------------

Claude Desktop
~~~~~~~~~~~~~~

To connect Claude Desktop to the OpenZync MCP server, add an MCP server
entry to your Claude Desktop configuration:

.. code-block:: json
   :caption: ~/Library/Application Support/Claude/claude_desktop_config.json (macOS)

   {
     "mcpServers": {
       "openzync": {
         "command": "python",
         "args": [
           "-m",
           "openzync_mcp",
           "--api-key",
           "oz_live_YOUR_API_KEY"
         ]
       }
     }
   }

Claude Desktop communicates with the MCP server over stdio — the server
inherits Claude's process environment.

Custom Application (HTTP transport)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For remote deployments or custom agent frameworks, start the server with
HTTP transport::

   docker run -d -p 8100:8100 \
     -e OPENZYN_API_KEY=oz_live_YOUR_API_KEY \
     -e OPENZYN_BASE_URL=https://api.openzync.com \
     openzync-mcp:latest

Then connect from any MCP client::

   from mcp import Client

   async with Client("http://localhost:8100/mcp") as client:
       result = await client.call_tool(
           "add_memory",
           arguments={
               "project_id": "...",
               "messages": [{"role": "user", "content": "Hello"}],
           },
       )
       print(result.content)


Testing
-------

The server includes a test injection hook for the SDK client.  In unit
tests, pre-set ``server._oz_client`` before creating ``Client(mcp)``::

   from unittest.mock import AsyncMock
   from openzync_mcp.server import mcp

   # Pre-set a mock client — the lifespan will use it as-is
   mock_client = AsyncMock()
   mock_client.memory.ingest = AsyncMock(return_value=...)
   mcp._oz_client = mock_client

   # Now create the MCP test client
   from fastmcp import Client
   client = Client(mcp)
   result = await client.call_tool("add_memory", arguments={...})

.. note::

   When ``_oz_client`` is pre-set, the lifespan does **not** create a new
   client and does **not** close it on shutdown — the test fixture owns
   the lifecycle.


Tool Patterns & Conventions
---------------------------

Every tool in the MCP server follows these conventions:

**Structured logging**

Every invocation, success, and error is logged with structured context::

   logger.info("mcp.tool.invoke tool=%s project_id=%s ...",
               "add_memory", project_id, ...)
   logger.info("mcp.tool.success tool=%s duration_ms=%d ...",
               "add_memory", elapsed, ...)
   logger.error("mcp.tool.error tool=%s duration_ms=%d ...",
                "add_memory", elapsed, exc_info=True)

**Input validation**

All inputs are validated at the tool boundary with immediate ``raise
ValueError`` — no silent truncation or coercion::

   if not messages:
       raise ValueError("At least one message is required.")
   if len(messages) > 1000:
       raise ValueError("Maximum 1000 messages per call.")

**Timing**

Every tool measures wall-clock time and includes it in success/error logs.

**No silent fallback**

As per the platform's design principle (see :doc:`core`), the MCP server
never falls back silently.  If the OpenZync API is unreachable, the API key
is invalid, or a rate limit is hit, the error propagates directly to the
MCP client.

**Mutable type validation**

The ``add_fact`` tool validates that each fact is a ``dict`` with the
required keys before making any API call, providing early, clear error
messages.


Cross-References
----------------

* :doc:`memory_context` — the ingestion and retrieval pipeline that the MCP
  tools call into.
* :doc:`auth` — the API key authentication model that the MCP server uses.
* :doc:`api_layer` — the full REST API that the SDK wraps.
* :doc:`workers` — the ARQ background workers that process enrichment
  tasks after ingestion.
* :doc:`../api/services.mcp` — legacy MCP service docs (the previous
  custom JSON-RPC implementation in the monolith).
* :doc:`../api/services.mcp.tools` — API reference for the tool modules.
* :doc:`core` — configuration and dependency injection patterns.


.. _mcp-repo-gaps:

Repository Gaps
---------------

The following files are missing from the MCP server repository
(``/home/rohan-linkai/code/personal/openzync/openzync-mcp/``):

.. list-table::
   :header-rows: 1

   * - File
     - Impact
     - Recommendation
   * - ``pyproject.toml``
     - The Dockerfile references ``pyproject.toml`` for ``pip install -e .``,
       but the file is not present in the repository.  The build process
       may rely on a parent project's config or an unpublished
       ``setup.py``.
     - Add ``pyproject.toml`` with project metadata,
       dependencies, and entry point configuration.
   * - ``LICENSE``
     - Legal status is unclear — no license file in the repository.
     - Add a license file matching the parent project (Apache 2.0 /
       MIT).
   * - ``README``
     - No documentation in the repository about what this is, how to
       build it, or how to contribute.
     - Add a ``README.md`` covering build, run, and development
       setup.
   * - ``.gitignore``
     - No gitignore means generated artifacts (``__pycache__``,
       ``*.egg-info``, ``.venv``) may be accidentally committed.
     - Add a standard Python ``.gitignore``.

These are packaging gaps only — the runtime code is functional and
self-contained.
