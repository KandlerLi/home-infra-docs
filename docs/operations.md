# Operations

## Controller preparation

All repositories live below:

```text
/home/julian/projects/
```

The infrastructure repository is:

```text
/home/julian/projects/home-infra
```

Do not use the obsolete `/home/julian/home-infra` path.

The local virtual environment was moved with the repository and contains old
absolute shebangs. Recreate it before normal use:

```bash
cd /home/julian/projects/home-infra
python3 -m venv --clear .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/ansible-galaxy collection install -r requirements.yml
```

`--clear` is appropriate only for this generated `.venv`, never for an
infrastructure or data directory.

## Pre-change checklist

```bash
cd /home/julian/projects/home-infra
git status --short --branch
.venv/bin/ansible home_servers -m ansible.builtin.ping
.venv/bin/ansible-playbook ansible/playbooks/site.yml --syntax-check
```

Before applying, review:

- the complete Git diff
- whether `/mnt/black-hdd` is mounted
- whether a task could recreate a production container or VM
- whether required SOPS secrets decrypt successfully
- whether the change affects public ports or Nextcloud storage

## Apply infrastructure

Full infrastructure:

```bash
.venv/bin/ansible-playbook ansible/playbooks/site.yml \
  --ask-become-pass
```

GitHub runner only:

```bash
.venv/bin/ansible-playbook ansible/playbooks/github-runner.yml \
  --ask-become-pass
```

The home-agent playbook exists only in uncommitted, unapplied work at the
snapshot date. Do not treat it as a current operational command.

## Inspect the host

Repository inspection playbook:

```bash
.venv/bin/ansible-playbook ansible/playbooks/inspect.yml \
  --ask-become-pass
```

Useful read-only checks:

```bash
ssh julian@192.168.178.100
df -hT
systemctl --failed
docker ps
docker network ls
docker volume ls
```

Do not use `docker inspect` in logs or documentation without filtering because
it can expose container environment variables and secrets.

## Edit encrypted secrets

```bash
cd /home/julian/projects/home-infra
sops ansible/inventory/group_vars/all/secrets.sops.yml
```

Verify the file remains encrypted before staging it.

## Change policy

- Git changes are prepared and reviewed before deployment.
- Production Ansible is never run automatically by an AI agent.
- Nextcloud storage and AIO volumes are not reorganised during unrelated work.
- Orphaned VM disks, libvirt domains, or Docker volumes require manual
  investigation before cleanup.
- DNS AAAA records require explicit IPv6 end-to-end validation.
