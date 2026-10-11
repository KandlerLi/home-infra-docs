# Runbook: Migrate AWS Account Bootstrap Ownership

Status: completed on 2026-08-19; retained as migration and recovery evidence.
The destination root this runbook describes, `aws-account-bootstrap`, was
itself retired and deleted on 2026-08-21 after its resources were migrated a
second time into [`repo-infra`](https://github.com/KandlerLi/repo-infra) (see
[ADR 0010](../adr/0010-consolidate-repo-and-identity-control-plane.md) and
[Manage repository infrastructure](manage-repo-infra.md) for
the current procedure). This runbook remains as historical evidence of the
first handoff pattern, which the second handoff reused unchanged.

## Scope

Transfer Terraform state ownership of these existing AWS resources from
`dyndns` to `aws-account-bootstrap` without changing the live resources:

- account-wide GitHub Actions OIDC provider
- DynDNS apply role and inline policy
- DynDNS plan role and inline policy

The new Terraform root is local-only. Do not add or use GitHub Actions for it.

## Preconditions

- Both repositories have clean worktrees at the reviewed migration commits.
- The existing DynDNS plan/apply workflows are healthy.
- A trusted, short-lived local AWS administrative or bootstrap identity is
  selected and verified.
- `aws-account-bootstrap/terraform.tfvars` contains the same Route 53 zone ID
  used by DynDNS.
- No concurrent Terraform plan or apply is running.

Do not read, copy, or paste raw Terraform state. Do not use manual state-file
downloads or `terraform state push` for this migration.

## Phase 1: relinquish the source bindings

1. Open the prepared `dyndns` change as a pull request.
2. Confirm the speculative plan reports only the five state removals and says
   each remote object will not be destroyed because `destroy = false`.
3. Complete the existing exact-commit owner approval and protected production
   approval.
4. Merge and allow the protected apply to remove only those state bindings.
5. Confirm the workflow completes and can still authenticate with its unchanged
   AWS role.

Stop if the plan proposes any AWS resource change, replacement, or destroy.

## Phase 2: import into the local control plane

From [`repo-infra`](https://github.com/KandlerLi/repo-infra):

```bash
export AWS_PROFILE=terraform-bootstrap
aws sts get-caller-identity
terraform init -input=false
terraform plan
```

At the time of migration, the plan imported the five existing resources
selected by the committed `imports.tf`. It was applied locally only after that
review and explicit confirmation. The one-time import file was removed from
the active root after successful verification; this runbook retains the
procedure.

## Phase 3: verify

- A second bootstrap plan reports no changes.
- A DynDNS plan reports no changes to IAM/OIDC resources.
- The protected apply and same-repository pull-request plan can still assume
  their distinct roles.
- GitHub repository variables still match the bootstrap outputs.
- The `aws-account-bootstrap` state key is not accessible to either DynDNS role.

## Completion evidence

- DynDNS PR #8 merged as `d8d6541`.
- PR validation, speculative plan, exact-commit approval, and protected apply
  completed successfully.
- The existing OIDC provider, both roles, and both inline policies were imported
  into the local-only bootstrap state without replacement or destruction.
- Final no-change plans and both GitHub role paths were user-confirmed.
- The one-time destination import configuration was removed after completion;
  its history remains available in Git and this runbook.
