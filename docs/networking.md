# Networking and Ingress

Current hostnames, auth per route, DNS records and forwarded ports are
listed in [`current-state.md`](current-state.md); the physical and network
placement is the [deployment view](architecture/current/deployment.md). This
page keeps the rules behind them.

## Topology

```text
Internet
   |
FRITZ!Box (NAT, port forwards, WireGuard VPN)         LAN 192.168.178.0/24
   |
homeserver 192.168.178.100
   |  iptables DNAT (k3s_ingress_forward): 80, 443, 53, 25, 465, 993
   v
k3s-node-1 192.168.101.10   (libvirt NAT network 192.168.101.0/24,
k3s-node-2 192.168.101.11    reachable only from the homeserver)
```

- The FRITZ!Box's IPv4 port forwards (TCP 25, 80, 443, 465, 993, all to
  the homeserver) are Terraform-managed in `KandlerLi/fritzbox` through
  `KandlerLi/terraform-provider-fritzbox`, applied locally only (no runner
  can reach the FRITZ!Box); `fritzbox` is private. Port 53 is
  not forwarded: there is no public DNS resolver, and
  `k3s_ingress_forward`'s port-53 DNAT only serves LAN clients.
- `k3s_ingress_forward` is a packet-level relay below TLS, so ACME and
  client certificates see the real handshake. Its rules match only traffic
  addressed to the homeserver, so the host's own outbound connections are
  not hairpinned into the cluster (see `history.md`, "k3s learning
  cluster").
- Traefik in k3s terminates TLS for every HTTP hostname. It uses the file
  provider only and has no Kubernetes API token.
- k3s Pods reach homeserver services on `192.168.101.1` (Nextcloud AIO
  `:11000`, Prometheus `:9090`, `home-tools` `:8095`); those binds are
  additive to the loopback ones.
- Raw-protocol ports (SMTP/IMAP to Stalwart, DNS to Blocky, Deluge's
  BitTorrent port) are the only traffic that bypasses Traefik.

## LAN DNS

LAN clients use the homeserver as DNS server. IPv4 queries on
`192.168.178.100:53` are DNATed to Blocky in k3s; IPv6 queries reach the
`blocky-ipv6-relay` service on `[::]:53`, which forwards them to Blocky over
IPv4. The homeserver's IPv6 address is stable (EUI-64) since 2026-09-24.

## IPv6 safety rule

A public AAAA record once sent traffic to the FRITZ!Box instead of the
homeserver, and users got the FRITZ!Box's self-signed certificate. Do not
publish an AAAA record for a homeserver-hosted name until all of these are
verified:

- the global IPv6 address is stable or updated automatically
- FRITZ!Box IPv6 firewall rules target the homeserver
- the ingress listens on IPv6
- TLS and hostname routing are correct over IPv6
- external tests confirm traffic terminates on the homeserver

`www.jkandler.de` has an AAAA record because it is served by CloudFront,
not the homeserver.
