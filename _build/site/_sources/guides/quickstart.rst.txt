Quickstart
==========

This guide takes you from zero to a running local OpenZync instance and your
first successful API call — using only the documentation.

If you already understand the architecture, this is all you need. For detailed
deployment options (production Docker Compose, Helm, observability stack), see
:doc:`/domains/infrastructure`.

.. contents:: Sections
   :local:
   :depth: 3
   :class: this-will-duplicate-information-and-it-is-still-useful-here

--------------
Prerequisites
--------------

Before you begin, ensure you have the following installed and available on your
``PATH``:

* **Docker** ``>=24.0`` and **Docker Compose** ``>=v2.20``
  (``docker compose`` plugin, not the standalone ``docker-compose``).
* **~3 GB** of free disk space for the backing containers (PostgreSQL, Redis,
  OpenBao) plus the Docker images.
* **OpenSSL** (``openssl``) and **Python 3.11+** (``python3``) — used to
  generate the required bootstrap secrets.
* **curl** for API smoke-testing (or any HTTP client of your choice).
* **Ports** ``8000``, ``5432``, ``6379``, and ``8200`` free on ``127.0.0.1``.
  These are bound by the backend stack.

---------------------------
Clone and Bootstrap
---------------------------

Step 1 — Clone the repository
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   git clone https://github.com/rohnsha0/openzync.git
   cd openzync/openzync-core

Step 2 — Generate bootstrap secrets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The stack uses four required environment variables. Three of them are
**bootstrap-only** — after first boot, every secret is stored in OpenBao and
auto-rotated. The fourth (``BAO_STATIC_SEAL_KEY``) must be backed up; if lost,
all secrets in OpenBao are irrecoverable.

.. code-block:: bash

   # Copy the template
   cp .env.example .env

   # Generate all four required secrets
   sed -i "s|<replace-with-64-char-hex>|$(openssl rand -hex 32)|" .env
   sed -i "s|<replace-with-32-byte-base64>|$(openssl rand -base64 32)|" .env
   sed -i "s|<replace-with-48+-char-url-safe-secret>|$(python3 -c "import secrets; print(secrets.token_urlsafe(48))")|" .env
   sed -i "s|<replace-with-32+-char-url-safe-secret>|$(python3 -c "import secrets; print(secrets.token_urlsafe(32))")|" .env

   # Verify the file has no remaining placeholders
   grep -c '<replace-with' .env && echo "ERROR: some secrets were not replaced" || echo "OK"

What each variable does:

.. list-table::
   :header-rows: 1

   * - Variable
     - Purpose
     - Generation
     - Stored in OpenBao after boot?
   * - ``BAO_STATIC_SEAL_KEY``
     - OpenBao auto-unseal key (AES-256-GCM). **Back this up** — if lost,
       all OpenBao secrets are irrecoverable.
     - ``openssl rand -hex 32`` (64 hex chars)
     - No (used by OpenBao directly)
   * - ``POSTGRES_PASSWORD``
     - Postgres superuser password for first-boot cluster init. Auto-rotated
       after boot.
     - ``openssl rand -base64 32``
     - Discarded after rotation
   * - ``OZ_SECRET_KEY``
     - JWT signing and application crypto operations.
     - ``python -c "import secrets; print(secrets.token_urlsafe(48))"``
     - Yes (in system secret)
   * - ``OZ_WEBHOOK_SIGNING_SECRET``
     - HMAC-SHA256 webhook signing key (Svix-compatible).
     - ``python -c "import secrets; print(secrets.token_urlsafe(32))"``
     - Yes (in system secret)

.. note::

   If you see ``<replace-with-*>`` values remaining in ``.env``, Docker Compose
   will refuse to start with an explicit error message naming the missing
   variable.  The ``:?`` modifier on every required env var is fail-fast by
   design.

Step 3 — Start the stack
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   docker compose -f infra/docker-compose.backend.yml up -d

This pulls the images and starts every service in the correct dependency order.
No ``make migrate``, no manual secret copy-paste.

Step 4 — Tail the bootstrap logs (optional, but instructive)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   docker compose -f infra/docker-compose.backend.yml logs -f

Watch for each phase completing. The full bootstrap takes **~60 seconds** on a
cold start (first pull may take longer depending on your connection speed).

----------------------------------------
Boot Sequence — What Happens in Those 60s
----------------------------------------

The bootstrap is an eight-phase directed acyclic graph enforced by Docker
Compose ``depends_on`` conditions:

.. code-block:: text

   openbao ─► openbao-init ─► postgres ─► postgres-init ─► postgres-migrate
                                                               │
                                   ┌───────────────────────────┘
                                   ▼
                               openbao-write-db
                                   │
                     ┌─────────────┴─────────────┐
                     ▼                           ▼
             openbao-agent-api            openbao-agent-worker
                (sidecar)                     (sidecar)
                     │                           │
                     ▼                           ▼
                    api                        worker
                     │                           │
                     └─────────► redis ◄─────────┘

Timeline:

+-----------------+---------------------------------------------------------------+
| Time (approx.)  | Phase                                                         |
+=================+===============================================================+
| **0–10s**       | ``openbao`` starts. The Raft storage node boots and the       |
|                 | ``static_kv`` seal reads ``BAO_STATIC_SEAL_KEY`` from the     |
|                 | environment. The healthcheck waits for ``"sealed":false``.    |
+-----------------+---------------------------------------------------------------+
| **10–30s**      | ``openbao-init`` runs. On first boot: initialises the cluster |
|                 | (5 key shares, 3 threshold), unseals, mounts KV v2, writes    |
|                 | ACL policies, creates AppRoles (``openzync-app`` and          |
|                 | ``openzync-worker``), enables the Transit engine, and writes  |
|                 | the system secret (all ``OZ_*`` vars minus ``DATABASE_URL``)  |
|                 | to ``system/config/data/system``.                             |
+-----------------+---------------------------------------------------------------+
| **30–40s**      | ``postgres`` starts (``pgvector/pgvector:pg15``). The         |
|                 | healthcheck waits for ``pg_isready``.                         |
+-----------------+---------------------------------------------------------------+
| **40–50s**      | ``postgres-init`` connects as the superuser, creates the      |
|                 | ``openzync`` database and two least-privilege roles:          |
|                 | ``openzync_migrator`` (DDL) and ``openzync_app`` (CRUD).      |
|                 | Passwords are auto-generated and written to                   |
|                 | ``/bao-init/db-creds.json`` (mode 0600).                      |
+-----------------+---------------------------------------------------------------+
| **40–50s**      | ``postgres-migrate`` runs ``alembic upgrade head`` using the  |
|                 | migrator credentials from ``db-creds.json``. Applies all      |
|                 | schema migrations (tables, indexes, extensions).              |
+-----------------+---------------------------------------------------------------+
| **50–55s**      | ``openbao-write-db`` reads ``db-creds.json``, constructs      |
|                 | ``DATABASE_URL`` (``postgresql+asyncpg://openzync_app:<pw>    |
|                 | @postgres:5432/openzync``), and merges it into the OpenBao    |
|                 | system secret using CAS (Check-And-Set) to prevent concurrent |
|                 | overwrites.                                                    |
+-----------------+---------------------------------------------------------------+
| **55–60s**      | ``openbao-agent-api`` and ``openbao-agent-worker`` sidecars   |
|                 | authenticate via AppRole, render the system secret to         |
|                 | ``/run/secrets/system.env`` (``KEY=VALUE`` lines). The        |
|                 | ``api`` and ``worker`` entrypoints source this file and       |
|                 | ``exec`` uvicorn / ARQ.                                       |
+-----------------+---------------------------------------------------------------+

After this sequence, the API is available at ``http://localhost:8000`` and the
ARQ worker is listening on the Redis queue for enrichment jobs.

----------------------
Verify the Installation
----------------------

Once the bootstrap is complete, run the health and readiness checks:

.. code-block:: bash

   curl -s http://localhost:8000/v1/health | python3 -m json.tool

Expected response:

.. code-block:: json

   {
     "status": "ok",
     "service": "openzync-api"
   }

Readiness probe (checks PostgreSQL and Redis):

.. code-block:: bash

   curl -s http://localhost:8000/v1/ready | python3 -m json.tool

Expected response:

.. code-block:: json

   {
     "status": "ok",
     "checks": {
       "database": true,
       "redis": true
     }
   }

If you get a ``503`` with ``"database": false`` or ``"redis": false``, wait a
few more seconds and retry — the api container may have started before the
sidecar finished rendering the secrets. See :ref:`quickstart-troubleshooting`
below.

-----------------
First API Calls
-----------------

Step 1 — Sign up and get JWT tokens
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The signup flow is two-step by design: the server sends a 6-digit OTP to the
registered email, and the client completes the flow by verifying it.  In a
development environment the OTP is printed to the server logs (no email
infrastructure is configured by default).

.. code-block:: bash

   # (a) Create organization + admin user
   curl -s -X POST http://localhost:8000/v1/auth/signup \
     -H "Content-Type: application/json" \
     -d '{
       "email": "alice@example.com",
       "password": "SecurePass123!",
       "organization_name": "Acme Corp"
     }' | python3 -m json.tool

Expected response:

.. code-block:: json

   {
     "message": "Verification code sent to email",
     "email": "alice@example.com"
   }

.. code-block:: bash

   # (b) Retrieve the OTP from the docker logs
   docker compose -f infra/docker-compose.backend.yml logs api 2>&1 \
     | grep -oP '(?<=OTP: )\d{6}'

   # If the grep finds nothing, try the worker logs or a broader search:
   docker compose -f infra/docker-compose.backend.yml logs 2>&1 \
     | grep -i "verification.*code\|otp.*code\|[Oo][Tt][Pp]"

   # (c) Verify email with the OTP to receive JWT tokens
   curl -s -X POST http://localhost:8000/v1/auth/verify-email \
     -H "Content-Type: application/json" \
     -d '{
       "email": "alice@example.com",
       "otp": "483926"
     }' | python3 -m json.tool

Expected response:

.. code-block:: json

   {
     "access_token": "eyJhbGciOiJIUzI1NiIs...",
     "refresh_token": "oz_ref_xxxxxxxxxx",
     "expires_in": 1800,
     "token_type": "Bearer"
   }

Set the access token in a shell variable for the subsequent calls:

.. code-block:: bash

   TOKEN="eyJhbGciOiJIUzI1NiIs..."    # ← paste your access_token here

Step 2 — Create a project
~~~~~~~~~~~~~~~~~~~~~~~~~~~

All memory operations are scoped to a project. Create one:

.. code-block:: bash

   curl -s -X POST http://localhost:8000/v1/projects \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{
       "name": "My First Project",
       "description": "Trying out OpenZync"
     }' | python3 -m json.tool

Expected response (abbreviated):

.. code-block:: json

   {
     "id": "a1b2c3d4-...",
     "name": "My First Project",
     "description": "Trying out OpenZync",
     ...
   }

Save the project ID:

.. code-block:: bash

   PROJECT_ID="a1b2c3d4-..."    # ← paste your project id here

Step 3 — Ingest a message
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Ingest a conversation message. The API returns immediately (HTTP 202) and an
ARQ worker enriches the message asynchronously — extracting entities, facts,
and embeddings.

.. code-block:: bash

   curl -s -X POST "http://localhost:8000/v1/projects/${PROJECT_ID}/memory" \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{
       "session_id": "conv-001",
       "messages": [
         {"role": "user", "content": "Hi, my name is Alice and I work at Acme Corp in San Francisco."},
         {"role": "assistant", "content": "Nice to meet you, Alice! How can I help you today?"}
       ]
     }' | python3 -m json.tool

Expected response:

.. code-block:: json

   {
     "job_id": "e5f6g7h8-...",
     "episode_count": 2,
     "status": "accepted",
     "message": "Messages accepted for enrichment"
   }

The ``job_id`` can be used to poll the enrichment status at
``/v1/projects/{project_id}/memory/jobs/{job_id}``.

Step 4 — Retrieve context
~~~~~~~~~~~~~~~~~~~~~~~~~~~

After enrichment completes (typically 5–20 seconds), query OpenZync for an
assembled context block ready for LLM injection:

.. code-block:: bash

   curl -s "http://localhost:8000/v1/projects/${PROJECT_ID}/context?query=What+does+Alice+do&limit=5" \
     -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

Expected response:

.. code-block:: json

   {
     "context": "---\nEpisodes:\n  [user] Hi, my name is Alice and I work at Acme Corp in San Francisco.\n  [assistant] Nice to meet you, Alice! How can I help you today?\n\nFacts:\n  - Alice works_for Acme Corp\n  - Alice located_in San Francisco\n\nEntities:\n  - Alice (person)\n  - Acme Corp (organization)\n  - San Francisco (location)\n",
     "metadata": {
       "cache_hit": false,
       "total_episodes": 2,
       "total_facts": 2,
       "total_entities": 3
     }
   }

Step 5 — Search
~~~~~~~~~~~~~~~~

Perform a hybrid search across the project's memory:

.. code-block:: bash

   curl -s "http://localhost:8000/v1/projects/${PROJECT_ID}/search?q=Acme+Corp+San+Francisco&limit=10" \
     -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

Expected response (abbreviated):

.. code-block:: json

   {
     "query": "Acme Corp San Francisco",
     "results": [
       {
         "type": "episode",
         "content": "Hi, my name is Alice and I work at Acme Corp in San Francisco.",
         "score": 0.89,
         ...
       }
     ],
     "total": 1
   }

---------------
Using the SDK
---------------

The OpenZync Python SDK wraps the REST API in typed, async-first client
classes.  Install it from PyPI:

.. code-block:: bash

   pip install openzync

The SDK requires Python ``>=3.11``.  For LangChain integration, install with:

.. code-block:: bash

   pip install openzync[langchain]

Quick-start example:

.. code-block:: python

   import asyncio
   from openzync import AsyncOpenZync

   async def main() -> None:
       async with AsyncOpenZync(
           api_key="oz_live_xxxxxxxxxxxxxxxx",
           base_url="http://localhost:8000",
       ) as client:
           # 1. Ingest messages
           resp = await client.memory.ingest([
               {"role": "user", "content": "My name is Bob and I work at Initech."},
               {"role": "assistant", "content": "Hello Bob!"},
           ])
           print(f"Ingested {resp.episode_count} episode(s)")

           # 2. Retrieve context for an LLM prompt
           ctx = await client.memory.get_context(
               query="What does Bob do?",
               limit=5,
           )
           print(ctx.context)

           # 3. List graph entities
           async for node in await client.graph.nodes():
               print(f"Entity: {node.name} ({node.type})")

   asyncio.run(main())

For the full SDK reference, see :doc:`/domains/sdk_python`.

Authentication with the SDK
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The SDK authenticates using an **API key**, not the JWT from the dashboard
signup flow.  Generate an API key after signing in to the dashboard:

.. code-block:: bash

   # Using the JWT from signup, create an API key scoped to your project
   curl -s -X POST "http://localhost:8000/v1/projects/${PROJECT_ID}/api-keys" \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name": "dev-key", "permissions": ["read", "write"]}' \
     | python3 -m json.tool

The response contains the ``key`` value — use it as ``api_key`` in the SDK.

.. note::

   The SDK's ``AsyncOpenZync`` client is a context manager.  It manages the
   HTTP connection pool and automatically authenticates every request with the
   ``Bearer`` token derived from your API key.  You never need to pass the key
   on individual calls.

-----------------------
Using the Dashboard
-----------------------

The OpenZync frontend provides a web dashboard for graph exploration, tenant
management, and configuration.  It runs separately from the backend stack.

Start the frontend:

.. code-block:: bash

   cd ../openzync-frontend
   docker compose -f deploy/docker-compose.yml up -d

The dashboard is available at ``http://localhost:3000``.

Login using the email and password from the signup step above.  After login,
the dashboard shows:

* **Overview** — org-level usage stats and health indicators.
* **Projects** — list of projects with member management.
* **Graph Explorer** — interactive knowledge graph visualisation with entity
  nodes, relationship edges, and community clusters.
* **Users** — user management within the organisation.
* **Settings** — LLM provider configuration, extraction schemas, classification
  templates, webhook endpoints, and prompt template manager.

.. note::

   For the dashboard to work, the backend API must be reachable at
   ``http://localhost:8000``.  The frontend proxy routes ``/v1/*`` requests
   to the API automatically.  See :doc:`/domains/frontend` for the full
   frontend documentation.

-----------------------
Development Setup
-----------------------

For local development with hot-reload (code changes trigger an automatic server
restart), run only the infrastructure in Docker and start the API locally:

.. code-block:: bash

   # Start only the supporting containers (OpenBao, Postgres, Redis, worker,
   # and the OpenBao Agent sidecars)
   docker compose -f infra/docker-compose.backend.yml up -d \
     openbao postgres redis worker openbao-agent-api

   # Wait for the bootstrap sequence to complete (~60s), then start the API
   # with hot-reload
   make dev

   # The API is now running at http://localhost:8000 with live reload.
   # Any change to a .py file triggers an automatic restart.

The local API process reads configuration from OpenBao (via the OpenBao Agent
sidecar) — just like the containerised API.  No additional configuration is
needed.

Useful Makefile targets for development:

.. list-table::
   :header-rows: 1

   * - Command
     - What it does
   * - ``make test``
     - Run unit tests only
   * - ``make test-all``
     - Run all tests (unit + integration + security)
   * - ``make lint``
     - Ruff check + format verification
   * - ``make lint-fix``
     - Auto-fix lint issues
   * - ``make migrate``
     - Apply pending Alembic migrations
   * - ``make migrate-new``
     - Auto-generate a new migration revision
   * - ``make docker-up``
     - Start the full backend stack
   * - ``make docker-down``
     - Stop the backend stack
   * - ``make docker-logs``
     - Tail logs from all containers
   * - ``make docker-reset``
     - Full reset — removes all volumes (OpenBao state, Postgres data, etc.)
   * - ``make docs-build``
     - Build the Sphinx documentation
   * - ``make docs-watch``
     - Serve documentation with live-reload on port 8600

.. code-block:: bash

   # Install development dependencies
   make install

   # Run a specific test by name
   make test ARGS="-k test_memory_ingest"

-----------------
.. _quickstart-troubleshooting:

Troubleshooting
-----------------

Issue: Port already in use
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If port 8000 (or 5432, 6379, 8200) is already allocated:

.. code-block:: bash

   # Check which service is using the port
   sudo lsof -i :8000

   # Stop the conflicting service, or update the port mapping in
   # infra/docker-compose.backend.yml and the API entrypoint.

Issue: Docker Compose fails with "required variable is unset"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``:?`` modifier in the compose file means every required env var must be
set.  Common causes:

* ``.env`` has placeholder values (``<replace-with-64-char-hex>``) instead of
  generated secrets.
* ``.env`` is missing one or more of the four required variables.

Fix:

.. code-block:: bash

   # Regenerate secrets for any placeholder values still present
   grep -n '<replace-with' .env
   # Replace as shown in Step 2 of Clone and Bootstrap

Issue: OpenBao seal key lost and OpenBao won't start
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If ``BAO_STATIC_SEAL_KEY`` is lost, **all secrets in OpenBao are
irrecoverable**.  The only option is a full reset:

.. code-block:: bash

   make docker-reset

This destroys all Docker volumes (OpenBao Raft data, PostgreSQL data, Redis
data).  You will need to re-bootstrap from scratch.

.. warning::

   ``make docker-reset`` deletes **all data**.  There is no recovery path.
   Back up ``BAO_STATIC_SEAL_KEY`` in a password manager or secrets vault.

Issue: API returns 503 on /v1/ready
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The readiness probe checks PostgreSQL and Redis connectivity.  If one or both
report ``false``:

.. code-block:: bash

   # Check the api logs for startup errors
   docker compose -f infra/docker-compose.backend.yml logs api

   # Check the Agent sidecar — the api waits for the rendered env file
   docker compose -f infra/docker-compose.backend.yml logs openbao-agent-api

   # Common cause: the Agent is still authenticating or rendering secrets.
   # The healthcheck allows up to 90s for this.  If it exceeds 90s, the
   # sidecar may be in a crash loop — check with:
   docker compose -f infra/docker-compose.backend.yml ps

Issue: Signup returns 429 (rate limited)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Auth endpoints are rate-limited per IP.  Wait 60 seconds before retrying.

Issue: Migration failures
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Check migration logs
   docker compose -f infra/docker-compose.backend.yml logs postgres-migrate

   # If the migration failed (e.g., Alembic revision conflict), the
   # bootstrap chain stops.  Fix the migration, then run:
   make docker-reset

   # Migrations can also be run manually against the bootstrapped database:
   make migrate

Administrative OpenBao access
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The root token is at ``/bao-init/root-token`` inside the ``openbao-init-data``
volume.  Access it with:

.. code-block:: bash

   docker compose -f infra/docker-compose.backend.yml run --rm openbao \
     bao login $(docker compose -f infra/docker-compose.backend.yml \
       run --rm --no-deps openbao cat /bao-init/root-token 2>/dev/null)

   # List the system secret
   docker compose -f infra/docker-compose.backend.yml exec openbao \
     bao kv get -namespace=system/ config/data/system

.. caution::

   The root token is powerful.  In production, set
   ``BAO_REVOKE_ROOT_TOKEN=true`` on the ``openbao-init`` service to revoke
   it after bootstrap.

--------
Next Steps
--------

* :doc:`/domains/infrastructure` — detailed deployment configuration (Helm
  chart, NGINX, observability stack).
* :doc:`/domains/auth` — authentication architecture, JWT and API key flows.
* :doc:`/domains/memory_context` — memory ingestion, context assembly, hybrid
  retrieval pipeline.
* :doc:`/domains/sdk_python` — Python SDK reference with all domain clients.
* :doc:`/domains/frontend` — dashboard setup, routes, and features.
* :doc:`/domains/graph_backends` — pluggable graph backends (PostgreSQL-native,
  FalkorDB, SurrealDB).
* :doc:`/domains/workers` — ARQ worker architecture, enrichment pipeline,
  job lifecycle.
* ``/docs`` endpoint on your running API — interactive Swagger UI for every
  endpoint.
