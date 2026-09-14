ADR 008: Production Installer Distribution
==========================================

**Status:** Accepted (2026-09-14)

----

Context
-------

OpenZync needed a production one-liner installer for full-stack
single-host deploys (backend compose + frontend dashboard). The
workspace is 7 independent repos with separate remotes, branches, CI,
and deploys. OpenBao is the sole runtime-config source, with only 4
bootstrap secrets allowed in ``.env`` (``BAO_STATIC_SEAL_KEY``,
``POSTGRES_PASSWORD``, ``OZ_SECRET_KEY``,
``OZ_WEBHOOK_SIGNING_SECRET``).

----

Decision
--------

D1 — Installer source lives in ``openzync-core/infra/install.sh``.
It owns the compose reference, the OpenBao bootstrap, and the GHCR
image references.

D2 — Distribution via a stable landing URL
(``openzync.tech/install.sh`` → versioned asset). Core owns installer
behavior; landing owns the URL.

D3 — The frontend is pulled as a GHCR image reference only, never
vendored. The container boundary keeps AGPLv3 core / MIT frontend
clean.

D4 — Tags float on ``:latest``. Drift risk is accepted and noted
in-script; a pinned manifest is deferred (see below).

D5 — TLS is out of scope: HTTP on loopback, terminate upstream.

D6 — Interactive prompts for every choice, with a ``--yes`` flag for
CI. Scope is full-stack single host, with a bundled local-db default
path.

----

Alternatives Considered
-----------------------

**Dedicated 8th repo for the installer**
   Rejected: overhead of a repo (CI, deploy, versioning) for one
   script.

**Docs-hosted distribution**
   Rejected: the docs repo is CC BY Sphinx — the wrong medium for a
   shell distribution artifact.

**Pinned manifest (semver-pinned image set)**
   Ideal but deferred: a single version file pinning every image.
   Revisit when multi-host or version-skew issues appear.

----

Consequences
------------

Positive
~~~~~~~~

- One-liner install: ``curl -fsSL openzync.tech/install.sh | bash``.
- No license mixing: frontend crosses the boundary only as a
  container image.
- No config drift: OpenBao stays the sole runtime-config source;
  the installer only bootstraps the 4 secrets.

Negative
~~~~~~~~

- ``:latest`` float means installer runs at different times may pull
  different images (drift risk, noted in-script).
- Cross-cutting version bumps require separate commits per repo.
- Docs quickstart/deployment must reference the one-liner and stay
  in sync with installer behavior.

----

References
----------

- ``openzync-core/infra/install.sh`` — installer source
- ``openzync-landing`` — stable ``/install.sh`` URL owner
- ``guides/quickstart.rst``, ``guides/deployment.rst`` — one-liner
  references
