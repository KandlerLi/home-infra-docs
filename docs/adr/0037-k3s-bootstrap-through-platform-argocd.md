# ADR 0037: k3s-bootstrap as GitOps manifests, synced by a platform Argo CD

- Status: Accepted (2026-11-27)
- Date: 2026-11-25
- Deployment status: all stages live since 2026-11-28 (tracker #80).
  Stage 0 (`k3s-bootstrap#105`, `#107`, `#108`, `k3s-apps#254`, `#255`):
  headless install, then the LAN-only UI with OIDC and ntfy notifications,
  tested end to end with the canary ConfigMap. Then `k3s-bootstrap#109`–`#120`:
  namespaces, RBAC, Reloader and ESO, ARC, node-exporter, PVs and the apps
  Argo CD; `#121`/`#122` moved the platform OIDC secret out of Helm values
  and dropped the applied migration blocks. Deviation from §6: no new ESO
  roles for `arc-system` and `argocd`; the GitHub App Secret, both OIDC
  Secrets and ESO's ServiceAccounts (their role ARN holds the account ID)
  stay a local Terraform apply instead. Supersedes the "local-only" rule
  for `k3s-bootstrap` except for that remainder, and narrows ADR 0031's
  "no ClusterRole" to the apps instance

## Context

The cluster has two layers, run like two teams:

- **`k3s-apps`** (the app team): manifests that Argo CD syncs, Secrets
  through ESO (ADR 0032). This is the usual GitOps setup and works well.
- **`k3s-bootstrap`** (the platform team): Terraform, applied by hand
  from the laptop with the node's admin kubeconfig. It holds everything
  that defines privilege: namespaces and PSA labels, every
  ServiceAccount, Role and binding, the cluster-scoped PersistentVolumes,
  ARC with the GitHub App key, and Argo CD, ESO and Reloader themselves.

The second layer is rare in practice. In setups with a platform team and
app teams, Terraform usually only builds the cluster and installs Argo CD
once. After that, a platform Argo CD syncs the cluster-wide layer from Git,
including itself, and app teams get a narrower slice. Git, with branch
protection and review, is the trust root, not an admin machine.

Here the local apply also creates real toil: since 2026-08-25, 37 commits
changed `argocd.tf` and 18 changed `storage.tf`. Most of them followed an
app change in `k3s-apps`, each needing a second PR and a local
`scripts/update.sh`.

Today's rule (owner's `check-security-invariants`): `k3s-bootstrap` runs
only from Julian's trusted local machine, and Argo CD has no ClusterRole.
This ADR proposes changing both.

## Decision

### 1. Two Argo CD instances

- **Platform Argo CD**, new, in namespace `argocd-platform`. It syncs
  only `main` of `k3s-bootstrap` and may write cluster-wide (in practice
  cluster-admin, because it creates RBAC and PVs). It also manages the
  apps instance, and itself once bootstrapped.
- **Apps Argo CD**, today's instance in `argocd`. It stays exactly as
  ADR 0031 and `argocd.tf` define it: Roles only, named objects, no
  Secrets, only `main` of `k3s-apps`. Its Roles become manifests in
  `k3s-bootstrap` instead of Terraform.

Rejected: **one instance with AppProjects** (the other common
pattern). Its controller would need cluster-admin, so the boundary for
`k3s-apps` would only be an Argo CD setting, not Kubernetes RBAC. Today
a push to `k3s-apps` can't reach RBAC, PVs or Secrets even if Argo CD's
own checks failed, and that should stay true.

### 2. What stays in Terraform

A small local-only stage 0 in `k3s-bootstrap`: namespace
`argocd-platform`, the platform Argo CD Helm release, its repository
credential, and one root Application that points at the repo's
manifests. It is applied again only to recover from a lost cluster or a
broken platform Argo CD. IAM stays in `terraform-state`, local only
(ADR 0032, unchanged).

### 3. Repository layout after the move

Plain YAML, Kustomize and Helm values, one Application per component
(app of apps):

- `namespaces/`: namespaces and PSA labels
- `rbac/`: every ServiceAccount, Role and binding, including the apps
  Argo CD's and Headlamp's
- `storage/`: the static PVs, keeping `Retain` and each `claimRef`
- `eso/`: ESO and Reloader per namespace, their SecretStores and
  ServiceAccounts (ADR 0032's model, unchanged)
- `arc/`: the ARC controller, plus one runner scale set per repository
  from an ApplicationSet that reads `repositories.json` (replaces the
  module's `for_each`)
- `argocd/`: the apps Argo CD (Helm values) and the platform Argo CD
  itself
- the remaining small RBAC files (kube-state-metrics, Alloy, kubelet
  metrics, stalwart-backup) and node-exporter

### 4. Secrets

Values Terraform reads today from Secrets Manager (the GitHub App key in
`home-infra/github-runner`, Argo CD's OIDC client secret in
`home-infra/argocd`) come through ESO: one more namespace-scoped ESO plus
IAM role each for `arc-system` and `argocd`, in `terraform-state`. These
roles follow ADR 0032: trust only that namespace's `external-secrets`
ServiceAccount, `GetSecretValue` on the named group only. The platform
Argo CD never gets a rule on Secret *contents* beyond what applying
ExternalSecrets needs.

### 5. Guards on the platform instance

Decided 2026-11-27. A taken-over GitHub account alone must not reach the
cluster: a change needs a merge **and** a sync, and the sync is only
possible from the home network, behind Authelia with a second factor.

- **No automated sync, no prune:** every Application on the platform
  instance syncs by hand only. A merge shows up as OutOfSync, and Julian
  syncs it in the UI. That click replaces `scripts/update.sh`.
- **UI only from the home network:** no public DNS record. The hostname
  resolves only through Blocky to the homeserver, and its Traefik route
  carries an IP allowlist for the LAN, which the FRITZ!Box WireGuard
  clients share (ADR 0025; not yet tested over the VPN). The FRITZ!Box's
  DNS rebind protection needs a host exception for the name (manual, see
  current-state). Authelia
  forward-auth (`user:julian`, two-factor) and its own OIDC client
  (`julian_only`). No local admin, Dex, anonymous access, exec or API keys.
- **Notifications:** Argo CD notifications for OutOfSync and for every
  sync started, succeeded or failed go to Julian (channel decided in
  stage 0b), so an unexpected change on `main` or an unexpected sync is
  visible at once.
- **Source:** its AppProject allows only `k3s-bootstrap` at `main`.
  `main` keeps branch protection and the required `trivy-config` and
  `rollout-ack` checks.
- **No commit-signature check:** without a hardware key, a signing key
  would sit on the same laptop as the GitHub session. The sync gate
  above is the control instead (2026-11-27).

### 6. Migration, in stages

Each stage is its own PR: Terraform `removed { destroy = false }`, then
the same object as a manifest, adopted by Argo CD in place. A plan or
diff that shows a create, replace or destroy of a live object is a stop
condition.

1. Stage 0a: platform Argo CD (Terraform, local), headless, with an
   empty root Application. Without a UI nobody can sync, so nothing
   reaches the cluster through it yet.
   Stage 0b: its UI (LAN only, OIDC) and notifications.
2. Namespaces and PSA labels
3. RBAC
4. ESO and Reloader, plus the new ESO roles for `arc-system` and `argocd`
5. ARC and the runner scale sets
6. PVs, only after Nextcloud's backup check (`verify-backups.sh`), with
   every `claimRef`, `Retain` and path checked against the live object
7. The apps Argo CD

## Alternatives

- **Keep local Terraform** (today). The strongest boundary, but not the
  usual pattern, and it costs a second PR plus a local apply for most app
  changes.
- **Loosen the coupling only.** Let the apps Argo CD change any object of
  its kinds in its namespaces instead of named ones, and add a `Retain`
  StorageClass so new apps don't need a PV here. This removes most of the
  toil but keeps the local apply and Terraform for the platform layer.
- **CI applies Terraform.** A GitHub Actions identity with cluster-admin
  on the self-hosted runners. Worse than Argo CD: the credential lives
  where any repo's workflow runs.
- **One Argo CD with AppProjects:** see Decision 1.

## Consequences

- **The trust root moves partly to GitHub.** A change to the platform
  layer needs a merge to `k3s-bootstrap`'s `main` and a sync, which needs
  access to the home network plus Julian's Authelia login with second
  factor (Decision 5). Today it needs the laptop with the node's admin
  kubeconfig, the SSH key and `pass`.
- **Answered 2026-11-27 (tracker #82):** until 2026-11-26, root on
  `k3s-node-2` was cluster-admin. A CI job reached it through the
  privileged `dind` sidecar, and the agents held the server token, while
  the ARC controller's Pod rights in a `privileged` namespace sat on the
  same node. Both paths are closed now: only `stalwart-config` keeps a
  self-hosted runner, without Docker or root under PSA `restricted`, the
  controller runs on node-1, and the agents join with their own agent
  token ([decisions](../decisions.md#ci-on-github-hosted-runners-self-hosted-only-for-the-home-network)).
  So today no GitHub-side path leads to cluster-admin, and the first
  point would add one rather than replace one.
- A second Argo CD to run and update, about 0.5 GiB of memory.
- `check-security-invariants` changes: `k3s-bootstrap` drops out of the
  "local machine only" list (stage 0 stays local), and the Argo CD rule
  splits into the platform and the apps instance.
- `k3s-bootstrap`'s docs, `current-state.md` and the architecture views
  change with each stage.
