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

The Terraform DNS repository is:

```text
/home/julian/projects/dyndns
```

The central Terraform state-bucket bootstrap project is:

```text
/home/julian/projects/terraform-state
```

The local-only AWS identity control-plane project is:

```text
/home/julian/projects/aws-account-bootstrap
```

It must never run through GitHub Actions or through a role it manages. Its
initial no-destroy state handoff completed on 2026-08-19 and is retained in the
dedicated migration runbook. Future plans and applies require a trusted,
short-lived local bootstrap identity.

It is a Git repository on `main`. Do not inspect or copy its ignored local
state/backup contents during routine work. Enumerate every top-level directory
under `/home/julian/projects` before workspace-wide tasks so future non-Git
projects are not omitted.

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

Home-agent only:

```bash
.venv/bin/ansible-playbook ansible/playbooks/home-agent.yml \
  --ask-become-pass
```

The dedicated playbook is opt-in. The main `site.yml` keeps the role disabled
unless explicitly enabled.

After deployment, verify from the homeserver or through Ansible:

```bash
curl --fail-with-body http://127.0.0.1:8090/healthz
curl --fail-with-body \
  --header 'Content-Type: application/json' \
  --data '{"message":"Check my homeserver."}' \
  http://127.0.0.1:8090/v1/chat
```

Public authenticated checks prompt for the password instead of storing it in
shell history:

```bash
curl --user julian https://ai.jkandler.de/healthz
curl --user julian \
  --header 'Content-Type: application/json' \
  --data '{"message":"Check my homeserver."}' \
  https://ai.jkandler.de/v1/chat
```

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

Shared-ingress checks:

```bash
docker inspect shared-ingress \
  --format 'running={{.State.Running}} health={{.State.Health.Status}}'
curl --fail-with-body http://127.0.0.1:8082/ping
curl --fail-with-body \
  --header 'Host: nextcloud.jkandler.de' \
  http://127.0.0.1:11000/status.php
curl --fail-with-body https://nextcloud.jkandler.de/status.php
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
- The deployed home-agent can observe but cannot apply Ansible or remediate.
- Nextcloud storage and AIO volumes are not reorganised during unrelated work.
- Orphaned VM disks, libvirt domains, or Docker volumes require manual
  investigation before cleanup.
- DNS AAAA records require explicit IPv6 end-to-end validation.
