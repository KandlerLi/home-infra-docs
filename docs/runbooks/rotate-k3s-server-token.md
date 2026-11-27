# Runbook: Rotate the k3s server token

The server token on `k3s-node-1` is the cluster's root credential. It can
join servers, and it is the passphrase that encrypts the bootstrap data
(CA certs and keys) in the datastore. Rotate it when a copy may have left
`k3s-node-1`.

Since 2026-11-26, agents join with their own agent token
(`home-infra`'s `k3s_node` role, `agent-token-file`), so rotating the
server token touches only `k3s-node-1`.

## How `k3s token rotate` works here

Checked against the k3s v1.37.2 source (`pkg/cli/token/token.go`,
`pkg/server/handlers/token.go`, `pkg/cluster/storage.go`):

- Without `--token`, it reads the old token from
  `/var/lib/rancher/k3s/server/token`. Without `--new-token`, it generates one.
- The server then updates the `server` entry in `server/cred/passwd`,
  rewrites `server/token`, and re-encrypts the bootstrap data in the
  datastore (SQLite through kine works).
- It changes the `node` (agent) entry only when the agent token still
  equals the old server token *and* no agent token is configured. Here an
  agent token is configured, so the agents are unaffected.
- On restart, k3s takes the token from `config.yaml` or, as here, from
  `server/cred/passwd`. No `token:` setting is needed.
- At the end it saves the bootstrap data (`cluster.Save(..., override)`),
  so the copy of `cred/passwd` in the datastore matches the file on disk
  again. See "`cred/passwd` newer than datastore" below.
- It touches neither the CA nor the service-account signing key, so
  ServiceAccount tokens, ESO's AWS login (ADR 0032) and kubeconfigs keep
  working.

Rotating the token does not revoke anything taken *with* the old one. If
the old token was actually used to fetch the bootstrap data, the CA keys
are exposed, and only a CA rotation (`k3s certificate rotate-ca`) helps.
That is a separate, much bigger job.

## Backup

There is no k3s datastore backup otherwise: the cluster is rebuilt from
IaC, and app data lives on PVs. So the token is kept nowhere but on
`k3s-node-1`, and the backup below only exists for the rollback. It
contains the old token, the datastore and the CA keys: keep it root-only
on the node and delete it once the rotation is verified.

## Steps

All on 🤖 `k3s-node-1`, reached from 💻 the laptop:

```bash
ssh -t -J julian@192.168.178.100 ansible@192.168.101.10
```

1. Fingerprint the current token, without printing it:

   ```bash
   sudo sha256sum /var/lib/rancher/k3s/server/token | cut -c1-12
   ```

2. Back up while k3s is stopped, so SQLite is consistent. Stopping `k3s`
   leaves running containers alone; only the API is down for a moment.

   ```bash
   sudo systemctl stop k3s
   sudo tar -C /var/lib/rancher/k3s -czf /root/k3s-server-before-token-rotate.tgz server/db server/token server/cred
   sudo chmod 600 /root/k3s-server-before-token-rotate.tgz
   sudo systemctl start k3s
   until sudo k3s kubectl get --raw /readyz >/dev/null 2>&1; do sleep 2; done
   ```

3. Rotate, then restart:

   ```bash
   sudo k3s token rotate
   sudo systemctl restart k3s
   until sudo k3s kubectl get --raw /readyz >/dev/null 2>&1; do sleep 2; done
   ```

4. Check:

   ```bash
   sudo sha256sum /var/lib/rancher/k3s/server/token | cut -c1-12   # must differ from step 1
   sudo k3s kubectl get nodes                                     # all three Ready
   sudo k3s kubectl get secretstores -A                           # ESO stores still Ready
   ```

   From 💻 the laptop, `kubectl get nodes` through the tunnel should work
   unchanged, since the kubeconfig uses client certificates, not the token.

5. Restart `k3s` once more (`sudo systemctl restart k3s`) and wait for
   `/readyz`: a clean second start proves the datastore copy is in sync.

6. After a day without problems, delete the backup on 🤖 `k3s-node-1`:

   ```bash
   sudo rm /root/k3s-server-before-token-rotate.tgz
   ```

## Rollback

If k3s doesn't come back after step 3, on 🤖 `k3s-node-1`:

```bash
sudo systemctl stop k3s
sudo rm -rf /var/lib/rancher/k3s/server/db /var/lib/rancher/k3s/server/cred
sudo tar -C /var/lib/rancher/k3s -xzf /root/k3s-server-before-token-rotate.tgz
sudo systemctl start k3s
```

The fingerprint from step 1 should match again.

## `cred/passwd` newer than datastore

k3s keeps a copy of `server/cred/passwd` in the bootstrap data and
compares it with the file at every start. When k3s itself changes the
file without saving the bootstrap data, the *next* start fails:

```text
level=fatal msg="/var/lib/rancher/k3s/server/cred/passwd newer than datastore and could cause a cluster outage. Remove the file(s) from disk and restart to be recreated from datastore."
```

Seen 2026-11-27: the restart that added `agent-token-file` (2026-11-26)
rewrote the agent entry in `cred/passwd` but not the datastore copy, so
the next stop/start, here the backup in step 2, left the API down until
the fix below. Every reboot of `k3s-node-1` in between would have failed
the same way. Changing the agent token again would do it again.

Fix, on 🤖 `k3s-node-1`: move the file aside (keep it), let k3s recreate
it from the datastore, then save the bootstrap data with a rotation
(steps 3–5). The rotation's save is what makes the next restart clean.

```bash
sudo mv /var/lib/rancher/k3s/server/cred/passwd /root/k3s-passwd-before-recreate
sudo chmod 600 /root/k3s-passwd-before-recreate
sudo systemctl restart k3s
```

Running Pods keep running while the API is down. Nothing gets scheduled,
and Argo CD and ESO stop until k3s is back.
