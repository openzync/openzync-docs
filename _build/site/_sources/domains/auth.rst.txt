Authentication & Authorization Domain
======================================

.. note::

   This document covers the authentication and authorisation subsystem **within
   the OpenZync monolith** (``openzync-core``).  Code examples assume the
   relevant packages are importable from the monolith's Python path.

   The Auth domain spans: dashboard user authentication (email/password, JWT),
   programmatic API key authentication, project-scoped access control, email
   verification, MFA, passwordless login, password reset, OTP generation and
   rate-limiting, PII detection/redaction for ingestion flows, and user CRUD.

   **Design principles**:

   * Dual-mode authentication — JWT for dashboard, API key for programmatic.
   * All secrets and tokens are hashed before storage (bcrypt for passwords,
     SHA-256 with salt for API keys, SHA-256 for refresh tokens).
   * OTPs are never stored in plaintext — only SHA-256 hashes go to Redis.
   * Zero silent fallback — if authentication infrastructure (Redis, DB) is
     unavailable, the error propagates as an HTTP 401/503 — no degraded auth.
   * Tenant isolation via ``organization_id`` on every query.

.. contents:: Sections
   :local:
   :depth: 2
   :class: this-will-duplicate-information-and-it-is-still-useful-here


Core Concepts
-------------

.. _auth-architecture-flow:

Authentication Architecture
~~~~~~~~~~~~~~~~~~~~~~~~~~~

OpenZync supports two independent authentication methods, routed by the
``Authorization`` header value:

**JWT (Dashboard users)**
   Bearer tokens with three base64url segments (two dots).  Signed with
   ``SECRET_KEY`` via HS256.  Used by the web dashboard UI.  Claims include
   ``sub`` (user UUID), ``org_id``, ``role``, and ``type`` (``"access"``).

**API Key (SDK clients)**
   Bearer tokens starting with ``oz_live_`` or ``oz_test_``.  Stored as
   salted SHA-256 hashes with an unsalted ``lookup_hash`` for fast Redis/DB
   lookups.  Scoped to a single project (no org-wide keys).  Created and
   revoked via the dashboard at ``/v1/projects/{project_id}/api-keys``.

.. rubric:: Request State (set by AuthMiddleware)

After successful authentication, the middleware sets the following keys on
``request.state``:

.. list-table::
   :header-rows: 1

   * - Key
     - JWT
     - API Key
     - Type
   * - ``org_id``
     - Extracted from JWT claims
     - From ``api_keys.organization_id``
     - ``str``
   * - ``user_id``
     - ``sub`` claim
     - ``api_keys.created_by`` (may be ``None``)
     - ``str | None``
   * - ``role``
     - ``role`` claim
     - ``None``
     - ``str | None``
   * - ``auth_type``
     - ``"jwt"``
     - ``"api_key"``
     - ``str``
   * - ``api_key_scopes``
     - ``["read", "write", "admin"]``
     - Key's scopes
     - ``list[str]``
   * - ``api_key_project_id``
     - ``None``
     - Key's project UUID
     - ``str | None``

Tenant Isolation
~~~~~~~~~~~~~~~~

Every entity in the system is scoped to an ``organization_id``.  All
repositories apply this scope in every query, and PostgreSQL Row-Level
Security (RLS) enforces it at the database level.

.. code-block:: python

    # Repository methods always accept organization_id
    async def get_user(self, organization_id: UUID, user_id: UUID) -> User:
        ...

    async def list_by_org(
        self,
        organization_id: UUID,
        project_id: UUID | None = None,
    ) -> list[ApiKey]:
        ...

The ``organization_id`` is extracted from the authenticated request state
by dependencies in :mod:`dependencies/auth.py`.

User Model Distinction
~~~~~~~~~~~~~~~~~~~~~~

OpenZync has two kinds of users, stored in the same ``users`` table but with
different attributes set:

.. list-table::
   :header-rows: 1

   * - Attribute
     - Dashboard User
     - SDK / End-User
   * - ``email``
     - Set (globally unique)
     - Optional
   * - ``password_hash``
     - Set (bcrypt)
     - ``None``
   * - ``external_id``
     - Same as email
     - Caller-defined (unique per org)
   * - ``role``
     - ``"admin"`` or ``"member"``
     - ``"member"`` (default)
   * - ``is_email_verified``
     - Relevant
     - Typically ``False``
   * - ``mfa_enabled``
     - Relevant
     - ``False``
   * - Authentication
     - JWT (email/password)
     - API key (project-scoped)

The ``User`` model (``models/user.py``) unifies both kinds via nullable
fields.


Configuration Settings
----------------------

The following settings in ``core/config.py`` control the auth subsystem:

.. list-table::
   :header-rows: 1

   * - Setting
     - Type
     - Default
     - Description
   * - ``SECRET_KEY``
     - ``str``
     - Required (no default)
     - HMAC secret for JWT signing (min 32 chars)
   * - ``JWT_ACCESS_TOKEN_TTL_MINUTES``
     - ``int``
     - ``30``
     - Access token lifetime in minutes (1-1440)
   * - ``JWT_REFRESH_TOKEN_TTL_DAYS``
     - ``int``
     - ``7``
     - Refresh token lifetime in days (1-90)

All values are loaded from OpenBao at startup — see :doc:`core` for the
configuration system documentation.


Models
------

User
~~~~

Module: ``models/user.py``

.. class:: User(TimestampMixin, Base)

   Represents both dashboard users and SDK end-users.  ``external_id`` is
   caller-chosen and unique per organization.  Dashboard users additionally
   have a ``password_hash`` and ``email`` set.

   .. tabularcolumns:: |p{4cm}|p{3cm}|p{9cm}|

   .. list-table::
      :header-rows: 1
      :widths: 30 15 55

      * - Column
        - Type
        - Description
      * - ``id``
        - ``UUID`` (PK)
        - Primary key, ``gen_random_uuid()``
      * - ``organization_id``
        - ``UUID`` (FK)
        - Owning organization; ``ON DELETE CASCADE``
      * - ``external_id``
        - ``Text``
        - Caller-defined identifier, unique per org via
          ``uq_user_organization_external``
      * - ``name``
        - ``Text | None``
        - Optional display name
      * - ``email``
        - ``Text | None``
        - Email address (globally unique via partial unique index
          ``ix_user_email_unique``)
      * - ``metadata_``
        - ``JSONB``
        - Arbitrary metadata (column name ``"metadata"`` in DB —
          SQLAlchemy ``metadata`` is reserved)
      * - ``role``
        - ``String(50)``
        - User role — ``"admin"``, ``"member"``. Default: ``"member"``
      * - ``password_hash``
        - ``Text | None``
        - bcrypt hash; set only for dashboard users
      * - ``is_active``
        - ``Boolean``
        - Soft activation toggle. Default: ``True``
      * - ``is_deleted``
        - ``Boolean``
        - Soft-delete flag for GDPR two-phase purge. Default: ``False``
      * - ``is_email_verified``
        - ``Boolean``
        - Email verification status. Default: ``False``
      * - ``email_verified_at``
        - ``DateTime(tz) | None``
        - When the email was verified (``None`` = not verified)
      * - ``mfa_enabled``
        - ``Boolean``
        - MFA enabled for this user. Default: ``False``
      * - ``summary``
        - ``Text | None``
        - Generated AI summary of the user
      * - ``summary_updated_at``
        - ``DateTime(tz) | None``
        - When the summary was last generated

   Constraints:

   * ``UNIQUE (organization_id, external_id)`` — unique per-tenant user ID.
   * Partial unique index on ``email WHERE email IS NOT NULL AND is_deleted = false``.
   * Index on ``organization_id`` for tenant-scoped queries.

ApiKey
~~~~~~

Module: ``models/api_key.py``

.. class:: ApiKey(TimestampMixin, Base)

   A project-scoped API key credential.  The raw key is never persisted —
   only its salted SHA-256 hash and an unsalted lookup hash are stored.

   .. list-table::
      :header-rows: 1
      :widths: 30 15 55

      * - Column
        - Type
        - Description
      * - ``id``
        - ``UUID`` (PK)
        - Primary key
      * - ``organization_id``
        - ``UUID`` (FK)
        - Owning organization; ``ON DELETE CASCADE``
      * - ``project_id``
        - ``UUID`` (FK)
        - Project scope — every key belongs to exactly one project;
          ``ON DELETE CASCADE``
      * - ``created_by``
        - ``UUID`` (FK) \| ``None``
        - JWT user who created this key; ``ON DELETE SET NULL``
      * - ``lookup_hash``
        - ``Text`` (UNIQUE)
        - Unsalted SHA-256 for fast cache/DB lookup
      * - ``key_hash``
        - ``Text``
        - Salted SHA-256 hash for verification
      * - ``salt``
        - ``Text``
        - 16-byte random hex salt
      * - ``prefix``
        - ``String(10)``
        - Human-readable prefix — ``oz_live_`` or ``oz_test_``
      * - ``name``
        - ``Text | None``
        - Optional label
      * - ``scopes``
        - ``ARRAY(String)``
        - Permission scopes. Default: ``["read", "write"]``
      * - ``last_used_at``
        - ``DateTime \| None``
        - Timestamp of most recent authentication
      * - ``expires_at``
        - ``DateTime \| None``
        - Optional expiration
      * - ``is_revoked``
        - ``Boolean``
        - Soft revocation. Default: ``False``

   Constraints:

   * ``CHECK (prefix IN ('oz_live_', 'oz_test_'))``
   * Index on ``organization_id`` and ``created_by``.

RefreshToken
~~~~~~~~~~~~

Module: ``models/refresh_token.py``

.. class:: RefreshToken(TimestampMixin, Base)

   A refresh token for dashboard session renewal.  Tokens form a rotation
   chain: each rotation creates a new token and links back to the previous
   one via ``rotated_by``.

   .. list-table::
      :header-rows: 1
      :widths: 30 15 55

      * - Column
        - Type
        - Description
      * - ``id``
        - ``UUID`` (PK)
        - Primary key
      * - ``user_id``
        - ``Text``
        - Dashboard user UUID (stored as text for schema simplicity)
      * - ``organization_id``
        - ``UUID`` (FK)
        - Owning organization; ``ON DELETE CASCADE``
      * - ``token_hash``
        - ``Text`` (UNIQUE)
        - SHA-256 hash of the opaque token string
      * - ``expires_at``
        - ``DateTime``
        - Expiration timestamp (naive UTC)
      * - ``is_revoked``
        - ``Boolean``
        - Revocation flag. Default: ``False``
      * - ``rotated_by``
        - ``UUID \| None``
        - FK to the replacement token (rotation chain)

Project
~~~~~~~

Module: ``models/project.py``

.. class:: Project(TimestampMixin, Base)

   A collaborative project workspace within an organisation.  Groups sessions,
   facts, graph knowledge, and API key scopes.

   .. list-table::
      :header-rows: 1
      :widths: 30 15 55

      * - Column
        - Type
        - Description
      * - ``id``
        - ``UUID`` (PK)
        - Primary key
      * - ``organization_id``
        - ``UUID`` (FK)
        - Owning organization; ``ON DELETE CASCADE``
      * - ``name``
        - ``Text``
        - Human-readable name, unique per org
      * - ``description``
        - ``Text \| None``
        - Optional description
      * - ``metadata_``
        - ``JSONB``
        - Arbitrary metadata (column name ``"metadata"``)
      * - ``is_archived``
        - ``Boolean``
        - Archive flag — preserves data. Default: ``False``
      * - ``created_by``
        - ``UUID`` (FK) \| ``None``
        - Creator user; ``ON DELETE SET NULL``

   Constraint: ``UNIQUE (organization_id, name)``.

ProjectMember
~~~~~~~~~~~~~

Module: ``models/project_member.py``

.. class:: ProjectMember(TimestampMixin, Base)

   A user's membership and role within a project.  A user can only have one
   role per project.

   .. list-table::
      :header-rows: 1
      :widths: 30 15 55

      * - Column
        - Type
        - Description
      * - ``id``
        - ``UUID`` (PK)
        - Primary key
      * - ``project_id``
        - ``UUID`` (FK)
        - Project; ``ON DELETE CASCADE``
      * - ``user_id``
        - ``UUID`` (FK)
        - User; ``ON DELETE CASCADE``
      * - ``role``
        - ``String(20)``
        - ``"owner"`` or ``"member"``. Default: ``"member"``

   Constraints:

   * ``UNIQUE (project_id, user_id)``.
   * ``CHECK (role IN ('owner', 'member'))``.
   * Index on both ``project_id`` and ``user_id``.

Organization
~~~~~~~~~~~~

Module: ``models/organization.py``

.. class:: Organization(TimestampMixin, Base)

   The top-level tenant entity.  Every user, project, API key, and session
   belongs to exactly one organisation.

   .. list-table::
      :header-rows: 1
      :widths: 30 15 55

      * - Column
        - Type
        - Description
      * - ``id``
        - ``UUID`` (PK)
        - Primary key
      * - ``name``
        - ``Text``
        - Human-readable name
      * - ``plan``
        - ``String(20)``
        - Billing plan — ``"free"``, ``"pro"``, ``"enterprise"``
      * - ``config``
        - ``JSONB``
        - Per-org UI-exposed configuration (LLM, embeddings, graph, behaviour)
      * - ``quotas``
        - ``JSONB``
        - Usage quotas — including PII config under ``quotas["pii"]``
      * - ``is_active``
        - ``Boolean``
        - Soft toggle for deactivation. Default: ``True``

   Constraint: ``CHECK (plan IN ('free', 'pro', 'enterprise'))``.


Schema (``TimestampMixin``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Module: ``models/base.py``

.. class:: TimestampMixin

   Adds ``created_at`` and ``updated_at`` to any model that inherits it.

   .. attribute:: created_at
      :type: DateTime(tz)
      :server_default: ``func.now()``

   .. attribute:: updated_at
      :type: DateTime(tz)
      :server_default: ``func.now()``
      :onupdate: ``func.now()``

.. class:: CreatedAtMixin

   Adds only ``created_at`` — for immutable / append-only tables.


End-to-End Authentication Flows
--------------------------------

Signup Flow
~~~~~~~~~~~

:Endpoint: ``POST /v1/auth/signup``
:Service: :meth:`AuthService.signup`
:Throttled: Yes — per-IP (3 per hour)

.. mermaid::

   sequenceDiagram
       participant Client
       participant Router
       participant AuthService
       participant AuthRepository
       participant OtpService
       participant Redis
       participant EmailService

       Client->>Router: POST /v1/auth/signup {email, password, org_name}
       Router->>AuthService: signup(payload)
       AuthService->>AuthService: _validate_password(password)
       AuthService->>AuthRepository: find_user_by_email(email)
       AuthRepository-->>AuthService: User | None
       alt Email already registered
           AuthService-->>Router: raise ConflictError(409)
           Router-->>Client: 409 Conflict
       else
           AuthService->>AuthRepository: create_organization(name, plan="free")
           AuthService->>AuthRepository: seed_prompts_for_org(org.id)
           AuthService->>AuthService: hash_password(password)
           AuthService->>AuthRepository: create_dashboard_user(...)
           AuthService->>OtpService: generate_and_send(email, "signup")
           OtpService->>Redis: store SHA-256(otp) with TTL 600s
           OtpService->>EmailService: send OTP email
           AuthService-->>Router: SignupResponse(message, email)
           Router-->>Client: 201 {message, email}
       end

**Step-by-step:**

#. The client sends email, password, and organisation name.
#. The router checks per-IP signup throttle (``AuthThrottle.check_signup_attempt``).
#. :meth:`AuthService.signup` validates password strength (min 8 chars, 1 upper,
   1 lower, 1 digit).
#. Email uniqueness is checked — ``ConflictError(409)`` if taken.
#. An :class:`Organization` is created with plan ``"free"``.
#. Default prompt templates are seeded for the new org.
#. The password is bcrypt-hashed and the dashboard user is created with role ``"admin"``.
#. A verification OTP is sent via email. **No JWT tokens are issued yet** —
   the user must first verify their email.
#. The client receives a ``SignupResponse`` with instructions to call
   ``POST /v1/auth/verify-email``.

Email Verification & Token Issuance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:Endpoint: ``POST /v1/auth/verify-email``
:Service: :meth:`AuthService.verify_email`
:Throttled: Yes — per-email (10/15min) and per-IP (20/15min)

.. code-block:: python
   :caption: Client flow after signup

   # 1. Check email for 6-digit OTP
   # 2. Submit OTP
   response = await client.post("/v1/auth/verify-email", json={
       "email": "admin@acme.com",
       "otp": "483926",
   })
   # 3. Receive JWT pair
   assert response.status_code == 200
   assert "access_token" in response.json()
   assert "refresh_token" in response.json()

The OTP is validated against its SHA-256 hash in Redis using constant-time
comparison (``hmac.compare_digest``).  After 5 failed attempts, the OTP is
invalidated and a new one must be requested via ``POST /v1/auth/resend-otp``.

Login Flow (Password)
~~~~~~~~~~~~~~~~~~~~~

:Endpoint: ``POST /v1/auth/login``
:Service: :meth:`AuthService.login`
:Throttled: Yes — per-email (5/15min) and per-IP (20/15min)

.. code-block:: python
   :caption: Login request

   response = await client.post("/v1/auth/login", json={
       "email": "admin@acme.com",
       "password": "secure-p@ssword-123",
   })

   # Response when MFA is disabled:
   assert response.status_code == 200
   data = response.json()
   assert data["requires_mfa"] is False
   assert data["access_token"]  # JWT string
   assert data["refresh_token"] # opaque token string
   assert data["expires_in"]    # seconds

   # Response when MFA is enabled:
   assert data["requires_mfa"] is True
   assert data["mfa_session_token"]  # opaque session key
   # → Client must call POST /v1/auth/mfa/verify

The login flow checks, in order:

#. Email exists (otherwise: ``AuthenticationError(401)``).
#. Password is set (not an OTP-only user).
#. Password matches bcrypt hash.
#. Account is active and not deleted.
#. Email is verified.
#. MFA gate: if ``mfa_enabled`` is ``True``, an OTP is sent and a pending
   MFA session is stored in Redis (TTL: 10 minutes).  The client receives a
   ``LoginResponse`` with ``requires_mfa=True`` and an ``mfa_session_token``.
#. Otherwise: JWT access + refresh tokens are issued immediately.

Passwordless Login Flow
~~~~~~~~~~~~~~~~~~~~~~~

:Endpoint: ``POST /v1/auth/login/otp/send`` → ``POST /v1/auth/login/otp/verify``
:Service: :meth:`AuthService.generate_login_otp` → :meth:`AuthService.passwordless_login`

Alternative to password authentication.  Uses the same OTP infrastructure
as email verification but with the ``passwordless_login`` purpose scope.
Key difference: the user's email is auto-verified on first login (the OTP
proves email ownership).

Refresh Token Rotation
~~~~~~~~~~~~~~~~~~~~~~

:Endpoint: ``POST /v1/auth/refresh``
:Service: :meth:`AuthService.refresh`

.. code-block:: python
   :caption: Token refresh

   response = await client.post("/v1/auth/refresh", json={
       "refresh_token": "abc123...",
   })
   assert response.status_code == 200
   data = response.json()
   new_access = data["access_token"]
   new_refresh = data["refresh_token"]
   # Old refresh token is now revoked and linked to the new one
   # via rotated_by for audit.

Refresh tokens are:

* **Opaque** — random 64-character hex string, stored as SHA-256 hash.
* **Rotated** on every use — the old token is revoked and linked via
  ``rotated_by``.
* **Revoked on password reset** — forces re-login everywhere.

MFA Flow
~~~~~~~~

:Enabled via: :meth:`AuthService.enable_mfa` — password re-auth + email notification.
:Disabled via: :meth:`AuthService.disable_mfa` — password + OTP verification.
:Verify during login: :meth:`AuthService.mfa_verify` — OTP + session token.

.. code-block:: python
   :caption: Step 1 — Login with MFA

   # First /v1/auth/login returns:
   {"requires_mfa": true, "mfa_session_token": "abc..."}
   # (MFA OTP is sent to email)

   # Step 2 — Complete MFA
   response = await client.post("/v1/auth/mfa/verify", json={
       "email": "admin@acme.com",
       "otp": "938472",
       "mfa_session_token": "abc...",
   })
   assert response.status_code == 200
   # → access_token, refresh_token

Password Reset Flow
~~~~~~~~~~~~~~~~~~~

:Endpoints: ``POST /v1/auth/forgot-password`` → ``POST /v1/auth/reset-password``
:Service: :meth:`AuthService.forgot_password` → :meth:`AuthService.reset_password`

.. code-block:: python
   :caption: Forgot password

   response = await client.post("/v1/auth/forgot-password", json={
       "email": "admin@acme.com",
   })
   # Always returns 200 (prevents email enumeration)
   assert response.json()["message"] == (
       "If an account exists with this email, "
       "a password reset code has been sent."
   )

   # Reset with OTP
   response = await client.post("/v1/auth/reset-password", json={
       "email": "admin@acme.com",
       "otp": "483926",
       "new_password": "n3w-s3cure-p@ss",
   })
   assert response.status_code == 200
   # All refresh tokens are revoked — user must log in again.

API Key Authentication Flow
~~~~~~~~~~~~~~~~~~~~~~~~~~~

:Middleware: :class:`AuthMiddleware` (``middleware/auth.py``)
:Caching: Redis — positive cache (TTL 300s), negative cache (TTL 60s)

.. mermaid::

   sequenceDiagram
       participant Client
       participant AuthMiddleware
       participant RedisCache
       participant AuthMiddleware as Middleware
       participant DB as PostgreSQL

       Client->>AuthMiddleware: GET /v1/sessions<br/>Authorization: Bearer oz_live_abc...
       AuthMiddleware->>AuthMiddleware: compute_lookup_hash(raw_key)
       AuthMiddleware->>RedisCache: GET auth:key:{lookup_hash}
       alt Cache hit
           RedisCache-->>AuthMiddleware: cached {org_id, scopes, ...}
           AuthMiddleware->>AuthMiddleware: set request.state
           AuthMiddleware-->>Client: → Route handler
       else Cache miss
           AuthMiddleware->>RedisCache: Check negative cache
           alt Negative cache hit
               AuthMiddleware-->>Client: 401 Invalid API Key
           else
               AuthMiddleware->>DB: SELECT * FROM api_keys WHERE lookup_hash=?
               alt Key found
                   AuthMiddleware->>AuthMiddleware: verify_api_key(salted_hash)
                   AuthMiddleware->>DB: UPDATE last_used_at (fire-and-forget)
                   AuthMiddleware->>RedisCache: SETEX auth:key:{hash} (300s)
                   AuthMiddleware->>AuthMiddleware: set request.state
                   AuthMiddleware-->>Client: → Route handler
               else Key not found
                   AuthMiddleware->>RedisCache: SETEX auth:neg:{hash} (60s)
                   AuthMiddleware-->>Client: 401 Invalid API Key
               end
           end
       end

**Rate limiting**: Per-IP auth miss-rate limiting — if an IP exceeds 10 DB
cache-misses per 60 seconds, further requests are rejected with HTTP 429.

**Key verification**: Uses ``SHA-256(salt || raw_key)``, then compares against
the stored salted hash.  The ``lookup_hash`` (unsalted SHA-256) is used only
for fast indexing and caching — it is **not** a security hash.

**Last-used tracking**: After successful API key auth, ``last_used_at`` is
updated fire-and-forget (exceptions are logged but do not block the request).


HTTP Endpoints
--------------

Auth Endpoints
~~~~~~~~~~~~~~

Router: ``routers/auth.py`` — prefix ``/v1/auth``, tag ``Authentication``

.. list-table::
   :header-rows: 1
   :widths: 8 20 20 12 40

   * - Method
     - Path
     - Summary
     - Auth
     - Description
   * - ``POST``
     - ``/signup``
     - Create org + admin user, send OTP
     - Public
     - See :ref:`Signup Flow <auth-architecture-flow>`
   * - ``POST``
     - ``/verify-email``
     - Verify email with OTP, return JWT
     - Public
     - See :ref:`Email Verification <auth-architecture-flow>`
   * - ``POST``
     - ``/resend-otp``
     - Resend verification OTP
     - Public
     - Rate-limited: 1 per 60s per email
   * - ``POST``
     - ``/login/otp/send``
     - Send passwordless login OTP
     - Public
     - Sends 6-digit code to email
   * - ``POST``
     - ``/login/otp/verify``
     - Verify login OTP, return JWT
     - Public
     - Auto-verifies email on first login
   * - ``POST``
     - ``/forgot-password``
     - Send password-reset OTP
     - Public
     - Returns same message whether email exists or not
   * - ``POST``
     - ``/reset-password``
     - Reset password with OTP
     - Public
     - Revokes all refresh tokens afterwards
   * - ``POST``
     - ``/login``
     - Authenticate by email/password
     - Public
     - Returns JWT or MFA challenge
   * - ``POST``
     - ``/mfa/verify``
     - Complete MFA login with OTP
     - Public
     - Second step of MFA login
   * - ``POST``
     - ``/mfa/enable``
     - Enable MFA
     - JWT
     - Requires password re-auth
   * - ``POST``
     - ``/mfa/disable``
     - Disable MFA
     - JWT
     - Requires password + OTP
   * - ``POST``
     - ``/refresh``
     - Rotate refresh token
     - Public
     - Returns new JWT pair
   * - ``GET``
     - ``/me``
     - Get dashboard user profile
     - JWT
     - Email, name, role, org, MFA status
   * - ``PATCH``
     - ``/me``
     - Update profile / change password
     - JWT
     - Optional fields; password change needs ``current_password``

API Key Endpoints
~~~~~~~~~~~~~~~~~

Router: ``routers/project_api_keys.py`` — prefix ``/v1/projects/{project_id}/api-keys``,
tag ``Project - API Keys``

.. list-table::
   :header-rows: 1
   :widths: 8 20 20 12 40

   * - Method
     - Path
     - Summary
     - Auth
     - Description
   * - ``GET``
     - ``/v1/projects/{project_id}/api-keys``
     - List project API keys
     - JWT + owner
     - Non-revoked keys only
   * - ``POST``
     - ``/v1/projects/{project_id}/api-keys``
     - Create project API key
     - JWT + owner
     - Returns raw key exactly once
   * - ``DELETE``
     - ``/v1/projects/{project_id}/api-keys/{key_id}``
     - Revoke project API key
     - JWT + owner
     - Soft-deletes; invalidates Redis cache

API Key Self-Service Endpoint
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Router: ``routers/api_key_self.py`` — prefix ``/v1/api-key``, tag ``API Key Self``

.. list-table::
   :header-rows: 1
   :widths: 8 20 20 12 40

   * - Method
     - Path
     - Summary
     - Auth
     - Description
   * - ``GET``
     - ``/project-id``
     - Resolve project ID for current API key
     - API Key or JWT
     - Returns ``{"project_id": "..."}`` or ``None`` for JWT

User Management Endpoints
~~~~~~~~~~~~~~~~~~~~~~~~~

Router: ``routers/users.py`` — prefix ``/v1/users``, tag ``Users``

.. list-table::
   :header-rows: 1
   :widths: 8 20 20 12 40

   * - Method
     - Path
     - Summary
     - Auth
     - Description
   * - ``POST``
     - ``/v1/users``
     - Create a new user
     - API Key / JWT
     - ``external_id`` unique per org
   * - ``GET``
     - ``/v1/users``
     - List users with pagination
     - API Key / JWT
     - Cursor-based, search, date filters
   * - ``GET``
     - ``/v1/users/{user_id}``
     - Get user with stats
     - API Key / JWT
     - Includes message, fact, session counts
   * - ``PATCH``
     - ``/v1/users/{user_id}``
     - Update user
     - API Key / JWT
     - Deep-merge metadata
   * - ``DELETE``
     - ``/v1/users/{user_id}``
     - Soft-delete user (GDPR two-phase)
     - API Key / JWT
     - Phase 2: hard purge after 30 days

All user endpoints require ``require_org_id`` — both JWT and API key
authentication are accepted.

Public Endpoints (No Auth Required)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following paths are defined in
:data:`middleware.auth.PUBLIC_ENDPOINTS` and pass through without
authentication:

* ``/health``, ``/ready``
* ``/docs``, ``/openapi.json``, ``/redoc``
* ``/v1/auth/signup``
* ``/v1/auth/login``
* ``/v1/auth/refresh``
* ``/v1/auth/verify-email``
* ``/v1/auth/resend-otp``
* ``/v1/auth/forgot-password``
* ``/v1/auth/reset-password``
* ``/v1/auth/login/otp/send``
* ``/v1/auth/login/otp/verify``
* ``/v1/auth/mfa/verify``
* ``/admin/organizations``, ``/admin/org/config/defaults``
* ``/metrics`` (exact path only)

The ``/metrics`` endpoint is handled by an exact-path check so that sub-paths
like ``/metrics/summary`` still require authentication.


Services
--------

AuthService
~~~~~~~~~~~

Module: ``services/auth_service.py``

The central orchestrator for all dashboard authentication flows.  Delegates
DB access to :class:`AuthRepository`, OTP operations to :class:`OtpService`,
and email delivery to :class:`EmailService`.

.. class:: AuthService(repo, otp_service, redis, email_service=None)

   :param AuthRepository repo: Repository for auth-related DB access.
   :param OtpService otp_service: OTP generation and verification.
   :param redis.asyncio.Redis redis: Async Redis client (MFA sessions, OTP storage).
   :param EmailService | None email_service: Optional email service for
       notification-only emails (e.g. password-change confirmation).

   .. method:: signup(payload)

      :param SignupRequest payload: Email, password, and organisation name.
      :returns: ``SignupResponse`` with confirmation message.
      :raises ConflictError: Email already registered.
      :raises ValidationError: Password too weak.

      See :ref:`Signup Flow` for details.

   .. method:: verify_email(payload)

      :param VerifyEmailRequest payload: Email and OTP code.
      :returns: ``TokenResponse`` with JWT pair.
      :raises AuthenticationError: Invalid/expired OTP.
      :raises NotFoundError: User not found.

   .. method:: resend_verification(email)

      :param str email: Registered email address.
      :returns: ``SignupResponse`` confirming code was sent.
      :raises NotFoundError: No user with this email.

   .. method:: forgot_password(email)

      :param str email: Registered email address.
      :returns: ``OtpResponse`` confirming code was sent.
      :raises ValidationError: No account found.

      Returns the same message whether or not the email exists (prevents
      email enumeration — the ``ValidationError`` is raised before the
      response is constructed when ``password_hash`` is ``None``).

   .. method:: reset_password(payload)

      :param ResetPasswordRequest payload: Email, OTP, new password.
      :returns: ``OtpResponse`` confirming password changed.
      :raises NotFoundError: User not found.
      :raises AuthenticationError: Invalid/expired OTP.
      :raises ValidationError: New password too weak.

      After success, all existing refresh tokens are revoked, forcing
      re-login everywhere.

   .. method:: generate_login_otp(email)

      :param str email: Registered email address.
      :returns: ``OtpResponse`` confirming code was sent.
      :raises NotFoundError: User not found.

   .. method:: passwordless_login(payload)

      :param VerifyOtpRequest payload: Email and OTP code.
      :returns: ``TokenResponse`` with JWT pair.
      :raises NotFoundError: User not found.
      :raises AuthenticationError: Invalid/expired OTP.

      Auto-verifies email if this is the user's first login.

   .. method:: login(payload)

      :param LoginRequest payload: Email and password.
      :returns: ``LoginResponse`` — tokens (MFA off) or MFA challenge.
      :raises AuthenticationError: Invalid credentials, inactive account,
          or unverified email.

   .. method:: mfa_verify(payload)

      :param MfaVerifyRequest payload: Email, OTP, MFA session token.
      :returns: ``TokenResponse`` with JWT pair.
      :raises AuthenticationError: Invalid/expired session or OTP.

   .. method:: enable_mfa(user_id, payload)

      :param UUID user_id: Authenticated user UUID.
      :param MfaEnableRequest payload: Current password.
      :returns: ``OtpResponse`` confirming MFA enabled.
      :raises NotFoundError: User not found.
      :raises AuthenticationError: Incorrect password.

   .. method:: disable_mfa(user_id, payload)

      :param UUID user_id: Authenticated user UUID.
      :param MfaDisableRequest payload: Current password + MFA OTP.
      :returns: ``OtpResponse`` confirming MFA disabled.
      :raises NotFoundError: User not found.
      :raises AuthenticationError: Incorrect password or invalid OTP.

   .. method:: refresh(raw_token)

      :param str raw_token: The opaque refresh token string.
      :returns: ``TokenResponse`` with fresh tokens.
      :raises AuthenticationError: Invalid or expired token.

      Implements rotation: the old token is revoked and linked to the new
      one via ``rotated_by``.

   .. method:: get_profile(user_id)

      :param UUID user_id: Authenticated user UUID.
      :returns: ``DashboardUserResponse`` with public profile data.
      :raises NotFoundError: User not found.

   .. method:: update_profile(user_id, payload)

      :param UUID user_id: Authenticated user UUID.
      :param UpdateProfileRequest payload: Fields to update.
      :returns: Updated ``DashboardUserResponse``.
      :raises NotFoundError: User not found.
      :raises ValidationError: Password change without current password.
      :raises ConflictError: New email already taken.

      Password change sends a notification email (best-effort — failures
      are logged but do not block the operation).

   .. method:: _validate_password(password)

      :param str password: The plaintext password.
      :raises ValidationError: If shorter than 8 characters, missing
          uppercase, lowercase, or digit.

   .. method:: _issue_tokens(user_id, organization_id, role)

      :param UUID user_id: Authenticated user UUID.
      :param UUID organization_id: User's organization UUID.
      :param str role: User role for JWT claims.
      :returns: ``TokenResponse`` with fresh JWT pair.

      Internal helper.  Creates JWT access token with ``sub``, ``org_id``,
      ``role``, and ``type`` claims.  Generates opaque refresh token,
      stores its SHA-256 hash, and returns both.

   .. method:: _hash_refresh_token(raw)

      :param str raw: The opaque refresh token string.
      :returns: Hex-encoded SHA-256 digest.

ApiKeyService
~~~~~~~~~~~~~

Module: ``services/api_key_service.py``

Business logic for project-scoped API key lifecycle management.

.. class:: ApiKeyService(repo, redis=None)

   :param ApiKeyRepository repo: API key repository.
   :param redis.asyncio.Redis | None redis: Optional Redis client for
       auth cache invalidation.

   .. method:: create_project_key(organization_id, project_id, payload, created_by=None)

      :param UUID organization_id: Owning organization.
      :param UUID project_id: Project scope.
      :param CreateApiKeyRequest payload: Key name.
      :param UUID | None created_by: Optional user UUID (from JWT session).
      :returns: ``(ApiKey, raw_key_string)`` tuple.

      Generates a cryptographically random key (48 CSPRNG bytes, base62
      encoded, ``oz_live_`` prefix), hashes it with a random 16-byte salt,
      computes an unsalted lookup hash, and persists all three.  The raw
      key string is returned exactly once.

   .. method:: list_project_keys(organization_id, project_id)

      :param UUID organization_id: Owning organization.
      :param UUID project_id: Project scope.
      :returns: List of ``ApiKey`` records, newest first, excluding revoked.

   .. method:: revoke_project_key(organization_id, project_id, key_id)

      :param UUID organization_id: Owning organization.
      :param UUID project_id: Project scope.
      :param UUID key_id: API key UUID.
      :returns: The revoked ``ApiKey``, or ``None`` if not found.
      :raises: Does not raise — returns ``None`` for not-found.

      Sets ``is_revoked = True``, then invalidates the Redis auth cache
      (both positive and negative cache entries) so the key is rejected
      on the next request.

OTPService
~~~~~~~~~~

Module: ``services/otp_service.py``

One-time passcode lifecycle — generate, send, verify, invalidate.  All
state lives in Redis with auto-expiring keys.  No DB persistence.

.. class:: OtpService(redis, email_service)

   :param redis.asyncio.Redis redis: Async Redis client.
   :param EmailService email_service: Email delivery service.

   OTPs are 6-digit numeric codes generated via ``secrets.randbelow``.
   Only SHA-256 hashes are stored in Redis — never plaintext.
   Verification uses ``hmac.compare_digest`` for constant-time comparison.

   **Redis key schema:**

   ======================================================  ========  =====
   Key                                                     Type      TTL
   ======================================================  ========  =====
   ``otp:v1:{purpose}:{email}:hash``                       SHA-256   600s
   ``otp:v1:{purpose}:{email}:attempts``                   counter   600s
   ``otp:v1:{purpose}:{email}:cooldown``                   flag      60s
   ``email:send_count:{email}:{yyyymmddhh}``               counter   3600s
   ======================================================  ========  =====

   Valid purposes (``OtpPurpose``): ``"signup"``, ``"password_reset"``,
   ``"passwordless_login"``, ``"mfa"``.

   **Rate limits:**

   * Max 5 sends per email per rolling hour.
   * Min 60s cooldown between sends.
   * Max 5 failed attempts per OTP (invalidated afterwards).

   .. method:: generate_and_send(email, purpose)

      :param str email: Recipient email.
      :param OtpPurpose purpose: Purpose scope.
      :raises RateLimitError: Send limit exceeded or cooldown active.

   .. method:: verify(email, purpose, code)

      :param str email: Email the OTP was sent to.
      :param OtpPurpose purpose: Purpose scope.
      :param str code: Plaintext OTP code.
      :returns: ``True`` if valid, ``False`` otherwise.
      :raises ValidationError: Attempt limit exceeded.

      On success, the OTP hash is deleted (single-use).

   .. method:: invalidate(email, purpose)

      :param str email: Email to invalidate.
      :param OtpPurpose purpose: Purpose scope.

      Removes all OTP-related keys (hash, attempts, cooldown).

UserService
~~~~~~~~~~~

Module: ``services/user_service.py``

Business logic for end-user (SDK) management — create, get-or-create,
update, soft-delete, and list with cursor-based pagination.

.. class:: UserService(repo, webhook_service=None)

   :param UserRepository repo: User repository.
   :param WebhookService | None webhook_service: Optional webhook emitter.

   .. method:: create_user(organization_id, external_id, name=None, email=None, metadata=None)

      :param UUID organization_id: Tenant scope.
      :param str external_id: Caller-defined unique user identifier.
      :param str | None name: Optional display name.
      :param str | None email: Optional email.
      :param dict | None metadata: Optional JSON metadata.
      :returns: ``UserResponse``.
      :raises ConflictError: External ID already exists in this org.
      :raises: Emits ``EventType.USER_CREATED`` webhook.

   .. method:: get_or_create_user(organization_id, external_id, name=None, email=None, metadata=None)

      :param UUID organization_id: Tenant scope.
      :param str external_id: Caller-defined unique user identifier.
      :param str | None name: Used only when creating.
      :param str | None email: Used only when creating.
      :param dict | None metadata: Used only when creating.
      :returns: ``UserResponse`` — pre-existing or newly created.
      :raises NotFoundError: Should never happen (DB inconsistency).

      Thread-safe via the ``(organization_id, external_id)`` unique
      constraint.  If two concurrent calls race, one raises
      ``IntegrityError``, which is caught and resolved by refetching.

   .. method:: get_user(organization_id, user_id)

      :param UUID organization_id: Tenant scope.
      :param UUID user_id: Internal user UUID.
      :returns: ``UserResponseWithStats`` with profile + counts.
      :raises NotFoundError: User not found or soft-deleted.

   .. method:: update_user(organization_id, user_id, update_fields)

      :param UUID organization_id: Tenant scope.
      :param UUID user_id: Internal user UUID.
      :param dict update_fields: Only fields the client explicitly set.
      :returns: Updated ``UserResponse``.
      :raises NotFoundError: User not found.

      Uses ``model_dump(exclude_unset=True)`` semantics: a key with
      value ``None`` means "set to null"; an absent key means "do not
      update".  Metadata is deep-merged, not replaced.

   .. method:: delete_user(organization_id, user_id)

      :param UUID organization_id: Tenant scope.
      :param UUID user_id: Internal user UUID.
      :raises NotFoundError: User not found.

      Sets ``is_deleted = True`` immediately.  Phase 2 (GDPR hard purge
      after 30 days) is a stub — see ``TODO(phase2)`` in the source.

   .. method:: list_users(organization_id, limit=50, cursor=None, search=None, created_after=None, created_before=None)

      :param UUID organization_id: Tenant scope.
      :param int limit: Max results per page (1-200).
      :param str | None cursor: Opaque pagination cursor.
      :param str | None search: Fuzzy match against external_id, name,
          email, metadata.
      :param datetime | None created_after: ISO-8601 timestamp filter.
      :param datetime | None created_before: ISO-8601 timestamp filter.
      :returns: ``UserListResponse``.

ProjectService
~~~~~~~~~~~~~~

Module: ``services/project_service.py``

Orchestrates project and project member lifecycle.

.. class:: ProjectService(repo)

   :param ProjectRepository repo: Project repository.

   .. method:: create_project(organization_id, user_id, payload)

      :param UUID organization_id: Tenant scope.
      :param UUID | None user_id: Creator (becomes initial owner), or
          ``None`` for API-key-authenticated requests.
      :param CreateProjectRequest payload: Name, description, metadata.
      :returns: ``ProjectResponse``.
      :raises ValidationError: Duplicate project name in org.

   .. method:: get_project(organization_id, project_id)

      :returns: ``ProjectResponse``.
      :raises NotFoundError: Project not found.

   .. method:: list_projects(organization_id, user_id, limit=50, offset=0)

      :param UUID | None user_id: When provided, only projects where the
          user is a member.  ``None`` returns all non-archived projects
          in the org.

   .. method:: add_member(project_id, payload)

      :param AddMemberRequest payload: User ID and role.
      :raises ValidationError: User already a member.

   .. method:: remove_member(project_id, user_id)

      :raises ValidationError: Cannot remove the last owner.
      :raises NotFoundError: Membership not found.

   .. method:: update_member_role(project_id, user_id, role)

      :raises ValidationError: Cannot downgrade the last owner.
      :raises NotFoundError: Membership not found.


PII Detection & Redaction Service
---------------------------------

Module: ``services/pii_service.py``

A two-layer PII detection and redaction service integrated into the memory
ingestion flow.  Configuration is read from the organisation's
``quotas["pii"]`` dict.

.. class:: PIIService(config=None)

   :param dict | None config: PII configuration from
       ``organizations.quotas["pii"]``.  Keys: ``"mode"`` (``"off"``,
       ``"mask"``, ``"block"``), ``"enabled_types"``, ``"min_confidence"``,
       ``"sensitivity"``.

   .. attribute:: mode
      :type: str

      The PII processing mode — ``"off"``, ``"mask"``, or ``"block"``.

   .. method:: process_message(content)

      :param str content: The raw message content to process.
      :returns: ``(redacted_content, detections, was_blocked)`` tuple.
      :raises ValidationError: If mode is ``"block"`` and PII was detected.

      **Modes:**

      * ``"off"`` — pass-through, no detection. Returns ``(content, [], False)``.
      * ``"mask"`` — replaces detected PII spans with ``[REDACTED:TYPE]``.
      * ``"block"`` — rejects messages containing PII with ``ValidationError``.

.. class:: PIIDetector(enabled_types=None, min_confidence=0.7, use_ner=True)

   Multi-layer PII detector with regex and optional spaCy NER.

   .. method:: detect(text)

      :param str text: Text to scan.
      :returns: List of :class:`PIIDetection` results, sorted by start offset.
      :raises ExternalServiceError: If NER is enabled but spaCy or model
          is unavailable.

   **Layer 1 — Regex (always runs):** Compiled patterns for emails, phone
   numbers, SSNs, credit cards, IP addresses, API keys (OpenAI, GitHub, AWS),
   and crypto wallet addresses (Ethereum, Bitcoin).  Regex detections have a
   confidence of 0.95.

   **Layer 2 — spaCy NER (optional):** Lazy-loaded ``en_core_web_sm`` model.
   Detects person names (``PERSON``), locations (``GPE``, ``LOC``),
   organisations (``ORG``), and dates (``DATE``).  Mapped to OpenZync PII
   types via :attr:`NER_LABEL_MAP`.  NER detections have a confidence of
   0.85.

   Layer sensitivity is controlled by the ``sensitivity`` config key:
   ``"low"`` disables NER, ``"medium"`` and ``"high"`` enable it.

.. class:: PIIRedactor(mode="mask")

   .. method:: apply(text, detections)

      :param str text: Original text.
      :param list[PIIDetection] detections: Detections to apply.
      :returns: Redacted text with PII replaced by ``[REDACTED:TYPE]``.
      :raises ValueError: If mode is ``"block"``.

      Detections are processed in reverse order (by start position) to
      preserve character offsets.

.. class:: PIIDetection(type, value, start, end, confidence, method="regex")

   A single PII detection result (frozen dataclass).

   :param str type: PII type identifier (``"email"``, ``"ssn"``, etc.).
   :param str value: Detected PII value (**NEVER logged** — redacted in
       ``__repr__``).
   :param int start: Character offset where detection begins.
   :param int end: Character offset where detection ends.
   :param float confidence: Detection confidence (0.0 to 1.0).
   :param str method: Detection method — ``"regex"`` or ``"spacy_ner"``.

.. rubric:: Recognised PII Types:

.. list-table::
   :header-rows: 1
   :widths: 20 20 20 40

   * - Type
     - Label
     - Method
     - Example Pattern
   * - ``"email"``
     - ``EMAIL``
     - Regex
     - ``user@example.com``
   * - ``"phone"``
     - ``PHONE``
     - Regex
     - ``+1-555-123-4567``
   * - ``"ssn"``
     - ``SSN``
     - Regex
     - ``123-45-6789``
   * - ``"credit_card"``
     - ``CARD``
     - Regex
     - ``4111-1111-1111-1111``
   * - ``"ip_address"``
     - ``IP``
     - Regex
     - ``192.168.1.1``
   * - ``"api_key"``
     - ``KEY``
     - Regex
     - ``sk-proj-...``, ``ghp_...``, ``AKIA...``
   * - ``"crypto_wallet"``
     - ``WALLET``
     - Regex
     - ``0x...`` (ETH), ``bc1...`` (BTC)
   * - ``"name"``
     - ``NAME``
     - NER
     - Person names from spaCy
   * - ``"address"``
     - ``ADDRESS``
     - NER
     - GPE/LOC entities from spaCy
   * - ``"organization"``
     - ``ORG``
     - NER
     - Organisation names from spaCy
   * - ``"date"``
     - ``DATE``
     - NER
     - Date entities from spaCy

Real-world usage in the ingestion pipeline::

    # Each org has a PII config in its quotas dict
    org = await repo.get_organization(org_id)
    pii_config = org.quotas.get("pii", {})

    pii_service = PIIService(pii_config)

    # The mode is "mask" — PII is redacted, not blocked
    redacted, detections, blocked = await pii_service.process_message(
        "Contact John at john@example.com or +1-555-123-4567"
    )
    # redacted == "Contact [REDACTED:NAME] at [REDACTED:EMAIL] or [REDACTED:PHONE]"
    # detections contains 3 PIIDetection objects
    # blocked == False

PII values are **never logged** — only type counts and detection metadata
are recorded in log entries.


Middleware
----------

AuthMiddleware
~~~~~~~~~~~~~~

Module: ``middleware/auth.py``

The primary authentication guard for all HTTP requests.  Runs as raw ASGI
middleware (no ``BaseHTTPMiddleware`` overhead) and sets ``scope["state"]``
for downstream middleware and route handlers.

.. class:: AuthMiddleware(app)

   :param ASGIApp app: The ASGI application.

   Authenticates each HTTP request via the ``Authorization: Bearer <token>``
   header.  Supports two modes:

   **JWT (dashboard users):**

   #. Token has exactly two dots (``_is_jwt_token`` heuristic).
   #. Signature verified via ``verify_jwt_token()`` with ``SECRET_KEY`` (HS256).
   #. Required claims: ``sub`` (user_id), ``org_id``, ``type`` (must be
      ``"access"``).
   #. Sets ``request.state``: ``auth_type="jwt"``, ``user_id``, ``org_id``,
      ``role``, ``api_key_scopes=["read", "write", "admin"]``.

   **API Key (SDK clients):**

   #. Token starts with ``oz_live_`` or ``oz_test_``.
   #. Computes ``compute_lookup_hash`` (unsalted SHA-256).
   #. Checks Redis positive cache (``auth:key:{lookup_hash}``, TTL 300s).
   #. On miss, checks negative cache (``auth:neg:{lookup_hash}``, TTL 60s).
   #. On negative miss, queries the database via ``ApiKeyRepository.get_by_lookup_hash``.
   #. Verifies the salted SHA-256 hash.
   #. Checks revocation and expiration.
   #. Sets ``request.state``: ``auth_type="api_key"``, ``org_id``,
      ``api_key_scopes``, ``api_key_project_id``, ``user_id`` (from
      ``created_by``).
   #. Updates ``last_used_at`` (fire-and-forget).
   #. Caches the result in Redis (fire-and-forget).

   **Public endpoints** (defined in :data:`PUBLIC_ENDPOINTS`) pass through
   without authentication.  Prometheus ``/metrics`` is handled by an
   exact-path check.

   All error responses use **RFC 7807** (Problem Details) format with
   content type ``application/problem+json``.

   .. rubric:: Constants

   .. list-table::
      :header-rows: 1

      * - Constant
        - Value
        - Description
      * - ``AUTH_CACHE_PREFIX``
        - ``"auth:key:"``
        - Redis prefix for cached auth data
      * - ``AUTH_CACHE_TTL``
        - ``300``
        - Positive cache TTL (5 minutes)
      * - ``AUTH_NEG_CACHE_PREFIX``
        - ``"auth:neg:"``
        - Redis prefix for negative cache entries
      * - ``AUTH_NEG_CACHE_TTL``
        - ``60``
        - Negative cache TTL (1 minute)
      * - ``AUTH_MISS_RATE_LIMIT_PREFIX``
        - ``"auth:miss_ip:"``
        - Redis prefix for per-IP miss-rate counters
      * - ``AUTH_MISS_RATE_LIMIT``
        - ``10``
        - Max DB auth misses per IP per window
      * - ``AUTH_MISS_RATE_WINDOW``
        - ``60``
        - Miss-rate sliding window (seconds)
      * - ``API_KEY_PREFIXES``
        - ``("oz_live_", "oz_test_")``
        - Recognised API key prefixes

AuthThrottle
~~~~~~~~~~~~

Module: ``middleware/auth_throttle.py``

Redis-backed rate limiting for public authentication endpoints.  Protects
against brute-force, credential-stuffing, and OTP-guessing attacks.

.. class:: AuthThrottle(redis, login_max_per_ip=20, login_window_sec=900, \
                        login_max_per_email=5, signup_max_per_ip=3, signup_window_sec=3600)

   :param redis.asyncio.Redis redis: Async Redis client.
   :param int login_max_per_ip: Max failed logins per IP (default 20).
   :param int login_window_sec: Login throttle window (default 900s = 15min).
   :param int login_max_per_email: Max failed logins per email (default 5).
   :param int signup_max_per_ip: Max signups per IP (default 3).
   :param int signup_window_sec: Signup throttle window (default 3600s = 1h).

   .. rubric:: Rate Limit Summary

   ==============================  ===================  ============  ========================
   Endpoint                        Per-Email            Per-IP        Window
   ==============================  ===================  ============  ========================
   ``/v1/auth/login``              5                    20            15 min
   ``/v1/auth/signup``             N/A                  3             1 hour
   ``/v1/auth/verify-email``       10                   20            15 min
   ``/v1/auth/forgot-password``    3                    10            1 hour / 15 min
   ``/v1/auth/reset-password``     10                   20            15 min
   ``/v1/auth/login/otp/send``     5                    10            1 hour / 15 min
   ``/v1/auth/login/otp/verify``   10                   20            15 min
   ==============================  ===================  ============  ========================

   All methods raise :class:`RateLimitError` when the limit is exceeded.


FastAPI Dependencies
--------------------

Module: ``dependencies/auth.py``

Six dependency levels for use in route handlers:

.. function:: get_org_id(request, credentials=None)

   :param Request request: Incoming HTTP request.
   :param credentials: Used only for OpenAPI schema generation.
   :type credentials: HTTPAuthorizationCredentials | None
   :returns: Organization UUID string, or ``None`` if not authenticated.
   :rtype: str | None

   Optional auth.  Works with both API keys and JWT tokens.

.. function:: require_org_id(org_id)

   :param org_id: From ``get_org_id``.
   :type org_id: str | None
   :returns: Authenticated organization UUID string.
   :raises HTTPException 401: If not authenticated.

   Mandatory auth.  Works with both API keys and JWT tokens.

.. function:: require_scope(required_scope)

   :param str required_scope: The scope the API key must possess (e.g.
       ``"admin:write"``, ``"sessions:read"``).
   :returns: A dependency callable that returns ``org_id``.
   :raises HTTPException 403: If the API key lacks the required scope.
   :raises HTTPException 401: If not authenticated.

   Dependency factory.  JWT-authenticated dashboard users implicitly have
   all scopes.

   Usage::

       @router.post("/admin/orgs")
       async def admin_action(
           org_id: str = Depends(require_scope("admin:write")),
       ):
           ...

.. function:: get_dashboard_user(request, org_id)

   :param Request request: Incoming HTTP request.
   :param str org_id: From ``require_org_id``.
   :returns: Dashboard user UUID string.
   :raises HTTPException 401: If not a JWT-authenticated session.

   Requires JWT authentication (dashboard session).  Rejects API key auth.

.. function:: get_current_user_id(request, org_id)

   :param Request request: Incoming HTTP request.
   :param str org_id: From ``require_org_id``.
   :returns: Authenticated user UUID.
   :rtype: UUID
   :raises HTTPException 401: If user not authenticated.

   Works with both JWT and API key auth.  Returns the user UUID (for JWT)
   or the API key's ``created_by`` value.

Module: ``dependencies/project_auth.py``

Project-scoped auth guards:

.. function:: require_project_membership(request, project_id, db)

   :param Request request: Incoming HTTP request.
   :param UUID project_id: From URL path.
   :param AsyncSession db: Database session.
   :raises HTTPException 401: If not authenticated.
   :raises HTTPException 403: If not a project member (JWT) or wrong
       project scope (API key).
   :raises HTTPException 404: If project does not exist.

   Dual-mode auth guard for project-scoped endpoints.  For API keys,
   verifies the key's ``project_id`` matches the request URL.  For JWT,
   verifies the user is a ``ProjectMember``.

.. function:: require_project_owner(request, project_id, db)

   :param Request request: Incoming HTTP request.
   :param UUID project_id: From URL path.
   :param AsyncSession db: Database session.
   :raises HTTPException 401: If not authenticated.
   :raises HTTPException 403: If not an owner or API key used.
   :raises HTTPException 404: If project does not exist.

   JWT-only (rejects API keys).  Requires the authenticated dashboard user
   to have the ``"owner"`` role in the project's members.


Utility Modules
---------------

Password Hashing
~~~~~~~~~~~~~~~~

Module: ``utils/password.py``

.. function:: hash_password(password)

   :param str password: Plaintext password.
   :returns: 60-character bcrypt hash string.
   :raises ValueError: If password is empty.

.. function:: verify_password(password, hashed)

   :param str password: Plaintext password to check.
   :param str hashed: Stored bcrypt hash.
   :returns: ``True`` if the password matches.
   :rtype: bool

Cryptographic Utilities
~~~~~~~~~~~~~~~~~~~~~~~

Module: ``utils/crypto.py``

.. function:: generate_api_key(prefix="oz_live_")

   :param str prefix: Key prefix (``"oz_live_"`` or ``"oz_test_"``).
   :returns: Full API key string, e.g. ``"oz_live_3Ab9...kQ7"`` (~73 chars).

   Generates 48 CSPRNG bytes encoded as base62.

.. function:: hash_api_key(raw_key)

   :param str raw_key: Full API key string.
   :returns: ``(hex_hash, hex_salt)`` tuple.
       ``hex_hash``: 64-char SHA-256 hex string.
       ``hex_salt``: 32-char random hex string (16 bytes).

   Uses ``SHA-256(salt || raw_key)``.

.. function:: verify_api_key(raw_key, stored_hash, salt)

   :param str raw_key: Full API key string from client.
   :param str stored_hash: 64-char hex hash from DB.
   :param str salt: 32-char hex salt from DB.
   :returns: ``True`` if the key matches.

.. function:: compute_lookup_hash(raw_key)

   :param str raw_key: Full API key string.
   :returns: 64-char hex string (unsalted SHA-256).

   For fast indexing/caching only — **not** a security hash.

.. function:: create_jwt_token(data, secret, expires_delta)

   :param dict data: Payload claims (``{"sub": ..., "org_id": ..., ...}``).
   :param str secret: HMAC secret key (min 32 chars).
   :param timedelta expires_delta: Relative duration until expiry.
   :returns: Encoded JWT string (three base64url segments).
   :raises PyJWTError: If encoding fails.

   Adds ``exp`` and ``iat`` claims automatically.

.. function:: verify_jwt_token(token, secret)

   :param str token: Encoded JWT string.
   :param str secret: HMAC secret key.
   :returns: Decoded payload dict.
   :raises AuthenticationError: If expired or invalid token.

.. function:: base62_encode(num)

   :param int num: Non-negative integer.
   :returns: Base62-encoded string (character set: ``0-9A-Za-z``).
   :raises ValueError: If ``num`` is negative.


Exception Classes
-----------------

The following exceptions are raised by auth-related code (all defined in
``core/exceptions.py``):

.. list-table::
   :header-rows: 1
   :widths: 30 10 60

   * - Exception
     - Status
     - Raised When
   * - :class:`AuthenticationError`
     - 401
     - Invalid credentials, expired/revoked token, wrong token type
   * - :class:`AuthorizationError`
     - 403
     - Insufficient API key scopes or project membership
   * - :class:`NotFoundError`
     - 404
     - User or token not found
   * - :class:`ConflictError`
     - 409
     - Email or ``external_id`` already taken
   * - :class:`ValidationError`
     - 422
     - Weak password, invalid OTP attempt limit
   * - :class:`RateLimitError`
     - 429
     - Too many login/signup/OTP attempts
   * - :class:`ExternalServiceError`
     - 502
     - spaCy model unavailable, email delivery failure

All error responses follow **RFC 7807** (Problem Details).  See
:class:`register_exception_handlers` in ``core/exceptions.py`` for
registration details.


Repositories
------------

AuthRepository
~~~~~~~~~~~~~~

Module: ``repositories/auth_repository.py``

All database access for dashboard authentication flows.  No business logic
— pure query construction and execution.

.. class:: AuthRepository(db)

   :param AsyncSession db: SQLAlchemy async session.

   .. method:: find_user_by_email(email)

      :param str email: User's email address.
      :returns: ``User`` with ``password_hash`` set, or ``None``.

      Global lookup (email is globally unique across all organisations).

   .. method:: get_user_by_id(user_id)

      :param UUID user_id: Internal user UUID.
      :returns: ``User`` if found, or ``None``.

   .. method:: create_organization(name, plan="free")

      :param str name: Organisation name.
      :param str plan: Billing plan (default ``"free"``).
      :returns: Newly created ``Organization``.

   .. method:: seed_prompts_for_org(org_id)

      :param UUID org_id: Organisation UUID.
      :returns: Number of prompt templates seeded.

      Reads from ``services/worker/prompts/manifest.yaml`` + ``.jinja2``
      files via :class:`PromptTemplateRepository`.

   .. method:: create_dashboard_user(organization_id, email, password_hash, name=None, role="admin")

      :param UUID organization_id: Owning organisation.
      :param str email: Email address.
      :param str password_hash: bcrypt hash.
      :param str | None name: Optional display name.
      :param str role: Role string (``"admin"`` or ``"member"``).
      :returns: Newly created ``User``.

      Sets ``external_id = email``.

   .. method:: create_refresh_token(user_id, organization_id, token_hash, expires_at)

      :param UUID user_id: Authenticated user UUID.
      :param UUID organization_id: User's organisation UUID.
      :param str token_hash: SHA-256 hash of opaque token.
      :param datetime expires_at: Expiration (naive UTC).
      :returns: Newly created ``RefreshToken``.

   .. method:: find_refresh_token(token_hash)

      :param str token_hash: SHA-256 hex digest.
      :returns: ``RefreshToken`` if found and not revoked and not expired,
          or ``None``.

   .. method:: revoke_refresh_token(token_id, rotated_by=None)

      :param UUID token_id: Token UUID to revoke.
      :param str | None rotated_by: UUID of replacement token.

   .. method:: revoke_all_refresh_tokens(user_id)

      :param UUID user_id: User UUID.

      Called after password reset to invalidate all sessions.

   .. method:: mark_email_verified(user_id)

      :param UUID user_id: User UUID.
      :returns: Updated ``User``.
      :raises NotFoundError: User not found.

   .. method:: reset_email_verification(user_id)

      :param UUID user_id: User UUID.
      :returns: Updated ``User`` with ``is_email_verified=False``.
      :raises NotFoundError: User not found.

   .. method:: set_mfa_enabled(user_id, enabled)

      :param UUID user_id: User UUID.
      :param bool enabled: ``True`` to enable, ``False`` to disable.
      :returns: Updated ``User``.
      :raises NotFoundError: User not found.

   .. method:: update_dashboard_user(user_id, name=None, email=None, password_hash=None)

      :param UUID user_id: User UUID.
      :param str | None name: New name (``None`` = no change).
      :param str | None email: New email (also updates ``external_id``).
      :param str | None password_hash: New bcrypt hash.
      :returns: Updated ``User``.
      :raises NotFoundError: User not found.

ApiKeyRepository
~~~~~~~~~~~~~~~~

Module: ``repositories/api_key_repository.py``

.. class:: ApiKeyRepository(db)

   :param AsyncSession db: SQLAlchemy async session.

   .. method:: list_by_org(organization_id, include_revoked=False, project_id=None)

      :param UUID organization_id: Tenant scope.
      :param bool include_revoked: If ``True``, include revoked keys.
      :param UUID | None project_id: Filter by project.
      :returns: All matching ``ApiKey`` records, newest first.

   .. method:: get_by_id(organization_id, key_id, project_id=None)

      :returns: ``ApiKey`` if found, or ``None``.

   .. method:: create(organization_id, lookup_hash, key_hash, salt, prefix, name, scopes=None, project_id=None, created_by=None)

      :returns: Newly created ``ApiKey``.

   .. method:: revoke(organization_id, key_id, project_id=None)

      :param UUID organization_id: Tenant scope.
      :param UUID key_id: Key to revoke.
      :param UUID | None project_id: Additional scope filter.
      :returns: Revoked ``ApiKey``, or ``None`` if not found.

   .. method:: get_by_lookup_hash(lookup_hash)

      :param str lookup_hash: Unsalted SHA-256 hex digest.
      :returns: ``ApiKey`` if found and not revoked.

      Used during API key authentication — the incoming key is hashed with
      SHA-256 and matched against ``lookup_hash`` for fast candidate lookup.

   .. method:: update_last_used(key_id)

      :param UUID key_id: API key UUID.

      Called after successful API key authentication.

UserRepository
~~~~~~~~~~~~~~

Module: ``repositories/user_repository.py``

See full documentation in the User Service section above; this module provides
query methods for ``create``, ``get_by_external_id``, ``get_by_uuid``,
``update``, ``soft_delete``, ``hard_delete``, ``list`` (cursor-paginated),
``exists_by_external_id``, and ``get_stats``.


Schemas (Pydantic)
------------------

Auth Schemas
~~~~~~~~~~~~

Module: ``schemas/auth.py``

.. class:: SignupRequest

   :param EmailStr email: Admin user email (``admin@acme.com``).
   :param str password: Min 8 chars, max 128.
   :param str organization_name: Min 1, max 255.

.. class:: SignupResponse

   :param str message: Confirmation message.
   :param EmailStr email: Email the verification code was sent to.

.. class:: VerifyEmailRequest

   :param EmailStr email: Email the OTP was sent to.
   :param str otp: 6-digit code (4-8 chars).

.. class:: LoginRequest

   :param EmailStr email: Dashboard user email.
   :param str password: Plaintext password.

.. class:: TokenResponse

   :param str access_token: JWT access token.
   :param str refresh_token: Opaque refresh token.
   :param int expires_in: Access token TTL in seconds.
   :param str token_type: Always ``"Bearer"``.

.. class:: RefreshRequest

   :param str refresh_token: The refresh token from login.

.. class:: DashboardUserResponse

   :param UUID id: User UUID.
   :param str email: Email address.
   :param str | None name: Display name.
   :param str role: User role (``"admin"``, ``"member"``).
   :param UUID organization_id: Owning organisation ID.
   :param bool is_email_verified: Email verification status.
   :param bool mfa_enabled: MFA status.

   ``model_config = ConfigDict(from_attributes=True)``.

.. class:: LoginResponse

   :param str | None access_token: JWT (``None`` when MFA required).
   :param str | None refresh_token: Opaque token (``None`` when MFA required).
   :param int | None expires_in: TTL in seconds.
   :param str | None token_type: ``"Bearer"``.
   :param bool requires_mfa: Whether MFA verification is needed.
   :param str | None mfa_session_token: Session key for MFA step 2.

.. class:: MfaVerifyRequest

   :param EmailStr email:
   :param str otp: 6-digit MFA code.
   :param str mfa_session_token: Session token from login response.

.. class:: MfaEnableRequest

   :param str password: Current password (re-authentication).

.. class:: MfaDisableRequest

   :param str password: Current password.
   :param str otp: MFA OTP.

.. class:: UpdateProfileRequest

   All fields optional:

   :param str | None name: New name (``None`` to clear).
   :param str | None email: New email (max 320).
   :param str | None current_password: Required when setting new password.
   :param str | None new_password: Min 8, max 128.

Email Schemas
~~~~~~~~~~~~~

Module: ``schemas/email.py``

.. class:: SendOtpRequest

   :param EmailStr email: Email to receive the OTP.

.. class:: VerifyOtpRequest

   :param EmailStr email: Email the OTP was sent to.
   :param str otp: 6-digit code (4-8 chars).

.. class:: ResetPasswordRequest

   :param EmailStr email: Email the OTP was sent to.
   :param str otp: 6-digit code.
   :param str new_password: Min 8, max 128.

.. class:: OtpResponse

   :param str message: Human-readable status.

API Key Schemas
~~~~~~~~~~~~~~~

Module: ``schemas/api_keys.py``

.. class:: CreateApiKeyRequest

   :param str name: Human-readable label (1-255 chars).

.. class:: ApiKeyResponse

   :param UUID id: API key UUID.
   :param str name: Label.
   :param str prefix: ``"oz_live_"`` or ``"oz_test_"``.
   :param UUID project_id: Project scope.
   :param UUID | None created_by: Creator user UUID.
   :param list[str] scopes: Permission scopes.
   :param bool is_revoked: Revocation status.
   :param datetime | None last_used_at: Last usage timestamp.
   :param datetime created_at: Creation timestamp.
   :param str | None raw_key: Only populated on creation.

   ``model_config = ConfigDict(from_attributes=True)``.

.. class:: ApiKeyCreatedResponse(ApiKeyResponse)

   Inherits all fields from :class:`ApiKeyResponse`.  ``raw_key`` is
   mandatory (shown once at creation).

   :param str raw_key: Full key string — save it, won't be shown again.
   :param str message: Warning to save the key.

.. class:: ApiKeyListResponse

   :param list[ApiKeyResponse] data: List of keys.
   :param int total: Total count (excludes revoked).


Security Considerations
-----------------------

Password Storage
~~~~~~~~~~~~~~~~

All dashboard user passwords are hashed with **bcrypt** (cost factor 12)
before storage.  The application never stores or logs plaintext passwords.
See ``utils/password.py``.

API Key Storage
~~~~~~~~~~~~~~~

API keys are hashed with **SHA-256 + random 16-byte salt**.  The unsalted
``lookup_hash`` is used only for fast indexing — it is not a security hash.
See :func:`generate_api_key`, :func:`hash_api_key`, :func:`verify_api_key`
in ``utils/crypto.py``.

Refresh Token Storage
~~~~~~~~~~~~~~~~~~~~~

Refresh tokens are stored as SHA-256 hashes.  The raw token is never
persisted.  Tokens are rotated on every use and revoked on password reset.

Email Enumeration Prevention
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* ``/v1/auth/forgot-password`` returns the same response whether the email
  exists or not (the ``ValidationError`` for non-existent accounts is
  raised internally and caught to return a generic success message).
  See :meth:`AuthService.forgot_password`.

* ``/v1/auth/login`` returns a generic "Invalid email or password" message
  rather than revealing which one is incorrect.

OTP Security
~~~~~~~~~~~~

* OTPs are never stored in plaintext — only SHA-256 hashes go to Redis.
* Verification uses ``hmac.compare_digest`` (constant-time comparison).
* After 5 failed attempts, the OTP is invalidated.
* Purpose-scoped — a ``signup`` OTP cannot be used for ``password_reset``.
* Per-email send limits (5/hour) and resend cooldowns (60s).

MFA Session Security
~~~~~~~~~~~~~~~~~~~~

* MFA pending sessions are stored in Redis with a 10-minute TTL.
* The ``mfa_session_token`` is 32 random bytes from ``secrets.token_hex``.
* The session is deleted immediately after retrieval (single-use).
* The OTP purpose is ``"mfa"`` — separate from all other OTP purposes.

Cache Poisoning Prevention
~~~~~~~~~~~~~~~~~~~~~~~~~~

* Auth cache entries are stored orjson-serialized and validated on read.
* Corrupted cache entries are deleted and ignored.
* Negative cache entries prevent replay of invalid keys for 60s.

Miss-Rate Limiting
~~~~~~~~~~~~~~~~~~

* Per-IP auth miss-rate limiting prevents brute-force enumeration of valid
  API keys via the database path.
* After 10 DB cache-misses per 60 seconds per IP, further requests are
  rejected with HTTP 429.


TODO / Known Gaps
-----------------

* **GDPR hard purge** (``workers/gdpr_jobs.py``) — the Phase 2 worker
  that hard-deletes user data 30 days after soft-delete is currently a
  stub.  See ``TODO(phase2)`` in :meth:`UserService.delete_user`.

* **Session-level RBAC** — JWT users currently get full scopes
  (``["read", "write", "admin"]``).  Fine-grained RBAC via dashboard
  roles is planned but not yet implemented.

* **Rate limits for ``/v1/auth/login/otp/send`` and ``/v1/auth/login/otp/verify``
  are managed through ``AuthThrottle``, but the OTP service itself also has
  Redis-based rate limits.  These two systems are independent and may
  occasionally report different rate-limit states.
