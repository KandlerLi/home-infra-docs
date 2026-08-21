# Runbook: Deploy Read-Only Nextcloud Tools

Status: **Deployed.** `nextcloud_tools_enabled` and `home_agent_enabled` are
`true`; list/search/read validation passed locally and through Open WebUI.

Since 2026-08-21, ADR 0011 adds write capability (create/update/delete/move,
gated behind explicit confirmation) on top of the same service this runbook
deploys — see
[Enable Nextcloud write tools](enable-nextcloud-writes.md) for that
follow-on step. This runbook's own scope remains list/search/read only; it
does not grant Open WebUI a credential, mount Nextcloud storage, analyse
image contents, extract PDF/Office documents, or index files.

## 1. Review the boundary

Read ADR 0008 and confirm the committed implementation revision. Verify
Nextcloud, `home-agent`, Open WebUI, and Traefik are healthy. Confirm that an
unauthenticated `PROPFIND` to the loopback WebDAV endpoint returns HTTP 401.

The service is restricted to `127.0.0.1:11000`, the HTTP host header
`nextcloud.jkandler.de`, and one relative root folder. Systemd permits only
Unix sockets and loopback IPv4. The agent receives only
`/run/nextcloud-tools/nextcloud-tools.sock` and a supplementary socket group.

## 2. Bootstrap the identity, token, and private service

The live Nextcloud version supports `occ user:add --generate-password` and
`occ user:auth-tokens:add --no-interaction`. The second command explicitly
creates a limited token when no login password is provided. The guarded
playbook uses those supported commands inside `nextcloud-aio-nextcloud`.

Only after explicitly approving creation of the Nextcloud account/token and
deployment of the host service, run:

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/ansible-playbook ansible/playbooks/nextcloud-tools.yml \
  --ask-become-pass \
  --extra-vars \
  nextcloud_tools_bootstrap_confirmation=BOOTSTRAP_NEXTCLOUD_TOOLS
```

The play:

1. Requires the exact confirmation sentinel.
2. Verifies that the AIO Nextcloud container is running.
3. Creates non-admin account `home-agent` with a generated login password that
   is never displayed.
4. Creates limited app password `home-agent-readonly` without receiving the
   login password.
5. Keeps the token only in non-cached Ansible memory under `no_log` and writes
   it directly as the service-owned `0400` host file.
6. Starts `nextcloud-tools.service`, attaches only its read-only Unix socket to
   `home-agent`, and verifies the private health endpoint.

The token is not stored in SOPS, shell arguments, Docker environment, Git,
chat, or documentation. The play refuses to silently create another managed
token if a prior token exists but the host file is missing; use the guarded
rotation/recovery play in section 6 for that state.

The bootstrap does not alter shares, personal files, Traefik, Open WebUI,
Nextcloud AIO configuration, or public routes.

The underlying supported commands are documented in Nextcloud's official
[user/token command reference](https://docs.nextcloud.com/server/stable/admin_manual/occ_users.html).

## 3. Share the approved folder manually

After the bootstrap succeeds, sign in to Nextcloud with the normal account that
owns the intended data:

1. Create or select a folder named `AI Workspace`.
2. Share it only with user `home-agent`.
3. Disable editing so the share is read-only (this milestone only reads;
   [Enable Nextcloud write tools](enable-nextcloud-writes.md) documents
   the separate, later step of turning editing back on for ADR 0011).
4. Keep downloading allowed because bounded text reads use WebDAV `GET`.
5. Add only files that may be sent to the configured cloud model when asked.

This deployment configures Nextcloud's received-share folder as `/Shared`.
Keep the owner-visible folder name `AI Workspace`, and set the service root to:

```yaml
nextcloud_tools_allowed_root: Shared/AI Workspace
```

This is deliberately the only manual identity/data step. Creating the share
through Ansible's supported OCS API would require a credential for the personal
owner account. Do not provide that credential to Ansible. The API's read-only
permission value is documented in the official [OCS Share API reference](https://docs.nextcloud.com/server/stable/developer_manual/client_apis/OCS/ocs-share-api.html).

## 4. Keep persistent inventory disabled during validation

Do not set `nextcloud_tools_enabled: true` until the live chat tests below
succeed. The guarded bootstrap explicitly enables the role for its own
apply; ordinary `site.yml` should not reconcile it before that. (This
milestone is now deployed and `nextcloud_tools_enabled: true` is
persisted — this section is kept for the next time this service is
bootstrapped from scratch, e.g. account recovery.)

## 5. Validate without exposing private data

On the homeserver, verify only bounded status and counts:

```bash
systemctl is-active nextcloud-tools.service
systemctl is-active docker.service
curl --fail-with-body --unix-socket \
  /run/nextcloud-tools/nextcloud-tools.sock \
  --header 'Content-Type: application/json' \
  --data '{"path":""}' \
  http://localhost/v1/list \
  | jq '{root, entry_count: (.entries | length)}'
```

Confirm a traversal request is rejected with HTTP 400. (As of ADR 0011 the
service does implement write routes, but only reachable through the gated
propose/confirm flow — see
[Enable Nextcloud write tools](enable-nextcloud-writes.md) for how those
are verified.) Do not paste filenames, metadata, document content, or logs
into documentation.

In Open WebUI, explicitly ask `home-agent` to list the approved root, search a
known filename fragment, and read one harmless small text file. This sends only
the requested results to the configured cloud model. Confirm requests for a
photo body, PDF/Office content, writes, paths above the root, or excessive data
are refused or reported unavailable.

After successful validation, set `nextcloud_tools_enabled: true` in
`ansible/inventory/group_vars/all/main.yml`, run a syntax check, and reapply the
full site only with separate approval. Recheck Nextcloud and Open WebUI.

## 6. Rotate or recover the managed token

Token rotation requires a separate exact confirmation. It records existing
token IDs without exposing their values, creates and validates a replacement
against loopback WebDAV, installs it, restarts the service, and only then
revokes the superseded tokens named `home-agent-readonly`:

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/ansible-playbook \
  ansible/playbooks/rotate-nextcloud-tools-token.yml \
  --ask-become-pass \
  --extra-vars \
  nextcloud_tools_rotation_confirmation=ROTATE_NEXTCLOUD_TOOLS_TOKEN
```

The same play recovers a partial bootstrap where the Nextcloud account/token
exists but `/etc/nextcloud-tools/app-password` is missing. It does not need the
old token value.

### Diagnose a generic tool 503

Revision `c720e00` logs only an upstream HTTP status or transport exception
class when a bounded tool request fails. It never logs the requested path,
filenames, content, or credentials. Reapply the bootstrap play with its exact
sentinel to install this service revision. When the host credential file
already exists, the play reuses the account and file and creates no token.
Repeat one bounded request, then inspect only the relevant
`nextcloud-tools.service` warning. Keep the inventory enable flag false until
the cause is corrected and list/search/read validation passes.

If `ss` reports the listener while the socket pathname is absent, deploy at
least revision `cb6a3d2`. It preserves systemd's runtime directory across
restarts so the container bind mount retains the same inode. Dedicated deploy,
rotation, and rollback plays also force `home-agent` recreation to remount the
current directory and apply socket attachment changes deterministically.

If journald has no service records, deploy at least revision `8ceb715`. Its 503
JSON response contains only an allowlisted upstream HTTP status, transport
exception class, or generic invalid-response category. It cannot return paths,
filenames, content, credentials, or an unstructured exception message.

## 7. Roll back

To detach the agent socket first and then stop the credential-bearing service:

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/ansible-playbook ansible/playbooks/disable-nextcloud-tools.yml \
  --ask-become-pass
```

Set `nextcloud_tools_enabled: false` in persistent inventory if it had been
enabled. The play preserves the configuration for diagnosis. Revoke the app
password in Nextcloud if compromise is suspected; do not delete the account or
share until recovery needs have been reviewed.
