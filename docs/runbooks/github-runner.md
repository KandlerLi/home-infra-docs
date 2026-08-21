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
| OS packages | `ca-certificates`, `curl`, `git`, `gzip`, `python3`, `tar`, `unzip` |

Since 2026-08-21, `dyndns`'s CI actually runs here: `checks.yml`'s `Validate`
and `Terraform plan` jobs and `apply.yml`'s `apply` job all use
`runs-on: [self-hosted, home, debian]` instead of `ubuntu-latest`. `python3`
and `unzip` were added to the package list for this — `python3` for the
Lambda unit tests, `unzip` for `hashicorp/setup-terraform`'s binary
extraction. This is safe against untrusted code because `dyndns`'s
`pull_request_creation_policy: collaborators_only` plus a single
collaborator (the owner) means only the owner can ever open a PR that
triggers these jobs.

## Apply runner configuration

```bash
cd /home/julian/projects/infra/home-infra
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

The repository entry in `github_runner_github_repositories` is generated,
not hand-edited, since 2026-08-21 — see
`/home/julian/projects/bootstrap/repo-infra/README.md`'s "Adding a
repository with a self-hosted GitHub Actions runner" section for the full
workflow. In short:

1. Grant the fine-grained GitHub token access to the repository.
2. Add `runner: true` to the repository's entry in
   `/home/julian/projects/bootstrap/repo-infra/config.yml`.
3. In `home-infra`, run
   `.venv/bin/python scripts/sync_github_runner_repositories.py`, review the
   diff to `ansible/inventory/group_vars/all/github_runner_repositories.yml`,
   and commit it.
4. Review the runner name and Unix account generated from the ID.
5. Apply the runner playbook.
6. Confirm the new runner is online in the repository settings.

Removing configuration does not automatically unregister or delete a runner.
Decommission it explicitly first, then remove `runner: true` from
`config.yml` and re-run the sync script.
