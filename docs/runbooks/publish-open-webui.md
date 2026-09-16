# Runbook: Deploy and Publish Open WebUI — retired

**Retired 2026-08-30, not applicable to the current architecture.**
This runbook described `infra/home-infra`'s own homeserver-Docker
deployment of Open WebUI (`ansible/playbooks/open-webui.yml`,
`publish-open-webui.yml`, `rollback-open-webui.yml`) — none of these
playbooks exist any more. Open WebUI itself moved to a k3s-native
Deployment the same day (`infra/k3s-apps`' own `modules/open_webui`,
reached via `ai_ingress.tf`'s path split with `home_agent` on
`ai.jkandler.de`); `home-infra`'s own `open_webui` role now keeps only
the service account and `open_webui_data_dir` host prerequisites the
k3s Pod's NFS mount still depends on, nothing deployable on its own.
Login has also changed since this runbook was current: Authelia OIDC
SSO only since 2026-09-08, not the Basic Auth/native-signup flow
described below — see
[Harden Open WebUI login](harden-open-webui-login.md).

There's no dedicated deploy/publish runbook for the current shape —
Open WebUI is deployed the same way every other `infra/k3s-apps`
module is, through that repo's own normal Terraform apply (CI on
merge to `main`). See
`docs/home-infra-ai-context/context/current-state.md`'s "k3s learning
cluster" section for the full cutover history.

The one piece of this runbook's original content that's still real and
unaddressed: **backing up `/var/lib/open-webui` before every image
upgrade** (this file's old section 6) was a genuine, correct concern —
and had zero actual backup coverage until `infra/home-infra`'s new
`aux_backup` role (2026-09-16) started covering it. See that role's
own README and `PARKED.md`'s "Nextcloud data has zero backup" entry
for the current backup story; there's still no dedicated
pre-upgrade-backup step wired into the k3s-native deploy flow itself,
just the daily `aux_backup` timer.
