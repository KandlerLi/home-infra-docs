# Runbook: Deploy and Publish Open WebUI

Status: **Private deployment, administrator onboarding, guarded public
publication, authenticated chat validation, and inventory-flag persistence
completed on 2026-08-19.**

This runbook adds a chat window without granting the frontend direct host,
Nextcloud, or OpenAI access. Production deployment is a separate human-
confirmed action.

## 1. Preflight

1. Review the infrastructure diff and exact Open WebUI tag/digest.
2. Confirm at least 2 GiB free on `/` for the image pull and growth of new
   state. The compressed image is approximately 1.44 GiB.
3. Confirm `home-agent`, `home-tools`, Traefik, and Nextcloud are healthy.
4. Confirm the current public API still returns 401 without credentials and
   HTTP 200 with credentials.
5. Run unit tests, production-profile `ansible-lint`, and syntax checks.

## 2. Deploy privately

Run only after explicitly approving a production apply:

```bash
cd /home/julian/projects/home-infra
.venv/bin/ansible-playbook ansible/playbooks/open-webui.yml \
  --ask-become-pass
```

The play creates the non-masqueraded `home-agent-frontend` bridge, recreates
`home-agent` with an additional network attachment, and starts Open WebUI on
`127.0.0.1:8091`. It does not alter the public Traefik route. Disabling IP
masquerading permits the loopback port mapping while blocking frontend
external egress.

Verify with filtered output only:

```bash
curl --fail-with-body http://127.0.0.1:8091/health
docker inspect open-webui \
  --format 'running={{.State.Running}} health={{.State.Health.Status}} user={{.Config.User}}'
docker network inspect home-agent-frontend \
  --format 'internal={{.Internal}} options={{json .Options}} containers={{len .Containers}}'
```

Do not record an unfiltered `docker inspect`; it includes environment values.

### Initial deployment recovery

The first apply created Open WebUI on a Docker network with `internal: true`.
Docker retained the requested port in `HostConfig` but suppressed the runtime
port mapping, so the final loopback readiness task failed with connection
refused. Open WebUI itself later became internally healthy; the current agent
and Nextcloud stayed healthy throughout.

Infrastructure commit `9c97e88` changed the bridge to
`enable_ip_masquerade=false`, added an idempotent one-time network migration,
enabled offline/RAG-bypass startup settings, and extended the readiness window.
A bounded host test proved loopback publication worked and outbound HTTPS was
blocked. The idempotent rerun then completed successfully. Final checks showed:

- healthy Open WebUI and `home-agent` containers with zero Open WebUI restarts
- listeners only on `127.0.0.1:8090` and `127.0.0.1:8091`
- all capabilities dropped, `no-new-privileges`, non-privileged mode, and the
  expected CPU, memory, and PID limits
- only Open WebUI and `home-agent` attached to the frontend bridge
- synthetic `home-agent` model discovery and a live compatibility completion
- blocked Open WebUI external HTTPS egress
- unchanged public agent HTTP 401 and healthy Nextcloud status

## 3. Bootstrap the first administrator

Before public publication, create the sole initial administrator through an
SSH tunnel:

```bash
ssh -L 8091:127.0.0.1:8091 julian@192.168.178.100
```

Open `http://127.0.0.1:8091` locally and create a dedicated strong Open WebUI
password. Do not reuse or record the Traefik Basic Auth password. Open WebUI
persists configuration and automatically disables public signup after the
first administrator; confirm that a second anonymous signup is unavailable.

Start a chat with `home-agent` and request “Check my homeserver.” Confirm the
response uses the current health tools. Confirm uploads, workspace tools,
plugins, web search, and code execution are unavailable.

Onboarding completed on 2026-08-19. The user reported the private UI working,
and a secret-free `/api/config` check confirmed onboarding false,
authentication/login enabled, and anonymous signup disabled. Open WebUI,
`home-agent`, the existing public 401 boundary, and Nextcloud remained healthy.

## 4. Publish the chat window

The publication play refuses an initial route switch unless the exact
confirmation is supplied:

```bash
.venv/bin/ansible-playbook ansible/playbooks/publish-open-webui.yml \
  --ask-become-pass \
  --extra-vars open_webui_publish_confirmation=PUBLISH_OPEN_WEBUI
```

This preserves exact `/healthz` and `/v1/chat` routes to the existing API and
sends other `ai.jkandler.de` requests to Open WebUI.

The guarded apply completed with 16 tasks successful, two expected skips, and
no failures. Public `/`, `/health`, and `/api/config` returned Open WebUI
successfully; signup remained disabled. Unauthenticated legacy health/chat
requests returned 401, Traefik reported healthy, and Nextcloud remained healthy.
The user subsequently confirmed an authenticated public chat, and infrastructure
commit `941bb52` persisted all deployed enable flags in inventory.

Verify:

1. An anonymous request to `https://ai.jkandler.de/` shows the Open WebUI login
   page but cannot read chats or use the model.
2. Open WebUI login and a streamed chat work from an external IPv4 network.
3. Unauthenticated `/healthz` still returns 401, and Basic-Auth-protected
   `/healthz` and `/v1/chat` still return HTTP 200.
4. A new anonymous account cannot be registered after the first administrator.
5. Nextcloud status, login, WebDAV, and AIO health remain unchanged.
6. Logs contain no password, API key, prompt, or tool payload.

The deployed enable flags are persisted in inventory so later `site.yml` runs
retain the verified state.

## 5. Roll back the route

If the UI or chat path fails, restore the previous direct API route:

```bash
.venv/bin/ansible-playbook ansible/playbooks/rollback-open-webui.yml \
  --ask-become-pass \
  --extra-vars open_webui_rollback_confirmation=ROLL_BACK_OPEN_WEBUI
```

This leaves the Open WebUI container and `/var/lib/open-webui` intact for
diagnosis. Verify the authenticated `/healthz` and `/v1/chat` endpoints and
Nextcloud again. Do not delete the data directory as part of routing rollback.

## 6. Backup and upgrade

- Back up `/var/lib/open-webui` before every image upgrade. Treat the backup as
  private because it contains accounts and chat history.
- Keep automatic container updates disabled. Review release notes and security
  advisories, then update both the tag and digest in one reviewed change.
- Validate private login and chat before publishing an upgraded container.
- Reassess whether the upstream image can run with a read-only root filesystem.
- Do not enable Nextcloud tools as part of an Open WebUI version upgrade.
