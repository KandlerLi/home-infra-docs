# Runbook: Publish the Home Agent

Status: **Local deployment completed and verified on 2026-08-18. Public
ingress remains a planning checklist and must not be executed without a
reviewed implementation and rollback plan.**

## Phase A: Local deployment — completed

1. Reviewed and committed the `home_agent` role.
2. Added the OpenAI API key through SOPS.
3. Passed unit tests, Python checks, production-profile `ansible-lint`, and
   Ansible syntax checks.
4. Applied the dedicated agent playbook with human-entered sudo credentials.
5. Confirmed that only `127.0.0.1:8090` is published.
6. Confirmed the agent container has no Docker socket, capabilities, or writable
   root filesystem.
7. Completed a local health report and compared it with direct read-only
   checks.

## Phase B: Shared ingress design

1. Select a lightweight reverse proxy.
2. Design authentication independent of model behaviour.
3. Configure request-size, rate, and upstream time limits.
4. Prepare the supported Nextcloud AIO reverse-proxy configuration.
5. Document current port bindings and a precise rollback.
6. Test the proxy configuration away from public ports when possible.

## Phase C: Controlled Nextcloud migration

1. Schedule a maintenance window.
2. Verify backups and AIO health.
3. Move AIO Apache to the supported internal proxy port/binding.
4. Place the shared proxy on ports 80/443.
5. Validate the existing Nextcloud hostname internally and externally.
6. Validate WebDAV performance and certificate chains.
7. Roll back immediately if the existing service is degraded.

## Phase D: Agent hostname

1. Add the authenticated `ai.jkandler.de` virtual host.
2. Test authentication and rate limits before public DNS.
3. Create the Route 53 record without an AAAA record.
4. Validate from an external IPv4 network.
5. Update current C4 diagrams and the deployment inventory.
