# Operational Notes

Run commands from the owning repository and inspect its current README/runbook
before use. These examples are secret-safe.

## Workspace inventory

```bash
find /home/julian/projects \
  -mindepth 2 -maxdepth 2 -type d -printf '%P\n' | sort
```

For each result, determine whether it is a Git root before assuming `git status`
is available. Do not scan state files to determine project purpose; inspect
source, README, ignore rules, and file metadata instead.

## Fast public checks

```bash
curl --output /dev/null --silent --write-out '%{http_code}\n' \
  https://ai.jkandler.de/healthz
# Expected without credentials: 401

curl --user julian https://ai.jkandler.de/healthz
# Prompts for the password; expected: {"status":"ok"}

curl --fail-with-body https://nextcloud.jkandler.de/status.php
```

## Host checks

```bash
ssh julian@192.168.178.100
docker ps
systemctl --failed
curl --fail-with-body http://127.0.0.1:8082/ping
curl --fail-with-body http://127.0.0.1:8090/healthz
```

Filter `docker inspect` with an explicit format. Do not print environments.

## Hardware-maintenance shutdown

The implementation repository contains the guarded helper:

```text
/home/julian/projects/infra/home-infra/scripts/shutdown-homeserver-for-maintenance.sh
```

Copy it to `/tmp` on the host, run `--check` through an interactive SSH session,
then run it without that option and enter the exact `POWER OFF` confirmation.
It checks Docker, `/mnt/black-hdd`, failed units, running libvirt VMs, and AIO
state. It uses AIO's supported `STOP_CONTAINERS=1` mechanism, verifies that AIO
application containers have stopped, flushes filesystem buffers, and invokes
`systemctl poweroff`. It does not create a backup or bypass the requirement to
verify one. See the human-facing Nextcloud AIO runbook for exact commands and
post-boot validation. The 2026-08-19 maintenance confirmed that the stopped AIO
application containers remained stopped after boot; start them through
`https://192.168.178.100:8080` before performing the endpoint checks.

## Agent API

```bash
curl --user julian \
  --header 'Content-Type: application/json' \
  --data '{"message":"Check my homeserver."}' \
  https://ai.jkandler.de/v1/chat
```

Public `GET /` returns Open WebUI. Exact `POST /v1/chat` remains the
Basic-Auth-protected legacy API interface.

## Private Open WebUI workflow (historical — superseded 2026-08-30)

**This entire section describes the original homeserver-Docker deployment,
stopped 2026-08-30 and superseded by a k3s-native Deployment**
(`infra/k3s-apps`' `modules/open_webui`, reached via `ai_ingress.tf`'s path
split with `home_agent`; see `repositories.md`'s `modules/open_webui`
entry and `current-state.md`'s Docker-vs-k3s table). Since 2026-09-08 its
login is Authelia OIDC SSO only — native login/signup are disabled at the
protocol level; see
`docs/home-infra-docs/docs/runbooks/harden-open-webui-login.md` for that
change and a deployment-specific gotcha it documents. Kept below only as a
historical record of the original bootstrap; the commands are no longer
runnable as-is (they target the retired Docker container).
`publish-open-webui.md` itself was the same way -- retired 2026-09-16
with a pointer to the current mechanism, rather than left presenting
dead commands as a valid runbook.

Do not combine private deployment and public publication into one approval.
Follow the (now-historical) runbook:

```text
/home/julian/projects/docs/home-infra-docs/docs/runbooks/publish-open-webui.md
```

The private deployment completed on 2026-08-19. Its reproducible command was:

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/ansible-playbook ansible/playbooks/open-webui.yml \
  --ask-become-pass
```

Private administrator onboarding completed on 2026-08-19, and signup is
disabled. The guarded publication playbook completed successfully with
`open_webui_publish_confirmation=PUBLISH_OPEN_WEBUI`; rollback requires
`open_webui_rollback_confirmation=ROLL_BACK_OPEN_WEBUI`. Never use unfiltered
container inspection or record the Open WebUI/Basic Auth passwords.

The first apply stopped at loopback readiness because Docker suppresses host
port mappings on `internal: true` networks. Commit `9c97e88` migrated the
frontend to a non-masqueraded bridge, preserving loopback publication while
blocking egress. The rerun and live compatibility completion succeeded.

Public route checks returned Open WebUI HTTP 200 with onboarding closed and
signup disabled. Exact legacy health/chat paths still returned 401 without
Basic Auth, and Nextcloud remained healthy. Authenticated public browser/chat
was user-confirmed, and the deployed inventory flags were persisted in
infrastructure commit `941bb52`.

A fresh iPhone load later exposed HTTP 429 responses during frontend bootstrap:
the HTML references 56 same-origin assets before startup API traffic, but the
deployed UI burst is 60. Infrastructure revision `784ed03` raises only the UI
burst to 240 while keeping the sustained 120 requests/minute limit. It was
deployed and validated on 2026-08-20. For future recovery, apply the existing
published route without the initial publication sentinel:

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/ansible-playbook ansible/playbooks/publish-open-webui.yml \
  --ask-become-pass
```

Validation passed in a new private tab on the affected phone. The
unauthenticated legacy `/healthz` boundary remained 401 and public Nextcloud
returned 200.

## Deployed read-only Nextcloud tools

The private service and data share are deployed, and the socket mount is fixed.
An initial list returned allowlisted `upstream_http_404`, revealing that the
configured received-share folder is `/Shared`. Revision `bffedd8` corrected the
root to `Shared/AI Workspace`; current local-only list/search/text-read checks
and Open WebUI acceptance pass. Revision `54a521e` persists
`home_agent_enabled` and `nextcloud_tools_enabled`. Follow the authoritative
runbook:

```text
/home/julian/projects/docs/home-infra-docs/docs/runbooks/deploy-nextcloud-tools.md
```

**Retired 2026-08-30, `018c5ff`**: the commands this section originally
documented (`ansible/playbooks/rotate-nextcloud-tools-token.yml`,
`ansible/playbooks/disable-nextcloud-tools.yml`) no longer exist --
`infra/home-infra`'s whole `nextcloud_tools` role and its playbook
family were deleted outright once `home_agent`/`nextcloud_tools` moved
to a k3s-native Pod (`infra/k3s-apps`' `modules/home_agent`). For
token rotation now, see
`docs/home-infra-docs/docs/runbooks/rotate-secrets.md`'s
`nextcloud_tools_app_password` section (no helper script yet -- a
manual `put-secret-value` + `terraform apply` in `infra/k3s-apps`).
Never place the app password
in AWS Secrets Manager outside a deliberate `put-secret-value` step, shell
arguments, chat, logs, or documentation.

Use only deliberately approved data for future Open WebUI requests because
requested metadata/content crosses the configured cloud-model boundary.

## Secrets

Secrets live in AWS Secrets Manager (as of the 2026-09-09
SOPS-to-Secrets-Manager cutover -- SOPS/`secrets.sops.yml` are gone
from this workspace entirely). Edit one by merging into its existing
JSON blob (each secret holds several related keys), never overwriting
it outright -- see `infra/home-infra`'s own README.md "First apply"
section for the exact `get-secret-value`/`put-secret-value` commands.

The Basic Auth password can be retrieved directly into a password manager with
`aws secretsmanager get-secret-value` against the `home-infra/ingress` group
(see the `## Secrets` commands above) -- the publish-home-agent runbook's own
extract command still shows `sops -d --extract`, the real command at the time
that rollout actually ran; it's a completed historical record, not live
guidance, so it wasn't rewritten for the 2026-09-09 cutover. Never paste the
password into a task/chat or this repository.

## Ingress playbooks

Dedicated playbooks exist for preparation, Nextcloud cutover, agent routing,
and rollback. Cutover/rollback sentinels are intentionally required. Consult:

```text
/home/julian/projects/docs/home-infra-docs/docs/runbooks/publish-home-agent.md
```

Do not improvise port ownership changes with direct `docker run` commands.

## DNS workflow

Change `dyndns/deployment.auto.tfvars` on a branch, validate, open a pull
request, review the speculative plan, merge, wait for the protected apply, then
verify using independent public resolvers. Do not add DNS records manually when
Terraform owns them.

## repo-infra workflow (GitHub settings + AWS identity control plane)

`/home/julian/projects/github/repo-infra` (renamed from `github-infra`; absorbed the
formerly separate `aws-account-bootstrap` root on 2026-08-21) is local-only.
Never add a GitHub Actions plan/apply workflow and never use one of its
managed identities to change the root.

```bash
cd /home/julian/projects/github/repo-infra
terraform fmt -check -recursive
terraform validate
terraform plan
```

`terraform plan`/`apply` require a human-selected, short-lived AWS identity
(e.g. exported from a password store, never left in shell history) plus a
`GITHUB_TOKEN`. Do not inspect or manually transfer raw state. Any
unexplained create, replacement, or destroy proposal is a stop condition.

The original DynDNS-identity no-destroy migration into `aws-account-bootstrap`
completed on 2026-08-19; its retained procedure and evidence are documented
in:

```text
/home/julian/projects/docs/home-infra-docs/docs/runbooks/migrate-aws-account-bootstrap.md
```

The current procedure for adding a repository (with or without pre-existing
manual GitHub/AWS configuration) is documented in:

```text
/home/julian/projects/docs/home-infra-docs/docs/runbooks/manage-repo-infra.md
```

## Documentation workflow

After relevant implementation, cloud/DNS, workflow, security, architecture, or
production changes, update this repository in the same work session
(`current-state.md` for live facts, an ADR for decisions, a runbook for
procedures). Verify code fences and `git diff --check` before committing.
