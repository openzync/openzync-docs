Add-ons — Developer Guide
==========================

This is the complete guide to building, installing, and operating
platform add-ons. The runnable reference implementation is
``addons/democrm/`` in ``openzync-core`` — every snippet below maps to
something you can read or run there.

.. contents:: Sections
   :local:
   :depth: 3

.. note::

   Backend paths below assume the API at ``http://localhost:8000`` and the
   dashboard at ``http://localhost:3000``. All ``/admin/system/addons``
   endpoints require **superadmin** auth (``require_superadmin``) — pass the
   JWT from signup as ``Authorization: Bearer $TOKEN``.

----------
Concepts
----------

* **First-party vs third-party.** There is no technical distinction. Every
  add-on — including ``democrm`` — is an out-of-tree bundle loaded through
  the same installer, manifest validation, and registry. ``democrm`` is
  simply the bundle shipped in-repo as a canonical sample.
* **Platform-wide enablement.** One row per add-on in the ``addon_states``
  table, regardless of organization. There is no per-org tier: an enabled
  add-on serves every org (its own code is responsible for org-scoping
  queries — see :ref:`addons-security`). Disabled add-ons 403 on every
  route; they never fall through to 404.
* **Versioned namespaces.** Each loaded revision imports under
  ``addons.{id}.v{major}`` (``core/addons/loader.py::module_name``). A new
  major version loads *beside* the old one; routes are stripped and
  re-mounted wholesale on upgrade, so enable/upgrade never double-mounts.
* **Hot-load.** ``enable()`` loads the module, resolves hooks, atomically
  swaps the registry (GIL-atomic dict assignment in
  ``core/addons/registry.py::publish_registry``), and mounts routes — no
  restart. Boot re-loads every ``enabled`` row the same way
  (``services/api/main.py``, lifespan); a broken add-on degrades loudly
  (``addon.boot_load_failed`` log) but never fails boot.
* **Kill-switch.** ``disable()`` unmounts API + UI routes, mounts a gated
  placeholder (so the id keeps 403ing instead of 404ing), bumps the
  generation, and marks the registry record disabled — worker dispatch to
  it fails closed from that moment.
* **Generations.** Every enable/disable/reload assigns a new generation
  number (``next_generation()``, a process-local counter). Route gates can
  pin a generation; a stale pin 404s via ``get_enabled()``. The number is
  also surfaced in ``GET .../health`` and the dashboard detail page.

-----------------------
Quickstart (democrm)
-----------------------

The fastest path from zero to a working panel in the browser:

.. code-block:: bash

   # 0. create a superadmin JWT first (see guides/quickstart.rst signup flow)
   TOKEN="eyJhbGciOiJIUzI1NiIs..."   # ← paste your access_token

   # 1. install the sample from the in-repo source (disabled by default)
   curl -s -X POST http://localhost:8000/admin/system/addons \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{"local_path": "addons/democrm"}' | python3 -m json.tool
   # → 201, {"id": "democrm", "enabled": false, ...}

   # 2. enable it (hot-loads + mounts /v1/ext/democrm/*)
   curl -s -X POST http://localhost:8000/admin/system/addons/democrm/enable \
     -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
   # → 200, {"enabled": true, ...}

   # 3. hit the add-on API directly (org-scoped by your JWT)
   curl -s http://localhost:8000/v1/ext/democrm/contacts \
     -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

   # 4. mint a panel ticket and open the panel
   curl -s -X POST http://localhost:8000/admin/system/addons/democrm/panels/ticket \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{"panel_path": "/ui/index.html"}' | python3 -m json.tool
   # → 201, {"ticket": "..."} (5-minute TTL)

   # 5. dashboard route (does steps 4 + token handoff for you):
   #    http://localhost:3000/ext/democrm/ui/index.html

For a new add-on, copy ``addons/democrm/`` as the scaffold — there is no
template or cookiecutter generator in the repo today:

.. code-block:: bash

   cp -r addons/democrm addons/myaddon
   cd addons/myaddon
   # edit addon.toml (id, version, tables, panels), hooks.py, pyproject.toml
   # (version must match addon.toml), migrations/versions/*, ui/*

Then install with ``{"local_path": "addons/myaddon"}`` and ``enable`` as
above. ``local_path`` must live under ``ALLOWED_ADDON_SRC``
(default: the ``addons/`` dir; OpenBao-managed ``OZ_ALLOWED_ADDON_SRC``) —
anything outside it is rejected with 422, as are symlinks.

.. _addons-manifest:

------------------
Manifest reference
------------------

``addon.toml`` is parsed by ``core/addons/manifest.py::parse_manifest``.
Flat keys are canonical; a ``[addon]`` table is accepted as a fallback
(``name`` may substitute for ``id`` there). Unknown sections (e.g.
``[signing]``) are ignored. Parsed manifests are cached by
``(resolved path, addon.toml mtime)``.

.. list-table:: Manifest fields
   :header-rows: 1

   * - Field
     - Required
     - Rules
   * - ``id``
     - yes
     - ``^[a-z0-9][a-z0-9_-]{1,62}$``. Also the table prefix
       (``addon_<id>_``), the route prefix (``/v1/ext/<id>``), and the
       install-dir name for local sources.
   * - ``version``
     - yes
     - Strict semver ``MAJOR.MINOR.MAJOR`` with optional ``+``/``-``
       suffix. Only the **major** pins the import namespace.
   * - ``min_core_version``
     - no (default ``0.0.0``)
     - Semver; install fails loudly when the running core
       (``core/_version.py::__version__``, leading-triple comparison) is
       older.
   * - ``hooks``
     - no (default ``[]``)
     - Allowlist: ``router, service, middleware, worker, graph_backend,
       embedder, llm, health, settings, admin_ui``. Anything else →
       422 ``Unknown hooks``. See :ref:`addons-hooks` for which hooks are
       actually wired today.
   * - ``permissions``
     - no (default ``[]``)
     - Must be a subset of the platform permission vocabulary
       (``core.rbac.ALL_PERMISSIONS``). Unknown names → 422. Declared,
       not enforced — enforcement is the add-on's own queries plus RLS
       (see :ref:`addons-security`).
   * - ``tables``
     - no (default ``()``)
     - Short names matching ``^[a-z][a-z0-9_]{1,62}$``. Real tables are
       ``addon_<id>_<table>`` and **only** these may be created by
       migrations (see :ref:`addons-migrations`).
   * - ``[[panels]]`` (``id``, ``title``, ``path``)
     - no
     - ``id`` follows the addon-id regex. ``path`` must be absolute,
       under ``/ui/`` (no backslashes, no ``..`` escapes, not bare
       ``/ui``) — e.g. ``/ui/index.html``. Served at
       ``/v1/ext/<id><path>?ticket=``.
   * - ``entrypoint``
     - no (default ``hooks.py``)
     - Explicit module file first; else the probe order
       ``hooks.py → addon.py → addon/__init__.py`` (first existing file
       wins via ``spec_from_file_location``); else the
       ``openzync.addons`` entry-points group. Nothing found → 422.

``democrm`` manifest (``addons/democrm/addon.toml``), annotated:

.. code-block:: toml

   id = "democrm"
   version = "0.3.0"          # keep in sync with pyproject.toml (upgrade
                              # of local sources keys off that version!)
   min_core_version = "1.0.0"
   hooks = ["router", "worker", "health"]
   permissions = ["project:read"]
   tables = ["contacts"]       # → addon_democrm_contacts
   entrypoint = "hooks.py"

   [[panels]]
   id = "overview"
   title = "Overview"
   path = "/ui/index.html"     # → /v1/ext/democrm/ui/index.html?ticket=

   [[panels]]
   id = "contacts"
   title = "Contacts"
   path = "/ui/contacts.html"

.. _addons-hooks:

----------------
Hooks reference
----------------

Add-on modules expose **functions** (preferred) or plain attributes. Only
two resolvers exist today — ``core/addons/hooks.py``:

* ``register_router() -> APIRouter`` (or a module-level ``router``
  attribute). Mounted at ``/v1/ext/<id>`` with the enablement gate as
  router-level dependencies. Optional — no router means API-less add-on
  (UI-only or worker-only). A *broken* factory raises loudly and fails
  ``enable()`` visibly.
* ``register_worker() -> dict[str, handler]`` (or module-level
  ``WORKER_OPS`` dict). Ops run via the ``addon_dispatch`` ARQ task
  (see :ref:`addons-operator`). Optional.

Plus one convention read directly off the module (no ``register_*``):

* ``SERVICE_OVERRIDES: dict[str, Any]`` — module attribute mapping
  service names to replacements, consulted by
  ``core/addons/registry.py::resolve_service_override`` (first enabled
  add-on wins; core default otherwise). Read at ``enable()`` and at boot.

.. warning::

   The remaining manifest hook names — ``service`` (beyond the
   ``SERVICE_OVERRIDES`` dict), ``middleware``, ``graph_backend``,
   ``embedder``, ``llm``, ``health``, ``settings``, ``admin_ui`` — are
   **accepted by the manifest allowlist but have no resolver or call-site
   in the current code**. In particular, ``democrm``'s
   ``register_health_check()`` is inert: nothing calls it, and
   ``GET .../addons/{id}/health`` returns only the platform snapshot
   (``id, enabled, loaded, generation, version``). Treat these names as
   reserved for the roadmap, not as working extension points. If you need
   one, wire a resolver in ``core/addons/hooks.py`` plus a call-site, and
   extend this doc.

``settings`` (manifest hook) likewise has no loader — add-on configuration
today is whatever the add-on reads itself (env/OpenBao); there is no
platform settings-injection contract.

Minimal ``hooks.py`` (API-only add-on):

.. code-block:: python

   from fastapi import APIRouter, Depends, Request
   from sqlalchemy import text
   from sqlalchemy.ext.asyncio import AsyncSession

   from dependencies.db import get_db

   def register_router() -> APIRouter:
       r = APIRouter()

       @r.get("/contacts")
       async def list_contacts(
           request: Request, db: AsyncSession = Depends(get_db)
       ) -> list[dict]:
           rows = (
               await db.execute(
                   text(
                       "SELECT id, name, email FROM addon_myaddon_contacts "
                       "WHERE organization_id = :org ORDER BY name"
                   ),
                   {"org": str(request.state.org_id)},
               )
           ).all()
           return [{"id": i, "name": n, "email": e} for i, n, e in rows]

       return r

   def register_worker() -> dict:
       return {"sync_contacts": sync_contacts}

.. note::

   Worker-handler calling convention (per ``workers/addon_dispatch.py``):
   the dispatcher inspects the handler signature — parameters named
   ``db`` and/or ``session`` receive an **add-on-only session** (see
   :ref:`addons-security`); every other parameter is filled from the
   task payload kwargs, so payload keys must match handler parameter
   names. A minimal op plus its enqueue form:

   .. code-block:: python

      # The dispatcher calls handler(**payload), injecting an add-on-only
      # session for parameters literally named "db" and/or "session".
      # No ctx is passed; every other parameter comes from the payload.
      async def sync_contacts(db, since: str | None = None) -> dict:
          total = (
              await db.execute(text("SELECT count(*) FROM addon_myaddon_contacts"))
          ).scalar()
          return {"synced": total, "since": since}

      # Enqueued as: addon_dispatch(ctx, "myaddon", "sync_contacts", since="...")
      # (ctx is the ARQ worker context — redis, addons, engine — not a handler arg.)

   Handlers do **not** receive ``ctx`` — despite what the ``democrm``
   sample's ``_sync_contacts(ctx, db, args)`` signature suggests, the dispatcher
   calls ``handler(**payload)`` plus ``db``/``session`` only, so that
   sample signature would raise ``TypeError`` if actually dispatched.
   (Flagged as sample drift; see Troubleshooting.)

.. _addons-panels:

----------------
Panel authoring
----------------

Panels are static files under the bundle's ``ui/`` dir, served by the
backend at ``GET /v1/ext/<id>/ui/<path>`` inside a dashboard ``<iframe
sandbox="allow-scripts">``. Three mechanisms compose the viewer contract
(full text in the header comment of
``src/app/(dashboard)/ext/[addonId]/[[...panel]]/page.tsx``):

**1. Ticket flow (mint → ``?ticket=`` → rewrite propagation).**

1. The viewer mints a ticket before mounting the frame::

      POST /admin/system/addons/{id}/panels/ticket   {"panel_path": "/ui/index.html"}
      → 201 {"ticket": "<opaque>"}   (Redis-backed, 5-min TTL)

   The ticket payload binds ``addon_id`` + ``panel_path`` (+ ``viewer``
   for audit) and derives ``allowed_dir`` as the panel's directory
   (``/ui`` for ``/ui/index.html``). Scope rule: the ticket authorizes
   the panel file itself **plus any file under ``allowed_dir``** —
   so a ticket minted for ``/ui/index.html`` covers sibling assets
   (``/ui/app.js``, ``/ui/style.css``) but does **not** authorize
   ``/ui/other.html`` or anything outside that dir. Legacy tickets with
   no (or a non-``/ui/``) ``allowed_dir`` fail closed with 403.
2. The iframe src carries it as ``?ticket=``:
   ``{API_BASE}/v1/ext/{id}/ui/index.html?ticket=...``. ``?token=`` is
   rejected (401). ``<iframe>`` elements cannot send ``Authorization``
   headers, which is why the ticket exists. **Every** request under
   ``/v1/ext/<id>/ui/`` requires a ticket — there is no ticketless
   fallback, not even for non-HTML assets.
3. On serving an ``.html`` file *with* a valid ticket, the backend rewrites
   relative ``href``/``src``/``action``/``poster`` attributes to append
   ``?ticket=`` (``_rewrite_html_for_ticket``), plus the **first** URL of
   each ``srcset`` (boot image only — further candidates are yours to
   propagate). Sub-pages, stylesheets and scripts keep working without
   JS. Absolute URLs (scheme or ``//``), ``#``-fragments, ``data:`` srcset
   entries, and refs already carrying ``ticket`` are left alone.
   Non-UTF-8 payloads are served untouched (fail-open to the pre-rewrite
   401 behavior for the follow-up request).

   JS ``fetch()`` and CSS ``url()`` are **not** rewritten — propagate the
   ticket yourself with the ``withTicket`` helper (shipped in
   ``democrm``'s ``ui/*.html``; keep it verbatim):

   .. code-block:: js

      function ozTicket() { return new URLSearchParams(location.search).get("ticket"); }
      function withTicket(p) {
        if (!p || /^([a-z][a-z0-9+.-]*:)?\/\//i.test(p) || p.charAt(0) === "#") return p;
        try {
          const u = new URL(p, location.href);
          if (u.searchParams.get("ticket")) return p;
          u.searchParams.set("ticket", ozTicket());
          return u.pathname + u.search + u.hash;
        } catch (e) { return p; }
      }
      document.addEventListener("click", (ev) => {
        const a = ev.target && ev.target.closest ? ev.target.closest("a[href]") : null;
        if (!a) return;
        const fixed = withTicket(a.getAttribute("href"));
        if (fixed !== a.getAttribute("href")) a.setAttribute("href", fixed);
      });

   Apply ``withTicket()`` to any URL fetched or constructed at runtime
   (``fetch(withTicket("../data.json"))``); the click listener covers
   plain ``<a href>`` navigation between panels.
4. Missing/unknown/expired ticket → **401** ``Panel ticket required`` /
   ``invalid or expired``. Ticket for another add-on, or for a file
   outside the ticket's dir → **403** ``not valid for this file`` (logged
   as ``addon.ticket_scope_denied`` with viewer + requested path).
   Redis unreachable → **503** (fail closed).

Panels are served by a gated route (not ``StaticFiles``): the same
``require_addon_enabled`` gate plus the ticket check run before any byte
is read, and every response carries ``Referrer-Policy: no-referrer``
(``X-Content-Type-Options: nosniff`` on non-HTML files).

**2. postMessage token contract.**

Panel JS API calls use the normal ``Authorization: Bearer <token>``
header — the token arrives via ``postMessage`` from the viewer wrapper:

.. code-block:: js

   // panel side (copy from addons/democrm/ui/index.html):
   window.addEventListener("message", (ev) => {
     if (!ev.data || ev.data.type !== "openzync:addon-token") return;
     if (!ev.data.token || ev.origin !== ev.data.origin) return;
     TOKEN = ev.data.token; boot();
   });
   async function api(path, opts) {
     const r = await fetch(path, Object.assign(
       {headers: {Authorization: "Bearer " + TOKEN}}, opts || {}));
     if (!r.ok) throw new Error("API " + r.status);
     return r.json();
   }

Viewer side: on every iframe load it posts
``{type: "openzync:addon-token", token, origin: window.location.origin}``
with ``targetOrigin "*"``. The ``"*"`` is mandatory — the sandbox gives
the frame an opaque origin, so specific target origins are silently
dropped and the panel would hang on ``Waiting for API token``. Trust is
established **in the panel**: it boots only when the browser-attested
``ev.origin === ev.data.origin`` (plus type match) — a spoofed claim from
any other page fails because ``ev.origin`` is browser-set. (Deriving the
expected origin from the panel's own URL is wrong: the panel is served by
the API, the viewer runs on the dashboard.)

A 401 inside the panel means the page token expired: democrm-style panels
may post ``openzync:addon-unauthorized`` and the wrapper refreshes the
session once (``refreshOnce``) then remounts the frame with a fresh ticket.
Viewer-side default-deny: the ``openzync:addon-unauthorized`` listener
accepts messages only from the API origin derived from ``API_BASE`` — when
``API_BASE`` is empty or unparseable the message is rejected outright.

**3. Null-origin CORS.**

Sandboxed fetches send ``Origin: null``, which Starlette's
``CORSMiddleware`` explicit allowlist rejects. ``ExtNullOriginCorsMiddleware``
(``middleware/ext_null_origin_cors.py``) handles exactly that case —
``Origin: null`` on paths under ``/v1/ext/`` only — and leaves every other
path to the explicit allowlist. It must stay registered OUTSIDE (after)
``CORSMiddleware`` so its preflight short-circuit runs first. Safety basis:
add-on APIs authenticate via bearer tokens only, no cookies, so no
``Allow-Credentials`` is emitted. Never extend the allowance beyond
``/v1/ext/*``. Allowed methods/headers:
``GET, POST, PUT, PATCH, DELETE, OPTIONS`` and ``Authorization,
Content-Type, X-Request-ID, Idempotency-Key`` (preflights cached
``max-age=600``; responses carry ``Vary: Origin``). A ``NetworkError``/failed
preflight from panel JS almost always means this path (see
Troubleshooting).

Relative API calls from a panel resolve against the UI URL, so use
``../`` to reach the add-on API root, e.g. from
``/v1/ext/democrm/ui/contacts.html`` fetch ``../contacts`` →
``/v1/ext/democrm/contacts`` (the democrm pattern).

.. _addons-migrations:

------------------------
Migrations authoring
------------------------

Add-on bundles ship an alembic-style ``migrations/versions/*.py`` chain,
run against the platform database but isolated per add-on
(``core/addons/alembic_runner.py``):

* Version tracking lives in ``addon_alembic_<slug>`` — never the core
  ``alembic_version`` table.
* Only ``addon_<slug>_<table>`` tables drawn from the manifest ``tables``
  allowlist may be created. **Enforced statically before anything runs**
  (source scan of each revision for ``create_table("...")``).
* **Every created table must carry an ``organization_id`` column**
  (same static scan). This is the multi-tenancy contract — RLS cannot do
  it for you on raw tables.
* Exactly one head, linear chain, no merges, no cycles — violations fail
  loudly at install/upgrade time.
* Upgrade is per-revision ``upgrade()`` with stamp-skip for already-applied
  revisions. The sync driver URL is derived from ``DATABASE_URL``
  (``+asyncpg://`` → ``+psycopg2://``); unsupported schemes fail closed.
* Downgrade (uninstall path) runs the chain in reverse then clears the
  version table; ``drop_addon_tables()`` drops exactly the allowlisted
  tables (``DROP TABLE ... CASCADE``).

Minimal revision (``addons/democrm/migrations/versions/0001_contacts.py``):

.. code-block:: python

   """Create demo CRM contacts."""
   revision = "0001"
   down_revision = None

   import sqlalchemy as sa
   from alembic import op


   def upgrade() -> None:
       op.create_table(
           "addon_democrm_contacts",          # addon_<id>_<tables-entry>
           sa.Column("id", sa.Text(), primary_key=True, nullable=False),
           sa.Column("organization_id", sa.Text(), nullable=False),
           sa.Column("name", sa.Text(), nullable=False),
           sa.Column("email", sa.Text(), nullable=True),
       )


   def downgrade() -> None:
       op.drop_table("addon_democrm_contacts")

No ``migrations/versions/`` dir → migrations are a silent no-op
(``addon.migrations_absent`` log).

.. _addons-operator:

-------------------------------
Lifecycle operator runbook
-------------------------------

Full endpoint reference (all under ``/admin/system/addons``, superadmin
only; ``services/addon_service.py`` + ``routers/admin_addons.py``):

.. list-table:: Add-on admin endpoints
   :header-rows: 1

   * - Method + path
     - Success
     - Notable errors
   * - ``POST /admin/system/addons`` ``{"local_path"|"github_url", "tag?"}``
     - ``201`` (disabled by default)
     - ``409`` id already installed; ``422`` both/neither source,
       ``tag`` with local, bad dir, manifest/policy violation;
       ``502`` git clone/ls-remote failure
   * - ``GET /admin/system/addons``
     - ``200`` ``{data, total}``
     - —
   * - ``POST /admin/system/addons/check-updates``
     - ``200`` refreshed list
     - per-source failures degrade to a warning log, never fail the call
   * - ``POST /admin/system/addons/{id}/panels/ticket``
       ``{"panel_path"}``
     - ``201`` ``{"ticket"}`` (path-scoped: panel file + its dir)
     - ``403`` disabled; ``404`` unknown id or undeclared panel path;
       ``503`` Redis unavailable
   * - ``GET /admin/system/addons/{id}``
     - ``200``
     - ``404`` unknown
   * - ``POST /admin/system/addons/{id}/enable``
     - ``200`` (hot-load + mount)
     - ``404`` unknown; broken entrypoint/router factory raises visibly
   * - ``POST /admin/system/addons/{id}/disable``
     - ``200`` (unmount + gated placeholder + generation bump)
     - ``404`` unknown
   * - ``POST /admin/system/addons/{id}/upgrade?tag=``
     - ``200`` (``tag`` is a **query** param — the dashboard's
       ``upgradeAddon(id, tag)`` sends ``?tag=``; a JSON body is ignored)
     - ``404`` unknown; ``422`` local source unchanged version,
       ``tag`` on local source, invalid tag charset (leading ``-``
       rejected: option injection), migration/policy failure
       (disables the add-on first)
   * - ``DELETE /admin/system/addons/{id}?remove_data=&confirm=``
     - ``204``
     - ``404`` unknown; ``422`` ``remove_data=true`` without
       ``confirm=true``
   * - ``GET /admin/system/addons/{id}/health``
     - ``200`` ``{id, enabled, loaded, generation, version}``
     - ``404`` unknown

Source types:

* ``local_path`` — directory copied into ``var/addons/<id>``. Must
  resolve under ``ALLOWED_ADDON_SRC`` (OpenBao ``OZ_ALLOWED_ADDON_SRC``,
  default ``addons``); confinement rejects symlinks at every path
  component up to the allowed root, and a full tree walk
  (``_reject_symlinked_tree``) rejects any symlink inside the source
  (``copytree`` would otherwise materialise outside targets).
  Staging uses a temp dir inside ``ADDON_ROOT`` with an atomic
  rename-over swap; the displaced live dir is parked under a unique
  ``.bak.<pid>.<rand>`` name (concurrent installs never share one) and
  restored if the swap fails. Version comes from the installed copy's
  ``pyproject.toml`` (missing file/version → 422); ``installed_sha`` is
  a ``local-<hash>`` of the install mtime. ``tag`` is rejected for local
  sources (422).
* ``github_url`` — ``owner/repo`` (https/ssh spellings normalised to
  ``https://github.com/<repo>.git``), shallow-cloned into
  ``var/addons/<owner>__<repo>`` via a single ``--branch=<tag>`` arg
  (no shell splitting). ``tag`` pins the clone and must match
  ``^[\w][\w.\-/]*$`` — anything else (notably a leading ``-``, which
  would be parsed as a git option) → 422. Omitted → latest tag via
  ``git ls-remote --tags`` (numeric-aware sort, ``v10`` > ``v9``;
  no tags → 502). Same atomic stage-and-swap as local sources.

``check-updates`` (manual endpoint + hourly ``reconcile_addon_versions``
ARQ cron): github compares newest ls-remote tag against ``installed_sha``
(``update_available = latest != installed_sha``); local compares the
current ``pyproject.toml`` version against ``row.version``. Concurrency is
semaphore-capped (5) with 15 s per-call timeouts; one failure never fails
the cron.

Upgrade semantics: github re-clones (pin with
``POST .../upgrade?tag=v1.2.0``, omitted → latest tag) and migrates; local re-copies
``source_ref`` and **422s when the ``pyproject.toml`` version is
unchanged** — bump the version on disk before re-syncing. Migration
failure disables the add-on (row + registry) and raises. When still
enabled afterwards, routes are reloaded (new generation). Note the
dashboard detail page labels local upgrades "Re-sync": same endpoint.
Stale UI/code after an upgrade is almost always "enabled state cached" —
``disable`` then ``enable`` forces a clean reload with a fresh generation.

Uninstall: ``DELETE ...?remove_data=false`` (default) drops the row and
unmounts routes, leaving tables in place. ``?remove_data=true&confirm=true``
first downgrades the bundle chain to base and drops exactly the
allowlisted ``addon_<slug>_<table>`` tables, then removes the row.

Worker contract (``workers/addon_dispatch.py``): the single ARQ task
``addon_dispatch(ctx, addon_id, op, **payload)`` resolves the op from the
registry snapshot (record → live module → ``register_worker()``),
fail-closed on unknown/disabled add-ons and unknown ops. Idempotency: a
Redis ``SET NX`` dedup key over canonical JSON
(``addon:dispatch:<id>:<op>:<sha256>``) with 24 h TTL — replays return
``{"status": "duplicate"}``. Handlers declaring ``db``/``session``
parameters get add-on-only sessions (see :ref:`addons-security`).

.. _addons-security:

--------------
Security model
--------------

* **Superadmin-only management.** Every admin endpoint and the ticket-issue
  endpoint sit behind ``require_superadmin``. There is no self-service
  install path for org users.
* **RLS posture for add-on code** (defense in depth, two layers):

  1. ``AddonSession`` (``core/addons/sessions.py``) is the only session
     class add-on code may hold (worker path). It raises
     ``AuthorizationError`` on ``set_bypass_rls()`` / ``rls_bypass()``,
     and ``core.rls`` additionally rejects bypass for any session carrying
     the ``_oz_addon_session`` marker — no frame-walking, no caller check
     to spoof.
  2. HTTP handlers run on the core ``RlsAsyncSession`` (via ``get_db``),
     but the route gate (``dependencies/addons.py::require_addon_enabled``)
     marks an ``addon_request`` ``ContextVar`` on entry (reset on exit,
     propagates into in-request ``asyncio`` tasks); ``core.rls`` denies
     bypass while the flag is set.

  Net effect: **add-on code can never escalate isolation — but it is not
  automatically scoped either**. ``democrm`` filters every query with
  ``WHERE organization_id = :org`` from ``request.state.org_id``. Copy
  that pattern: the ``organization_id`` column requirement exists
  precisely so this filter is always available.
* **Ticket scope.** Panel tickets are opaque (``secrets.token_urlsafe(32)``),
  Redis-backed, 5-minute TTL, bound to one add-on id. The payload carries
  ``allowed_dir`` (the panel's directory): each ticket authorizes the
  panel file itself plus anything under that dir — a ticket minted for
  ``/ui/index.html`` does **not** authorize ``/ui/other.html`` (403,
  logged as ``addon.ticket_scope_denied``), and legacy tickets without a
  valid ``allowed_dir`` fail closed. They authorize *static UI files
  only* — data calls still need the viewer's bearer token via postMessage.
  Cross-addon ticket use → 403.
* **Sandbox flags.** Panels run in ``sandbox="allow-scripts"`` iframes
  (opaque origin, no same-origin, no forms/top-navigation by default).
  Responses carry ``Referrer-Policy: no-referrer``.
* **Allowlist.** Manifest ``hooks``/``permissions``/``tables``/panel paths
  are validated against fixed allowlists at parse time; migration table
  creation is statically confined to the allowlist; local sources are
  confined under ``ALLOWED_ADDON_SRC`` with symlink rejection. GitHub
  sources are cloned, not executed pre-validation — but review third-party
  code before enabling: **an enabled add-on runs arbitrary Python in the
  API process**.

---------------
Troubleshooting
---------------

.. list-table:: Symptoms, causes, fixes
   :header-rows: 1

   * - Symptom
     - Cause
     - Fix
   * - ``401 Panel ticket required`` / ``ticket is invalid or expired``
       on ``/v1/ext/<id>/ui/...``
     - Iframe loaded without ``?ticket=`` (viewer always mints first), or
       the 5-min TTL lapsed, or ``?token=`` used (removed contract).
       Every ``ui/`` request needs the ticket — including JS/CSS/assets.
     - Reload the viewer page (re-mints). Never hand-craft ``?token=``.
   * - Panel stuck on ``Waiting for API token``
     - Stale viewer bundle posting a different event type, or
       ``postMessage`` called with a specific ``targetOrigin`` instead of
       ``"*"`` (silently dropped to opaque-origin frames).
     - Hard-reload the dashboard; check the viewer posts
       ``openzync:addon-token`` with ``"*"`` and the panel checks
       ``ev.origin === ev.data.origin`` + type.
   * - ``403 not valid for this file`` on a panel sub-path
     - Ticket minted for one panel used on a file outside its dir
       (e.g. ticket for ``/ui/index.html`` fetching ``/ui/other.html``),
       or a legacy scopeless ticket.
     - Mint the ticket for the panel you actually load; sibling assets
       under the same dir share the scope. Check logs for
       ``addon.ticket_scope_denied`` (viewer + requested path).
   * - Add-on API fetch fails with CORS ``NetworkError`` from panel JS
     - ``Origin: null`` preflight not handled (middleware order /
       non-``/v1/ext/*`` path) or panel fetching an absolute API URL
       outside ``/v1/ext/``.
     - Keep panel fetches relative under ``/v1/ext/<id>/*``; ensure
       ``ExtNullOriginCorsMiddleware`` stays outside ``CORSMiddleware``.
   * - ``403 Addon ... is not enabled`` right after install
     - Installs land **disabled by default** (normal). Or the add-on was
       disabled by a failed migration/upgrade.
     - ``POST .../enable``. Check logs for ``addon.migrations_failed``.
   * - ``422 Local addon source must be under ...`` / ``Symlinks not
       allowed``
     - ``local_path`` outside ``ALLOWED_ADDON_SRC``, a symlinked path
       component, or any symlink inside the source tree.
     - Move the source under ``<repo>/addons/`` (or set
       ``OZ_ALLOWED_ADDON_SRC`` in OpenBao) and remove symlinks.
   * - ``422 Invalid git tag`` on install/upgrade
     - ``tag`` outside ``^[\w][\w.\-/]*$`` (leading ``-`` rejected:
       option injection).
     - Use a plain tag name (e.g. ``v1.2.0``); pin with
       ``POST .../upgrade?tag=v1.2.0``.
   * - ``422 Addon ... already at latest`` on local upgrade
     - Local upgrade keys off ``pyproject.toml`` version, not file diff.
     - Bump the version in both ``pyproject.toml`` and ``addon.toml``.
   * - Stale routes/UI after upgrade
     - Registry/generation staleness (old module still mounted).
     - ``disable`` then ``enable`` for a clean reload + fresh generation.
   * - ``409 already installed``
     - Id reinstall without uninstall.
     - ``DELETE`` first (choose ``remove_data`` deliberately), or upgrade.
   * - ``422 remove_data=true requires confirm=true``
     - Destructive-uninstall guard.
     - Append ``&confirm=true`` (the dashboard always sends it).
   * - Worker op raises ``TypeError`` (unexpected ``ctx``/``args``)
     - Handler uses the ``democrm`` sample signature
       ``(ctx, db, args)``; the dispatcher only injects ``db``/``session``
       and payload kwargs.
     - Declare ``async def op(db, <payload kwargs...>)`` — no ``ctx``,
       payload keys matching parameter names (see Hooks reference).

-------------------
Runnable reference
-------------------

``addons/democrm/`` in ``openzync-core`` is the canonical sample — install
it, click through it, then copy it:

* ``addon.toml`` — manifest with router + worker + health hooks, one table,
  two panels.
* ``hooks.py`` — ``register_router()`` (org-scoped ``/contacts`` CRUD +
  ``/stats``), ``register_worker()`` (``sync_contacts`` op),
  ``register_health_check()`` (**currently inert** — see warning in
  :ref:`addons-hooks`).
* ``migrations/versions/0001_contacts.py`` — single-revision chain creating
  ``addon_democrm_contacts`` with ``organization_id``.
* ``ui/index.html``, ``ui/contacts.html`` — ticket propagation helper +
  postMessage token contract + ``../``-relative API calls.
* ``pyproject.toml`` — version source of truth for local upgrade checks
  (keep in sync with ``addon.toml``).
