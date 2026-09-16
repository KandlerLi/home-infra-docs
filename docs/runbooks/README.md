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
- [Deploy read-only Nextcloud tools](deploy-nextcloud-tools.md) — **retired**
  2026-08-30, the Ansible/Docker deployment it described no longer exists;
  see the file itself for where the service actually lives now
  (`infra/k3s-apps`)
- [Enable Nextcloud write tools](enable-nextcloud-writes.md) — **retired**
  2026-08-30, same reason; the feature itself (ADR 0013) is still
  implemented, not yet deployed, just via a different mechanism now
- [Enable the Nextcloud Shopping List tools](enable-nextcloud-shopping-list.md) —
  **retired** 2026-08-30, same reason; the feature itself (ADR 0014) is
  still implemented, not yet deployed, just via a different mechanism now
- [Enable OpenAI voice transcription](enable-openai-voice-transcription.md) —
  routes Open WebUI's speech-to-text through home-agent instead of the
  local Whisper engine this deployment's network can't reach (deployed and
  confirmed working)
- [Manage repository infrastructure](manage-repo-infra.md) — Terraform-managed
  GitHub repository settings and per-repository AWS deploy identities,
  including how to import a repository's pre-existing manual configuration
- [Deploy and publish Deluge](publish-deluge.md) — private bootstrap, Nextcloud
  downloads integration, guarded route switch, and rollback
- [Enable the Finanzfluss Sankey exporter](enable-sankey-export.md) —
  dedicated Nextcloud account, WebDAV-only access, a cheap 1-minute ETag
  poll that only runs Playwright when the workbook actually changed
  (implemented, not yet deployed)
- [Deploy the personal website](deploy-personal-website.md) — static site
  at `www.jkandler.de` via S3 + CloudFront + ACM, GitHub-hosted CI, leaves
  the `jkandler.de` apex untouched (deployed, confirmed live)
- [Rotate and manage AWS Secrets Manager secrets](rotate-secrets.md) —
  per-secret rotation procedures, automated-rotation feasibility, and
  which secrets do/don't belong in Secrets Manager at all
- [Close Open WebUI's native-login MFA-bypass gap](harden-open-webui-login.md) —
  a stale PersistentConfig row, not the Terraform env vars, was the actual
  cause; the fix and how to redo it (deployed and confirmed live)
- [Fix Open WebUI's stale home-agent backend URL](fix-open-webui-stale-backend-url.md) —
  a second, independent PersistentConfig staleness incident (no model
  selectable, root cause traced to a Docker-era URL never replaced by
  the correct k3s Service address); fixed and confirmed live

Runbooks prefer reversible observations before changes. Commands that alter
production state require explicit human review and approval.
