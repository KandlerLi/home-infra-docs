# Repository Infrastructure — Container Diagram

- **C4 type:** Container
- **Scope:** GitHub repository configuration and the AWS identity control
  plane for CI
- **Status:** Current as of 2026-10-11

```mermaid
flowchart LR
    operator["Julian<br/>[Person]<br/>Runs control-plane changes locally"]
    github_api["GitHub API<br/>[External Software System]<br/>Repository settings, branch protection, OIDC tokens"]
    aws["AWS IAM<br/>[External Software System]<br/>OIDC provider and per-repo roles"]

    subgraph infrastructure["Home Infrastructure [Software System]"]
        repo_infra["repo-infra<br/>[Container: local-only Terraform]<br/>config.yml + aws_policies.tf for every repo"]
        workloads["CI-applied repos<br/>[Containers: Terraform + GitHub Actions]<br/>k3s-apps, dyndns, website, ses-relay, aws-budget, homeserver-health-check, secrets-manager"]
        state["Terraform state<br/>[Container: S3 data store]<br/>One key per root"]
    end

    operator -->|"Applies with the julian IAM user + gh login"| repo_infra
    repo_infra -->|"Settings, branch protection, PR policy, environments"| github_api
    repo_infra -->|"OIDC provider, plan and apply role per repo"| aws
    repo_infra -->|"Own state key"| state
    github_api -->|"Repository-bound OIDC tokens"| aws
    aws -->|"Short-lived plan/apply sessions"| workloads
    workloads -->|"Each its own state key"| state

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef external fill:#666,color:#fff,stroke:#333
    classDef container fill:#438dd5,color:#fff,stroke:#0b4884
    classDef datastore fill:#2e8b57,color:#fff,stroke:#1d5c3a
    class operator person
    class github_api,aws external
    class repo_infra,workloads container
    class state datastore
```

| Container | Technology | Responsibility |
| --- | --- | --- |
| repo-infra | Terraform (GitHub + AWS providers), local only | Every repo's GitHub settings; the account-wide OIDC provider; a read-only plan role (PR subject) and an apply role (`production` environment subject) per AWS-deploying repo |
| CI-applied repos | Terraform via `gha-common` workflows | Manage their own resources; can never change their own trust or policies |
| Terraform state | S3 (`bootstrap/terraform-state`) | Isolated state per root; the plan role can only read its own key |

Adding a repo: the `onboard-new-repo` skill. Decisions: ADR 0009, 0010,
0022.

## Legend

- Dark blue: person
- Grey: external system
- Medium blue: C4 application container
- Green: C4 data-store container
