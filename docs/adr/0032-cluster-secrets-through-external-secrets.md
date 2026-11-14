# ADR 0032: Cluster Secrets through the External Secrets Operator, authenticated by OIDC federation

- Status: Proposed
- Date: 2026-11-14
- Deployment status: not deployed

## Context

Since 2026-11-14 Argo CD deploys every k3s workload from `k3s-apps`'
`argocd/` (decisions: "Argo CD trial", ADR 0031). Terraform in
`k3s-apps` is down to the bridge from AWS Secrets Manager into the
cluster: 22 Kubernetes Secrets, four `random_*` values stored only in
its state (Grafana's admin password, Paperless' secret key and Postgres
password, Bulwark's session secret), and the `checksum/*` Pod-template
annotations that restart a Pod after a rotation. Julian wants
`k3s-apps` to end up without Terraform, so it needs no CI apply, no AWS
role and no `k3s-apps-ci` ServiceAccount any more.

Today secret values reach the cluster only through a Terraform plan or
apply: the `k3s-apps` CI role or Julian's session reads Secrets Manager,
and the values also land in the encrypted S3 state. Nothing in the
cluster can read Secrets Manager.

## Decision

### 1. External Secrets Operator, with Reloader for restarts

ESO (pinned chart and image, installed by `k3s-bootstrap`) turns an
`ExternalSecret` (key names, no values) into a Kubernetes Secret from
Secrets Manager and keeps it in sync. Composite secrets (Authelia's
`configuration.yml`, Grafana's `grafana.ini`, Alertmanager's config)
become ESO templates, with the non-secret parts in Git. Reloader
restarts a Deployment when a Secret it references changes, replacing
Terraform's `checksum/*` annotations; it runs with
`reloadStrategy: annotations` (Argo CD ignores fields its manifests
leave out) and only acts on Deployments that opt in by annotation.

Rejected: keeping Terraform for Secrets (the current state, which this
ADR exists to end); SOPS or Sealed Secrets in Git (encrypted values in
Git again, after the 2026-09-09 move to Secrets Manager); copying values
by hand (no rotation path, no audit).

### 2. ESO authenticates to AWS through OIDC federation, never static keys

Static access keys for a cluster identity break the workspace's rule of
OIDC over static keys, so the cluster becomes an OIDC identity provider
for AWS, like GitHub Actions already is:

- **Issuer.** k3s gets a public issuer URL as its first
  `--service-account-issuer`, so new ServiceAccount tokens carry it, and
  keeps `https://kubernetes.default.svc.cluster.local` as a second one,
  so every existing token (Argo CD, runners, every Pod) stays valid.
  `--service-account-jwks-uri` points at the public JWKS. Set in
  `home-infra`'s `k3s_node` role.
- **Discovery documents** (`/.well-known/openid-configuration` and the
  JWKS, public key material only) are served from a private S3 bucket
  behind CloudFront with Origin Access Control: the account-wide S3
  Block Public Access stays on, and CloudFront gives the valid TLS
  certificate AWS requires. The issuer is the distribution's own
  `*.cloudfront.net` name (no DNS or certificate to manage), protected
  with `prevent_destroy`, since a new distribution would mean a new
  issuer everywhere. The JWKS is committed as a file (public key) and
  refreshed by hand if k3s' service-account signing key ever changes.
- **AWS side.** An IAM OIDC provider for that issuer, and one role per
  namespace (`apps`, `auth`, `monitoring`). Each role's trust allows
  exactly `system:serviceaccount:<namespace>:external-secrets` with
  audience `sts.amazonaws.com`; its policy allows only
  `secretsmanager:GetSecretValue` on the named secret groups that
  namespace's apps use, nothing else.

### 3. The trust root lives in `terraform-state`, applied locally

The bucket, distribution, OIDC provider and the three roles define who
may assume AWS identities, so they belong with the account baseline and
the `julian` operator in `terraform-state`: local apply only, the
highest bar in the workspace. Rejected: `k3s-bootstrap` (its
`k3s-bootstrap-local` identity would need IAM write, i.e. power to grant
itself anything) and a CI-applied repo (a plan/apply role must never
manage trust).

### 4. SecretStores from `k3s-bootstrap`, ExternalSecrets from Argo CD

A `SecretStore` binds a namespace's `external-secrets` ServiceAccount to
its IAM role, which is privilege-defining, so `k3s-bootstrap` creates
the ServiceAccounts and SecretStores (one per namespace, no
`ClusterSecretStore`). The `ExternalSecret`s sit next to each app in
`k3s-apps`' `argocd/<app>/`, and Argo CD's Role gains them by name like
every other object. Whoever can create an `ExternalSecret` in a
namespace can pull any secret that namespace's role allows into it;
that is the same set the namespace's Pods consume today, and the
reviewed merge to `main` stays the only way in.

### 5. Migration, one app at a time

The four Terraform-only random values are first copied into Secrets
Manager once (Paperless' Postgres password must match the database;
the others could be regenerated, but copying avoids logouts), read from
the live Kubernetes Secret and written without printing them. Then per
app: its `ExternalSecret` takes over the existing Secret, the Deployment
gets Reloader's annotation, and `k3s-apps` drops the Secret and its
`kubernetes_annotations` behind `removed { destroy = false }`. A cheap
app goes first (Bulwark) to prove that ESO adopts an existing Secret
without a gap; Authelia goes last. When the last Secret is gone,
`k3s-apps`' Terraform, CI workflows, AWS role (repo-infra),
`k3s-apps-ci` ServiceAccount and Roles, its runner's cluster egress and
the leftover Stalwart provider all go.

## Consequences

- **The cluster can read Secrets Manager.** Each namespace's role reads
  exactly the groups its Pods already hold as Kubernetes Secrets, so
  taking over the cluster yields the same values as today, plus their
  future rotations.
- **k3s' service-account signing key becomes an AWS credential.** Today
  it only matters inside the cluster; with federation, whoever has it
  can mint tokens that AWS accepts for the three roles from anywhere.
  It stays on `k3s-node-1`'s disk (as root-owned as today); the roles'
  narrow read-only scope bounds the damage. An `aws:SourceIp` condition
  isn't practical with the dynamic home IP.
- **Values leave Terraform state.** The S3 state no longer holds the
  cluster's secrets.
- **Rotation gets simpler.** Write the new value to Secrets Manager; ESO
  picks it up within its refresh interval (or at once on request), and
  Reloader restarts the Pod. The per-app `terraform apply` step goes,
  and so does `rotate-oidc-client-secret.sh`'s apply.
- **Two new controllers** (ESO and Reloader, together a few hundred MiB),
  pinned and Renovate-tracked; Reloader reads every Secret in the three
  namespaces, which only Pods and Terraform could until now.
- **A rebuilt cluster** depends on the federation: k3s must come up with
  the public issuer before ESO can fetch anything.

## Open points for review

- Issuer on the default `*.cloudfront.net` name (proposed) or a
  dedicated subdomain with its own certificate.
- Refresh interval for ExternalSecrets (proposed: 1 h, plus a forced
  refresh after every rotation).
