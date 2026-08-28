Deployment Guide
================

This guide covers production and development deployment of the OpenZync
backend stack.  For detailed infrastructure documentation (service topology,
OpenBao bootstrap sequence, Helm chart reference, NGINX config, and
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
     - Required for relational data, vector embeddings, and the default graph
       backend.  Managed RDS / Cloud SQL / AlloyDB work as long as pgvector
       is available.
   * - **Redis**
     - 7+ (7-alpine recommended)
     - Required for ARQ job queues, rate limiting, caching, and pub/sub.
   * - **Python**
     - 3.11+
     - Runtime for the API server and ARQ worker.
   * - **OpenBao**
     - 2.5+ (Vault-compatible)
     - Required for secrets management.  See ADR-003 for the architectural rationale.
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

   OpenZync follows a **zero-fallback secrets model**: OpenBao is the exclusive
   source of truth for all runtime configuration.  The bootstrap sequence
   generates and escrows secrets automatically.  You must provide only the
   four bootstrap environment variables (``BAO_STATIC_SEAL_KEY``,
   ``POSTGRES_PASSWORD``, ``OZ_SECRET_KEY``, ``OZ_WEBHOOK_SIGNING_SECRET``).
   All other secrets (database credentials, API keys) are auto-generated or
   configured via OpenBao at runtime.

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

   # Generate bootstrap secrets (POSTGRES_PASSWORD is Option A only)
   export BAO_STATIC_SEAL_KEY=$(openssl rand -hex 32)
   export POSTGRES_PASSWORD=$(openssl rand -base64 32)
   export OZ_SECRET_KEY=$(python -c "import secrets; print(secrets.token_urlsafe(48))")
   export OZ_WEBHOOK_SIGNING_SECRET=$(python -c "import secrets; print(secrets.token_urlsafe(32))")

   # Start the full stack (8-phase DAG, 41-48s)
   docker compose --env-file .env -f infra/docker-compose.backend.yml --profile local-db up -d --build

   # Tail logs until bootstrap completes
   docker compose --env-file .env -f infra/docker-compose.backend.yml --profile local-db logs -f

**Option B — External Postgres (host / managed DB):**

::

   # Generate bootstrap secrets (no POSTGRES_PASSWORD needed)
   export BAO_STATIC_SEAL_KEY=$(openssl rand -hex 32)
   export OZ_SECRET_KEY=$(python -c "import secrets; print(secrets.token_urlsafe(48))")
   export OZ_WEBHOOK_SIGNING_SECRET=$(python -c "import secrets; print(secrets.token_urlsafe(32))")
   # External DB URL — persisted to .env so OpenBao can render it
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
     - ``postgres``, ``postgres-init``, ``postgres-migrate``, ``openbao-write-db``, ``minio``, ``minio-init``
     - ``--profile local-db`` — embedded Postgres + MinIO; omit for external DB
   * - ``llm``
     - ``ollama``
     - ``--profile llm`` — local LLM inference
   * - ``observability``
     - ``prometheus``, ``grafana``, ``alloy``
     - ``--profile observability`` — metrics + dashboards
   * - ``alt-graph``
     - ``surrealdb``, ``falkordb``
     - ``--profile alt-graph`` — alternative graph backends (set ``OZ_SURREALDB_URL`` / ``OZ_FALKORDB_URL``)

The API becomes available at ``http://localhost:8000`` once the bootstrap
sequence completes (``~60s`` Option A / ``~30s`` Option B).  See
:doc:`/domains/infrastructure` for the full bootstrap phase table.

Service Topology
~~~~~~~~~~~~~~~~

**With ``--profile local-db`` — Full DAG (Option A, 8 phases, 41–48s):**

.. code-block::

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

**Without ``--profile local-db`` — Slim DAG (Option B, external DB, 21–39s):**

.. code-block::

   openbao ─► openbao-init ─► openbao-agent-api ─► api ─┐
                              └─► openbao-agent-worker ─► worker ─► redis

   # No postgres, postgres-init, postgres-migrate, or openbao-write-db.
   # Agent static_secret_render_interval is 10s; grep -c DATABASE_URL is 1
   # (vs 2 for Option A where openbao-write-db merges a second entry).

Each phase must complete before the next begins, enforced by ``depends_on:
condition: service_healthy`` and ``service_completed_successfully``.  See
:doc:`/domains/infrastructure` for the full service graph and bootstrap details.

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
     - Full reset — ``down -v`` then ``up -d`` (removes all volumes)
   * - ``make dev``
     - Local uvicorn with hot-reload (bypasses Docker)

.. note::

   If your Makefile still exposes only ``make docker-up`` / ``make docker-down``
   without the ``-local`` variants, they map to the external-DB form (no
   ``--profile local-db``). Add ``docker-up-local`` / ``docker-down-local``
   targets that include ``--profile local-db --build`` for Option A.

.. warning::

   ``make docker-reset`` removes all data volumes (OpenBao Raft state,
   PostgreSQL data, Redis data).  Use with extreme caution in any environment
   with real data.

External Postgres (production/CI)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For managed or host Postgres, skip ``--profile local-db`` and inject
``OZ_DATABASE_URL`` via OpenBao:

.. code-block:: bash

   # .env — persisted so OpenBao can render it at runtime
   echo 'OZ_DATABASE_URL=postgresql+asyncpg://openzync:pass@host.docker.internal:5432/openzync' >> .env
   docker compose --env-file .env -f infra/docker-compose.backend.yml up -d

``init_openbao.sh`` writes all ``OZ_*`` vars (including ``OZ_DATABASE_URL``
when set) into the OpenBao system secret at ``system/config/data/system``;
the ``openbao-agent-api`` / ``openbao-agent-worker`` sidecars render it to
``/run/secrets/system.env`` (``grep -c DATABASE_URL`` → ``1`` vs ``2`` for
Option A where ``openbao-write-db`` merges a second entry). The
``postgres-init`` / ``postgres-migrate`` / ``openbao-write-db`` services are
not started in this mode.

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
   # Option B shows no postgres / postgres-init / postgres-migrate / openbao-write-db
   curl -s http://localhost:8000/v1/health | python3 -m json.tool  # → {"status": "ok"}
   curl -s http://localhost:8000/v1/ready | python3 -m json.tool   # → {"checks": {"database": true, ...}}

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
     graphBackend: "postgres"
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

The OpenBao bootstrap runs with ``tls_disable = true`` and a static-KV seal by
default.  For production:

#. **Replace the seal** — use Shamir (5-key, 3-threshold) or a cloud KMS
   (AWS KMS, GCP Cloud KMS, Azure Key Vault).  Remove ``BAO_STATIC_SEAL_KEY``.
#. **Enable TLS** — remove ``tls_disable = true`` from
   ``infra/openbao/config.hcl`` and configure proper certificates.
#. **Revoke the root token** — set ``BAO_REVOKE_ROOT_TOKEN=true`` on the
   ``openbao-init`` container.
#. **Persist audit logs** — mount the ``openbao-audit-logs`` volume on
   persistent storage with a suitable retention policy.
#. **Deploy a Raft cluster** — use 3 or 5 OpenBao nodes with Raft storage
   on persistent volumes for high availability.
#. **Deploy the OpenBao Agent sidecars** — each service authenticates via
   AppRole (``role_id`` + ``secret_id``) and renders secrets to a shared
   ``emptyDir`` volume backed by memory (``medium: Memory`` in Kubernetes).

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
   * - **OpenBao**
     - Raft cluster (3–5 nodes), performance standby nodes for read scaling
     - Each org gets its own namespace — total namespace count is bounded by
       OpenBao's storage backend.

Backup and Disaster Recovery
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Two pieces of state require regular backups:

#. **PostgreSQL data** — standard ``pg_dump`` or managed-DB snapshots.  The
   binary ``pgvector`` data is included in the dump.
#. **OpenBao Raft storage** (``openbao-data`` volume) — contains all secrets
   and encryption keys.  Without this, secrets are irrecoverable.
#. **OpenBao init data** (``openbao-init-data`` volume) — contains unseal
   keys, root token (if not revoked), AppRole credentials, and DB credentials.

Recovery procedure if the init data volume is lost:

#. Re-run ``openbao-init`` — regenerates AppRole ``secret_id``\ s.
#. Re-run ``init_postgres.sh`` — regenerates DB user passwords.
#. Re-run ``openbao-write-db`` — pushes new ``DATABASE_URL`` to OpenBao.

.. important::

   If the OpenBao Raft storage volume is lost **and** no backup exists, all
   secrets are irrecoverable.  This includes per-org LLM API keys, webhook
   signing secrets, PII encryption keys, and the system secret.  Regular
   backups of the OpenBao data volume are mandatory in production.

Networking
~~~~~~~~~~

- Replace named volume ``tmpfs``-like mounts (``api-secrets``,
  ``worker-secrets``) with Kubernetes ``emptyDir.medium: Memory``.
- Change Prometheus scrape targets from ``host.docker.internal:8000`` to
  ``api:8000`` (internal DNS).
- Configure proper DNS records and TLS certificates for the NGINX SSL server
  block (see ``infra/nginx/conf.d/openzync.ssl.conf``).
- Set ``OZ_HOSTS_ALLOWED`` to the production domain (e.g.
  ``"api.openzync.tech"``) to enable ``TrustedHostMiddleware``.

PostgreSQL
~~~~~~~~~~

- Use an external managed PostgreSQL instance (RDS, Cloud SQL) in production.
- The bootstrap credentials must then be pre-seeded into OpenBao; the
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

   * - API returns ``OpenBaoAuthError: [sys/namespaces/*] permission denied``
     - ``/sys/namespaces`` is restricted to the **root namespace** — the
       API's AppRole token (in ``system/``) cannot create namespaces.
     - Ensure ``/openbao-bootstrap/root-token`` is mounted in the API
       container. ``OpenBaoClient.create_namespace()`` reads it
       automatically and uses the root token for namespace creation.

   * - API returns ``OpenBaoSecretNotFoundError: [sys/mounts/config] namespace not found``
     - ``/sys/mounts`` is also root-namespace-only; the AppRole token
       cannot enable secrets engines in child namespaces.
     - ``create_org_namespace()`` passes ``use_root_token=True`` to
       ``enable_kv_v2()`` — no manual intervention needed.

   * - AppRole token returns ``namespace not found`` on child namespace operations
     - ``X-Vault-Namespace`` resolves **from root**, not from the token's
       namespace. Use the full path ``system/org_<uuid>/``, not just
       ``org_<uuid>/``.
     - ``_org_ns()`` returns ``system/org_<uuid>/`` automatically.
       Verify no code path calls ``_headers()`` with a bare
       ``org_<uuid>`` value.

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
  graph, OpenBao configuration, NGINX, observability, Helm chart, Makefile
  targets, production considerations)
- :doc:`/domains/core` — Configuration system (``BootstrapSettings``,
  ``Settings``), async DB/Redis/ARQ connection management
- :doc:`/domains/workers` — ARQ worker architecture and task registry
- :doc:`/domains/api_layer` — FastAPI application layer and middleware
- :doc:`/guides/overview` — System overview and architecture
