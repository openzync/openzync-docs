Rotating runtime keys (dual-key rotation)
=========================================

All runtime configuration lives in ``OZ_*`` environment variables
(``.env``), read directly at startup — there is no secret store, no
sidecar, and no seal key. Secrets at rest (webhook signing secrets, PII
fields) are encrypted with Fernet under per-purpose subkeys derived from
``OZ_SECRET_KEY`` (see ``core.crypto``). ``OZ_SECRET_KEY_PREVIOUS``
decrypts only, providing the rotation window: set it while rotating so
old ciphertexts still verify while new ones use the fresh key.

For the initial deploy and the plain-env model, see
:doc:`/guides/deployment`. For the full service topology, see
:doc:`/domains/infrastructure`.

Rotate OZ_SECRET_KEY
--------------------

1. Generate the new key without echo or history::

    set +x
    NEW_KEY=$(python3 -c 'import secrets; print(secrets.token_urlsafe(48))')

2. Open the dual-key window — keep the old key as decrypt-only.
   In ``.env``::

    OZ_SECRET_KEY=<new value from step 1>
    OZ_SECRET_KEY_PREVIOUS=<old OZ_SECRET_KEY value>

   Restart both services so new encryptions use the fresh key while old
   ciphertexts still decrypt::

    docker restart openzync-api openzync-worker

3. Re-encrypt live secrets under the new key. Webhook signing secrets
   are single-key Fernet rows — **re-create each webhook** (create with
   the new key, verify delivery, delete the old one). Per-user LLM keys
   and PII fields in ``users.config_override`` are re-encrypted on next
   write; to force it, re-save the platform config.

4. Verify, then close the window::

    curl -s http://localhost:8000/health | python3 -m json.tool
    curl -s http://localhost:8000/ready | python3 -m json.tool

   Once every secret has been re-created/re-saved under the new key,
   remove ``OZ_SECRET_KEY_PREVIOUS`` from ``.env`` and restart again.
   A row still encrypted under the old key fails loudly after this —
   that is the signal it was missed, not a bug.

Rotate any other OZ_* value
----------------------------

Plain values (``OZ_SMTP_PASSWORD``, feature flags, URLs) need no
window. This uses ``OZ_SMTP_PASSWORD`` as the worked example — swap the
key name for any other ``OZ_*`` key; the steps are identical.

1. Capture the new value without echo or history::

    set +x
    read -rsp 'New OZ_SMTP_PASSWORD: ' NEW_VAL; echo

2. Write it to ``.env`` and restart::

    # edit .env: OZ_SMTP_PASSWORD=<new value>
    docker restart openzync-api openzync-worker

3. Verify::

    curl -s http://localhost:8000/health | python3 -m json.tool
    curl -s http://localhost:8000/ready | python3 -m json.tool
    docker logs openzync-api 2>&1 | grep -i email | tail -5
    unset NEW_VAL; set -x

Security notes
--------------

* Never ``echo`` or log a secret value.
* ``unset`` the shell variables when done (last line above).
* Redact ``PASSWORD|SECRET|KEY|TOKEN`` in any pasted output.

Rollback
--------

Inside the dual-key window, roll back by setting ``OZ_SECRET_KEY`` to
the previous value (or keeping ``OZ_SECRET_KEY_PREVIOUS`` set) and
restarting. After the window is closed, restore ``.env`` from backup
and re-create any secrets written under the discarded key.

Related documentation
---------------------

* :doc:`/guides/deployment` — deploy runbook, plain-env model
* :doc:`/domains/infrastructure` — service topology
* :doc:`/guides/quickstart` — first-boot walkthrough
