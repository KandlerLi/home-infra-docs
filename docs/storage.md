# Storage

## Layout

```text
/dev/sdb2 (ext4, about 51 GiB)
└── /

/dev/sda1 (ext4, about 916 GiB)
└── /mnt/black-hdd
    ├── nextcloud/                 configured Nextcloud data directory
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
