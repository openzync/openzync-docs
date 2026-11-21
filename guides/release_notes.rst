Release Notes
=============

Operator-facing changes for each OpenZync release. Entries start at v1.0.1;
v1.0.0 is not backfilled here.

v1.0.1 (2026-11-05)
-------------------

Breaking changes
~~~~~~~~~~~~~~~~

* **The project memory wipe is now organization-gated.** The wipe endpoint
  (``DELETE /v1/projects/{project_id}/memory``) resolves the project inside
  the caller's organization before deleting anything. Wiping a project that
  belongs to another organization, or that is archived or missing, returns
  ``404`` instead of deleting by the bare ``project_id``. Callers that cached
  project IDs across organizations must re-resolve them through an org-scoped
  listing.

* **Episode and fact ``UPDATE`` statements carry an ``organization_id``
  predicate.** Every episode and fact update is now filtered by the caller's
  organization, so a row belonging to another tenant can no longer be matched
  or updated by row ID alone.

* **A wildcard CORS origin is a startup error.** A ``CORS_ORIGINS`` value
  containing ``*`` now aborts startup instead of being accepted. The API
  always serves ``allow_credentials=True``, which a wildcard cannot legally
  combine with — browsers reject the pair, and tolerating it leaks
  credentials. List explicit origins only.

Upgrade steps
~~~~~~~~~~~~~

1. Apply migration ``0060`` (``0060_force_rls_all_org_tables``), which sets
   ``FORCE ROW LEVEL SECURITY`` on every table carrying an
   ``organization_id`` column. The migration **raises** and names any such
   table that has no ``org_isolation_*`` policy — add that policy first. There
   is no fallback: forcing RLS onto a policyless table denies every row, so
   the upgrade refuses to continue and the table stays fail-closed.
2. Set explicit ``OZ_CORS_ORIGINS`` (comma-separated origins, no ``*``) before
   restarting. A wildcard now prevents the API from starting.
3. Bump companion clients to their 1.0.1 releases (Python SDK, MCP server,
   dashboard) so the new wipe semantics and org-scoped behavior match the
   server.
4. No re-embedding is required — the embedder is the same frozen model as
   v1.0.0, so stored vectors remain compatible.

Security fixes
~~~~~~~~~~~~~~

* **MFA is bound to the login session's email.** Verification uses the email
  recorded when the session was created, not caller-supplied input, closing a
  session-fixation path where an attacker could verify their own OTP against a
  victim's login. Sessions minted before the binding existed fail closed and
  require a fresh login.
* **Deactivated and soft-deleted accounts are nonexistent across every OTP
  flow** — verify, password reset, send, and login-OTP. Codes can no longer be
  issued for or verified against a disabled account, and responses stay
  indistinguishable from an unknown email.
* **API keys are capped at their creator's permissions** and restricted to the
  ``project:*`` permission vocabulary, so a key can never escalate beyond the
  user who created it. Creation with no in-scope permission is rejected.
* **Self-permission changes are rejected.** A member can no longer change
  their own permission set, non-admin grants are capped at the actor's own
  effective permissions, and permission changes fail loudly while the
  authorization cache is unavailable.
* **The hardcoded admin root password is gone.** An unset root password now
  gets a random one-time password, and the seed still forces a change at first
  login.
* **Webhook signing secrets are encrypted at rest** with Fernet
  (``core.crypto``, AAD = endpoint ID): encrypted before create, rotate, or
  update, decrypted only at sign time. Crypto failures surface as a
  fail-loud ``502`` with no plaintext fallback, and a missing
  ``OZ_SECRET_KEY`` fails startup.
* **RBAC cache keys are scoped per ``(org_id, user_id)``** for roles,
  permission sets, and the must-change-password flag, so a user with roles in
  two organizations can no longer read one organization's cached role from the
  other's lookup.
* **RLS bypass reset is hardened** with a nested rollback, so a failed reset
  always releases the transaction instead of leaving tenant bypass enabled.
* Blob uploads verified with libmagic before dispatch, with generic
  ``application/octet-stream`` claims resolved by content detection instead of
  trusted as-is.

Reliability and performance
~~~~~~~~~~~~~~~~~~~~~~~~~~~

* **HNSW index usage restored** on episode and fact vector search: results are
  ordered by the raw ``<=>`` operator instead of the ``score`` SELECT alias,
  which previously forced a sequential scan plus full sort. Result sets are
  unchanged — only the query plan.
* **Hybrid retrieval degrades per leg.** A failing leg reports ``degraded``
  with the offending ``failed_legs`` instead of failing the whole query; legs
  run concurrently on sibling RLS-bound sessions and fall back to sequential
  execution when no sibling-session factory is available.
* **Ingest deduplication is atomic.** The dedupe claim is a single
  ``SET NX EX`` with token-checked release, accepted batches return ``202``
  with a ``Location`` header, and dedupe entries expire after 48 hours.
* **Post-commit failures keep the ``202``.** Context-cache purge and webhook
  emission failures no longer turn a committed ingest into a ``500``; the
  failure is logged and the response contract is preserved.
* **Uploads are capped at 10 files and 200 MB per request.**
* **Degraded startup is explicit.** ``/health`` returns ``200`` for liveness
  while ``/ready`` returns ``503`` with ``pending_edge_sync`` until graph edge
  sync catches up.
* **Leader-guarded scheduled work.** Cron jobs, cache prewarm, and seeding run
  on a single replica instead of every replica.
* **Pool budgets are explicit** — 4 + 1 for the database, 10 for FalkorDB.
* **Batch ingest latency work** removes per-item round trips from the batch
  path, so large writes no longer pay call overhead per record.

Companion releases
~~~~~~~~~~~~~~~~~~

* **openzync-core 1.0.1** — organization-gated wipes, org-scoped update
  predicates, the CORS wildcard startup guard, and the security and retrieval
  fixes above.
* **Python SDK 1.0.1** — unified retry loops across JSON and multipart
  requests, pagination that stops on a null cursor, a 60-second project-ID
  cache TTL, and a loud failure when version metadata is missing instead of a
  fabricated version.
* **MCP server 1.0.1** — added a ``/health`` route with a matching container
  probe, returned per-type memory results unsliced, and respected the
  ``get_user_graph`` limit with null-safe search scores.
* **Dashboard 1.0.1** — token access centralized behind ``tokenStorage`` with
  a bounded GET cache and security headers, plus sessions that survive reloads
  and cross-tab token rotation.
