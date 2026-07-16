Welcome to OpenZync's documentation!
=====================================

**OpenZync** is an open-source agent memory platform — persistent, queryable,
graph-based memory for AI agents. It provides hybrid retrieval (semantic +
graph), knowledge graphs with temporal queries, and multi-tenant isolation for
production LLM applications.

.. note::

   OpenZync is currently a **monolith** — all backend domains live in a single
   repository (``openzync-core``). Future work may decompose into separate
   microservices, but today everything shares a single FastAPI process, a
   single PostgreSQL database, and a single Redis instance.

.. toctree::
   :maxdepth: 2
   :caption: Getting Started

   guides/overview
   guides/quickstart
   guides/architecture
   guides/deployment
   guides/contributing

.. toctree::
   :maxdepth: 2
   :caption: Core Infrastructure

   domains/core
   domains/llm
   domains/graph_backends
   domains/reranker

.. toctree::
   :maxdepth: 2
   :caption: Domain Services

   domains/auth
   domains/memory_context
   domains/admin_webhooks
   domains/idempotency

.. toctree::
   :maxdepth: 2
   :caption: API & Workers

   domains/api_layer
   domains/workers

.. toctree::
   :maxdepth: 2
   :caption: SDK, MCP & Frontend

   domains/sdk_python
   domains/mcp_server
   domains/frontend

.. toctree::
   :maxdepth: 2
   :caption: Deployment & Infra

   domains/infrastructure

.. toctree::
   :maxdepth: 4
   :caption: API Reference (Auto-generated)

   api/core
   api/routers
   api/models
   api/schemas
   api/services
   api/repositories
   api/middleware
   api/dependencies
   api/workers
   api/utils
   api/packages

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
