# Networking and Ingress

## Current topology

The homeserver is connected to the home LAN behind a Fritzbox and a dynamic
1&1 internet connection.

```text
Internet
   |
Fritzbox / NAT
   |
192.168.178.0/24
   |
homeserver: 192.168.178.100
```

Port forwarding rules are controlled outside Ansible and were not directly
enumerated during the snapshot.

## DNS

| Name | Provider | State on 2026-08-19 |
| --- | --- | --- |
| `jkandler.dedyn.io` | Dynamic DNS provider | A record present; no AAAA record |
| `jkandler.de` | AWS Route 53 | A record present; no AAAA record |
| `nextcloud.jkandler.de` | AWS Route 53 | CNAME to `jkandler.de`; resulting A record; no AAAA record |
| `ai.jkandler.de` | AWS Route 53/Terraform | CNAME to `jkandler.de`; resulting A record; no AAAA record |

Transient public IP values are intentionally not recorded because the
connection is dynamic.

## IPv6 safety rule

The server has IPv6 addresses, but public AAAA records previously caused
traffic to reach the Fritzbox rather than Nextcloud. Users then received the
Fritzbox self-signed certificate. Removing the AAAA record restored correct
external behaviour and substantially improved WebDAV performance.

Do not publish a new AAAA record until all of the following are deliberately
verified:

- the intended global IPv6 address is stable or updated automatically
- Fritzbox IPv6 firewall rules target the homeserver
- the reverse proxy or service listens on IPv6
- TLS certificates and hostname routing are correct over IPv6
- external tests confirm traffic terminates on the homeserver

## Observed listening TCP ports

| Bind | Port | Likely owner | Management status |
| --- | ---: | --- | --- |
| All IPv4 | 22 | OpenSSH | Debian host; not explicitly configured by current roles |
| All IPv6 | 22 | OpenSSH | Debian host; not explicitly configured by current roles |
| All IPv4 and IPv6 | 80 | Traefik shared ingress | Ansible `shared_ingress` role |
| All IPv4 and IPv6 | 443 | Traefik shared ingress | Ansible `shared_ingress` role |
| All IPv4 | 8080 | Nextcloud AIO management interface | Ansible/AIO |
| All IPv4 | 8443 | Nextcloud AIO master | Ansible/AIO |
| All IPv4 and IPv6 | 3478 | Nextcloud Talk TURN | AIO |
| All addresses | 5201 | Likely iperf3 | Unknown; verify owner |
| Loopback | 5900 | libvirt VM VNC | Ansible/libvirt |
| Loopback | 43117 | Unknown | Verify owner |
| Loopback IPv4 | 8090 | `home-agent` HTTP API | Ansible `home_agent` role |
| Loopback IPv4 | 8091 | Open WebUI HTTP frontend | Privately deployed; public route disabled |
| Loopback IPv4 | 8082 | Traefik health endpoint | Ansible `shared_ingress` role |
| Loopback IPv4 | 11000 | Nextcloud AIO Apache upstream | AIO reverse-proxy mode |
| VM bridges | 53 | libvirt dnsmasq | libvirt |

UDP listeners were not fully inventoried. Nextcloud Talk publishes UDP 3478.
The shared ingress currently terminates HTTPS over TCP; UDP 443 is not part of
the documented Traefik configuration.

## Reverse proxy status

Pinned Traefik `v3.7.1` is deployed with host networking and owns TCP ports 80
and 443. Its configuration comes from the file provider; it has no Docker
socket. It routes `nextcloud.jkandler.de` to the loopback-only AIO Apache
upstream at `127.0.0.1:11000` and obtains the hostname certificate through
ACME. It routes `ai.jkandler.de` to `127.0.0.1:8090` only after Basic Auth,
rate limiting, a 16 KiB request-size limit, and security-header middleware.

The completed agent publication is documented in the
[public home-agent ingress view](architecture/proposed/deployment.md). The
application container remains loopback-only even though the authenticated
route is public.

The prepared Open WebUI publication retains exact `/healthz` and `/v1/chat`
routes to port 8090 and sends the hostname catch-all to loopback port 8091.
That routing change is guarded and has not been applied. The private Open WebUI
container reaches `home-agent` through the `home-agent-frontend` bridge. The
bridge has IP masquerading disabled: loopback publication works, while a live
outbound HTTPS test from Open WebUI was blocked.
