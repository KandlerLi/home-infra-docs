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

/dev/sdc1 (ext4)
└── /mnt/red-hdd               Nextcloud AIO's Borg backup target
```

The tree shows managed paths, not a complete listing of private data.

## Current capacity

Snapshot date: 2026-08-18 (`/`, `/mnt/black-hdd`); `/mnt/red-hdd` added
2026-09-15, not yet snapshotted here.

| Mount | Total | Used | Available | Used percentage |
| --- | ---: | ---: | ---: | ---: |
| `/` | 51 GiB | 26 GiB | 23 GiB | 54% |
| `/mnt/black-hdd` | 916 GiB | 138 GiB | 732 GiB | 16% |
| `/mnt/red-hdd` | — | — | — | — |

## Mount management

The `storage` Ansible role mounts both HDDs by filesystem UUID and configures:

```text
mount: /mnt/black-hdd
filesystem: ext4
owner: www-data
group: www-data
mode: 02770

mount: /mnt/red-hdd
filesystem: ext4
owner: root
group: root
mode: 0755
```

`/mnt/red-hdd` is `root:root` rather than `black_hdd`'s `www-data:www-data`
-- it's written to only by Nextcloud AIO's own backup container (root-
equivalent inside Docker), not the Nextcloud app container directly. `sdc1`
contained a leftover Windows OEM recovery partition
(`$RECYCLE.BIN`/`Recovery`/`System Volume Information`, no `Windows`/`Users`/
`Program Files`) before being reformatted -- confirmed via a read-only mount
before wiping, not assumed.

The role creates or adjusts only each mount point directory itself.
Infrastructure automation must never recursively change ownership or
permissions below either mount.

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

`/mnt/red-hdd` exists now (2026-09-15) as a target, but AIO's own Borg
backup still needs pointing at it through the AIO interface -- see
`docs/runbooks/nextcloud-aio.md`'s "Enable the Borg backup" section. Once
that's live and daily backups are enabled, it covers Nextcloud user data and
AIO's own configuration/database state (see AIO's own docs for the exact
scope) -- everything else below is still **not** covered by anything:

- SOPS-encrypted infrastructure secrets and access to the GPG private key
- GitHub runner configuration only if preserving the runner is preferable to
  rebuilding it
- this documentation repository and the infrastructure repository
- Deluge downloads and config (`downloads/`, `deluge-config/`) if downloaded
  files should survive a disaster recovery — not covered by AIO's backup
  even though the folder is visible from within Nextcloud

The documentation does not currently prove a complete backup policy even
once AIO's own backup is enabled.
