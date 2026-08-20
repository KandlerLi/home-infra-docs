# System Inventory

## Physical host

| Property | Value |
| --- | --- |
| Inventory name | `homeserver` |
| Hardware | Fujitsu ESPRIMO P900 |
| CPU | Intel Core i5-2400, 4 cores / 4 threads, 3.10 GHz |
| Memory | 16 GiB DDR3 online; approximately 15.5 GiB usable |
| GPU | None dedicated |
| Operating system | Debian 13.6 (`trixie`) |
| Architecture | x86-64 |
| LAN address | `192.168.178.100/24` |
| Ansible user | `julian` |
| Ansible Python | `/usr/bin/python3` |

The host is suitable for orchestration and lightweight services. It is not a
sensible primary inference host for modern large language models.

The memory was upgraded on 2026-08-19. Post-boot validation reported 16 GiB
online, no offline memory blocks, and no memory, EDAC, MCE, or hardware-error
indicators in the current boot's accessible kernel journal. This confirms
normal recognition and initial operation, not an exhaustive offline memory
test.

## Storage devices

| Device | Filesystem | Mount | Purpose |
| --- | --- | --- | --- |
| `/dev/sdb2` | ext4 | `/` | Debian system filesystem |
| `/dev/sdb1` | vfat | `/boot/efi` | EFI system partition |
| `/dev/sda1` | ext4 | `/mnt/black-hdd` | Nextcloud data and VM storage |

See [Storage](storage.md) for invariants and data locations.

## Network segments

| Segment | Interface/bridge | Purpose | Status at snapshot |
| --- | --- | --- | --- |
| `192.168.178.0/24` | `enp0s25` | Home LAN | Active |
| `172.17.0.0/16` | `docker0` | Default Docker bridge | Bridge down; no attached running containers observed |
| `172.18.0.0/16` | `br-fe7e9491a57b` | Nextcloud AIO Docker network | Active |
| `192.168.100.0/24` | `virbr10` | GitHub runner private NAT network | Active |
| `192.168.122.0/24` | `virbr0` | libvirt default network | Bridge down |

The server has global and private IPv6 addresses on its LAN interface. Public
DNS intentionally has no AAAA record for the documented external names.

## Software inventory

| Software | Management | Responsibility |
| --- | --- | --- |
| Debian base packages | Ansible `base` role | Administrative tools and base accounts |
| HDD mount | Ansible `storage` role | Persistent large storage |
| Docker Engine | Ansible `docker` role | Nextcloud and home-agent runtime |
| Nextcloud AIO master | Ansible `nextcloud_aio` role | Lifecycle management for the AIO stack |
| Nextcloud sibling containers | Nextcloud AIO | Files, database, cache, office, talk, and supporting services |
| Traefik `v3.7.1` | Ansible `shared_ingress` role | Public ACME TLS termination and hostname routing without Docker socket access |
| KVM/QEMU and libvirt | Ansible `libvirt_host` role | Virtual machine runtime |
| GitHub runner VM | Ansible `github_runner` role | Isolated self-hosted CI runner |
| SOPS/GPG | Ansible controller | Encrypted infrastructure secrets |
| Terraform DynDNS stack | `/home/julian/projects/dyndns` | Route 53 CNAMEs plus API Gateway/Lambda IPv4 updater |
| Terraform state bootstrap | `/home/julian/projects/terraform-state` (Git `main`) | Versioned, encrypted, public-blocked central S3 state bucket |
| AWS account bootstrap | `/home/julian/projects/aws-account-bootstrap` (local Git `main`) | Local-only owner of GitHub OIDC and repository plan/apply roles |
| GitHub repository protection | `/home/julian/projects/github-infra` (local Git `main`, no remote) | Terraform management of GitHub repository/branch protection, Actions permissions, and environments for `dyndns` and `testing`; local-only state backend |
| `home-tools` | Ansible `home_agent` role | Fixed, read-only host observations over a Unix socket |
| `home-agent` | Ansible `home_agent` role | Loopback-bound health API, published through authenticated Traefik routing |

## Docker networks

Observed networks:

- `bridge`
- `host`
- `none`
- `nextcloud-aio`

## Docker volumes

Observed AIO-managed volumes:

- `nextcloud_aio_mastercontainer`
- `nextcloud_aio_apache`
- `nextcloud_aio_database`
- `nextcloud_aio_database_dump`
- `nextcloud_aio_nextcloud`
- `nextcloud_aio_nextcloud_data`
- `nextcloud_aio_redis`

Observed legacy or unidentified volumes:

- `owncloud-docker-server_files`
- `owncloud-docker-server_mysql`
- `owncloud-docker-server_redis`
- one anonymous hash-named volume

Volume presence is not proof that data is unused. Never delete an unidentified
volume as routine cleanup.
