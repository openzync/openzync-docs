Rotating runtime keys (single-key pattern)
===========================================

OpenBao is the sole runtime-config source. The ``system/`` namespace
holds one KV-v2 secret at ``config/data/system`` and the API/worker
sidecars render it to ``/run/secrets/system.env`` at boot.

This runbook rotates **one** key inside that object. It uses
``OZ_SMTP_PASSWORD`` as the worked example — swap the key name for
any other ``OZ_*`` key; the steps are identical.

For the initial deploy and the zero-fallback secrets model, see
:doc:`/guides/deployment`. For the full service topology and the
bootstrap sequence, see :doc:`/domains/infrastructure`.

Prerequisites
-------------

* SSH access to the production host (all commands run there).
* ``VAULT_ADDR=http://127.0.0.1:8200`` — OpenBao listens on
  loopback; never expose it publicly.
* A root token, read from the init-data volume::

    TOKEN=$(cat /var/lib/docker/volumes/infra_openbao-init-data/_data/root-token)

* Every API call **must** send ``X-Vault-Namespace: system/``.
  Reads against the root namespace return ``404`` even when the
  path is correct.

.. warning::

    ``POST config/data/system`` **replaces the whole data object**.
    Never write a single key. Always read-modify-write **all** keys
    (expect ``27``). Writing one field alone deletes the rest.

Rotate one key
--------------

1. Capture the new value without echo or history::

    set +x
    read -rsp 'New OZ_SMTP_PASSWORD: ' NEW_VAL; echo

2. Back up the current key list (key names only, no values)::

    curl -s -H "X-Vault-Token: $TOKEN" \
      -H 'X-Vault-Namespace: system/' \
      "$VAULT_ADDR/v1/config/data/system" \
      | python3 -c 'import json,sys; print("\n".join(sorted(json.load(sys.stdin)["data"]["data"])))' \
      | tee /tmp/oz-keys-before.txt
    wc -l /tmp/oz-keys-before.txt  # expect 27

3. Read-modify-write: assert the key exists, overwrite one field,
   post the full object back. Print the key count only::

    curl -s -H "X-Vault-Token: $TOKEN" \
      -H 'X-Vault-Namespace: system/' \
      "$VAULT_ADDR/v1/config/data/system" > /tmp/oz-cur.json
    NEW_VAL="$NEW_VAL" TOKEN="$TOKEN" python3 - <<'EOF'
    import json, os, urllib.request
    base = 'http://127.0.0.1:8200/v1/config/data/system'
    ns = 'system/'
    token = os.environ['TOKEN']
    with urllib.request.urlopen(urllib.request.Request(
            base, headers={'X-Vault-Token': token,
                           'X-Vault-Namespace': ns})) as r:
        cur = json.load(r)['data']['data']
    assert 'OZ_SMTP_PASSWORD' in cur, 'key missing, aborting'
    cur['OZ_SMTP_PASSWORD'] = os.environ['NEW_VAL']
    req = urllib.request.Request(
        base, data=json.dumps({'data': cur}).encode(),
        headers={'X-Vault-Token': token,
                 'X-Vault-Namespace': ns,
                 'Content-Type': 'application/json'}, method='POST')
    with urllib.request.urlopen(req):
        pass
    print(f'keys={len(cur)}')  # expect keys=27, value never printed
    EOF

4. Wait ~12s for the agent sidecars to re-render, then confirm the
   rendered file changed (names only, values redacted)::

    sleep 12
    grep -o '^[A-Z0-9_]*' \
      /var/lib/docker/volumes/infra_api-secrets/_data/system.env \
      | sort | diff /tmp/oz-keys-before.txt - && echo 'key set unchanged'

5. Restart both services. **Mandatory** — the entrypoint sources
   ``/run/secrets/system.env`` at boot, so ``docker exec`` env is
   stale until restart::

    docker restart openzync-api openzync-worker

6. Verify: key-name listing (redacted), in-container check, probes,
   and mail logs::

    curl -s -H "X-Vault-Token: $TOKEN" \
      -H 'X-Vault-Namespace: system/' \
      "$VAULT_ADDR/v1/config/data/system" \
      | python3 -c 'import json,sys; print(sorted(json.load(sys.stdin)["data"]["data"]))'
    docker exec openzync-api sh -c 'set | grep -o "^OZ_[A-Z_]*" | sort | head -30'
    curl -s http://localhost:8000/health | python3 -m json.tool
    curl -s http://localhost:8000/ready | python3 -m json.tool
    docker logs openzync-api 2>&1 | grep -i email | tail -5
    unset NEW_VAL TOKEN; set -x

Security notes
--------------

* Never ``echo`` or log a secret value. Print key names and counts.
* ``unset`` the shell variables when done (last line above).
* Redact ``PASSWORD|SECRET|KEY|TOKEN`` in any pasted output.

Troubleshooting
---------------

* **Config looks stale** — you forgot the restart in step 5.
  The sidecar file updates, but the running process keeps its
  boot-time environment. Restart and re-verify.
* **Key count is not 27** — a partial write happened. Restore from
  ``/tmp/oz-keys-before.txt``: re-run step 3 against the previous
  KV-v2 version, confirm ``keys=27``, then restart.
* **Signup returns 502** — check for mail-send failures::

    docker logs openzync-api 2>&1 | grep -i 'send_failed' | tail -5

Rollback
--------

KV-v2 keeps prior versions of ``config/data/system``. To roll back,
write back the previous version's full data object (same
read-modify-write path, no single-key writes), confirm ``keys=27``,
restart both services, and re-run the step 6 probes.

Related documentation
---------------------

* :doc:`/guides/deployment` — deploy runbook, zero-fallback model
* :doc:`/domains/infrastructure` — OpenBao topology and bootstrap
* :doc:`/guides/quickstart` — first-boot walkthrough
