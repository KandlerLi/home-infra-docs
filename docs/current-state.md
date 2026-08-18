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

The private home-agent stack is deployed and verified. The `home-tools`
systemd service is active, the `home-agent` Docker container is running, and
an end-to-end model request successfully called the restricted health tool.
The agent API remains loopback-only and has no public hostname.

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
| `home-tools` service | Active; fixed read-only collectors over a protected Unix socket |
| `home-agent` container | Running; healthy API on `127.0.0.1:8090` |
| Agent end-to-end check | Successful OpenAI tool call and health report |
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
3c24b530f4202f180b8d2ab14ab8ed8cb95add33
Configure encrypted home-agent API credential
```

The committed repository contains these roles:

- `base`
- `storage`
- `docker`
- `nextcloud_aio`
- `libvirt_host`
- `github_runner`
- `home_agent`

The `home_agent` role, dedicated playbook, tests, and encrypted API credential
are committed. The dedicated playbook was applied with human-entered sudo
credentials on 2026-08-18.

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

- `ai.jkandler.de` DNS record
- shared reverse proxy
- public agent authentication
- agent web UI
- Nextcloud document retrieval or indexing

## Agent verification evidence

Observed on 2026-08-18:

- `home-tools.service` active under systemd
- `home-agent` running with a read-only root filesystem
- Docker privileged mode disabled and every Linux capability dropped
- only the protected tool socket directory and API-key file mounted read-only
- host publication restricted to `127.0.0.1:8090`
- live health request reported `/` at 50.9% and `/mnt/black-hdd` at 15.0%
- no failed systemd units reported
