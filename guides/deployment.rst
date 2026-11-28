Deployment Guide
================

This guide covers production and development deployment of the OpenZync
backend stack.  For detailed infrastructure documentation (service topology,
Helm chart reference, NGINX config, and
observability stack), see :doc:`/domains/infrastructure`.

.. contents:: Sections
   :local:
   :depth: 3
   :class: this-will-duplicate-information-and-it-is-still-useful-here

Environment requirements
------------------------

Before deploying OpenZync, ensure the following infrastructure is available:

.. list-table:: Core requirements
   :header-rows: 1

   * - Component
     - Requirement
     - Notes
    * - **PostgreSQL**
      - 15+ with pgvector extension
      - Required for relational data and vector embeddings (pgvector).  The
        PostgreSQL-native graph backend is **deprecated** and will be removed in
        ``v1.1.0`` — new deployments must use **FalkorDB** (default graph backend,
        always on, no profile needed).  Managed RDS / Cloud SQL / AlloyDB work as
        long as pgvector is available.
   * - **Redis**
     - 7+ (7-alpine recommended)
     - Required for ARQ job queues, rate limiting, caching, and pub/sub.
   * - **Python**
     - 3.11+
     - Runtime for the API server and ARQ worker.
   * - **Docker**
     - 24+ (for Compose deployment)
     - Required for the Docker Compose stack.
   * - **Kubernetes**
     - 1.25+ (for Helm deployment)
     - Required if using the Helm chart.
   * - **NGINX**
     - 1.24+ (optional)
     - Required only if deploying a reverse proxy separately.  The Compose
       stack includes NGINX; the Helm chart uses ``Ingress``.

.. important::

   OpenZync reads all runtime configuration from ``OZ_*`` environment
   variables (``.env``).  Generate ``OZ_SECRET_KEY`` and
   ``OZ_WEBHOOK_SIGNING_SECRET`` once, set ``OZ_DATABASE_URL`` and
   ``OZ_REDIS_URL``, and treat the ``.env`` file itself as the secret —
   there is no sidecar, no seal key, and no external secret store.

One-liner Installer (Production Runbook)
-----------------------------------------

Recommended path for a single production host. One command downloads the
installer, which provisions its own checkout, then runs the prompts below.
See :doc:`/guides/quickstart` for the quickstart hero block.

.. code-block:: bash

   curl -fsSL https://raw.githubusercontent.com/openzync/openzync-core/master/infra/install.sh -o install.sh && bash install.sh

   # Non-interactive (accepts defaults)
   curl -fsSL https://raw.githubusercontent.com/openzync/openzync-core/master/infra/install.sh -o install.sh && bash install.sh --yes

Checkout provisioning
~~~~~~~~~~~~~~~~~~~~~

The installer keeps its checkout under ``INSTALL_DIR/src`` (default
``~/.openzync/src``) with ``openzync-core`` and ``openzync-frontend`` as
siblings. It tracks ``master`` by default; pin fleets to a tag with
``OZ_INSTALL_REF`` (e.g. ``OZ_INSTALL_REF=v1.2.0 bash install.sh``).
Re-running offline reuses the existing checkout instead of re-cloning.
Uninstall also offers to remove the checkout directory via a remove-dir
prompt.

Preconditions
~~~~~~~~~~~~~

Supported hosts: Ubuntu, Fedora, Arch, and macOS — with Docker ``24+`` and
the Compose v2 plugin (``docker compose``, not standalone
``docker-compose``). The installer uses distro repos only
(apt/dnf/pacman/brew). Keep these host ports free on ``127.0.0.1``:

.. list-table:: Installer host ports
   :header-rows: 1

   * - Port
     - Service
   * - ``8000``
     - API
   * - ``3000``
     - Frontend (when enabled)
   * - ``5432``
     - Bundled Postgres (``local-db`` profile only)
   * - ``6380``
     - Redis
   * - ``6381``
     - FalkorDB

Prompts (normal flow)
~~~~~~~~~~~~~~~~~~~~~

* **Frontend on/off** — include the dashboard (host port ``3000``) or run
  API-only.
* **Bundled vs external Postgres** — ``local-db`` starts the embedded
  Postgres; otherwise set ``OZ_DATABASE_URL`` to your managed instance.
  Use a plain reachable hostname in the URL — container ``localhost``
  resolves to the container itself, so point it at the host or DB host.
* **CORS_ORIGINS / HOSTS_ALLOWED** — the public origin(s) and host(s) the
  API serves (e.g. your domain); must match where the frontend and clients
  reach the API.
* **Custom ports** — via Compose override files alongside the generated
  stack; the installer never edits the base Compose file for ports.

Secrets backup
~~~~~~~~~~~~~~

.. important::

   Generated secrets live in ``~/.openzync/.env`` with mode ``0600``.
   Back it up in a password manager or vault. Losing it means
   regenerating every secret and re-creating per-user credentials
   from scratch.

First login
~~~~~~~~~~~

.. warning::

   Sign in to the dashboard with id ``root`` and password ``admin``. You
   are forced to rotate this password at first login — do so immediately
   before exposing the host. Never reuse these defaults anywhere else.

Status and uninstall
~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   bash openzync-core/infra/install.sh --status
   bash openzync-core/infra/install.sh --uninstall

.. note::

   The installer serves plain HTTP only. Terminate TLS upstream (NGINX,
   Ingress, or your load balancer) — see the NGINX and Helm sections below.

Docker Compose Deployment (Backend Stack)
-----------------------------------------

The Docker Compose stack is defined in
``openzync-core/infra/docker-compose.backend.yml``.  It deploys 11 services
across four profiles (``local-db``, ``llm``, ``observability``, ``alt-graph``)
plus the default (no profile — external Postgres).

Quick Start
~~~~~~~~~~~

**Option A — Local one-box (embedded Postgres):**

::

   # Generate secrets (POSTGRES_PASSWORD is Option A only)
   export POSTGRES_PASSWORD=$(openssl rand -base64 32)
   export OZ_SECRET_KEY=$(python -c "import secrets; print(secrets.token_urlsafe(48))")
   export OZ_WEBHOOK_SIGNING_SECRET=$(python -c "import secrets; print(secrets.token_urlsafe(32))")

   # Start the full stack (8-phase DAG, 41-48s)
   docker compose --env-file .env -f infra/docker-compose.backend.yml --profile local-db up -d --build

   # Tail logs until bootstrap completes
   docker compose --env-file .env -f infra/docker-compose.backend.yml --profile local-db logs -f

**Option B — External Postgres (host / managed DB):**

::

   # Generate secrets (no POSTGRES_PASSWORD needed)
   export OZ_SECRET_KEY=$(python -c "import secrets; print(secrets.token_urlsafe(48))")
   export OZ_WEBHOOK_SIGNING_SECRET=$(python -c "import secrets; print(secrets.token_urlsafe(32))")
   # External DB URL — read directly from .env at startup
   echo 'OZ_DATABASE_URL=postgresql+asyncpg://openzync:pass@host.docker.internal:5432/openzync' >> .env
   # extra_hosts: ["host.docker.internal:host-gateway"] is already in compose for api/worker

   # Start without --profile local-db (slim DAG, 21-39s — no postgres phases)
   docker compose --env-file .env -f infra/docker-compose.backend.yml up -d

   # Host Postgres must listen on 0.0.0.0 (socat relay) or listen_addresses='*'
   # — localhost:5432 fails inside the container (netns isolation)

   # Tail logs
   docker compose --env-file .env -f infra/docker-compose.backend.yml logs -f

Profiles:

.. list-table:: Available profiles
   :header-rows: 1

   * - Profile
     - Services added
     - Usage
   * - ``local-db``
     - ``postgres``, ``postgres-init``, ``postgres-migrate``, ``minio``, ``minio-init``
     - ``--profile local-db`` — embedded Postgres + MinIO; omit for external DB
   * - ``llm``
     - ``ollama``
     - ``--profile llm`` — local LLM inference
   * - ``observability``
     - ``prometheus``, ``grafana``, ``alloy``
     - ``--profile observability`` — metrics + dashboards
   * - ``alt-graph``
      - ``surrealdb``
      - ``--profile alt-graph`` — SurrealDB alternative graph backend (set ``OZ_SURREALDB_URL``); FalkorDB is now the default graph backend (always on, host port 6381, no profile needed; PostgreSQL graph backend is deprecated and will be removed in v1.1.0)

The API becomes available at ``http://localhost:8000`` once the stack
is healthy (``~60s`` Option A / ``~30s`` Option B).  See
:doc:`/domains/infrastructure` for the full service graph.

Service Topology
~~~~~~~~~~~~~~~~

**With ``--profile local-db`` — Full DAG (Option A, embedded Postgres):**

.. code-block::

   postgres ─► postgres-init ─► postgres-migrate ─► api ─► redis ◄── worker

.. note::

   ``api``/``worker`` wait on ``redis`` and ``falkordb``
   (``service_healthy``).

**Without ``--profile local-db`` — Slim DAG (Option B, external DB, 21–39s):**

.. code-block::

   api ─► redis ◄── worker

   # No postgres, postgres-init, or postgres-migrate.

Each phase must complete before the next begins, enforced by ``depends_on:
condition: service_healthy`` and ``service_completed_successfully``.  See
:doc:`/domains/infrastructure` for the full service graph.

Makefile Targets
~~~~~~~~~~~~~~~~

The ``Makefile`` at the monolith root exposes common Compose operations:

.. list-table:: Makefile infrastructure targets
   :header-rows: 1

   * - Target
     - Description
   * - ``make docker-up``
     - ``docker compose --env-file .env -f infra/docker-compose.backend.yml up -d`` (external DB — no postgres)
   * - ``make docker-up-local``
     - ``docker compose --env-file .env -f infra/docker-compose.backend.yml --profile local-db up -d --build`` (embedded Postgres)
   * - ``make docker-down``
     - ``docker compose --env-file .env -f infra/docker-compose.backend.yml down``
   * - ``make docker-down-local``
     - ``docker compose --env-file .env -f infra/docker-compose.backend.yml --profile local-db down``
   * - ``make docker-logs``
     - ``docker compose --env-file .env -f infra/docker-compose.backend.yml logs -f`` (add ``--profile local-db`` for Option A)
   * - ``make docker-reset``
     - Full reset — ``down -v`` then ``up -d --build`` (removes all
       volumes; add ``--profile local-db`` for Option A)
   * - ``make dev``
     - Local uvicorn with hot-reload (bypasses Docker)

.. note::

   If your Makefile still exposes only ``make docker-up`` / ``make docker-down``
   without the ``-local`` variants, they map to the external-DB form (no
   ``--profile local-db``). Add ``docker-up-local`` / ``docker-down-local``
   targets that include ``--profile local-db --build`` for Option A.

.. warning::

   ``make docker-reset`` removes all data volumes (PostgreSQL data,
   Redis data, MinIO blobs).  Use with extreme caution in any environment
   with real data.  A cold-boot loop must use ``down -v`` (not plain
   ``down``): ``init_postgres.sh`` rotates DB passwords on every boot, so a
   restart without ``-v`` can leave a stale ``DATABASE_URL`` behind.
   Re-run with ``--build`` so one-shot init images are not stale.

External Postgres (production/CI)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For managed or host Postgres, skip ``--profile local-db`` and set
``OZ_DATABASE_URL`` directly in ``.env``:

.. code-block:: bash

   # .env — read directly from the environment at startup
   echo 'OZ_DATABASE_URL=postgresql+asyncpg://openzync:pass@host.docker.internal:5432/openzync' >> .env
   docker compose --env-file .env -f infra/docker-compose.backend.yml up -d

The API, worker, and migrator (``postgres-migrate``) all read
``OZ_DATABASE_URL`` straight from the environment. The
``postgres-init`` / ``postgres-migrate`` services are not started in
this mode.

``extra_hosts: ["host.docker.internal:host-gateway"]`` is already configured
for ``api`` and ``worker`` in the compose file — use
``host.docker.internal`` in the URL, not ``localhost``.
``localhost:5432`` inside the container resolves to the container's own network
namespace and will fail (connection refused). Ensure the host Postgres is
reachable from containers:

* **socat relay (no config change):** ``socat TCP-LISTEN:5432,bind=0.0.0.0,fork TCP:[::1]:5432 &``
  — forwards ``0.0.0.0:5432`` to the host's loopback. Useful when Postgres
  only listens on ``127.0.0.1`` / ``::1``.
* **or** set ``listen_addresses='*'`` in ``postgresql.conf`` and restart
  Postgres — makes it listen on all interfaces.

Verify:

.. code-block:: bash

   docker compose --env-file .env -f infra/docker-compose.backend.yml ps
   # Option B shows no postgres / postgres-init / postgres-migrate
   curl -s http://localhost:8000/health | python3 -m json.tool  # → {"status": "ok"}
   curl -s http://localhost:8000/ready | python3 -m json.tool   # → {"checks": {"database": true, ...}}

Frontend Docker Deployment
--------------------------

The frontend is a Next.js 16 application deployed from its own ``Dockerfile``
at ``openzync-frontend/Dockerfile``.

Building the frontend image::

   docker build -t openzync-frontend:latest openzync-frontend/

   # Or using the Compose-override pattern (if included in your compose file)
   docker compose -f infra/docker-compose.backend.yml \
     -f infra/docker-compose.frontend.yml up -d frontend

The frontend expects the API at ``http://localhost:8000`` by default.  In
production, configure ``NEXT_PUBLIC_API_URL`` as a build arg or environment
variable.

The frontend container should sit behind NGINX (as configured in
``infra/nginx/conf.d/openzync.conf``) which routes ``/v1/*``, ``/docs``, and
``/health`` to the API, and everything else to the frontend.

Kubernetes / Helm Deployment
----------------------------

The Helm chart is at ``infra/helm/openzync/``.  It supports Kubernetes 1.25+
with the following resources:

.. list-table:: Helm chart resources
   :header-rows: 1

   * - Resource
     - Description
   * - ``deployment-api.yaml``
     - API Deployment with liveness (``GET /health``) and readiness
       (``GET /ready``) probes
   * - ``deployment-worker.yaml``
     - Worker Deployment with liveness probe (Redis ping)
   * - ``hpa.yaml``
     - HorizontalPodAutoscaler for API (CPU + memory) and worker (CPU)
   * - ``configmap.yaml``
     - Non-sensitive configuration (log level, LLM backend, JWT TTLs, rate
       limits, CORS origins, OTLP endpoint)
   * - ``secret.yaml``
     - Conditionally created Secret for sensitive values (database URL, Redis
       URL, API keys)
   * - ``service.yaml``
     - ClusterIP service on port 80 → container port 8000
   * - ``ingress.yaml``
     - Ingress with optional cert-manager TLS, conditionally enabled
   * - ``pvc.yaml``
     - PersistentVolumeClaims for PostgreSQL and Redis (optional)

Deploying with Helm
~~~~~~~~~~~~~~~~~~~

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

   # Scale API replicas and enable autoscaling
   helm upgrade --install openzync infra/helm/openzync/ \
     --set api.replicaCount=3 \
     --set api.autoscaling.enabled=true \
     --set api.autoscaling.minReplicas=3 \
     --set api.autoscaling.maxReplicas=20

Helm values reference
~~~~~~~~~~~~~~~~~~~~~

Key sections of ``values.yaml``:

.. code-block:: yaml

   api:
     replicaCount: 2
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

   worker:
     replicaCount: 1
     autoscaling:
       enabled: true
       minReplicas: 1
       maxReplicas: 5
       targetCPUUtilizationPercentage: 75

   postgresql:
     enabled: true
     image: pgvector/pgvector:pg15
     database: openzync
     persistence:
       enabled: true
       size: 20Gi

   redis:
     enabled: true
     image: redis:7-alpine
     persistence:
       enabled: true
       size: 5Gi

    config:
      llmBackend: "ollama"
      llmModel: "llama3.2:3b"
      embeddingDim: 768
      graphBackend: "falkordb"
      otlpEndpoint: "http://alloy:4317"

Secrets are **not** inlined in ``values.yaml`` — inject them via ``--set``,
an external values file, or the ``external-secrets`` operator.

Database Migrations (Alembic Runbook)
-------------------------------------

Migrations are managed via Alembic with async-compatible configuration.

Migration Commands
~~~~~~~~~~~~~~~~~~

.. list-table:: Alembic commands
   :header-rows: 1

   * - Command
     - Description
   * - ``alembic upgrade head``
     - Apply all pending migrations (idempotent)
   * - ``alembic downgrade -1``
     - Roll back the last migration
   * - ``alembic downgrade <revision>``
     - Roll back to a specific revision
   * - ``alembic revision --autogenerate -m "description"``
     - Auto-generate a new migration from model changes
   * - ``alembic check``
     - Verify the migration history matches the models (exit 1 if out of sync)
   * - ``alembic history``
     - Show the full migration chain

The comparable ``make`` shortcuts are ``make migrate`` (upgrade head),
``make migrate-check`` (check), ``make migrate-new`` (autogenerate), and
``make migrate-downgrade`` (roll back -1).

Production Migration Runbook
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Pre-deployment check**::

   alembic check

If this exits non-zero, a migration is needed.  Never deploy new code without
running pending migrations first.

**Standard upgrade** (zero-downtime compatible for additive changes)::

   alembic upgrade head

**Rolling back**::

   # Identify the current revision
   alembic current

   # Roll back one step
   alembic downgrade -1

   # Verify the rollback
   alembic check

**In Docker Compose**, migrations run automatically during bootstrap (the
``postgres-migrate`` one-shot container).  To run migrations manually against a
running Compose stack::

   docker compose -f infra/docker-compose.backend.yml \
     run --rm postgres-migrate alembic upgrade head

.. warning::

   In production, always run migrations **before** rolling out new application
   code.  The new code should be backward-compatible with the old schema for
   at least one release cycle (additive-only pattern).  Destructive changes
   (column drops, table renames) must be a separate, carefully planned
   migration step.

Monitoring Setup
----------------

The observability stack consists of three components deployed via the
``observability`` Compose profile:

.. list-table:: Observability components
   :header-rows: 1

   * - Component
     - Image
     - Purpose
   * - **Prometheus**
     - ``prom/prometheus:v2.53.0``
     - Metrics collection and alerting
   * - **Grafana**
     - ``grafana/grafana:11.0.0``
     - Dashboards and visualisation
   * - **Alloy**
     - ``grafana/alloy:latest``
     - OpenTelemetry collector (metrics, logs, traces)

Starting Observability
~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   docker compose -f infra/docker-compose.backend.yml \
     --profile observability up -d

Prometheus is pre-configured to scrape the API's ``/metrics`` endpoint every
15 seconds.  Grafana is auto-provisioned with a Prometheus datasource and the
``OpenZync-overview.json`` dashboard, which includes:

.. list-table:: Pre-built dashboard panels
   :header-rows: 1

   * - Panel
     - Type
     - Description
   * - API Request Rate & Error Rate
     - Time series
     - HTTP request rate by status code with 5xx error overlay
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
     - Up/down status for API, Worker, PostgreSQL, Redis, graph backends

.. seealso::

   :doc:`/domains/infrastructure` for detailed Prometheus config
   (``prometheus.yml``), Grafana provisioning, and the Alloy OTLP pipeline.

Production Observability
~~~~~~~~~~~~~~~~~~~~~~~~

In production, replace the dev-mode stack with:

- **Mimir** (long-term metrics storage, replacing local Prometheus)
- **Tempo** (distributed tracing)
- **Loki** (log aggregation)

Configure Alloy as the single OTLP ingestion point for all three signals:

.. code-block:: alloy

   // Alloy scrapes /metrics, collects container logs, and receives OTLP traces
   prometheus.scrape "openzync_api" {
     targets = [{ __address__ = "api:8000", job = "openzync-api" }]
     forward_to = [prometheus.remote_write.mimir.receiver]
   }

   loki.source.docker "openzync" {
     forward_to = [loki.process.default.receiver]
   }

   otelcol.receiver.otlp "default" {
     grpc { endpoint = "0.0.0.0:4317" }
     output { traces = [otelcol.processor.batch.default.input] }
   }

Enable Grafana authentication and replace the anonymous-access settings with
your identity provider (OAuth, SAML, or LDAP).

Production Considerations
-------------------------

Secrets Management
~~~~~~~~~~~~~~~~~~

All runtime configuration lives in ``OZ_*`` environment variables
(``.env``, mode ``0600``).  For production:

#. **Restrict the env file** — ``chmod 600 .env``, owned by the service
   user; never commit it, never log its values.
#. **Terminate TLS upstream** — the API serves plain HTTP on loopback;
   put NGINX or your load balancer in front with certificates.
#. **Back up ``.env`` and the database** — losing ``OZ_SECRET_KEY``
   invalidates JWTs and per-purpose encryption subkeys; rotate with the
   ``OZ_SECRET_KEY_PREVIOUS`` dual-key window (see
   :doc:`/guides/rotating-keys`).

.. seealso::

   Rotating a live key? See :doc:`/guides/rotating-keys` (dual-key rotation).

Scaling
~~~~~~~

.. list-table:: Scaling guidance
   :header-rows: 1

   * - Component
     - Scaling approach
     - Limits
   * - **API server**
     - Horizontal (increase replicas) via HPA or Compose ``scale``
     - CPU-bound under LLM proxy load; memory-bound under concurrent
       retrieval.  Recommend 2–10 replicas.
   * - **ARQ Worker**
     - Horizontal (increase replicas) plus vertical (increase
       ``MAX_WORKERS`` per pod)
     - High-priority queue runs ``min(MAX_WORKERS, 8)`` concurrent jobs.
       Low-priority runs ``max(1, MAX_WORKERS - high_workers)``.  Recommend
       1–5 replicas.
   * - **PostgreSQL**
     - Vertical (larger instance) first; read replicas for query scaling
     - pgvector HNSW index build is memory-intensive.  Ensure
       ``maintenance_work_mem`` is set appropriately.
   * - **Redis**
     - Vertical (larger instance) for cache; cluster mode for queue HA
     - ARQ queues benefit from ``maxmemory-policy allkeys-lru`` and
       sufficient memory for peak queue depth.

Backup and Disaster Recovery
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Two pieces of state require regular backups:

#. **PostgreSQL data** — standard ``pg_dump`` or managed-DB snapshots.  The
   binary ``pgvector`` data is included in the dump.
#. **The** ``.env`` **file** — holds ``OZ_SECRET_KEY``,
   ``OZ_WEBHOOK_SIGNING_SECRET``, and credentials.  Without it, JWTs and
   encrypted webhook secrets cannot be verified or decrypted.

Recovery procedure if the env file is lost:

#. Regenerate ``OZ_SECRET_KEY`` / ``OZ_WEBHOOK_SIGNING_SECRET``.
#. Re-run ``init_postgres.sh`` — regenerates DB user passwords.
#. Update ``OZ_DATABASE_URL`` in ``.env`` and restart.

.. important::

   If the ``.env`` file is lost **and** no backup exists, all
   secrets are irrecoverable.  This includes per-user LLM API keys, webhook
   signing secrets, and PII encryption keys.  Regular
   backups of ``.env`` are mandatory in production.

Networking
~~~~~~~~~~

- Change Prometheus scrape targets from ``host.docker.internal:8000`` to
  ``api:8000`` (internal DNS).
- Configure proper DNS records and TLS certificates for the NGINX SSL server
  block (see ``infra/nginx/conf.d/openzync.ssl.conf``).
- Set ``OZ_HOSTS_ALLOWED`` to the production domain (e.g.
  ``"api.openzync.tech"``) to enable ``TrustedHostMiddleware``.

PostgreSQL
~~~~~~~~~~

- Use an external managed PostgreSQL instance (RDS, Cloud SQL) in production.
- Set ``OZ_DATABASE_URL`` in ``.env`` directly; the
  ``postgres-init`` and ``postgres-migrate`` services are skipped.
- Set ``pool_recycle`` to 3600 seconds (the default in ``init_db_engine``)
  to handle connection expiration from managed-DB proxies.
- Configure appropriate ``work_mem`` for vector index scans and
  ``maintenance_work_mem`` for ``CREATE INDEX CONCURRENTLY`` on pgvector
  HNSW indexes.

Known Deployment Pitfalls
---------------------------

.. important::

   The following issues are **blockers** that will prevent a fresh deployment
   from starting.  They are documented here so operators can diagnose and
   resolve them quickly.

.. list-table::
   :header-rows: 1

   * - Symptom
     - Root Cause
     - Resolution

   * - Container shows ``(unhealthy)`` but logs show the app is running
     - HEALTHCHECK uses ``wget`` which is not installed in the image.
     - The fix has been applied to the API Dockerfile (using
       ``python3 -c``). For custom images, ensure the healthcheck
       command is available in the base image.

   * - Frontend settings pages return 404
     - Nginx only routes ``/v1/*`` to the API; ``/admin/*`` hits the
       frontend.
     - Add ``location /admin/ { proxy_pass $api_upstream; ... }``
       to the nginx configuration.

Related documentation
---------------------

- :doc:`/domains/infrastructure` — Full infrastructure reference (service
  graph, NGINX, observability, Helm chart, Makefile
  targets, production considerations)
- :doc:`/domains/core` — Configuration system (``Settings``,
  ``get_settings``), async DB/Redis/ARQ connection management
- :doc:`/domains/workers` — ARQ worker architecture and task registry
- :doc:`/domains/api_layer` — FastAPI application layer and middleware
- :doc:`/guides/overview` — System overview and architecture
