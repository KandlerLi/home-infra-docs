# Runbook: Publish the Home Agent

Status: **Local agent deployment completed on 2026-08-18. Traefik preparation
and the controlled Nextcloud ingress migration completed and were verified on
2026-08-19. Nextcloud and the authenticated agent endpoint are now live through
the shared proxy at their respective hostnames.**

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

## Phase B: Shared ingress preparation — completed

The prepared implementation uses pinned Traefik `v3.7.1`, host networking,
file-based routing, and no Docker socket. The agent router uses Basic Auth,
ten requests per minute with a burst of five, a 16 KiB request-body limit, and
security headers. Nextcloud is not placed behind additional authentication.

1. Set `shared_ingress_acme_email` in the non-secret group variables.
2. Keep `shared_ingress_agent_enabled` false for the initial Nextcloud move.
3. Confirm the generated Basic Auth password and bcrypt verifier remain
   encrypted in SOPS.
4. Run unit tests, production-profile `ansible-lint`, and syntax checks.
5. Prepare files and validate Traefik without binding public ports:

   ```bash
   .venv/bin/ansible-playbook \
     ansible/playbooks/shared-ingress-prepare.yml \
     --ask-become-pass
   ```

6. Verified that the pinned image is present, no `shared-ingress` container is
   running, Nextcloud still owns its original ports and returns HTTP 200, and
   the loopback-only home-agent health endpoint returns `{"status":"ok"}`.

## Phase C: Controlled Nextcloud migration — completed

1. Schedule a maintenance window.
2. Verify a recent AIO backup, free disk capacity, and current AIO health.
3. Open the AIO interface using the homeserver IP and stop all application
   containers. Leave the master container running.
4. Run the guarded cutover:

   ```bash
   .venv/bin/ansible-playbook ansible/playbooks/shared-ingress.yml \
     --ask-become-pass \
     --extra-vars \
     ingress_cutover_confirmation=MIGRATE_NEXTCLOUD_INGRESS
   ```

5. In the AIO interface, start the application containers again.
6. Confirm AIO Apache publishes `127.0.0.1:11000` and Traefik owns TCP ports
   80/443.
7. Validate `https://nextcloud.jkandler.de/status.php`, WebDAV transfers,
   browser access, certificate chains, and the AIO health view internally and
   externally.
8. Persist `nextcloud_aio_reverse_proxy_enabled: true`,
   `shared_ingress_enabled: true`, and `shared_ingress_start: true` in group
   variables so ordinary `site.yml` runs retain the deployed mode. Review and
   commit that state change.
9. Roll back immediately if the existing service is degraded.

The first cutover run stopped safely before creating Traefik because the local
`community.docker` collection expects `capabilities` rather than `cap_add`.
Infrastructure commit `2c50768` corrected the field and added a regression
assertion. The idempotent retry succeeded. Final checks showed every AIO
container healthy, Apache bound only to `127.0.0.1:11000`, Traefik healthy and
hardened on ports 80/443, and public Nextcloud returning HTTP/2 200 with a
valid Let's Encrypt certificate. Commit `2f8039d` persists the deployed mode.

### Rollback

1. Stop all AIO application containers through the AIO interface.
2. Run:

   ```bash
   .venv/bin/ansible-playbook \
     ansible/playbooks/shared-ingress-rollback.yml \
     --ask-become-pass \
     --extra-vars \
     ingress_cutover_confirmation=ROLLBACK_NEXTCLOUD_INGRESS
   ```

3. Start the AIO application containers again.
4. Confirm AIO master owns port 80 and AIO Apache owns TCP/UDP port 443.
5. Restore the persisted group variables to direct-ingress/disabled-proxy
   values and commit that state.
6. Revalidate Nextcloud externally. Do not delete Traefik ACME state during
   rollback.

## Phase D: Agent hostname — completed

1. Add `ai` to the `subdomains` set in the `KandlerLi/dyndns` Terraform
   repository. The merged apply creates a CNAME for `ai.jkandler.de` pointing
   to `jkandler.de`. Do not create an AAAA record.
2. Wait until the CNAME and resulting A record resolve externally.
3. Retrieve the generated Basic Auth password directly into a password manager
   without copying it into documentation or chat:

   ```bash
   sops -d --extract \
     '["shared_ingress_agent_auth_password"]' \
     ansible/inventory/group_vars/all/secrets.sops.yml
   ```

4. Enable the authenticated router with the separate guarded playbook:

   ```bash
   .venv/bin/ansible-playbook \
     ansible/playbooks/shared-ingress-agent.yml \
     --ask-become-pass \
     --extra-vars \
     ingress_cutover_confirmation=MIGRATE_NEXTCLOUD_INGRESS
   ```

5. Confirm an unauthenticated request returns HTTP 401.
6. Use `curl --user julian` so the password is prompted rather than stored in
   shell history, then validate `/healthz` and one small `/v1/chat` request.
7. Validate from an external IPv4 network and confirm Nextcloud remains
   healthy.
8. Update current C4 diagrams and the deployment inventory.

Terraform change `0fde1dd` created the record. Both Cloudflare and Google
resolvers returned the CNAME and resulting IPv4 address without an AAAA address.
The public endpoint presented a valid Let's Encrypt certificate for
`ai.jkandler.de`, returned 401 without credentials, and returned 200 for
authenticated `/healthz` and `/v1/chat` calls. Nextcloud remained healthy.
Infrastructure commit `c07fa3f` persists the enabled agent router.
