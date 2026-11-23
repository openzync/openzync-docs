ADR 008: Production Installer Distribution
==========================================

**Status:** Accepted (2026-09-14)

----

Context
-------

OpenZync needed a production one-liner installer for full-stack
single-host deploys (backend compose + frontend dashboard). The
workspace is 7 independent repos with separate remotes, branches, CI,
and deploys. Runtime configuration lives in ``OZ_*`` environment variables
(``.env``) — the full runtime set, read directly at startup with no sidecar
or seal key. (This supersedes the earlier OpenBao sole-source model; see the
supersede note below — history preserved as written.)

----

Decision
--------

D1 — Installer source lives in ``openzync-core/infra/install.sh``.
It owns the compose reference, the ``.env`` bootstrap, and the GHCR
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
- No config drift: ``.env`` holds the full runtime set;
  the installer only bootstraps the secrets.

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

----

Amendment (2026-09-14)
----------------------

D2 corrected: distribution is GitHub raw on ``master``
(``https://raw.githubusercontent.com/openzync/openzync-core/master/infra/install.sh``,
verified live; both ``main`` and ``master`` exist on origin, ``master`` is
canonical and already includes the RLS fix commit). Rationale: zero landing
coupling, version = branch head, no vendoring/license mixing, no release
overhead. One-liner uses the raw URL directly. Landing redirect retained
as a deferred alternative.

----

Supersede note (2026-11-21)
---------------------------

The OpenBao sole-source model in Context/D1 above is superseded:
runtime configuration now lives in ``OZ_*`` environment variables
(``.env``), encryption is Fernet via ``core.crypto`` (``OZ_SECRET_KEY`` /
``OZ_SECRET_KEY_PREVIOUS`` dual-key window), and there is no sidecar,
seal key, or external secret store. The history above is preserved as
written; new work follows the env model.
