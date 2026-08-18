# Current State

- Snapshot date: **2026-08-18**
- Production host: `homeserver` (`192.168.178.100`)

## Executive summary

The physical homeserver is running Debian 13.6. Docker and libvirt are active.
Nextcloud AIO is the primary production application and all ten observed AIO
containers report healthy. The large HDD is mounted and has substantial free
capacity. No failed systemd units were observed.

The GitHub Actions runner is configured by committed Ansible code. The
dedicated bridge, guest interface, static neighbour entry, and VNC listener
were observed. A non-privileged `virsh` query timed out, so the libvirt domain
state was not independently enumerated during this snapshot.

The home-agent code has **not** been applied. It must not appear in current
deployment diagrams.

## Live evidence

| Area | Observed state |
| --- | --- |
| Operating system | Debian 13.6 (`trixie`) |
| Kernel | `6.12.95+deb13-amd64` |
| CPU | 4 logical CPUs; Intel Core i5-2400 documented hardware |
| Memory | 7.8 GiB usable |
| Root filesystem | 51 GiB total, 26 GiB used, 23 GiB available, 54% used |
| Large HDD | 916 GiB total, 138 GiB used, 732 GiB available, 16% used |
| Failed systemd units | None observed |
| Docker service | Active |
| libvirt service | Active |
| Nextcloud AIO | Ten running containers, all reporting healthy |
| Runner network | `virbr10` active at `192.168.100.1/24` |
| Runner guest evidence | `vnet9` present; `192.168.100.10` reachable in neighbour table |

Disk values above are human-readable `df` values and may differ slightly from
byte-level filesystem statistics.

## Repository state

Infrastructure repository root:

```text
/home/julian/projects/home-infra
```

Current committed revision at snapshot time:

```text
7dde15de1afcee81376c84225a60d12ab3a4500a
Simplify GitHub runner automation
```

The committed repository contains these roles:

- `base`
- `storage`
- `docker`
- `nextcloud_aio`
- `libvirt_host`
- `github_runner`

Uncommitted changes add `home_agent`, a dedicated playbook, tests, and README
documentation. Those changes are **proposed**, not deployed.

## Managed services

### Nextcloud AIO

Observed running containers:

- `nextcloud-aio-mastercontainer`
- `nextcloud-aio-apache`
- `nextcloud-aio-nextcloud`
- `nextcloud-aio-database`
- `nextcloud-aio-redis`
- `nextcloud-aio-imaginary`
- `nextcloud-aio-whiteboard`
- `nextcloud-aio-notify-push`
- `nextcloud-aio-talk`
- `nextcloud-aio-collabora`

The AIO master container is managed directly by Ansible. AIO itself creates
and manages the sibling containers.

### GitHub Actions runner

Committed automation defines a Debian 13 VM with:

- 2 virtual CPUs
- 2048 MiB RAM
- a 40 GiB qcow2 disk
- static address `192.168.100.10`
- private NAT network `192.168.100.0/24`
- one runner installation and Unix account per configured repository

The currently configured repository is `KandlerLi/dyndns`.

## Known gaps and drift candidates

| Observation | Classification | Follow-up |
| --- | --- | --- |
| No monitoring role or monitoring containers | Current gap | Design lightweight monitoring separately |
| No reverse proxy in Ansible | Current gap | Required before safely publishing more hostnames |
| `ai.jkandler.de` does not resolve | Expected | Create only when authenticated ingress exists |
| `iperf3`-typical port `5201` is listening | Unknown | Identify owner and decide whether to manage or remove it |
| Loopback port `43117` is listening | Unknown | Identify owning process with privileged diagnostics if needed |
| Legacy `owncloud-docker-server_*` Docker volumes exist | Unknown/legacy | Do not delete until contents and recovery need are confirmed |
| One anonymous Docker volume exists | Unknown | Identify owner without exposing stored data |
| Local `.venv` contains the obsolete repository path | Controller issue | Recreate the virtual environment at the current path |

## Explicitly not deployed

- `home-tools` restricted collector
- `home-agent` container
- OpenAI API credential
- `ai.jkandler.de` DNS record
- shared reverse proxy
- public agent authentication
- agent web UI
- Nextcloud document retrieval or indexing
