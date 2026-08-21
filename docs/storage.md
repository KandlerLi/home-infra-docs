# Storage

## Layout

```text
/dev/sdb2 (ext4, about 51 GiB)
└── /

/dev/sda1 (ext4, about 916 GiB)
└── /mnt/black-hdd
    ├── nextcloud/                 configured Nextcloud data directory
    ├── downloads/                 Deluge downloads (owner: deluge:deluge)
    ├── deluge-config/             Deluge state, separate from downloads
    └── github-runner/
        ├── github-runner.qcow2
        └── github-runner-cloud-init.iso
```

The tree shows managed paths, not a complete listing of private data.

## Current capacity

Snapshot date: 2026-08-18.

| Mount | Total | Used | Available | Used percentage |
| --- | ---: | ---: | ---: | ---: |
| `/` | 51 GiB | 26 GiB | 23 GiB | 54% |
| `/mnt/black-hdd` | 916 GiB | 138 GiB | 732 GiB | 16% |

## Mount management

The `storage` Ansible role mounts the HDD by filesystem UUID and configures:

```text
mount: /mnt/black-hdd
filesystem: ext4
owner: www-data
group: www-data
mode: 02770
```

The role creates or adjusts only the mount-point directory. Infrastructure
automation must never recursively change ownership or permissions below the
mount.

## Nextcloud data

The committed AIO configuration sets:

```text
NEXTCLOUD_DATADIR=/mnt/black-hdd/nextcloud
```

Nextcloud AIO also uses Docker volumes for configuration, databases, cache,
and application state. Both the filesystem data and AIO volumes are required
for a complete recovery.

Do not reorganise `/mnt/black-hdd`, change the Nextcloud data directory, or
delete AIO volumes as part of unrelated work.

## Deluge downloads mounted into Nextcloud

`deluge_downloads_dir` (`/mnt/black-hdd/downloads`) is optionally bind-mounted
into the `nextcloud-aio-nextcloud` app container via AIO's `NEXTCLOUD_MOUNT`
environment variable (set on the mastercontainer, not the app container
directly — see `ansible/roles/nextcloud_aio/tasks/main.yml`), then registered
as a Nextcloud "Local" external storage mount via `occ files_external:create`.
This is deliberately **not wired up automatically** when Deluge is enabled —
`nextcloud_aio_mount_dir` stays empty until explicitly set in
`group_vars/all/main.yml`, once Deluge itself has been validated.

**This mount is not covered by AIO's built-in backup.** Its contents (and
`deluge-config/`) need their own backup coverage if you want downloaded files
to survive a disaster recovery, separate from whatever backs up
`nextcloud/`.

## GitHub runner data

The runner role uses:

```text
/mnt/black-hdd/github-runner/github-runner.qcow2
/mnt/black-hdd/github-runner/github-runner-cloud-init.iso
```

Safety behaviour in the role:

- refuses to overwrite an orphaned disk
- refuses to redefine a mismatched libvirt domain
- does not recursively change permissions on the HDD
- requires the HDD to be mounted before creating VM storage

## Backup scope

The documentation does not currently prove a complete backup policy. A future
backup design must explicitly cover:

- Nextcloud user data
- AIO configuration and database state
- SOPS-encrypted infrastructure secrets and access to the GPG private key
- GitHub runner configuration only if preserving the runner is preferable to
  rebuilding it
- this documentation repository and the infrastructure repository
- Deluge downloads and config (`downloads/`, `deluge-config/`) if downloaded
  files should survive a disaster recovery — not covered by AIO's backup
  even though the folder is visible from within Nextcloud
