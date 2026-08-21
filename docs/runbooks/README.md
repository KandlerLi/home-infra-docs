# Runbooks

- [Rebuild the homeserver](rebuild-homeserver.md)
- [Operate Nextcloud AIO](nextcloud-aio.md)
- [Operate the GitHub runner VM](github-runner.md)
- [Migrate AWS account bootstrap ownership](migrate-aws-account-bootstrap.md) —
  historical: the first identity handoff, superseded by repo-infra
- [Update this documentation](update-documentation.md)
- [Operate and publish the home-agent](publish-home-agent.md) — local operation
  and proposed public-ingress checklist
- [Deploy and publish Open WebUI](publish-open-webui.md) — private bootstrap,
  guarded route switch, validation, backup, and rollback
- [Deploy read-only Nextcloud tools](deploy-nextcloud-tools.md) — dedicated
  account, scoped share, encrypted app password, validation, and rollback
- [Manage repository infrastructure](manage-repo-infra.md) — Terraform-managed
  GitHub repository settings and per-repository AWS deploy identities,
  including how to import a repository's pre-existing manual configuration

Runbooks prefer reversible observations before changes. Commands that alter
production state require explicit human review and approval.
