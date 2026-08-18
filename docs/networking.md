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

| Name | Provider | State on 2026-08-18 |
| --- | --- | --- |
| `jkandler.dedyn.io` | Dynamic DNS provider | A record present; no AAAA record |
| `jkandler.de` | AWS Route 53 | A record present; no AAAA record |
| `ai.jkandler.de` | AWS Route 53 | No A or AAAA record |

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
| All IPv4 | 80 | Nextcloud AIO master | Ansible/AIO |
| All IPv4 and IPv6 | 443 | Nextcloud AIO Apache | AIO |
| All IPv4 | 8080 | Nextcloud AIO management interface | Ansible/AIO |
| All IPv4 | 8443 | Nextcloud AIO master | Ansible/AIO |
| All IPv4 and IPv6 | 3478 | Nextcloud Talk TURN | AIO |
| All addresses | 5201 | Likely iperf3 | Unknown; verify owner |
| Loopback | 5900 | libvirt VM VNC | Ansible/libvirt |
| Loopback | 43117 | Unknown | Verify owner |
| VM bridges | 53 | libvirt dnsmasq | libvirt |

UDP listeners were not fully inventoried. Nextcloud Talk also publishes UDP
3478, and Nextcloud Apache publishes UDP 443 for HTTP/3.

## Reverse proxy status

There is no shared reverse proxy in the committed infrastructure repository.
Nextcloud AIO Apache directly owns port 443. Hosting another HTTPS service on
the same public address and standard port requires a shared reverse proxy and
a controlled Nextcloud AIO migration.

The proposed routing is documented separately in the
[home-agent deployment view](architecture/proposed/deployment.md).
