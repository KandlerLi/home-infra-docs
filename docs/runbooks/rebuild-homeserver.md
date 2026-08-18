# Runbook: Rebuild the Homeserver

Status: baseline outline; disaster-recovery details still need an exercised
backup and restore procedure.

## Preconditions

- A verified backup of Nextcloud files and application state
- A clone of `home-infra` and this documentation repository
- Access to the SOPS GPG private key
- The HDD filesystem UUID from the infrastructure defaults
- Console access to the physical machine
- Current Debian installation media

## Safety constraints

- Do not format `/dev/sda1`.
- Do not recursively change ownership below `/mnt/black-hdd`.
- Confirm device identity and filesystem UUID before mounting or installing.
- Do not allow an empty mount point to hide a mount failure before Docker
  services start.

## Outline

1. Install the supported Debian release on the system SSD only.
2. Configure the administrative user and SSH access.
3. Clone repositories below `/home/julian/projects/`.
4. Restore the SOPS/GPG decryption capability.
5. Create the Python virtual environment and install pinned dependencies.
6. Confirm `/dev/sda1` identity without mounting it over unexpected content.
7. Run Ansible connectivity and syntax checks.
8. Apply base and storage configuration first.
9. Confirm `/mnt/black-hdd` is mounted and contains the expected data.
10. Apply Docker and Nextcloud AIO configuration.
11. Validate Nextcloud locally and externally before proceeding.
12. Apply libvirt and GitHub runner configuration.
13. Validate DNS, certificates, container health, failed units, and backups.

## Validation

```bash
findmnt /mnt/black-hdd
df -hT
systemctl --failed
docker ps
```

Do not declare the rebuild complete until a representative Nextcloud file can
be read and a restore path has been confirmed.
