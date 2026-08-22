# Runbook: Enable the Finanzfluss Sankey exporter

Status: **Implemented, not yet deployed.** See [ADR 0015](../adr/0015-sankey-export-automation.md)
for the design. This replaces the hand-maintained
`/home/julian/sankey` script and its manually-created
`sankey-export.service` unit with an Ansible-managed role, a dedicated
Nextcloud account, and a 1-minute polling timer.

## 1. Prerequisite: share the folder (manual, not Ansible)

Sign in to Nextcloud as `admin`, open the sharing settings for
`Documents/Finanzen`, and share it with a new internal share to the
`sankey-export` account (created in step 3 below — share it after the
first bootstrap run, or pre-create the share once the account exists),
granting **Can edit** (needed for both reading the workbook and writing
the PNG). This is a manual step, same reasoning as every other Nextcloud
integration in this repo (ADR 0008): automating a share grant would need
`admin`'s own credential.

## 2. Run tests and syntax checks

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/python -m unittest discover -s tests -v
.venv/bin/ansible-playbook ansible/playbooks/site.yml --syntax-check
```

## 3. Bootstrap the dedicated Nextcloud account

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/ansible-playbook ansible/playbooks/sankey-export.yml \
  --ask-become-pass \
  --extra-vars sankey_export_bootstrap_confirmation=BOOTSTRAP_SANKEY_EXPORT
```

This creates the `sankey-export` Nextcloud account, generates its app
password via `occ user:auth-tokens:add`, writes it to
`/etc/sankey-export/app-password` (mode `0400`, host-only — this is
intentionally not in SOPS; see ADR 0015 for why a Nextcloud app password
can't be pre-chosen the way a login password can), creates the dedicated
Linux service account, installs the venv and pinned dependencies, installs
Chromium for Playwright, and starts the `sankey-export.timer`.

If you hadn't shared the folder yet (step 1), do it now that the account
exists.

## 4. Enable it for routine deploys

Add `sankey_export_enabled: true` to
`ansible/inventory/group_vars/all/main.yml` so future `site.yml` runs
(code updates, not just the one-time account bootstrap) keep it deployed:

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/ansible-playbook ansible/playbooks/site.yml --ask-become-pass
```

## 5. Verify live, end to end

1. `systemctl list-timers sankey-export.timer` — confirm it's active and
   due within a minute.
2. `journalctl -u sankey-export.service -n 20` after a tick with an
   unchanged workbook — confirm it printed "Arbeitsmappe unverändert,
   nichts zu tun." and exited quickly, with no Chromium process spawned
   (`ps aux | grep chromium` should show nothing right after).
3. Edit `Finanzfluss_Nextcloud.xlsx` in Nextcloud (change any amount).
   Within about a minute, confirm `journalctl -u sankey-export.service`
   shows a real run (workbook downloaded, Highcharts export intercepted,
   PNG uploaded), and that `finanzfluss-diagramm.png` in
   `Documents/Finanzen` has a new timestamp/content.
4. Confirm the dedicated account genuinely can't see anything outside
   `Documents/Finanzen` — e.g. a WebDAV `PROPFIND` against a different
   path with its credential returns 403/404, not a listing.
5. If step 3 fails at the Playwright stage, check
   `journalctl -u sankey-export.service` for the specific warning
   (`download_png_via_menu`'s return value is printed as a warning, not
   swallowed) before assuming it's a permissions problem — a failed
   export leaves the cached ETag stale on purpose, so simply waiting
   another minute retries it automatically once the underlying issue is
   fixed.

## 6. Roll back

```bash
systemctl disable --now sankey-export.timer
```

removes the automation without touching the Nextcloud account or the old
manually-created script (if it's still present at `/home/julian/sankey`,
which was left alone and not deleted by this rollout). To remove the
dedicated Nextcloud account entirely, revoke its token via `occ
user:auth-tokens:delete` and delete the account via `occ user:delete`,
then unshare `Documents/Finanzen` from it.
