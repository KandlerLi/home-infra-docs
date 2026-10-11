# Runbook: Update Documentation

Run this after every change that makes something in this repository
untrue: a service, hostname, port, auth rule, backup, disk, node, DNS
record, secret, CI identity, or a manual procedure.

## Where it goes

| What changed | Update |
| --- | --- |
| A fact about what is live | [`current-state.md`](../current-state.md): facts only, date anything checked live |
| A durable choice between real alternatives | A new ADR in [`adr/`](../adr/README.md) (next number); amend old ADRs only for factual accuracy |
| A smaller decision, or the incident behind a workaround | [`decisions.md`](../decisions.md) |
| A manual step | A runbook here, and the [index](README.md) |
| A container, network path or trust relationship | The views in [`architecture/current/`](../architecture/current/) |
| Disk layout or storage rules | [`storage.md`](../storage.md) |
| Everyday commands | [`operations.md`](../operations.md) |
| A security boundary | Also the owner's agent rule set (`check-security-invariants`, kept outside this repo) |

[`history.md`](../history.md) is frozen: never append to it. Code comments
get one line plus a pointer to where the reasoning lives.

## Checks before the PR

- No secret values, Terraform state, document contents or transient public
  IPs.
- Relative links resolve; Mermaid blocks render (check the PR's rich diff).
- `git diff --check` is clean.

`main` is protected: every change goes through a PR with the gitleaks check.
