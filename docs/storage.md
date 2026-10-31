# Storage

Which disk holds what, and the rules for touching it. Capacity and backup
status: [`current-state.md`](current-state.md).

## Layout (checked 2026-10-31)

```text
/                      system SSD, ext4, 231 GB
/mnt/black-hdd         ext4, 916 GB, live data      www-data:www-data 2770
├── nextcloud/         Nextcloud AIO data directory (NEXTCLOUD_DATADIR)
├── downloads/         qBittorrent downloads; NFS to k3s; mounted into Nextcloud
├── deluge-config/     qBittorrent state (qBittorrent/), old Deluge files; NFS to k3s
├── paperless-media/   Paperless documents; NFS to k3s
├── k3s/               previous k3s VM disks, rollback until removed (tracker #47)
└── blocky/            leftover data of the stopped Docker Blocky
/mnt/red-hdd           ext4, 1.8 TB, backups only   root:root 0755
├── borg/              Nextcloud AIO Borg repository
├── aux-backups/       aux-backup tarballs (Deluge config, Open WebUI data)
├── authelia-backup/   Authelia CronJob output; NFS to k3s
└── paperless-export/  Paperless document_exporter output; NFS to k3s
/var/lib/open-webui    Open WebUI data (SQLite); NFS to k3s
/var/lib/libvirt/k3s   qcow2 disks of k3s-node-1 and k3s-node-2 (system SSD)
```

The tree shows managed paths only. Live data on `black-hdd`, backups on
`red-hdd`: keep it that way when placing anything new.

## Rules

- The `storage` role mounts both disks by UUID and sets only the mount
  point's own owner and mode. Automation never changes ownership or
  permissions recursively below either mount.
- Do not reorganise `/mnt/black-hdd`, move the Nextcloud data directory or
  delete Nextcloud AIO Docker volumes during unrelated work. A full
  Nextcloud restore needs both the data directory and the AIO volumes
  (database, config); the Borg backup covers both.
- Never delete an unidentified directory or Docker volume as routine
  cleanup; prove what owns it first.
- NFS exports go to one client IP each, with `root_squash` (the
  `nfs_server` role asserts this). A k3s workload writing to an export
  runs as the uid/gid that owns the directory on the host (qBittorrent
  993/986 via the `deluge` account, Paperless 983/978).
- k3s hostPath PV directories on node-1 whose workload runs as non-root
  are created by `home-infra`'s `k3s_node_host_path_dirs` with that
  workload's uid/gid (sankey-export 10010, both Postgres 70), since a
  Pod under PodSecurity `restricted` can't chown its own volume.
- `downloads/` is visible in Nextcloud as an external storage, but it is
  not part of the Nextcloud Borg backup and has no backup of its own
  (deliberately: large, re-downloadable).
- The k3s VM disks on the system SSD are not backed up; anything stateful
  in k3s that matters gets a backup to `red-hdd` (see `current-state.md`).
