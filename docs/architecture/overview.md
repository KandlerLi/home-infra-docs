# Architecture Summary

## Public request paths

```text
Remote Nextcloud client
  -> Route 53 CNAME/A
  -> FRITZ!Box NAT
  -> Traefik :443
  -> nextcloud-svc (k3s)
  -> Nextcloud Pod (Apache, PostgreSQL, Redis, notify_push), data over NFS
```


## DNS update path

```text
FRITZ!Box
  -> authenticated AWS API Gateway endpoint
  -> Python Lambda
  -> Route 53 jkandler.de A record

Terraform/GitHub Actions
  -> stable nextcloud and ai CNAME records
```

## AWS automation trust path

Current deployed ownership is:

```text
Trusted local operator with short-lived bootstrap identity + GitHub token
  -> local-only repo-infra Terraform root
  -> account-wide GitHub OIDC provider
  -> repository-bound DynDNS plan/apply roles and policies
  -> GitHub repository settings (dyndns, testing)

GitHub Actions OIDC token
  -> plan or protected-environment apply role
  -> dyndns Terraform state and allowed workload resources only
```

Neither DynDNS automation role may access the `repo-infra` state key or modify
its own trust or permissions. `repo-infra` (renamed from `github-infra`)
absorbed this trust path from the retired `aws-account-bootstrap` root on
2026-08-21.

## Technology choices

- configuration: Ansible for the host; Terraform for AWS/DNS; local-only
  Terraform for AWS automation identities
- runtime: k3s for the apps, Nextcloud included (ADR 0033); Docker on the host for monitoring
- secrets: AWS Secrets Manager for everything (the 2026-09-09
  SOPS-to-Secrets-Manager cutover retired SOPS/GPG entirely)
- ingress: pinned Traefik `v3.7.1`, host network, file provider, no Docker socket
- documentation: C4 views in `home-infra-docs`

See `docs/architecture/README.md` for the
authoritative diagrams and notation.
