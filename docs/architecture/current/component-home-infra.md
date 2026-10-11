# Infrastructure Automation — Container Diagram

- **C4 type:** Container
- **Scope:** The repositories that define the infrastructure, and what each
  one manages
- **Status:** Current as of 2026-10-11

```mermaid
flowchart LR
    julian["Julian<br/>[Person]"]

    subgraph local["Applied locally by Julian"]
        ansible["infra/home-infra<br/>[Container: Ansible]<br/>site.yml: homeserver; k3s.yml: k3s VMs"]
        tfstate["bootstrap/terraform-state<br/>[Container: Terraform]<br/>State bucket, operator identity, account baseline"]
        k3sboot["bootstrap/k3s-bootstrap<br/>[Container: Terraform]<br/>CI runners, PVs, privilege-defining RBAC"]
        repoinfra["github/repo-infra<br/>[Container: Terraform]<br/>GitHub settings, OIDC provider, per-repo CI roles"]
    end

    subgraph ci["Applied by CI on merge to main"]
        k3sapps["infra/k3s-apps<br/>[Container: Terraform]<br/>All app workloads in k3s"]
        awsrepos["aws/dyndns, website, ses-relay,<br/>aws-budget, homeserver-health-check,<br/>secrets-manager<br/>[Containers: Terraform]"]
        gha["github/gha-common<br/>[Container: reusable workflows]<br/>Checks, plan, apply, scans"]
    end

    homeserver["Homeserver and k3s VMs<br/>[Deployment Nodes]"]
    cluster["k3s cluster<br/>[Deployment Node]"]
    awsn["AWS account<br/>[External Software System]"]
    githubn["GitHub<br/>[External Software System]"]
    state[("Terraform state<br/>[Data store: S3]")]
    secrets[("Secrets Manager<br/>[Data store]")]

    julian -->|"ansible-playbook --ask-become-pass"| ansible
    julian -->|"terraform apply"| tfstate & k3sboot & repoinfra
    julian -->|"Merges PRs"| githubn
    githubn -->|"Runs"| gha
    gha -->|"Plan on PR, apply on merge"| k3sapps & awsrepos
    ansible -->|"Configures"| homeserver
    ansible -->|"Reads secrets"| secrets
    k3sboot --> cluster
    k3sapps --> cluster
    k3sapps -->|"Reads secrets"| secrets
    repoinfra --> githubn
    repoinfra -->|"OIDC provider, CI roles"| awsn
    awsrepos --> awsn
    tfstate --> awsn
    tfstate & k3sboot & repoinfra & k3sapps & awsrepos -->|"State"| state

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef external fill:#666,color:#fff,stroke:#333
    classDef container fill:#438dd5,color:#fff,stroke:#0b4884
    classDef datastore fill:#2e8b57,color:#fff,stroke:#1d5c3a
    classDef node fill:#f5e6c8,color:#000,stroke:#8a6d3b
    class julian person
    class awsn,githubn external
    class ansible,tfstate,k3sboot,repoinfra,k3sapps,awsrepos,gha container
    class state,secrets datastore
    class homeserver,cluster node
```

## Why the split

The local-only roots define the identities and privileges CI itself runs
with (state bucket, OIDC roles, runner RBAC), so CI must never apply them.
Ansible needs an interactive become password and stays local too. Everything
else is applied by CI on merge, with the PR review as the only gate. Details:
the repo map in `/home/julian/projects/CLAUDE.md` and
[`repositories.md`](../../repositories.md).

## Legend

- Dark blue: person
- Grey: external system
- Medium blue: repository (C4 container: deployable unit of automation)
- Green: data store
- Beige: what gets configured
