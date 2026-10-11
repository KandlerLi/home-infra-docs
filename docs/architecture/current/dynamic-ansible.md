# Change Rollout — Dynamic Diagrams

- **C4 type:** Dynamic
- **Scope:** How a change reaches production, for a CI-applied Terraform
  repo and for the Ansible-managed homeserver
- **Status:** Current as of 2026-10-11

## CI-applied Terraform repo (e.g. [`k3s-apps`](https://github.com/KandlerLi/k3s-apps))

```mermaid
sequenceDiagram
    autonumber
    actor J as Julian
    participant G as GitHub
    participant R as Self-hosted runner (k3s-node-2)
    participant A as AWS (OIDC, S3 state, Secrets Manager)
    participant T as Target (k3s cluster or AWS)

    J->>G: Push branch, open PR
    G->>R: checks.yml (gha-common): validate, trivy, gitleaks
    R->>A: Assume plan role (OIDC, pull_request subject)
    R->>T: terraform plan (read-only)
    R-->>G: Plan posted as PR comment
    J->>G: Review and merge
    G->>R: apply.yml on main (production environment)
    R->>A: Assume apply role (OIDC, environment subject)
    R->>T: terraform apply
```

## Ansible ([`home-infra`](https://github.com/KandlerLi/home-infra))

```mermaid
sequenceDiagram
    autonumber
    actor J as Julian
    participant L as Laptop (ansible-playbook)
    participant A as AWS Secrets Manager
    participant H as Homeserver

    J->>L: ansible-playbook site.yml --ask-become-pass
    L->>A: Fetch secret groups with Julian's own credentials
    L->>H: SSH, become root
    L->>H: Apply roles in order (base ... aux_backup)
    H-->>L: Changed/ok per task
```

`k3s.yml` follows the same pattern for the k3s VMs and is deliberately run
separately from `site.yml`. The locally applied Terraform roots
(`repo-infra`, `terraform-state`, `k3s-bootstrap`) are run by Julian
through each repo's `scripts/roll-out.sh`.
