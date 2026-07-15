Infrastructure — Docker Compose, OpenBao, Observability & Helm
==============================================================

.. note::

   This document describes the **OpenZync backend infrastructure** as deployed
   via ``docker-compose.backend.yml`` and the Kubernetes Helm chart.  All paths
   are relative to ``openzync-core/infra/`` unless otherwise noted.

   The infrastructure follows a **zero-fallback secrets model**: OpenBao is the
   exclusive source of truth for all runtime configuration.  Environment
   variables are used only for bootstrap credentials.  See :doc:`/adr/003-openbao-zero-fallback`
   and :doc:`/adr/004-self-bootstrapping-postgres` for the architectural
   rationale.

.. contents:: Sections
   :local:
   :depth: 3
   :class: this-will-duplicate-information-and-it-is-still-useful-here

Quick Start
-----------

Bring up the entire stack from a cold start::

   # 1. Generate required secrets (or populate from infra/.env.example)
   export BAO_STATIC_SEAL_KEY=$(openssl rand -hex 32)
   export POSTGRES_PASSWORD=$(openssl rand -base64 32)
   export OZ_SECRET_KEY=$(python -c "import secrets; print(secrets.token_urlsafe(48))")
   export OZ_WEBHOOK_SIGNING_SECRET=$(python -c "import secrets; print(secrets.token_urlsafe(32))")

   # 2. Start all services
   docker compose -f infra/docker-compose.backend.yml up -d

   # 3. Tail logs until the bootstrap sequence completes
   docker compose -f infra/docker-compose.backend.yml logs -f

   # With local LLM inference (optional):
   docker compose -f infra/docker-compose.backend.yml --profile llm up -d

   # With observability (optional):
   docker compose -f infra/docker-compose.backend.yml --profile observability up -d

The api service is available at ``http://localhost:8000`` once the bootstrap
sequence completes (~60s on a cold start).

---

Docker Compose Backend Stack
----------------------------

The compose file at ``infra/docker-compose.backend.yml`` defines 14 services
across three Docker Compose profiles.

**Default profile** (always on)::

   openbao, openbao-init, postgres, postgres-init, postgres-migrate,
   openbao-write-db, openbao-agent-api, api, openbao-agent-worker,
   worker, redis

**Optional profiles**::

   --profile llm             ollama
   --profile observability   prometheus, grafana, alloy

Service Graph
~~~~~~~~~~~~~

The boot sequence is a strict eight-phase directed acyclic graph,
enforced by ``depends_on: condition: service_healthy`` and
``service_completed_successfully``::

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

Phases at a glance:

#. **Phase 1 — OpenBao bootstrap.**  ``openbao`` starts; ``openbao-init``
   initialises, unseals, mounts KV v2, writes policies, creates AppRoles,
   and enables the Transit engine.
#. **Phase 2 — System secrets (pre-DB).**  ``openbao-init`` writes all
   non-DB system config to ``system/config/data/system``.
#. **Phase 3 — AppRole credential files.**  ``openbao-init`` writes
   ``role_id`` and ``secret_id`` files for the api/worker Agent sidecars.
#. **Phase 4 — Postgres starts.**  ``postgres`` boots with pgvector.
#. **Phase 5 — DB users + least-privilege roles.**  ``postgres-init`` creates
   the ``openzync`` database, ``openzync_migrator`` (DDL), and
   ``openzync_app`` (CRUD) roles with auto-generated passwords.
#. **Phase 6 — Migrations.**  ``postgres-migrate`` runs ``alembic upgrade head``
   as the migrator user.
#. **Phase 7 — DB credentials into OpenBao.**  ``openbao-write-db`` reads
   ``db-creds.json``, constructs ``DATABASE_URL``, and merges it into the
   OpenBao system secret.
#. **Phase 8 — api + worker start.**  Each service's OpenBao Agent sidecar
   authenticates via AppRole and renders the system secret to a shared tmpfs
   volume.  The api/worker entrypoints source the rendered env file, then
   ``exec`` uvicorn / ARQ.

Logging defaults (all services)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: yaml

   driver: json-file
   options:
     max-size: "10m"
     max-file: "3"

All restart-loop services use ``restart: unless-stopped``.

Named Volumes
~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1

   * - Volume
     - Purpose
     - Mount points
   * - ``openbao-data``
     - Raft storage + seal state
     - ``/vault/data``
   * - ``openbao-init-data``
     - Shared bootstrap artefacts
     - ``/bao-init`` (writers), ``/openbao-bootstrap`` (readers)
   * - ``openbao-audit-logs``
     - JSON audit log
     - ``/vault/logs``
   * - ``postgres-data``
     - Postgres data directory
     - ``/var/lib/postgresql/data``
   * - ``api-secrets``
     - Shared render path for api sidecar → api app
     - ``/openbao/agent`` (sidecar), ``/run/secrets`` (api)
   * - ``worker-secrets``
     - Shared render path for worker sidecar → worker app
     - ``/openbao/agent`` (sidecar), ``/run/secrets`` (worker)
   * - ``redis-data``
     - Redis persistence
     - ``/data``
   * - ``ollama-data``
     - LLM model storage
     - ``/root/.ollama``

.. warning::

   ``api-secrets`` and ``worker-secrets`` are Docker named volumes (not
   ``tmpfs``) because Compose cannot express a shared ``tmpfs`` across two
   containers.  In production/Kubernetes, replace these with ``emptyDir``
   backed by memory (``medium: Memory``).

Services — Default Profile
~~~~~~~~~~~~~~~~~~~~~~~~~~

openbao
"""""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Image
     - ``openbao/openbao:2.5``
   * - Container name
     - ``openzync-openbao``
   * - Ports
     - ``127.0.0.1:8200:8200``
   * - Capabilities
     - ``IPC_LOCK`` (prevents seal memory from being swapped)
   * - Restart
     - ``unless-stopped``

.. list-table:: Environment
   :header-rows: 1

   * - Variable
     - Source
     - Purpose
   * - ``BAO_STATIC_SEAL_KEY``
     - ``${BAO_STATIC_SEAL_KEY:?...}``
     - 64-char hex key for auto-unseal (``openssl rand -hex 32``)
   * - ``BAO_ADDR``
     - ``http://0.0.0.0:8200``
     - Listen address

.. list-table:: Volumes
   :header-rows: 1

   * - Host/Volume
     - Mount
     - Mode
   * - ``openbao-data``
     - ``/vault/data``
     - rw
   * - ``./openbao/config.hcl``
     - ``/etc/openbao/config.hcl``
     - ro,z
   * - ``openbao-audit-logs``
     - ``/vault/logs``
     - rw

The healthcheck verifies OpenBao is initialized AND unsealed by checking
``bao status -format=json`` for ``"sealed":false`` (interval 5s, timeout 5s,
retries 10, start period 10s).

openbao-init
""""""""""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Image
     - ``openbao/openbao:2.5``
   * - Container name
     - ``openzync-openbao-init``
   * - Depends on
     - ``openbao: service_started``
   * - Restart
     - ``no`` (one-shot bootstrap)
   * - Entrypoint
     - ``/bin/sh /init_openbao.sh``

.. list-table:: Environment
   :header-rows: 1

   * - Variable
     - Source / Default
     - Purpose
   * - ``BAO_ADDR``
     - ``http://openbao:8200``
     - OpenBao address for the ``bao`` CLI
   * - ``BAO_SKIP_VERIFY``
     - ``true``
     - TLS skip for dev
   * - ``OZ_REDIS_URL``
     - ``redis://redis:6379/0``
     - Redis connection string
   * - ``OZ_SECRET_KEY``
     - **REQUIRED** (``${OZ_SECRET_KEY:?...}``)
     - App signing key (JWT, crypto)
   * - ``OZ_ENVIRONMENT``
     - ``development``
     - Deployment environment label
   * - ``OZ_CORS_ORIGINS``
     - ``http://localhost:3000``
     - Allowed CORS origins
   * - ``OZ_LOG_LEVEL``
     - ``INFO``
     - Log verbosity
   * - ``OZ_MAX_WORKERS``
     - ``4``
     - Uvicorn worker count
   * - ``OZ_JWT_ACCESS_TOKEN_TTL_MINUTES``
     - ``30``
     - JWT access token lifetime
   * - ``OZ_JWT_REFRESH_TOKEN_TTL_DAYS``
     - ``7``
     - JWT refresh token lifetime
   * - ``OZ_WEBHOOK_SIGNING_SECRET``
     - **REQUIRED**
     - HMAC-SHA256 webhook signing key
   * - ``OZ_PROMETHEUS_URL``
     - ``http://prometheus:9090``
     - Prometheus endpoint
   * - ``OZ_FALKORDB_URL``
     - ``redis://redis:6379``
     - FalkorDB connection string
   * - ``OZ_FALKORDB_MAX_CONNECTIONS``
     - ``20``
     - FalkorDB pool limit
   * - ``OZ_FALKORDB_SOCKET_TIMEOUT``
     - ``30``
     - FalkorDB socket timeout (seconds)
   * - ``OZ_RATE_LIMIT_IP_MAX``
     - ``10``
     - Max requests per IP per window
   * - ``OZ_RATE_LIMIT_WINDOW_SEC``
     - ``60``
     - Rate limit window (seconds)
   * - ``OZ_HOSTS_ALLOWED``
     - ``localhost:8000``
     - Allowed Host header values
   * - ``OZ_PROMPT_CACHING_ENABLED``
     - ``true``
     - Enable Anthropic prompt caching
   * - ``OZ_PROMPT_CACHING_ANTHROPIC_MIN_TOKENS``
     - ``1024``
     - Min tokens for caching
   * - ``OZ_PROMPT_CACHING_ANTHROPIC_TTL``
     - ``5m``
     - Cache TTL

.. list-table:: Volumes
   :header-rows: 1

   * - Host/Volume
     - Mount
     - Mode
   * - ``openbao-init-data``
     - ``/bao-init``
     - rw
   * - ``./openbao/policies``
     - ``/policies``
     - ro
   * - ``../scripts/init_openbao.sh``
     - ``/init_openbao.sh``
     - ro

This is an **idempotent** one-shot container.  It writes a marker file to
``/bao-init/init-complete`` on success; subsequent runs exit 0 immediately.

postgres
""""""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Image
     - ``pgvector/pgvector:pg15``
   * - Container name
     - ``openzync-postgres``
   * - Ports
     - ``127.0.0.1:5432:5432``
   * - Restart
     - ``unless-stopped``

.. list-table:: Environment
   :header-rows: 1

   * - Variable
     - Source / Default
     - Purpose
   * - ``POSTGRES_PASSWORD``
     - **REQUIRED** on first boot
     - Superuser password
   * - ``POSTGRES_USER``
     - ``postgres``
     - Superuser name
   * - ``POSTGRES_DB``
     - ``postgres``
     - Initial database

.. list-table:: Volumes
   :header-rows: 1

   * - Host/Volume
     - Mount
   * - ``postgres-data``
     - ``/var/lib/postgresql/data``

Healthcheck via ``pg_isready -U postgres -d postgres`` (interval 5s).

postgres-init
"""""""""""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Image
     - ``pgvector/pgvector:pg15``
   * - Container name
     - ``openzync-postgres-init``
   * - Depends on
     - ``postgres: service_healthy``
   * - Restart
     - ``no`` (one-shot bootstrap)
   * - Entrypoint
     - ``/bin/bash /init_postgres.sh``

Creates the ``openzync`` database and two least-privilege roles:

* ``openzync_migrator`` — DDL (used by Alembic)
* ``openzync_app`` — CRUD (used by api + worker at runtime)

Passwords are auto-generated with ``openssl rand -base64 32`` and written to
``/bao-init/db-creds.json`` (mode 0600).

.. list-table:: Environment
   :header-rows: 1

   * - Variable
     - Source / Default
     - Purpose
   * - ``POSTGRES_HOST``
     - ``postgres``
     - Hostname of the postgres container
   * - ``POSTGRES_PORT``
     - ``5432``
     - Postgres port
   * - ``DB_NAME``
     - ``openzync``
     - Target database name
   * - ``POSTGRES_SUPERUSER_PASSWORD``
     - ``${POSTGRES_PASSWORD}``
     - Superuser password (env-provided; optional, auto-rotates if absent)

postgres-migrate
""""""""""""""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Build context
     - ``..`` (monolith root)
   * - Dockerfile
     - ``services/api/Dockerfile``
   * - Image tag
     - ``openzync-api:latest``
   * - Container name
     - ``openzync-postgres-migrate``
   * - Depends on
     - ``postgres-init: service_completed_successfully``
   * - Restart
     - ``no`` (one-shot migration)
   * - Entrypoint
     - ``/bin/sh /entrypoint_migrate.sh``
   * - Command
     - ``alembic upgrade head``

Unlike the api and worker, the migration container reads the migrator password
directly from ``/bao-init/db-creds.json`` rather than through an OpenBao Agent
sidecar.  This keeps the migrator URL (which uses the DDL-privileged user)
separate from the app URL.

openbao-write-db
""""""""""""""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Image
     - ``openbao/openbao:2.5``
   * - Container name
     - ``openzync-openbao-write-db``
   * - Depends on
     - ``postgres-migrate: service_completed_successfully``
   * - Restart
     - ``no`` (one-shot bootstrap)
   * - Entrypoint
     - ``/bin/bash /write_db_to_openbao.sh``

Reads ``db-creds.json``, constructs ``DATABASE_URL``
(``postgresql+asyncpg://openzync_app:<pw>@postgres:5432/openzync``), reads the
existing system secret from OpenBao KV, **merges** the new key, and writes the
combined result back.  Uses CAS (Check-And-Set) on the OpenBao secret version
to prevent concurrent-overwrite races.

The merged system secret now contains every key the api/worker need
(``database_url``, ``secret_key``, ``redis_url``, etc.) as a single flat
object.

openbao-agent-api / openbao-agent-worker
""""""""""""""""""""""""""""""""""""""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Image
     - ``openbao/openbao:2.5``
   * - Container names
     - ``openzync-openbao-agent-api``, ``openzync-openbao-agent-worker``
   * - Depends on
     - ``openbao-write-db: service_completed_successfully``
   * - Restart
     - ``unless-stopped``
   * - Command
     - ``agent -config=/etc/bao/agent.hcl``

These sidecar containers run OpenBao Agent, which:

#. Authenticates to OpenBao via **AppRole** using the ``role_id`` and
   ``secret_id`` files written by ``openbao-init``.
#. Maintains a long-lived token with automatic renewal.
#. Using Consul Template syntax, reads the system secret from
   ``system/config/data/system`` and renders it to ``/openbao/agent/system.env``
   as a flat KEY=VALUE env file.
#. Re-renders every 5 minutes (``static_secret_render_interval``).

The healthcheck verifies that ``system.env`` exists, is non-empty, and contains
``DATABASE_URL=`` (interval 5s, retries 18, start period 10s).

The api and worker containers depend on their respective Agent sidecars with
``condition: service_healthy``.

.. list-table:: Volumes

   * - Mount
     - Purpose
   * - ``openbao-init-data:/openbao-bootstrap:ro``
     - AppRole credential files
   * - ``api-secrets:/openbao/agent:rw`` (api) / ``worker-secrets:/openbao/agent:rw`` (worker)
     - Rendered env file output
   * - ``./openbao/agent/api.hcl:/etc/bao/agent.hcl:ro`` (api) / ``./openbao/agent/worker.hcl:/etc/bao/agent.hcl:ro`` (worker)
     - Agent configuration

api
"""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Build context
     - ``..``
   * - Dockerfile
     - ``services/api/Dockerfile``
   * - Image
     - ``ghcr.io/rohnsha0/openzync/api:latest``
   * - Container name
     - ``openzync-api``
   * - Depends on
     - ``openbao-agent-api: service_healthy``, ``redis: service_healthy``
   * - Ports
     - ``127.0.0.1:8000:8000``
   * - Restart
     - ``unless-stopped``

.. list-table:: Resource limits
   :header-rows: 1

   * - Limit
     - Value
   * - CPU limit
     - 1 core
   * - CPU reservation
     - 0.5 cores
   * - Memory limit
     - 2 GB
   * - Memory reservation
     - 1 GB

.. list-table:: Environment
   :header-rows: 1

   * - Variable
     - Source
     - Purpose
   * - ``OZ_OPENBAO_ADDR``
     - ``http://openbao:8200``
     - OpenBao address for runtime org config access
   * - ``UVICORN_WORKERS``
     - ``1``
     - Uvicorn worker processes (entrypoint handles the rest)

The entrypoint (``/entrypoint_api.sh``, baked into the image) waits for
``/run/secrets/system.env`` (up to 90s), sources it with ``set -a`` to export
every variable, optionally reads AppRole credentials from
``/openbao-bootstrap/api-{role_id,secret_id}``, then ``exec``\ s uvicorn so
the process becomes PID 1 and receives signals directly.

worker
""""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Build context
     - ``..``
   * - Dockerfile
     - ``services/worker/Dockerfile``
   * - Image
     - ``ghcr.io/rohnsha0/openzync/worker:latest``
   * - Container name
     - ``openzync-worker``
   * - Depends on
     - ``openbao-agent-worker: service_healthy``, ``redis: service_healthy``
   * - Restart
     - ``unless-stopped``

.. list-table:: Resource limits
   :header-rows: 1

   * - Limit
     - Value
   * - CPU limit
     - 0.5 cores
   * - Memory limit
     - 1 GB

Same pattern as the api service: the entrypoint (``/entrypoint_worker.sh``)
waits for the rendered env file, sources it, then ``exec``\ s
``python -m services.worker.worker``.

redis
"""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Image
     - ``redis:7-alpine``
   * - Container name
     - ``openzync-redis``
   * - Ports
     - ``127.0.0.1:6379:6379``
   * - Restart
     - ``unless-stopped``

.. list-table:: Volumes
   :header-rows: 1

   * - Volume
     - Mount
   * - ``redis-data``
     - ``/data``

Healthcheck via ``redis-cli PING`` (interval 5s).

Services — LLM Profile
~~~~~~~~~~~~~~~~~~~~~~

ollama
""""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Profile
     - ``llm``
   * - Image
     - ``ollama/ollama:latest``
   * - Ports
     - ``11434:11434``
   * - Restart
     - ``unless-stopped``

Services — Observability Profile
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

prometheus
""""""""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Profile
     - ``observability``
   * - Image
     - ``prom/prometheus:v2.53.0``
   * - Ports
     - ``9090:9090``
   * - Restart
     - ``unless-stopped``

.. list-table:: Volumes
   :header-rows: 1

   * - Host/Volume
     - Mount
     - Mode
   * - ``./prometheus``
     - ``/etc/prometheus``
     - ro
   * - ``prometheus-data``
     - ``/prometheus``
     - rw

The ``extra_hosts`` entry ``host.docker.internal:host-gateway`` allows the
scrape target to reach the host's api port (``host.docker.internal:8000``).

grafana
"""""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Profile
     - ``observability``
   * - Image
     - ``grafana/grafana:11.0.0``
   * - Ports
     - ``127.0.0.1:3000:3000``
   * - Restart
     - ``unless-stopped``

.. list-table:: Environment
   :header-rows: 1

   * - Variable
     - Value
     - Purpose
   * - ``GF_AUTH_ANONYMOUS_ENABLED``
     - ``"true"``
     - Allow unauthenticated access (dev only)
   * - ``GF_AUTH_ANONYMOUS_ORG_ROLE``
     - ``"Viewer"``
     - Read-only role for anonymous users
   * - ``GF_SECURITY_ADMIN_PASSWORD``
     - ``admin``
     - Admin password (dev only)

.. list-table:: Volumes
   :header-rows: 1

   * - Host/Volume
     - Mount
     - Mode
   * - ``./grafana/datasources``
     - ``/etc/grafana/provisioning/datasources``
     - ro
   * - ``./grafana/dashboards``
     - ``/etc/grafana/provisioning/dashboards``
     - ro

alloy
"""""

.. list-table::
   :header-rows: 1

   * - Attribute
     - Value
   * - Profile
     - ``observability``
   * - Image
     - ``grafana/alloy:latest``
   * - Ports
     - ``4317:4317`` (OTLP gRPC), ``12345:12345`` (HTTP admin)
   * - Restart
     - ``unless-stopped``
   * - Command
     - ``run --server.http.listen-addr=0.0.0.0:12345 /etc/alloy/config.alloy``

.. list-table:: Volumes
   :header-rows: 1

   * - Host/Volume
     - Mount
     - Mode
   * - ``./alloy``
     - ``/etc/alloy``
     - ro

---

Environment Variables
---------------------

Bootstrap Environment
~~~~~~~~~~~~~~~~~~~~~

The following are the **only** environment variables the system reads directly
from the deployment environment.  All other config is loaded from OpenBao at
runtime.

.. list-table:: Required environment variables
   :header-rows: 1

   * - Variable
     - Required
     - Purpose
     - Generation
   * - ``BAO_STATIC_SEAL_KEY``
     - Yes
     - OpenBao auto-unseal key (AES-256-GCM)
     - ``openssl rand -hex 32`` (64 hex chars)
   * - ``POSTGRES_PASSWORD``
     - Yes (first boot)
     - Postgres superuser password for cluster init
     - ``openssl rand -base64 32``
   * - ``OZ_SECRET_KEY``
     - Yes
     - JWT signing and app crypto operations
     - ``python -c "import secrets; print(secrets.token_urlsafe(48))"``
   * - ``OZ_WEBHOOK_SIGNING_SECRET``
     - Yes
     - HMAC-SHA256 webhook signature (Svix-compatible)
     - ``python -c "import secrets; print(secrets.token_urlsafe(32))"``

.. list-table:: Optional environment variables
   :header-rows: 1

   * - Variable
     - Default
     - Purpose
   * - ``OZ_ENVIRONMENT``
     - ``development``
     - Deployment environment label
   * - ``OZ_CORS_ORIGINS``
     - ``http://localhost:3000``
     - Comma-separated CORS allowed origins
   * - ``OZ_LOG_LEVEL``
     - ``INFO``
     - Log verbosity (DEBUG, INFO, WARNING, ERROR)
   * - ``OZ_MAX_WORKERS``
     - ``4``
     - Uvicorn worker count
   * - ``OZ_JWT_ACCESS_TOKEN_TTL_MINUTES``
     - ``30``
     - JWT access token expiry
   * - ``OZ_JWT_REFRESH_TOKEN_TTL_DAYS``
     - ``7``
     - JWT refresh token expiry
   * - ``OZ_PROMETHEUS_URL``
     - ``http://prometheus:9090``
     - Prometheus remote write endpoint
   * - ``OZ_FALKORDB_URL``
     - ``redis://redis:6379``
     - FalkorDB connection string
   * - ``OZ_FALKORDB_MAX_CONNECTIONS``
     - ``20``
     - FalkorDB connection pool size
   * - ``OZ_FALKORDB_SOCKET_TIMEOUT``
     - ``30``
     - FalkorDB socket timeout (seconds)
   * - ``OZ_RATE_LIMIT_IP_MAX``
     - ``10``
     - Max requests per IP per window
   * - ``OZ_RATE_LIMIT_WINDOW_SEC``
     - ``60``
     - Rate limit time window (seconds)
   * - ``OZ_HOSTS_ALLOWED``
     - ``localhost:8000``
     - Space-separated allowed Host header values
   * - ``OZ_PROMPT_CACHING_ENABLED``
     - ``true``
     - Enable Anthropic prompt caching
   * - ``OZ_PROMPT_CACHING_ANTHROPIC_MIN_TOKENS``
     - ``1024``
     - Minimum tokens to trigger caching
   * - ``OZ_PROMPT_CACHING_ANTHROPIC_TTL``
     - ``5m``
     - Prompt cache TTL

System Secret (OpenBao KV)
~~~~~~~~~~~~~~~~~~~~~~~~~~

The optional env vars above are written into the OpenBao system secret at
``system/config/data/system`` (first by ``init_openbao.sh``, then merged with
``database_url`` by ``write_db_to_openbao.sh``).  The env var keyword mapping
in ``init_openbao.sh`` converts ``OZ_*`` uppercase names to lowercase underscore
keys:

.. list-table:: OpenBao system secret keys
   :header-rows: 1

   * - Env var
     - Secret key
     - Written by
   * - ``OZ_REDIS_URL``
     - ``redis_url``
     - init_openbao.sh
   * - ``OZ_SECRET_KEY``
     - ``secret_key``
     - init_openbao.sh
   * - ``OZ_PROMETHEUS_URL``
     - ``prometheus_url``
     - init_openbao.sh
   * - ``OZ_CORS_ORIGINS``
     - ``cors_origins``
     - init_openbao.sh
   * - ``OZ_HOSTS_ALLOWED``
     - ``hosts_allowed``
     - init_openbao.sh
   * - ``OZ_ENVIRONMENT``
     - ``environment``
     - init_openbao.sh
   * - ``OZ_LOG_LEVEL``
     - ``log_level``
     - init_openbao.sh
   * - ``OZ_MAX_WORKERS``
     - ``max_workers``
     - init_openbao.sh
   * - ``OZ_JWT_ACCESS_TOKEN_TTL_MINUTES``
     - ``jwt_access_token_ttl_minutes``
     - init_openbao.sh
   * - ``OZ_JWT_REFRESH_TOKEN_TTL_DAYS``
     - ``jwt_refresh_token_ttl_days``
     - init_openbao.sh
   * - ``OZ_WEBHOOK_SIGNING_SECRET``
     - ``webhook_signing_secret``
     - init_openbao.sh
   * - ``OZ_FALKORDB_URL``
     - ``falkordb_url``
     - init_openbao.sh
   * - ``OZ_FALKORDB_MAX_CONNECTIONS``
     - ``falkordb_max_connections``
     - init_openbao.sh
   * - ``OZ_FALKORDB_SOCKET_TIMEOUT``
     - ``falkordb_socket_timeout``
     - init_openbao.sh
   * - ``OZ_RATE_LIMIT_IP_MAX``
     - ``rate_limit_ip_max``
     - init_openbao.sh
   * - ``OZ_RATE_LIMIT_WINDOW_SEC``
     - ``rate_limit_window_sec``
     - init_openbao.sh
   * - ``OZ_PROMPT_CACHING_ENABLED``
     - ``prompt_caching_enabled``
     - init_openbao.sh
   * - ``OZ_PROMPT_CACHING_ANTHROPIC_MIN_TOKENS``
     - ``prompt_caching_anthropic_min_tokens``
     - init_openbao.sh
   * - ``OZ_PROMPT_CACHING_ANTHROPIC_TTL``
     - ``prompt_caching_anthropic_ttl``
     - init_openbao.sh
   * - *(constructed)*
     - ``database_url``
     - write_db_to_openbao.sh

---

OpenBao Configuration
---------------------

OpenBao is the system's **sole source of truth** for all runtime configuration.
The stack uses:

* **KV v2** at ``system/config/`` for flat system-level config and per-org
  config (``org_<uuid>/config/``).
* **AppRole** auth at ``auth/approle/`` for machine-to-machine authentication.
* **Transit** engine at ``transit/`` for server-side encryption/decryption of
  sensitive org data (API keys, webhook secrets).

Server Config
~~~~~~~~~~~~~

File: ``infra/openbao/config.hcl``

.. code-block:: hcl

   storage "raft" {
     path    = "/vault/data"
     node_id = "node1"
   }

   listener "tcp" {
     address       = "0.0.0.0:8200"
     tls_disable   = true       # dev only — enable TLS in production
   }

   seal "static_kv" {}          # reads BAO_STATIC_SEAL_KEY env var

   audit "file" {
     path    = "/vault/logs/audit.log"
     format  = "json"
   }

   api_addr     = "http://0.0.0.0:8200"
   cluster_addr = "https://0.0.0.0:8201"
   log_level    = "info"

Key points:

* **Raft storage** with a single node for dev.  Production should use a Raft
  cluster (3 or 5 nodes).
* **Static KV seal** reads the ``BAO_STATIC_SEAL_KEY`` environment variable
  (64-char hex).  The container's ``IPC_LOCK`` capability prevents the seal key
  from being swapped to disk.
* **File audit log** records every authenticated request in JSON format.
  Mount ``openbao-audit-logs`` as a persistent volume in production.
* ``tls_disable = true`` is **development only**.  Production must enable TLS
  and use a proper seal mechanism (Shamir, cloud KMS, or HSM).

AppRole Configuration
~~~~~~~~~~~~~~~~~~~~~

Two AppRole identities are created during bootstrap:

.. list-table:: AppRole identities
   :header-rows: 1

   * - AppRole
     - Bound Policy
     - Token TTL
     - Max TTL
     - Used by
   * - ``openzync-app``
     - ``openzync-app``
     - 24h
     - 72h
     - api service (via openbao-agent-api)
   * - ``openzync-worker``
     - ``openzync-worker``
     - 72h
     - 168h
     - worker service (via openbao-agent-worker)

ACL Policies
~~~~~~~~~~~~

openzync-app (``infra/openbao/policies/openzync-app.hcl``)
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

Grants the API server permission to:

* Read system-level config from the ``system/`` namespace.
* Create, read, update, and delete org-level namespaces.
* Enable secrets engines within org namespaces.
* Read and write org-level config keys.
* Encrypt, decrypt, and rewrap with any Transit key (but **not** create or
  delete keys).

Key paths:

.. code-block:: hcl

   path "system/config/data/*"              { capabilities = ["read", "list"] }
   path "sys/namespaces/*"                  { capabilities = ["create", "read", "update", "delete", "list"] }
   path "+/config/data/*"                   { capabilities = ["create", "read", "update", "delete", "list"] }
   path "transit/encrypt/*"                 { capabilities = ["create", "update"] }
   path "transit/decrypt/*"                 { capabilities = ["create", "update"] }
   path "transit/keys/*"                    { capabilities = ["read", "list"] }

openzync-worker (``infra/openbao/policies/openzync-worker.hcl``)
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

Grants the ARQ worker permission to:

* Read system-level config.
* Read org-level config keys (read-only — workers never write config).
* Decrypt with Transit keys (but **not** encrypt).

Key paths:

.. code-block:: hcl

   path "system/config/data/*"              { capabilities = ["read", "list"] }
   path "+/config/data/*"                   { capabilities = ["read", "list"] }
   path "transit/decrypt/*"                 { capabilities = ["create", "update"] }

Transit Engine
~~~~~~~~~~~~~~

Three AES256-GCM96 encryption keys are created during bootstrap:

.. list-table:: Transit keys
   :header-rows: 1

   * - Key name
     - Purpose
     - Write access
     - Read access
   * - ``org-api-key``
     - Encrypts org-level LLM/embedding API keys
     - api
     - worker
   * - ``webhook-secret``
     - Encrypts webhook signing secrets
     - api
     - worker
   * - ``pii-encryption``
     - Encrypts PII data at rest
     - api
     - worker

Each encrypt/decrypt operation uses the caller's UUID as additional
authenticated data (AAD), so ciphertext from one org context cannot be
decrypted by another.

Agent Configuration
~~~~~~~~~~~~~~~~~~~

api.hcl (``infra/openbao/agent/api.hcl``)
"""""""""""""""""""""""""""""""""""""""""

The api Agent sidecar authenticates as ``openzync-app`` and renders the
system secret to ``/openbao/agent/system.env``.

Key configuration:

.. code-block:: hcl

   auto_auth {
     method "approle" {
       mount_path = "auth/approle"
       config = {
         role_id_file_path                    = "/openbao-bootstrap/api-role_id"
         secret_id_file_path                  = "/openbao-bootstrap/api-secret_id"
         remove_secret_id_file_after_reading   = true
       }
     }
   }

   template {
     destination          = "/openbao/agent/system.env"
     perms                = "0600"
     error_on_missing_key = true
     contents = <<EOT
   {{- with secret "system/config/data/system" -}}
   {{- range $k, $v := .Data.data }}
   {{ $k | upper }}={{ $v }}
   {{ end -}}
   {{- end }}
   EOT
   }

   vault {
     address = "http://openbao:8200"
     retry {
       backoff     = "exponential"
       max_retries = 10
     }
   }

Security properties:

* ``secret_id`` file deleted after first successful read.
* ``system.env`` is written with mode 0600.
* ``error_on_missing_key`` causes the Agent to fail render if the template
  references a missing key.
* ``exit_on_retry_failure`` terminates the Agent if it cannot reach OpenBao
  after all retries.
* Tokens auto-renew via periodic AppRole login.

worker.hcl (``infra/openbao/agent/worker.hcl``)
"""""""""""""""""""""""""""""""""""""""""""""""

Same pattern as api.hcl, but authenticates as ``openzync-worker`` using
``worker-role_id`` and ``worker-secret_id``.

---

Bootstrap Scripts
-----------------

``scripts/init_openbao.sh``
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Idempotent one-shot bootstrap for OpenBao.  Triggered by the
``openbao-init`` Compose service.  Flow:

#. Checks for marker file ``/bao-init/init-complete`` — exits 0 if present.
#. Waits for OpenBao to be reachable (``bao status`` returning valid JSON,
   max 60 attempts / 2s interval).
#. Determines if OpenBao is already initialised via ``bao status -format=json``.
#. **First run path:**
   a. ``bao operator init`` — 5 key shares, 3 key threshold.
   b. Saves unseal keys to ``/bao-init/unseal-keys.json`` (mode 0600).
   c. Saves root token to ``/bao-init/root-token`` (mode 0600).
   d. Unseals with 3 of 5 keys.
#. **Re-run path (already initialised):**
   a. Unseals from saved keys + root token.
#. Authenticates with root token.
#. Creates ``system`` namespace.
#. Enables KV v2 at ``system/config``.
#. Writes combined system secret (all ``OZ_*`` env vars minus ``DATABASE_URL``)
   to ``system/config/data/system`` via a Python subprocess (args are passed
   as separate argv items to avoid shell interpolation of special characters).
#. Enables AppRole auth.
#. Writes ACL policies from ``/policies/*.hcl``.
#. Creates ``openzync-app`` and ``openzync-worker`` AppRoles.
#. Enables Transit engine and creates the three encryption keys.
#. Retrieves AppRole credentials via ``bao read`` / ``bao write -f``.
#. Writes credential files to ``/bao-init/``:
   ``api-role_id``, ``api-secret_id``, ``worker-role_id``, ``worker-secret_id``.
   All mode 0600, written via ``printf '%s'`` to avoid newline.
#. Optionally revokes root token (if ``BAO_REVOKE_ROOT_TOKEN=true`` — not
   the default).
#. Writes ``/bao-init/init-complete`` marker.

``scripts/init_postgres.sh``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Idempotent one-shot bootstrap for PostgreSQL.  Triggered by the
``postgres-init`` Compose service.  Flow:

#. Waits for postgres to be reachable (``pg_isready``, max 60 attempts / 2s).
#. If ``POSTGRES_SUPERUSER_PASSWORD`` is set, uses it as-is (no rotation).
   Otherwise, auto-generates a 32-byte base64 password and rotates the
   postgres superuser.
#. Always generates fresh 32-byte base64 passwords for ``openzync_migrator``
   and ``openzync_app`` (rotation-friendly).
#. Checks if the ``openzync`` database already exists.
#. Rotates superuser password if auto-generated.
#. Creates the ``openzync`` database (first run only).
#. Creates or alters ``openzync_migrator`` and ``openzync_app`` roles.
#. Applies GRANTs:
   * Migrator: ``ALL ON DATABASE``, ``ALL ON SCHEMA public``, default
     privileges on tables and sequences.
   * App: ``CONNECT``, ``USAGE``, ``SELECT/INSERT/UPDATE/DELETE`` on all
     tables, ``USAGE/SELECT`` on all sequences, plus default privileges
     **for objects created by the migrator role**.
#. Writes credentials JSON to ``/bao-init/db-creds.json`` (mode 0600, built
   safely via Python ``os.open`` with atomically-set mode).
#. Clears all passwords from shell memory (``unset``).

``scripts/write_db_to_openbao.sh``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Idempotent one-shot script that merges the auto-generated ``DATABASE_URL``
into the OpenBao system secret.  Triggered by the ``openbao-write-db`` Compose
service.  Flow:

#. Checks for marker file ``/bao-init/db-creds-written`` — exits 0 if present.
#. Waits for ``db-creds.json`` (up to 60 attempts / 2s).
#. Waits for ``root-token`` (up to 60 attempts / 2s).
#. Waits for OpenBao reachable + initialised + unsealed.
#. Authenticates with root token.
#. Reads ``db-creds.json``, constructs ``DATABASE_URL``
   (``postgresql+asyncpg://openzync_app:<pw>@postgres:5432/openzync``).
#. Reads the existing system secret from OpenBao
   (``bao kv get -format=json system/config/data/system``).
#. Merges ``database_url`` into the existing data.
#. Writes the merged secret back with **CAS** (check-and-set) using the
   current version — prevents concurrent-overwrite races.
#. **Verifies** by reading back the secret and asserting ``database_url`` is
   present and matches.
#. Writes ``/bao-init/db-creds-written`` marker.

``scripts/entrypoint_api.sh``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Runs as PID 1 in the ``api`` container.  Flow:

#. Waits for ``/run/secrets/system.env`` to exist and contain
   ``DATABASE_URL=`` (up to 90s, 1s intervals).
#. Times out with FATAL if the file never appears.
#. Sources the env file with ``set -a`` (auto-export).
#. Reads AppRole credentials from ``/openbao-bootstrap/api-role_id`` and
   ``/openbao-bootstrap/api-secret_id`` as fallback for ``OZ_OPENBAO_ROLE_ID``
   / ``OZ_OPENBAO_SECRET_ID`` (these are not part of the system secret).
#. Logs redacted ``DATABASE_URL`` setup confirmation.
#. ``exec``\ s ``uvicorn services.api.asgi:app`` — becomes PID 1.

``scripts/entrypoint_worker.sh``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Same pattern as the api entrypoint, but:

* Reads ``/openbao-bootstrap/worker-role_id`` / ``worker-secret_id``.
* ``exec``\ s ``python -m services.worker.worker``.

``scripts/entrypoint_migrate.sh``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Runs as PID 1 in the ``postgres-migrate`` container.  Unlike the api/worker,
this does **not** use an OpenBao Agent sidecar.  Flow:

#. Waits for ``/bao-init/db-creds.json`` to appear (up to 60s).
#. Constructs ``DATABASE_URL`` using **migrator** credentials (not app
   credentials) via a Python heredoc — never interpolates passwords through
   the shell (defence-in-depth).
#. Exports ``DATABASE_URL``.
#. ``exec``\ s the CMD (typically ``alembic upgrade head``).

---

NGINX Configuration
-------------------

The ``infra/nginx/`` directory contains three files:

* ``nginx.conf`` — base configuration
* ``conf.d/openzync.conf`` — HTTP reverse proxy (IP-based, no TLS)
* ``conf.d/openzync.ssl.conf`` — HTTPS reverse proxy with Cloudflare origin
  certificate

Base Configuration (nginx.conf)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: nginx

   user nginx;
   worker_processes auto;
   worker_connections 1024;
   client_max_body_size 10M;

Features:

* ``worker_processes auto`` — lets nginx scale to available CPU cores.
* ``sendfile``, ``tcp_nopush``, ``tcp_nodelay`` — optimised file serving and
  latency.
* ``server_tokens off`` — hides nginx version from response headers.
* ``gzip`` enabled for text content types (JSON, JS, CSS, SVG) with
  compression level 6 and minimum length 256 bytes.
* Custom ``main_ext`` log format includes ``$http_x_forwarded_for`` and
  ``$request_id``.

HTTP Proxy (openzync.conf)
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: nginx

   server {
       listen 80;
       server_name _;   # Responds on any IP/domain

       set $api_upstream "http://api:8000";
       set $frontend_upstream "http://frontend:3000";

       location /v1/ { proxy_pass $api_upstream; ... }
       location /docs { proxy_pass $api_upstream/docs; ... }
       location /openapi.json { proxy_pass $api_upstream/openapi.json; ... }
       location /health { access_log off; proxy_pass $api_upstream; ... }
       location / { proxy_pass $frontend_upstream; ... }
   }

Routing:

* ``/v1/*``, ``/docs``, ``/openapi.json``, ``/health`` → FastAPI backend
  (api:8000)
* All other routes → Next.js frontend (frontend:3000)

Important: ``proxy_pass`` uses **variables** (``$api_upstream``,
``$frontend_upstream``) so nginx re-resolves DNS at runtime.  Fixed-string
``proxy_pass`` resolves once at startup and caches the IP forever — container
IP changes after redeploy would cause 502 errors.

The resolver is set to ``127.0.0.11`` (Docker's embedded DNS) with
``valid=10s``.

The ``/v1/`` location is configured for WebSocket upgrades (``proxy_http_version
1.1``, ``proxy_set_header Upgrade $http_upgrade``, ``Connection "upgrade"``)
to support SSE streaming from the API.

HTTPS Proxy (openzync.ssl.conf)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Same routing as the HTTP config, but:

* Listens on port 443 with SSL.
* Uses Cloudflare Origin Certificate from ``/etc/nginx/certs/``.
* TLSv1.2 and TLSv1.3 only, with a curated cipher suite.
* Includes ``Strict-Transport-Security`` header (max-age 31536000).

```{note}
The SSL server block is only activated when a domain name is configured
(currently ``app.openzync.tech``).  To use it, deploy the Cloudflare Origin CA
certificate and key to ``/etc/nginx/certs/`` on the host.
```

---

Observability Stack
-------------------

Prometheus
~~~~~~~~~~

Configuration: ``infra/prometheus/prometheus.yml``

.. code-block:: yaml

   global:
     scrape_interval: 15s
     evaluation_interval: 15s

   scrape_configs:
     - job_name: "openzync-api"
       metrics_path: /metrics
       static_configs:
         - targets: ["host.docker.internal:8000"]
           labels:
             job: "openzync-api"

Scrapes the API's ``/metrics`` endpoint (FastAPI + Prometheus metrics) every
15 seconds.  Uses ``host.docker.internal`` to reach the host from a container
on Docker Desktop (macOS/Windows).  On Linux, change the target to
``api:8000``.

Grafana
~~~~~~~

Datasource provisioning file: ``infra/grafana/datasources/alloy.yaml``

.. code-block:: yaml

   datasources:
     - name: Prometheus
       type: prometheus
       url: http://prometheus:9090
       access: proxy
       isDefault: true
       editable: false

Dashboard provisioning file: ``infra/grafana/dashboards/dashboards.yaml``

.. code-block:: yaml

   providers:
     - name: "OpenZync"
       folder: "OpenZync"
       type: file
       options:
         path: /etc/grafana/provisioning/dashboards

The auto-loaded dashboard is ``OpenZync-overview.json``
(``infra/grafana/dashboards/OpenZync-overview.json``), which includes:

.. list-table:: Dashboard panels
   :header-rows: 1

   * - Panel
     - Type
     - Description
   * - API Request Rate & Error Rate
     - Time series
     - HTTP request rate by status code, with a 5xx error rate overlay
   * - Context Latency Percentiles
     - Time series
     - p50, p95, p99 latency for warm/cold context retrieval
   * - Worker Queue Depth
     - Time series
     - High- and low-priority ARQ queue depths
   * - LLM Token Usage
     - Bar chart
     - Cumulative token consumption by model
   * - Graph Node Growth
     - Time series
     - Total knowledge graph node count
   * - Service Health
     - Row + Stat panels
     - Up/down status for API, Worker, MCP, PostgreSQL, Redis, FalkorDB

In production, point the Grafana datasource to Mimir (metrics), Tempo (traces),
and Loki (logs) instead of the local Prometheus.

Grafana Alloy
~~~~~~~~~~~~~

Configuration: ``infra/alloy/config.alloy``

Alloy runs as an OpenTelemetry collector.  In development mode, it scrapes
the API's ``/metrics`` endpoint and performs a Prometheus remote write to the
local Prometheus instance:

.. code-block:: alloy

   prometheus.scrape "openzync_api" {
     targets = [
       { __address__ = "host.docker.internal:8000", job = "openzync-api" },
     ]
     scrape_interval = "15s"
     metrics_path    = "/metrics"
     forward_to      = [prometheus.remote_write.default.receiver]
   }

   prometheus.remote_write "default" {
     endpoint {
       url = "http://prometheus:9090/api/v1/write"
     }
   }

In production, Alloy is the single ingestion point for:

* **Metrics** — scraped from api/worker and remote-written to Mimir.
* **Logs** — collected from container stdout and forwarded to Loki.
* **Traces** — received via OTLP gRPC on port 4317 and forwarded to Tempo.

Alloy exposes a health/administration interface on port 12345.

---

Helm Chart
----------

Structure
~~~~~~~~~

The Helm chart is at ``infra/helm/openzync/``::

   helm/openzync/
   ├── Chart.yaml
   ├── values.yaml
   └── templates/
       ├── _helpers.tpl
       ├── configmap.yaml
       ├── deployment-api.yaml
       ├── deployment-worker.yaml
       ├── hpa.yaml
       ├── ingress.yaml
       ├── pvc.yaml
       ├── secret.yaml
       └── service.yaml

Chart.yaml
~~~~~~~~~~

.. code-block:: yaml

   apiVersion: v2
   name: openzync
   description: OpenZync — Long-term memory for AI agents (Helm chart)
   type: application
   version: 0.1.0
   appVersion: "1.0.0"
   kubeVersion: ">=1.25.0-0"
   home: https://openzync.tech

values.yaml (Key Sections)
~~~~~~~~~~~~~~~~~~~~~~~~~~

API service
"""""""""""

.. code-block:: yaml

   api:
     replicaCount: 2
     image:
       repository: openzync-api
       tag: latest
     containerPort: 8000
     autoscaling:
       enabled: true
       minReplicas: 2
       maxReplicas: 10
       targetCPUUtilizationPercentage: 70
       targetMemoryUtilizationPercentage: 80
     ingress:
       enabled: false
       host: api.openzync.tech
       tls:
         enabled: true
         secretName: openzync-tls

Worker service
""""""""""""""

.. code-block:: yaml

   worker:
     replicaCount: 1
     image:
       repository: openzync-worker
       tag: latest
     autoscaling:
       enabled: true
       minReplicas: 1
       maxReplicas: 5
       targetCPUUtilizationPercentage: 75

PostgreSQL
""""""""""

.. code-block:: yaml

   postgresql:
     enabled: true
     image: pgvector/pgvector:pg15
     database: openzync
     persistence:
       enabled: true
       size: 20Gi

Redis
"""""

.. code-block:: yaml

   redis:
     enabled: true
     image: redis:7-alpine
     persistence:
       enabled: true
       size: 5Gi

Application config
""""""""""""""""""

.. code-block:: yaml

   config:
     llmBackend: "ollama"
     llmModel: "llama3.2:3b"
     embeddingDim: 768
     graphBackend: "postgres"
     otlpEndpoint: "http://alloy:4317"

Secrets
"""""""

.. code-block:: yaml

   secrets:
     databaseUrl: ""
     redisUrl: ""
     secretKey: ""
     openaiApiKey: ""
     # ...

Secrets are not inlined in values.yaml; they are injected via ``--set``, an
external values file, or the ``external-secrets`` operator.

Templates
~~~~~~~~~

``_helpers.tpl`` — defines ``openzync.name``, ``openzync.fullname``,
``openzync.labels``, and ``openzync.selectorLabels``.

``configmap.yaml`` — creates a ConfigMap with non-sensitive configuration:
``OZ_ENVIRONMENT``, ``OZ_LOG_LEVEL``, ``OZ_LLM_BACKEND``, ``OZ_LLM_MODEL``,
``OZ_EMBEDDING_DIM``, ``OZ_GRAPH_BACKEND``, JWT TTLs, rate limits, CORS
origins, and OTLP endpoint.

``secret.yaml`` — conditionally creates a Secret for sensitive values (set
via ``.Values.secrets.*``):
``OZ_DATABASE_URL``, ``OZ_REDIS_URL``, ``OZ_FALKORDB_URL``, ``OZ_SECRET_KEY``,
``OPENAI_API_KEY``, ``OPENROUTER_API_KEY``, ``ANTHROPIC_API_KEY``.

``deployment-api.yaml`` — api Deployment with:
* ConfigMap and optional Secret injected as environment.
* Liveness probe: ``GET /health`` on port 8000.
* Readiness probe: ``GET /ready`` on port 8000.

``deployment-worker.yaml`` — worker Deployment with:
* Liveness probe: Python command that pings Redis (``redis.from_url(...).ping()``).
* No readiness probe (worker is a queue consumer, not an HTTP service).

``hpa.yaml`` — HorizontalPodAutoscaler for api (CPU + memory) and worker
(CPU only).  Conditional on ``.Values.api.autoscaling.enabled`` /
``.Values.worker.autoscaling.enabled``.

``service.yaml`` — ClusterIP service on port 80 → ``api.containerPort``.

``ingress.yaml`` — Ingress resource with optional cert-manager TLS annotation.
Conditional on ``.Values.api.ingress.enabled``.

``pvc.yaml`` — PersistentVolumeClaim for PostgreSQL and Redis.  Conditional
on ``.Values.postgresql.persistence.enabled`` /
``.Values.redis.persistence.enabled``.

Deploying with Helm
~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   # Package the chart
   helm package infra/helm/openzync/

   # Deploy with inline secrets
   helm upgrade --install openzync infra/helm/openzync/ \
     --set secrets.databaseUrl="postgresql+asyncpg://..." \
     --set secrets.redisUrl="redis://..." \
     --set secrets.secretKey="<your-secret-key>" \
     --set secrets.openaiApiKey="<key>" \
     --set config.llmBackend="openai" \
     --set config.llmModel="gpt-4o"

   # Deploy with a custom values file
   helm upgrade --install openzync infra/helm/openzync/ \
     -f my-production-values.yaml

   # Enable ingress with TLS
   helm upgrade --install openzync infra/helm/openzync/ \
     --set api.ingress.enabled=true \
     --set api.ingress.host="api.openzync.tech"

---

Makefile Targets
----------------

The ``Makefile`` at the monolith root provides the following infrastructure-
related targets:

.. list-table:: Makefile infrastructure targets
   :header-rows: 1

   * - Target
     - Command
     - Purpose
   * - ``make dev``
     - ``uvicorn services.api.asgi:app --reload --port 8000``
     - Start the API server locally (hot-reload)
   * - ``make docker-up``
     - ``docker compose -f infra/docker-compose.backend.yml up -d``
     - Start all backend infrastructure containers
   * - ``make docker-down``
     - ``docker compose -f infra/docker-compose.backend.yml down``
     - Stop infrastructure containers
   * - ``make docker-logs``
     - ``docker compose -f infra/docker-compose.backend.yml logs -f``
     - Tail logs from all containers
   * - ``make docker-reset``
     - ``docker compose -f infra/docker-compose.backend.yml down -v && up -d``
     - Full reset (removes volumes, including OpenBao state and Postgres data)
   * - ``make migrate``
     - ``alembic upgrade head``
     - Apply pending Alembic migrations (local, not inside Docker)
   * - ``make migrate-check``
     - ``alembic check``
     - Check if migrations are up-to-date
   * - ``make migrate-new``
     - ``alembic revision --autogenerate -m "<name>"``
     - Auto-generate a new migration revision
   * - ``make migrate-downgrade``
     - ``alembic downgrade -1``
     - Roll back the last migration
   * - ``make install``
     - ``pip install -e ".[dev]" && pre-commit install``
     - Install dev dependencies and pre-commit hooks
   * - ``make lint``
     - ``ruff check . && ruff format --check .``
     - Run ruff linter and formatter check
   * - ``make lint-fix``
     - ``ruff check --fix . && ruff format .``
     - Auto-fix lint issues and format
   * - ``make test``
     - ``pytest tests/unit/ -v``
     - Run unit tests only
   * - ``make test-all``
     - ``pytest tests/ -v``
     - Run all tests (unit + integration + security)
   * - ``make test-coverage``
     - ``pytest tests/unit/ --cov=... --cov-report=term --cov-report=html``
     - Run unit tests with coverage report
   * - ``make test-integration``
     - ``pytest tests/integration/ -v --timeout=60``
     - Run integration tests (requires running services)
   * - ``make benchmark``
     - ``pytest tests/benchmarks/ --run-benchmark -v``
     - Run LongMemEval benchmarks
   * - ``make docs-build``
     - ``sphinx-build -b html docs/ docs/_build/html``
     - Build Sphinx documentation
   * - ``make docs-watch``
     - ``sphinx-autobuild docs/ docs/_build/html --port 8600``
     - Serve docs with live-reload on port 8600
   * - ``make clean``
     - ``find ... -delete; rm -rf .pytest_cache ...``
     - Remove build artefacts and cache directories

---

Production Considerations
-------------------------

Secrets
~~~~~~~

* Replace the static KV seal with **Shamir** (5-of-3) or a **cloud KMS**
  (AWS KMS, GCP CKMS, Azure Key Vault).
* Enable **TLS** on OpenBao — remove ``tls_disable = true`` from
  ``config.hcl``.
* Set ``BAO_REVOKE_ROOT_TOKEN=true`` in ``openbao-init`` to revoke the root
  token after bootstrap.
* Deploy the OpenBao audit log volume with a persistent storage class.

OpenBao High Availability
~~~~~~~~~~~~~~~~~~~~~~~~~

* Deploy a Raft cluster with 3 or 5 OpenBao nodes.
* Place the Raft storage on persistent volumes with a suitable storage class.
* Configure performance standby nodes for read scaling.

Networking
~~~~~~~~~~

* Replace ``api-secrets`` and ``worker-secrets`` named volumes with ``tmpfs``
  mounts (Kubernetes ``emptyDir.medium: Memory``).
* Switch NGINX scrape target to ``api:8000`` (internal Docker DNS) instead of
  ``host.docker.internal``.
* Configure proper DNS and TLS certificates for the NGINX SSL server block.

Observability
~~~~~~~~~~~~~

* Replace the dev-mode Prometheus + Grafana with a production observability
  stack: **Mimir** (metrics), **Tempo** (traces), **Loki** (logs).
* Configure Alloy as the single OTLP ingestion point for all three signals.
* Set long-term retention policies and remote storage for Prometheus metrics.
* Enable Grafana authentication (disable anonymous access).

PostgreSQL
~~~~~~~~~~

* Use an external managed PostgreSQL instance (RDS, Cloud SQL) in production.
* The bootstrap credentials must then be pre-seeded into OpenBao; the
  ``postgres-init`` and ``postgres-migrate`` services are skipped.
* Set ``pool_recycle`` to 1 hour in the SQLAlchemy engine to handle
  connection expiration.

Disaster Recovery
~~~~~~~~~~~~~~~~~

Two pieces of state require regular backups:

#. **OpenBao Raft storage** (``openbao-data`` volume) — contains all secrets
   and encryption keys.  Without this, secrets are irrecoverable.
#. **OpenBao init data** (``openbao-init-data`` volume) — contains the unseal
   keys, root token (if not revoked), AppRole credentials, and DB credentials.

If the init data volume is lost: re-run ``openbao-init`` (regenerates AppRole
``secret_id``\ s), re-run ``init_postgres.sh`` (regenerates DB user passwords),
then re-run ``openbao-write-db`` (pushes new ``DATABASE_URL`` to OpenBao).

---

Related Documentation
---------------------

* :doc:`/adr/003-openbao-zero-fallback` — Architecture decision for OpenBao as
  sole source of truth.
* :doc:`/adr/004-self-bootstrapping-postgres` — Architecture decision for the
  self-bootstrapping Postgres credential flow.
* :doc:`/domains/core` — The Python ``core.config`` module (``BootstrapSettings``
  and ``Settings``).
* :doc:`/domains/workers` — The ARQ background worker service.
* :doc:`/domains/api_layer` — The FastAPI application layer.
* :doc:`/guides/deployment` — Deployment guide and checklist.

.. toctree::
   :hidden:

   /adr/003-openbao-zero-fallback
   /adr/004-self-bootstrapping-postgres
