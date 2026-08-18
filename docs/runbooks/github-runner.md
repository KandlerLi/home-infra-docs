# Runbook: Operate the GitHub Runner VM

## Current configuration

| Setting | Value |
| --- | --- |
| VM name | `github-runner` |
| Guest address | `192.168.100.10` |
| Network | `github_runner_network` on `virbr10` |
| vCPUs | 2 |
| Memory | 2048 MiB |
| Disk | 40 GiB qcow2 |
| Managed repository | `KandlerLi/dyndns` |

## Apply runner configuration

```bash
cd /home/julian/projects/home-infra
.venv/bin/ansible-playbook ansible/playbooks/github-runner.yml \
  --ask-become-pass
```

## Safety behaviour

The role deliberately fails when:

- a VM exists without the expected disk
- a disk exists without the expected VM
- the existing domain points to unexpected disk, network, MAC, or cloud-init
  media
- a remote runner registration conflicts with local state
- a runner is busy while an update would interrupt it

Resolve these states manually. Do not bypass the checks with forced disk or VM
recreation.

## Runtime validation

Read-only evidence from the 2026-08-18 snapshot:

- `libvirtd` active
- `virbr10` active at `192.168.100.1/24`
- guest interface `vnet9` present
- neighbour `192.168.100.10` at configured MAC address reachable
- VNC bound to loopback port 5900

A direct non-privileged `virsh` enumeration timed out. Use an approved
privileged session when exact domain state is required.

## Adding a repository

1. Grant the fine-grained GitHub token access to the repository.
2. Add a unique repository entry to group variables.
3. Review the runner name and Unix account generated from the ID.
4. Apply the runner playbook.
5. Confirm the new runner is online in the repository settings.

Removing configuration does not automatically unregister or delete a runner.
Decommission it explicitly first.
