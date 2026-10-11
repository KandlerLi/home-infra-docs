# Runbook: Enable the Finanzfluss Sankey exporter — retired

**Retired 2026-09-07, not applicable to the current architecture.**
This runbook described [`home-infra`](https://github.com/KandlerLi/home-infra)'s own systemd-timer-based
deployment of the Sankey exporter (`ansible/playbooks/sankey-export.yml`,
the `sankey_export` role and its `sankey-export.timer`/`.service`) — the
whole role was deleted outright once the exporter moved to a k3s
CronJob ([`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' own `modules/sankey_export/`), the same
precedent `landing_page`'s own earlier retirement set: it owned no host
prerequisite anything else depended on, so nothing was left behind the
way `nextcloud_tools`/`open_webui` left a service account and data
directory in place.

`finanzfluss_export.py` itself needed zero code changes in the move —
it was already entirely env-var-configured. There's no dedicated deploy
runbook for the current shape — it's deployed the same way every other
[`k3s-apps`](https://github.com/KandlerLi/k3s-apps) module is, through that repo's own normal Terraform
apply (CI on merge to `main`). See
[`current-state.md`](../current-state.md)'s own writeup of
the 2026-09-07 cutover for the real bugs found moving it (a CPU-request
QoS gotcha, a `hostPath` ownership fix, and a remote-directory
mismatch that silently 404'd every tick until caught).

The Nextcloud-side setup this runbook originally walked through (share
`Documents/Finanzen` with the dedicated `sankey-export` account,
confirm the recipient's own `target-path` via `occ share:list` rather
than assuming it matches the sharer's path) is still the real, correct
prerequisite for the current k3s CronJob too — that part of the design
didn't change, only how the export job itself runs.
