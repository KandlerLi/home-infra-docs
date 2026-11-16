# Runbooks

Step-by-step procedures. Everyday commands are in
[`../operations.md`](../operations.md). Commands that change production
need Julian's review; prefer read-only checks first.

## Current

| Runbook | Use when |
| --- | --- |
| [Move to the 256 GB SSD](migrate-to-ssd.md) | Clone the system disk, grow root, move the k3s VM disks off the HDD (#40) |
| [Rebuild the homeserver](rebuild-homeserver.md) | Disaster recovery (outline; restore steps in Verify and restore backups) |
| [Operate Nextcloud AIO](nextcloud-aio.md) | Borg backup setup, AIO quirks, planned hardware shutdown |
| [Stalwart mail server](stalwart-mail-server.md) | Any Stalwart setting, app passwords, SSO/admin access |
| [Rotate secrets](rotate-secrets.md) | Rotating or adding a Secrets Manager value |
| [Verify and restore backups](verify-backups.md) | Restore drill (`verify-backups.sh`), restoring Nextcloud, Authelia, Paperless |
| [Manage repository infrastructure](manage-repo-infra.md) | Adding a repo to `repo-infra`, importing existing settings |
| [Deploy the personal website](deploy-personal-website.md) | `www.jkandler.de` (S3 + CloudFront) |
| [FRITZ!Box WireGuard VPN](fritzbox-wireguard-vpn.md) | Adding, checking or removing a VPN device (ADR 0025) |
| [Retire a k3s app](retire-k3s-app.md) | Removing an Argo CD-deployed app everywhere, optionally with its data |
| [Update this documentation](update-documentation.md) | After any change that alters what these docs say |

## Known issues

| Runbook | Status |
| --- | --- |
| [Homeserver silent freeze](homeserver-silent-freeze.md) | Root cause unknown; watchdog did not fire on 2026-09-24 |
| [e1000e NIC watchdog hang](homeserver-e1000e-nic-watchdog-hang.md) | Mitigated, root cause unconfirmed |

## Historical

Kept because code comments or ADRs point at them; the setups they
describe no longer exist. Each file says where the thing lives now.

- [GitHub runner VM](github-runner.md): runners are in k3s since
  2026-08-31, ARC scale sets in [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap) since 2026-11-01
- [Enable the Sankey exporter](enable-sankey-export.md): moved to
  [`k3s-apps`](https://github.com/KandlerLi/k3s-apps) (2026-09-07). The home-agent, Open WebUI and
  Nextcloud-tools runbooks were deleted when those apps were retired
  (2026-11-16, [decisions](../decisions.md#open-webui-and-home_agent-retired)).
- [Migrate AWS account bootstrap ownership](migrate-aws-account-bootstrap.md):
  the first identity handoff, superseded by `repo-infra` (ADR 0010)
