# ADR 0037: k3s-bootstrap as GitOps manifests, synced by a platform Argo CD

- Status: Proposed
- Date: 2026-11-25
- Deployment status: not started; would supersede the "local-only"
  rule for `k3s-bootstrap` and narrow ADR 0031's "no ClusterRole" to the
  apps instance

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

- **Sync:** `prune` off everywhere. Auto-sync off for `namespaces/`,
  `rbac/`, `storage/` and `argocd/`: a merge shows up as OutOfSync and
  Julian syncs it in the UI. That click replaces `scripts/update.sh`.
  `arc/` and `eso/` may auto-sync.
- **Source:** its AppProject allows only `k3s-bootstrap` at `main`.
  `main` keeps branch protection and the required `trivy-config` and
  `rollout-ack` checks.
- **Commit signatures:** the AppProject requires GPG-signed commits. A
  squash merge in GitHub's web UI is signed with GitHub's own key, so
  trusting that key helps only against a stolen repository token, not a
  stolen GitHub account. Merging locally with Julian's key would cover
  both, at the cost of the web merge button. To be decided when it is
  implemented.
- **Login:** same as ADR 0031. Authelia forward-auth limited to
  `user:julian`, OIDC (`julian_only`), no local admin, Dex, anonymous
  access, exec or API keys. Its UI gets its own hostname and OIDC client.

### 6. Migration, in stages

Each stage is its own PR: Terraform `removed { destroy = false }`, then
the same object as a manifest, adopted by Argo CD in place. A plan or
diff that shows a create, replace or destroy of a live object is a stop
condition.

1. Stage 0: platform Argo CD, with an empty root Application
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

- **The trust root moves to GitHub.** Whoever can merge to
  `k3s-bootstrap`'s `main` (Julian's GitHub account and session) can get
  cluster-admin once a sync runs. Manual sync for the privileged
  components adds an Authelia login with second factor in between, but
  is not a hard boundary against someone holding both.
- **Open question:** how far today's privileged `dind` runners on
  `k3s-node-2` already reach. Root on node-2 gives its kubelet
  credential, which the Node authorizer restricts, not cluster-admin.
  Check before this is accepted, so the comparison in the first point is
  honest.
- A second Argo CD to run and update, about 0.5 GiB of memory.
- `check-security-invariants` changes: `k3s-bootstrap` drops out of the
  "local machine only" list (stage 0 stays local), and the Argo CD rule
  splits into the platform and the apps instance.
- `k3s-bootstrap`'s docs, `current-state.md` and the architecture views
  change with each stage.
