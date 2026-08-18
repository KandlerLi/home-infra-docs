# System Inventory

## Physical host

| Property | Value |
| --- | --- |
| Inventory name | `homeserver` |
| Hardware | Fujitsu ESPRIMO P900 |
| CPU | Intel Core i5-2400, 4 cores / 4 threads, 3.10 GHz |
| Memory | 8 GB DDR3; approximately 7.8 GiB usable |
| GPU | None dedicated |
| Operating system | Debian 13.6 (`trixie`) |
| Architecture | x86-64 |
| LAN address | `192.168.178.100/24` |
| Ansible user | `julian` |
| Ansible Python | `/usr/bin/python3` |

The host is suitable for orchestration and lightweight services. It is not a
sensible primary inference host for modern large language models.

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
| Docker Engine | Ansible `docker` role | Nextcloud and future application runtime |
| Nextcloud AIO master | Ansible `nextcloud_aio` role | Lifecycle management for the AIO stack |
| Nextcloud sibling containers | Nextcloud AIO | Files, database, cache, office, talk, and supporting services |
| KVM/QEMU and libvirt | Ansible `libvirt_host` role | Virtual machine runtime |
| GitHub runner VM | Ansible `github_runner` role | Isolated self-hosted CI runner |
| SOPS/GPG | Ansible controller | Encrypted infrastructure secrets |

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
