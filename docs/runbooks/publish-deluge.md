# Runbook: Deploy and Publish Deluge — retired

**Retired 2026-08-29/30, not applicable to the current architecture.**
This runbook described [`home-infra`](https://github.com/KandlerLi/home-infra)'s own Docker-based Deluge
deployment (a container with a web UI on `127.0.0.1:8112`, published
publicly through the retired `shared_ingress` role's own
`publish-deluge.yml`/`rollback-deluge.yml` playbooks). `deluge.yml`
itself still exists as a file, but **its content changed completely**
-- it no longer deploys a container or web UI at all, only the host
prerequisites (`deluge` service account, `/mnt/black-hdd/downloads`
and `/mnt/black-hdd/deluge-config`, NFS-exported to k3s) that the
current k3s-native copy still depends on. Running it will not do what
this runbook describes, even though the command itself doesn't error.
`publish-deluge.yml`/`rollback-deluge.yml` are both fully deleted.

`torrent.jkandler.de` is served by [`k3s-apps`](https://github.com/KandlerLi/k3s-apps)' own
`modules/deluge` now, reached through k3s-native Traefik
(`k3s_ingress_forward`'s DNAT relay) with Authelia forward-auth gating
the route instead of Traefik Basic Auth (since 2026-09-08). There's no
dedicated deploy/publish runbook for the current shape -- it's
deployed the same way every other [`k3s-apps`](https://github.com/KandlerLi/k3s-apps) module is, through
that repo's own normal Terraform apply. See
[`history.md`](../history.md)'s own "k3s
learning cluster" section for the full cutover history.

Two pieces of this runbook's original content are still genuinely
relevant, unrelated to which deployment mechanism serves the route:

- **Inbound peer connectivity** (the old section 5): BitTorrent peer
  connections can't be proxied through Traefik -- they're a raw TCP/UDP
  protocol. There is deliberately **no** FRITZ!Box forward for Deluge's
  peer port (decided 2026-10-12): Deluge works with outbound peer
  connections only. Should inbound peers ever be wanted, the forward goes
  into `KandlerLi/fritzbox` as a `fritzbox_port_forward`, not into the
  FRITZ!Box UI.
- **This data still isn't covered by AIO's built-in Nextcloud
  backup** -- see `docs/storage.md` and [`home-infra`](https://github.com/KandlerLi/home-infra)'s own
  `aux_backup` role (added 2026-09-16), which now covers
  `deluge-config` specifically (not the bulk `downloads` directory,
  deliberately excluded as large and re-attainable by re-downloading).
