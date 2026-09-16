# Runbooks

- [Rebuild the homeserver](rebuild-homeserver.md)
- [Operate Nextcloud AIO](nextcloud-aio.md)
- [Operate the GitHub runner VM](github-runner.md) — **retired** 2026-08-31,
  the runner moved from a dedicated VM to a k3s Deployment per repository;
  see the file itself for the current architecture and the still-valid
  "adding a repository" steps
- [Migrate AWS account bootstrap ownership](migrate-aws-account-bootstrap.md) —
  historical: the first identity handoff, superseded by repo-infra
- [Update this documentation](update-documentation.md)
- [Operate and publish the home-agent](publish-home-agent.md) — **retired**
  2026-09-01, described bootstrapping the now-deleted `shared_ingress` role;
  see the file itself for where public routing lives now (`infra/k3s-apps`)
- [Deploy and publish Open WebUI](publish-open-webui.md) — **retired**
  2026-08-30, the Ansible/Docker deployment it described no longer exists;
  see the file itself for where the service actually lives now
  (`infra/k3s-apps`)
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
  deploy steps **retired** 2026-08-30 (routes Open WebUI's speech-to-text
  through home-agent instead of the local Whisper engine this deployment's
  network can't reach; deployed and working, but the Admin Panel
  configuration/verification steps in this file are still the real
  procedure)
- [Manage repository infrastructure](manage-repo-infra.md) — Terraform-managed
  GitHub repository settings and per-repository AWS deploy identities,
  including how to import a repository's pre-existing manual configuration
- [Deploy and publish Deluge](publish-deluge.md) — **retired** 2026-08-29/30,
  the Ansible/Docker deployment it described no longer exists (`deluge.yml`
  still exists as a file but now only sets up k3s host prerequisites); see
  the file itself for where the service actually lives now (`infra/k3s-apps`)
- [Enable the Finanzfluss Sankey exporter](enable-sankey-export.md) —
  **retired** 2026-09-07, the whole role was deleted outright once the
  exporter moved to a k3s CronJob; see the file itself for the current
  architecture and the still-valid Nextcloud-share prerequisite
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
