# Contributing

## Update workflow

Follow [`docs/runbooks/update-documentation.md`](docs/runbooks/update-documentation.md):
which file takes which kind of change, and the checks before a PR. `main` is
protected; every change is a PR with the gitleaks check. Open work is tracked
in `/home/julian/projects/PARKED.md`, not here.

## C4 diagram rules

Every diagram must state its type and scope. Every element must have a name,
type, responsibility, and technology where relevant. Every relationship must
have a directional label. Every diagram must include a legend.

The diagrams use Mermaid flowcharts as a notation for C4 concepts. The C4
model is notation- and tooling-independent, so the semantic element labels are
more important than the shapes.

## Sensitive information

Allowed examples include LAN addresses, mount points, service names, public
hostnames, and secret variable names. Do not record:

- API keys or tokens
- private keys
- secret values from AWS Secrets Manager
- container environments (`docker inspect`, Kubernetes Secrets)
- document names or contents from private Nextcloud collections
- transient public IP addresses
- raw Terraform state or saved plan files
