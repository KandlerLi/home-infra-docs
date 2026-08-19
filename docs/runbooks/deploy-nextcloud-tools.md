# Runbook: Deploy Read-Only Nextcloud Tools

Status: **Implementation prepared and validated locally; no account,
credential, inventory enable flag, or production service has been created.**

This milestone lets `home-agent` list/search files and read small text files
below one approved folder. It does not grant Open WebUI a credential, mount
Nextcloud storage, analyse image contents, extract PDF/Office documents, index
files, or perform writes.

## 1. Review the boundary

Read ADR 0008 and confirm the committed implementation revision. Verify
Nextcloud, `home-agent`, Open WebUI, and Traefik are healthy. Confirm that an
unauthenticated `PROPFIND` to the loopback WebDAV endpoint returns HTTP 401.

The service is restricted to `127.0.0.1:11000`, the HTTP host header
`nextcloud.jkandler.de`, and one relative root folder. Systemd permits only
Unix sockets and loopback IPv4. The agent receives only
`/run/nextcloud-tools/nextcloud-tools.sock` and a supplementary socket group.

## 2. Create the dedicated Nextcloud identity

In the Nextcloud administration UI, create a non-admin account such as
`home-agent`. Give it a small quota and do not add it to privileged groups.
Do not reuse the human administrator account.

From the human account that owns the intended data:

1. Create or select a folder named `AI Workspace`.
2. Share it only with the dedicated account.
3. Disable editing so the share is read-only.
4. Keep downloading allowed because bounded text reads use WebDAV `GET`.
5. Add only files that may be sent to the configured cloud model when asked.

Log in as the dedicated account and create one device/app password named
`home-agent-readonly` under personal Security settings. Nextcloud shows the
plain value only when it is created, and a unique token can later be revoked
independently. See the official [device password documentation](https://docs.nextcloud.com/server/stable/user_manual/en/session_management.html)
and [WebDAV file-operation documentation](https://docs.nextcloud.com/server/stable/developer_manual/client_apis/WebDAV/basic.html).

## 3. Store the credential with SOPS

Edit only the encrypted inventory:

```bash
cd /home/julian/projects/home-infra
sops ansible/inventory/group_vars/all/secrets.sops.yml
```

Add the dedicated username and app password:

```yaml
nextcloud_tools_username: "home-agent"
nextcloud_tools_app_password: "generated-device-password"
```

Never paste the value into chat, shell arguments, documentation, or Git in
plaintext. Do not enable the persistent inventory flag yet.

## 4. Deploy privately

Only after explicit production approval, run:

```bash
cd /home/julian/projects/home-infra
.venv/bin/ansible-playbook ansible/playbooks/nextcloud-tools.yml \
  --ask-become-pass
```

The play installs `nextcloud-tools.service`, writes the token as a root-managed
`0400` file owned by the service account, starts the Unix socket service, and
recreates `home-agent` with the read-only socket mount. It does not alter
Traefik, Open WebUI, Nextcloud AIO configuration, shares, or public routes.

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

Confirm a traversal request is rejected with HTTP 400 and that the service has
no route for write methods. Do not paste filenames, metadata, document content,
or logs into documentation.

In Open WebUI, explicitly ask `home-agent` to list the approved root, search a
known filename fragment, and read one harmless small text file. This sends only
the requested results to the configured cloud model. Confirm requests for a
photo body, PDF/Office content, writes, paths above the root, or excessive data
are refused or reported unavailable.

After successful validation, set `nextcloud_tools_enabled: true` in
`ansible/inventory/group_vars/all/main.yml`, run a syntax check, and reapply the
full site only with separate approval. Recheck Nextcloud and Open WebUI.

## 6. Roll back

To detach the agent socket first and then stop the credential-bearing service:

```bash
cd /home/julian/projects/home-infra
.venv/bin/ansible-playbook ansible/playbooks/disable-nextcloud-tools.yml \
  --ask-become-pass
```

Set `nextcloud_tools_enabled: false` in persistent inventory if it had been
enabled. The play preserves the configuration for diagnosis. Revoke the app
password in Nextcloud if compromise is suspected; do not delete the account or
share until recovery needs have been reviewed.
