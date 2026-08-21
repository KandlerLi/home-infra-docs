# Runbook: Enable Nextcloud Write Tools

Status: **Implemented, not yet deployed.** See ADR 0011 for the design.
This is a follow-on to [Deploy Read-Only Nextcloud Tools](deploy-nextcloud-tools.md)
(already deployed) — it adds create/update/delete/move on top of the same
`nextcloud-tools` service and `home-agent` integration, gated behind
explicit human confirmation.

## 1. Prerequisite: widen the existing share (manual, not Ansible)

The dedicated `home-agent` Nextcloud account's share of `AI Workspace` was
created **read-only** (see `deploy-nextcloud-tools.md` step 3). That was a
deliberate manual action by the folder's owner — automating it would
require giving Ansible a credential for the personal account that owns the
data, which ADR 0008 explicitly rejected. The same reasoning applies here:
widening it is also a manual step.

Sign in to Nextcloud as the account that owns `AI Workspace`, open its
sharing settings for the `home-agent` share, and enable **Can edit** (which
covers create, change, and delete). Do this before deploying the code
below — otherwise every confirmed write will fail at the WebDAV layer with
403, independent of whether the confirmation code was correct.

## 2. Run tests and syntax checks

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/python -m unittest discover -s tests -v
.venv/bin/ansible-playbook ansible/playbooks/site.yml --syntax-check
```

## 3. Deploy

This is a code-only change to the existing `nextcloud-tools` service and
`home-agent`; no new playbook or role. Redeploy both:

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/ansible-playbook ansible/playbooks/nextcloud-tools.yml \
  --ask-become-pass \
  --extra-vars nextcloud_tools_bootstrap_confirmation=BOOTSTRAP_NEXTCLOUD_TOOLS
.venv/bin/ansible-playbook ansible/playbooks/home-agent.yml --ask-become-pass
```

The bootstrap confirmation sentinel is required by the existing guarded
play even though no new account/token is being created here — it's the
same play that installs the updated service file. It detects an existing
account/token and reuses them rather than creating new ones.

## 4. Verify live, end to end

Drive this through a real chat with `home-agent` (via Open WebUI or
directly against `/v1/chat`), not just a clean deploy:

1. Ask the agent to create a small test file in `Shared/AI Workspace`
   (e.g. a one-line note). It should call `propose_nextcloud_write` and
   show you a confirmation code and summary in its reply — verify the
   code and summary are shown to you verbatim, not paraphrased away.
2. Reply with a message that includes that exact code (e.g. "yes, ABC123,
   go ahead"). Confirm the agent reports success, and independently
   confirm the file actually landed — either by asking the agent to read
   it back, or via `occ`/WebDAV on the host.
3. **Wrong/missing code**: propose another write, then reply *without*
   the code (e.g. just "yes, do it"). Confirm the write is refused
   (`confirmation_not_verified`) and nothing was written — this is the
   actual security boundary, so it's worth confirming deliberately, not
   assuming the code review was correct.
4. **Replay**: after a successful confirm, try sending the same code
   again in a later message. Confirm it's rejected (`not_found` — codes
   are single-use).
5. **Conflict**: propose an update to an existing file, then edit that
   same file another way (e.g. through the Nextcloud web UI) before
   confirming. Confirm the agent reports a `conflict` error rather than
   silently overwriting your other edit.
6. **Expiry**: propose a write and wait past the ~10 minute window
   (`NEXTCLOUD_PENDING_WRITE_TTL_SECONDS`) before confirming. Confirm a
   clear `expired` error, not a silent no-op.
7. Confirm a folder delete on a non-empty folder is refused, and that an
   empty folder delete and a folder create (`mkcol`, propose with no
   content) both work.

None of this needs privileged read-only diagnostics beyond what
`deploy-nextcloud-tools.md` already establishes is safe — `docker exec`
into the service/agent containers, `occ` inside `nextcloud-aio-nextcloud`,
or asking the agent itself. Do not paste file content into documentation
or chat transcripts; describe outcomes ("the test file appeared with the
expected content"), not content itself.

## 5. Roll back

Writes share the same service and enable flags as reads — there is no
separate `write`-specific flag to flip off. To remove write capability
without touching reads, redeploy an earlier revision of
`nextcloud_tools_service.py`/`nextcloud_tools.py`/`agent.py` (before ADR
0011), or narrow the share back to read-only in the Nextcloud UI, which
makes every write fail at the WebDAV layer regardless of what the code
allows. To remove Nextcloud access entirely, use
[Deploy Read-Only Nextcloud Tools](deploy-nextcloud-tools.md)'s existing
rollback (`disable-nextcloud-tools.yml`).
