# AWS Account Bootstrap — Container Diagram

- **C4 type:** Container
- **Scope:** AWS identity and Terraform deployment control plane
- **Status:** Current/deployed as of 2026-08-19

```mermaid
flowchart LR
    operator["Julian<br/>[Person]<br/>Reviews and runs privileged bootstrap changes locally"]
    github["GitHub Actions<br/>[External Software System]<br/>Issues repository-bound OIDC tokens"]
    aws["AWS IAM<br/>[External Software System]<br/>Authenticates OIDC and enforces role permissions"]

    subgraph infrastructure["Home Infrastructure [Software System]"]
        bootstrap["AWS Account Bootstrap<br/>[Container: local-only Terraform]<br/>Owns GitHub OIDC and repository plan/apply roles"]
        dyndns_iac["DynDNS Automation<br/>[Container: Terraform and GitHub Actions]<br/>Owns DNS updater application resources"]
        state["Terraform State Backend<br/>[Container: S3 data store]<br/>Stores isolated bootstrap and DynDNS state keys"]
    end

    operator -->|"Runs reviewed plans with a short-lived bootstrap identity"| bootstrap
    bootstrap -->|"Manages OIDC provider, trust policies, and least-privilege roles"| aws
    bootstrap -->|"Reads, locks, and writes aws-account-bootstrap state"| state
    github -->|"Exchanges repository-bound OIDC tokens"| aws
    aws -->|"Issues short-lived plan/apply sessions"| dyndns_iac
    dyndns_iac -->|"Reads or updates only the DynDNS state key"| state

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef external fill:#666,color:#fff,stroke:#333
    classDef container fill:#438dd5,color:#fff,stroke:#0b4884
    classDef datastore fill:#2e8b57,color:#fff,stroke:#1d5c3a
    class operator person
    class github,aws external
    class bootstrap,dyndns_iac container
    class state datastore
```

## Deployed boundary

| Container | Technology | Responsibility |
| --- | --- | --- |
| AWS Account Bootstrap | Terraform, AWS provider, trusted local AWS session | Own account-wide OIDC and repository-bound plan/apply identities |
| DynDNS Automation | Terraform, GitHub Actions OIDC | Manage application resources without changing its own automation identity |
| Terraform State Backend | Amazon S3 | Isolate bootstrap and workload state under separate encrypted keys |

## Legend

- Blue: person
- Grey: external system
- Light blue: C4 application container
- Green: C4 data-store container
- Arrow label: protocol or intent of the relationship
