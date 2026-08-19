# Runbook: Migrate AWS Account Bootstrap Ownership

Status: completed on 2026-08-19; retained as migration and recovery evidence.

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

From `/home/julian/projects/aws-account-bootstrap`:

```bash
export AWS_PROFILE=terraform-bootstrap
aws sts get-caller-identity
terraform init -input=false
terraform plan
```

Review the plan. It must import the five existing resources selected by
`imports.tf`. Apply locally only after that review and explicit confirmation.

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
