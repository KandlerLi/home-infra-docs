# Home Infrastructure — Deployment Diagram

- **C4 type:** Deployment
- **Scope:** Production, physical and network placement
- **Status:** Current as of 2026-10-11

```mermaid
flowchart TB
    internet["Internet<br/>[External]"]

    subgraph lan["Home LAN 192.168.178.0/24"]
        fritz["FRITZ!Box<br/>[Deployment Node: router]<br/>NAT, port forwards, WireGuard VPN"]

        subgraph homeserver["homeserver 192.168.178.100<br/>[Deployment Node: Debian 13, i5-2400, 32 GiB]"]
            docker["Docker<br/>[Deployment Node]<br/>Nextcloud AIO, Prometheus, exporters"]
            systemd["systemd<br/>[Deployment Node]<br/>home-tools, ntfy-relay, blocky-ipv6-relay, aux-backup"]
            dnat["iptables DNAT<br/>[Infrastructure Node: k3s_ingress_forward]<br/>80, 443, 53, 25, 465, 993 to 192.168.101.10"]
            nfs["NFS server<br/>[Infrastructure Node]<br/>Single-client exports, root_squash"]
            disks[("black-hdd 916 GB live data<br/>red-hdd 1.8 TB backups<br/>[Infrastructure Node]")]

            subgraph libvirt["libvirt NAT network 192.168.101.0/24, reachable only from the homeserver"]
                node1["k3s-node-1 192.168.101.10<br/>[Deployment Node: VM, 3 vCPU, 10 GiB]<br/>Control plane and all app Pods"]
                node2["k3s-node-2 192.168.101.11<br/>[Deployment Node: VM, 8 GiB, 100 GB]<br/>Tainted: GitHub Actions runners only"]
            end
        end
    end

    subgraph aws["AWS [Deployment Node]"]
        r53["Route 53"]
        sm["Secrets Manager"]
        sesn["SES eu-central-1"]
        cf["S3 + CloudFront: www.jkandler.de"]
    end

    internet -->|"Public IPv4"| fritz
    fritz -->|"Port forwards"| dnat
    dnat -->|"Packet-level forward"| node1
    node1 -->|"NFS mounts"| nfs
    nfs --- disks
    docker --- disks
    node1 -->|"192.168.101.1:11000, :9090, :8095"| docker
    node1 -->|"192.168.101.1:8095"| systemd
    node2 -->|"CI jobs: outbound internet only"| internet
    node1 -->|"Secrets, SES relay, DNS-01"| aws
    internet -->|"DNS, website"| aws

    classDef node fill:#f5e6c8,color:#000,stroke:#8a6d3b
    classDef external fill:#666,color:#fff,stroke:#333
    class fritz,docker,systemd,dnat,nfs,disks,node1,node2 node
    class internet external
```

## Notes

- The k3s VMs' disks are qcow2 files on the system SSD (`/var/lib/libvirt/k3s/`). VM sizes and
  networking are set in [`home-infra`](https://github.com/KandlerLi/home-infra) `ansible/playbooks/k3s.yml`.
- Nothing on the LAN can route to `192.168.101.0/24`; inbound traffic
  enters the cluster only through the DNAT relay.
- The homeserver itself answers LAN DNS on port 53 by forwarding it to
  Blocky (IPv4 via DNAT, IPv6 via `blocky-ipv6-relay`).
- Admin access from outside goes through the FRITZ!Box WireGuard VPN
  (ADR 0025), not through any forwarded port.

## Legend

- Beige: deployment or infrastructure node
- Grey: external
- Arrow label: protocol, port or intent
