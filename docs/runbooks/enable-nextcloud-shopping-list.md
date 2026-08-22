# Runbook: Enable the Nextcloud Shopping List tools

Status: **Implemented, not yet deployed.** See [ADR 0014](../adr/0014-nextcloud-shopping-list-tools.md)
for the design. This is a follow-on to
[Deploy Read-Only Nextcloud Tools](deploy-nextcloud-tools.md) (already
deployed) and reuses the same `nextcloud-tools` service, `home-agent`
integration, and dedicated `home-agent` Nextcloud account/app password —
no new credential. Writes execute immediately, the same as
[Enable Nextcloud Write Tools](enable-nextcloud-writes.md) (ADR 0013).

## 1. Prerequisite: install the app, create a list, share it (manual, not Ansible)

1. If the Shopping List app (`shopping_list`, by otherworld-dev) isn't
   already installed, install it from the Nextcloud Apps admin page
   (`Apps` → search "Shopping List").
2. Create (or use an existing) list in the app as the account that will
   own it.
3. Share that list internally with the dedicated `home-agent` account from
   the app's own share panel, granting **edit** permission (create,
   change, and delete items) — the shopping-list equivalent of the
   `AI Workspace` folder's "Can edit" share widened in
   `enable-nextcloud-writes.md`. This is a manual, in-app action; per ADR
   0008's original reasoning, automating a share grant would need the
   owner's own personal credential, which Ansible never gets.

Nothing will be visible to the agent, and every shopping-list tool call
will fail with "no shopping lists are shared with the agent yet", until
this step is done. If more than one list is ever shared with the account,
the agent will ask which one you mean (or you can name it explicitly) —
no extra configuration is needed for that.

## 2. Run tests and syntax checks

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/python -m unittest discover -s tests -v
.venv/bin/ansible-playbook ansible/playbooks/site.yml --syntax-check
```

## 3. Deploy

Code-only change to the existing `nextcloud-tools` service and
`home-agent`; no new playbook, role, or environment variable. Redeploy
both (the same pair `enable-nextcloud-writes.md` uses, since this is the
same first-time rollout of that whole code path):

```bash
cd /home/julian/projects/infra/home-infra
.venv/bin/ansible-playbook ansible/playbooks/nextcloud-tools.yml \
  --ask-become-pass \
  --extra-vars nextcloud_tools_bootstrap_confirmation=BOOTSTRAP_NEXTCLOUD_TOOLS
.venv/bin/ansible-playbook ansible/playbooks/home-agent.yml --ask-become-pass
```

## 4. Verify live, end to end

Drive this through a real chat with `home-agent` (via Open WebUI or
directly against `/v1/chat`), not just a clean deploy.

1. **Add**: "Put milk on my shopping list." Confirm the agent reports
   success, and independently confirm the item actually appears in the
   Shopping List app (not as a text file anywhere).
2. **Check off**: "Mark milk as bought." Confirm it's checked in the app.
3. **Uncheck**: "Milk isn't actually bought yet." Confirm it's unchecked
   again.
4. **Remove**: "Take milk off the list." Confirm it's gone from the app.
5. **Ambiguous/unknown name**: ask about an item that isn't on the list,
   and (if you share a second list for this test) ask without naming
   which list — confirm the agent gets a clear error back and either asks
   you to clarify or offers the available names, rather than silently
   picking the wrong thing or fabricating success.

This also exercises the live-verification assumption flagged in ADR
0014 (JSON request bodies against the app's OCS API) — if step 1 fails
with an unexpected error, check that first before assuming it's a
permissions problem.

## 5. Roll back

Shopping list tools share the same service and enable flags as the file
tools — there is no separate flag to flip off. To remove shopping-list
capability without touching file reads/writes, redeploy an earlier
revision of `nextcloud_tools_service.py`/`nextcloud_tools.py`/`agent.py`
(before ADR 0014), or unshare the list from the `home-agent` account in
the Nextcloud UI, which makes every shopping-list tool call fail cleanly
regardless of what the code allows. To remove Nextcloud access entirely,
use [Deploy Read-Only Nextcloud Tools](deploy-nextcloud-tools.md)'s
existing rollback (`disable-nextcloud-tools.yml`).
