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
| [Verify and restore backups](verify-backups.md) | Restore drill (`verify-backups.sh`), restoring Nextcloud, Authelia, Paperless, Deluge, Open WebUI |
| [Manage repository infrastructure](manage-repo-infra.md) | Adding a repo to `repo-infra`, importing existing settings |
| [Deploy the personal website](deploy-personal-website.md) | `www.jkandler.de` (S3 + CloudFront) |
| [FRITZ!Box WireGuard VPN](fritzbox-wireguard-vpn.md) | Adding, checking or removing a VPN device (ADR 0025) |
| [Open WebUI voice transcription](enable-openai-voice-transcription.md) | The Admin Panel audio settings (deploy part is historical) |
| [Harden Open WebUI login](harden-open-webui-login.md) | Native login form reappears despite OIDC-only settings |
| [Fix Open WebUI's stale backend URL](fix-open-webui-stale-backend-url.md) | No model selectable in Open WebUI |
| [Update this documentation](update-documentation.md) | After any change that alters what these docs say |

## Known issues

| Runbook | Status |
| --- | --- |
| [Homeserver silent freeze](homeserver-silent-freeze.md) | Root cause unknown; watchdog did not fire on 2026-09-24 |
| [e1000e NIC watchdog hang](homeserver-e1000e-nic-watchdog-hang.md) | Mitigated, root cause unconfirmed |

## Historical

Kept because code comments or ADRs point at them; the setups they
describe no longer exist. Each file says where the thing lives now.

- [GitHub runner VM](github-runner.md): runners are k3s Deployments in
  [`k3s-bootstrap`](https://github.com/KandlerLi/k3s-bootstrap) since 2026-08-31
- [Publish the home-agent](publish-home-agent.md),
  [Deploy and publish Open WebUI](publish-open-webui.md),
  [Deploy read-only Nextcloud tools](deploy-nextcloud-tools.md),
  [Enable Nextcloud write tools](enable-nextcloud-writes.md),
  [Enable the Shopping List tools](enable-nextcloud-shopping-list.md),
  [Deploy and publish Deluge](publish-deluge.md),
  [Enable the Sankey exporter](enable-sankey-export.md): all moved to
  [`k3s-apps`](https://github.com/KandlerLi/k3s-apps) (2026-08-29 to 2026-09-07)
- [Migrate AWS account bootstrap ownership](migrate-aws-account-bootstrap.md):
  the first identity handoff, superseded by `repo-infra` (ADR 0010)
