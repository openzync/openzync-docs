ADR 007: Unified Org Permission Model
=====================================

**Status:** Accepted (2026-08-16)
**Supersedes:** ``users.role`` checks, ``project_members.role`` owner checks
(``require_project_owner``), ``api_keys.scopes`` + the ``require_scope``
bridge

----

Context
-------

OpenZync previously enforced authorization through **three overlapping
models**, each with its own vocabulary and its own dependency family:

1. **``users.role`` checks** — org-level role gates (``admin`` /
   ``member`` / ``superadmin``) enforced by ``require_org_admin`` and
   ``require_org_admin_or_self``.
2. **``project_members.role`` owner checks** — project-level ownership
   gates enforced by ``require_project_owner`` (JWT-only; rejected API
   keys outright).
3. **``api_keys.scopes`` + the ``require_scope`` bridge** — API keys
   carried scope strings (``read`` / ``write`` / ``admin`` /
   ``admin:write``), while JWT users were implicitly granted the full
   scope set (``["read", "write", "admin"]``) by the middleware.  The
   ``require_scope`` dependency then had to *translate* between the two:
   admin-prefixed scopes were re-checked against the org role via
   ``core.rbac.get_org_role`` (Redis-cached 60 s, fail-closed to
   member), while member-level scopes passed for any authenticated JWT
   user.

The result was four dependency families (``require_scope``,
``require_org_admin``, ``require_org_admin_or_self``,
``require_project_owner``), two vocabularies (role strings and scope
strings), and divergent JWT vs API-key authorization paths.  A single
endpoint could be gated by a scope string for API keys and a role check
for JWT users — with the translation logic buried inside one dependency.

----

Decision
--------

Replace all three models with **one org-scoped permission vocabulary**,
materialized on both users and API keys, and enforced by a single
dependency factory that treats JWT and API-key authentication
identically.

D1 — The permission vocabulary
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Seven org-scoped permissions:

.. list-table::
   :header-rows: 1

   * - Permission
     - Grants
   * - ``project:read``
     - Read projects and their memory, context, search, and graph data
   * - ``project:write``
     - Write project data (ingest, sessions, facts)
   * - ``project:manage``
     - Project lifecycle — create, archive, delete, project settings
   * - ``configuration:read``
     - Read org/project configuration
   * - ``configuration:write``
     - Modify org/project configuration
   * - ``members:read``
     - List org/project members
   * - ``members:write``
     - Add/remove members and change member permissions

Permissions are org-scoped: project membership (``require_project_membership``)
remains the project-scope gate, and the permission set decides *what* a
member may do within that scope.

D2 — Admin/superadmin = wildcard
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Org ``admin`` and platform ``superadmin`` roles are represented by an
**empty permissions array** — ``[]`` means *all permissions*.  No
wildcard string exists; the empty array is the wildcard.  This keeps the
vocabulary closed (no ``*`` permission to enumerate or leak) and makes
"admin" a role property rather than a permission.

D3 — Materialized defaults + optional grants
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Every new user gets the member defaults **materialized** in a new
``users.permissions ARRAY(String)`` column:

.. code-block:: text

   member defaults = {project:read, project:write}

Optional grants are appended to the array.  Defaults are stored, not
derived — the permission set is always readable from the row without
role inference.

D4 — Unified ``require_permission(perm)``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

One dependency factory replaces all four deleted dependencies:

.. code-block:: python

   org_id: str = Depends(require_permission("project:manage"))

``require_permission`` handles JWT and API-key auth identically:

* **JWT** — the user's permissions are read from ``users.permissions``
  (DB-verified, **never** from JWT claims), cached 60 s in Redis.
* **API key** — the key's permissions are read from
  ``api_keys.permissions`` (same cache).
* **Fail-closed** — any infrastructure error (Redis/DB unavailable)
  resolves to *deny*, preserving the existing zero-fallback auth
  semantics.

D5 — API-key unification
~~~~~~~~~~~~~~~~~~~~~~~~

``api_keys.scopes`` is renamed to ``api_keys.permissions`` and carries
the **same permission strings** as users.  Legacy scope values are
remapped in the migration:

.. list-table::
   :header-rows: 1

   * - Legacy scope
     - Remapped to
   * - ``read``
     - ``project:read``
   * - ``write``
     - ``project:write``
   * - ``admin``
     - (empty array — wildcard)
   * - ``admin:write``
     - (empty array — wildcard)

This is a **BREAKING change** for SDK/MCP consumers that create or
inspect API keys by scope name.

D6 — Owner-role decommission
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``project_members.role`` survives as a **display-only** field — it is no
longer an authorization input.  ``require_project_owner`` is deleted;
owner-only endpoints are gated by ``require_permission("project:manage")``
instead, which also restores API-key parity (API keys were previously
rejected by the owner check).

D7 — Deleted dependencies
~~~~~~~~~~~~~~~~~~~~~~~~~

``require_scope``, ``require_org_admin``, ``require_org_admin_or_self``,
and ``require_project_owner`` are deleted.  ``require_project_membership``
survives unchanged as the project-scope gate.

Platform superadmin (``require_superadmin`` + ``PLATFORM_ORG_ID`` + RLS
bypass in ``get_db_superadmin``) is **unchanged** — it operates above the
org permission model.

----

Alternatives Considered
-----------------------

**Per-project permission tables** (``project_members.permissions``)
   A permission set per project membership would allow independent
   project-level grants.  Rejected: the org-scoped vocabulary plus the
   existing project-membership gate covers the current product surface
   with one table and one cache key; per-project sets would multiply
   cache keys, grant/revoke paths, and migration surface for no current
   requirement.  Deferred — see below.

**Keeping the owner bypass**
   Retain ``project_members.role`` as an authorization input alongside
   the new vocabulary.  Rejected: it preserves the second vocabulary the
   ADR exists to remove, and it keeps API keys locked out of
   owner-gated endpoints.  The display-only role keeps the UI semantics
   without the authorization cost.

----

Consequences
------------

Positive
~~~~~~~~

- One vocabulary, one dependency factory, identical JWT/API-key
  enforcement — the ``require_scope`` translation layer is gone.
- Fail-closed semantics preserved: infrastructure errors still deny.
- Roles are DB-verified and Redis-cached (60 s), never trusted from JWT
  claims.
- Owner-gated endpoints gain API-key parity via ``project:manage``.

Negative
~~~~~~~~

- **BREAKING:** ``api_keys.scopes`` → ``api_keys.permissions`` rename
  affects SDK/MCP consumers; legacy values are remapped but the field
  name and value vocabulary change.
- Router sweep: every endpoint previously gated by the four deleted
  dependencies must be re-gated with ``require_permission``.
- Cache invalidation: grant/revoke of a user's or key's permissions must
  invalidate the 60 s Redis cache entry, or changes take up to a minute
  to apply.
- ``users.permissions`` is a new column on every user row; the
  migration must backfill member defaults for existing users.

----

Deferred Items
--------------

1. **Per-project permission sets** — if a project ever needs a
   permission set independent of the org vocabulary, revisit the
   per-project table alternative (the org-scoped model is the
   foundation it would extend).
2. **Role→permission mapping UI** — the dashboard currently manages
   roles; a UI for editing the underlying permission arrays is future
   work.

----

References
----------

- ``dependencies/auth.py`` — ``require_permission`` factory (replaces
  ``require_scope`` / ``require_org_admin`` /
  ``require_org_admin_or_self``)
- ``dependencies/project_auth.py`` — ``require_project_membership``
  (unchanged); ``require_project_owner`` deleted
- ``models/user.py`` — ``users.permissions ARRAY(String)``
- ``models/api_key.py`` — ``api_keys.permissions`` (renamed from
  ``scopes``)
- ``middleware/auth.py`` — ``request.state.permissions``
- ``core/rbac.py`` — role verification, 60 s Redis cache, fail-closed
- ``migrations/versions/...`` — ``scopes`` → ``permissions`` rename +
  legacy remap + member-default backfill
