# ADR 0024: WireGuard on the homeserver for admin access from outside

- Status: Superseded by [ADR 0025](0025-fritzbox-wireguard-admin-vpn.md)
- Date: 2026-09-26
- Deployment status: built and applied 2026-09-26 (`home-infra#69`), torn down by `home-infra#70`

## Context

The public web services (`nextcloud`, `ai`, `grafana`, `home`,
`torrent`, `mail`, `docs`) are already reachable from anywhere through
FRITZ!Box → DNAT → k3s Traefik → Authelia (MFA). Everything
administrative is LAN-only: SSH to the homeserver, Nextcloud AIO's
mastercontainer on `:8080`, the k3s API, Prometheus (loopback, via SSH
tunnel). There was no way to reach those from outside at all.

The line has a public IPv4 (DynDNS keeps `jkandler.de`'s A record
current), so an inbound UDP port works and NAT traversal isn't needed.

Options considered:

- **Expose SSH publicly** (key-only, fail2ban): a new internet-facing
  login surface next to Authelia, and it still wouldn't cover `:8080`
  or the k3s API without more tunnels. Rejected.
- **FRITZ!Box built-in WireGuard**: quick, and keeps working while the
  homeserver is frozen. But its config lives only in the router UI, not
  in code, which the workspace's IaC-only rule doesn't allow, and a
  frozen homeserver can't be fixed from remote anyway.
- **Tailscale / Headscale**: convenient, but their main advantage (NAT
  traversal) isn't needed here. Tailscale adds a third-party control
  plane, and Headscale adds one more service to run.
- **WireGuard on the homeserver, managed by Ansible**: one UDP port, the
  config and peer list in Git, keys generated on each device.

## Decision

Run a **WireGuard server on the homeserver** as the `wireguard` role in
[`home-infra`](https://github.com/KandlerLi/home-infra):

- Endpoint `vpn.jkandler.de:51820/udp`: a CNAME in [`dyndns`](https://github.com/KandlerLi/dyndns), IPv4
  only (public AAAA records stay off-limits, see `decisions.md`). The
  FRITZ!Box forwards UDP 51820 to the homeserver, a manual router
  setting like the existing 80/443/25 forwards.
- VPN subnet `10.13.13.0/24`, masqueraded as the homeserver so the
  FRITZ!Box and the k3s VM need no return routes. VPN clients reach the
  LAN (`192.168.178.0/24`) and the k3s network (`192.168.101.0/24`).
- The server private key is generated on the host and never leaves it.
  No new Secrets Manager entry is needed; a rebuilt host means updating
  each client's server public key.
- Clients use Blocky (`192.168.178.100`) as DNS and a split tunnel by
  default.

Public web services stay as they are. The VPN is for admin access only
and doesn't replace Authelia.

## Consequences

- SSH, AIO `:8080`, `kubectl` and Prometheus become reachable from
  outside, but only for devices with a peer entry.
- One new public UDP port. WireGuard doesn't answer unauthenticated
  packets, so a scan can't find it.
- The VPN depends on the homeserver: when it's down or frozen, the VPN
  is down too.
- Adding a device means a `home-infra` inventory change and an apply.
