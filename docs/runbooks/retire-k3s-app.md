# Runbook: Retire a k3s app

Removes an Argo CD-deployed app from `k3s-apps` everywhere it reaches, and
optionally its data. Worked out on qBittorrent, 2026-11-15
([decisions](../decisions.md#qbittorrent-retired)), extended on Open WebUI, 2026-11-16
([decisions](../decisions.md#open-webui-and-home_agent-retired)). The order matters:
Argo CD prunes only while its Application exists and its Role still names
the objects, and it never deletes PVCs.

## 1. Find every touch point

`grep -rni` all repos for the app's name, its hostname, its PVC/PV names,
its host account and uid. The usual places:

| Repo | Where |
| --- | --- |
| `k3s-apps` | `argocd/<app>/`, its route in `argocd/ingress/values.yaml`, Authelia's access rule (`argocd/authelia/externalsecret.yaml`), the landing-page tile, Grafana dashboards, the CI checksum step, `docs/README.md` |
| `k3s-bootstrap` | `argocd_applications`, the names in `argocd_objects`, NFS/hostPath PVs in `storage.tf`, `docs/README.md` |
| `dyndns` | `subdomains` |
| `home-infra` | host-prerequisite role and playbook, `nfs_server_exports`, backup roles and markers, `verify-backups-remote.sh`, monitoring probes and dashboard, Nextcloud mounts, tests |
| `terraform-state`, `secrets-manager` | only for an app with ExternalSecrets: its groups in `k3s-eso-<namespace>`, its Secrets Manager values |
| `fritzbox` | only if it has a port forward |
| this repo | current-state docs, runbooks, ADRs; the `check-security-invariants` skill |

Ask Julian what happens to the data: keep it unmanaged, or delete it.

## 2. Roll out in this order

1. **`k3s-apps`: everything but the PVCs.** Keep `storage.yaml` in the
   kustomization; automated sync refuses to prune an app down to nothing
   (`allowEmpty`). Route, Authelia rule, tile, panels and CI go in the
   same PR. After merge, `kubectl -n <ns> get all,cm,networkpolicy,pvc`
   shows only the PVCs. An app without PVCs keeps another harmless object
   the Role already names (`home-agent` kept its source ConfigMap), deleted
   by hand in step 3 along with the PVCs.
2. **`k3s-bootstrap`: the Application and the Role's names**, applied
   locally with `scripts/roll-out.sh apply`.
3. **Delete the PVCs by hand**, once: `kubectl -n <ns> delete pvc <names>`.
   Earlier, selfHeal would recreate them.
4. **`k3s-bootstrap`: the PVs**, in their own PR. With a PVC still bound,
   the destroy hangs on `pv-protection`. PVs are `Retain`, so the files stay.
5. **`k3s-apps`: delete `argocd/<app>/`**, no rollout.
6. **`dyndns`**: drop the subdomain; CI applies on merge.
7. **`home-infra`**: host prerequisites, exports, backups, probes; rolled
   out with `scripts/roll-out.sh apply`.
8. **Secrets**, for an app with ExternalSecrets: `terraform-state` drops
   its groups from `k3s-eso-<namespace>` and the operator policy. Delete
   the containers by hand as root (`aws secretsmanager delete-secret
   --region eu-central-1 …`; `prevent_destroy`, and CI can't delete),
   then merge the `secrets-manager` PR, whose plan is then empty.

## 3. Clean up by hand

Ansible and Terraform remove what they manage, not what a role once
created on the host:

- **Nextcloud mount**: delete the external storage
  (`occ files_external:delete`). AIO keeps a stored `NEXTCLOUD_MOUNT`
  until the variable overwrites it; `nextcloud_aio` sets it to `""` for
  that reason. Then AIO "Stop containers", "Start containers".
- **Data**: delete a directory only after `docker inspect` shows no
  running container bind-mounting it. A missing bind source keeps the
  container from starting.
- **Host account, old backup tarballs**: `userdel`, `groupdel`, `rm`.
  Look for accounts older roles left too (`getent passwd | grep <app>`).
- **systemd units and timers** a removed role installed: `disable --now`,
  remove the unit files and drop-ins, `daemon-reload`.

## 4. Verify live

Nothing left in: `kubectl get all,cm,networkpolicy,pvc -A` and `get pv`,
Argo CD's Applications, the ingress and Authelia config, the landing page,
Grafana's dashboards, Prometheus targets and rules, `exportfs -v`, public
DNS. Nextcloud's `status.php` answers. Then this repo's docs
([update-documentation](update-documentation.md)).
