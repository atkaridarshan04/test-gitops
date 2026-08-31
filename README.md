# my-platform-GitOps

The **GitOps repo**: ArgoCD watches it and applies everything in this repo
automatically (`prune` + `selfHeal` on - direct `kubectl` edits get
reverted). `argocd/` holds the `Application` objects; `apps/` holds what
each one deploys (Helm values, Kargo's own CRDs).

## Contents

- [Setup](#setup)
- [Layout](#layout)
- [Reaching Grafana](#reaching-grafana)
- [Reaching Kargo](#reaching-kargo)
- [Reaching Kiali](#reaching-kiali)
- [Not yet in this repo](#not-yet-in-this-repo)
- [Promotion pipeline](#promotion-pipeline)
- [How this works](#how-this-works)

## Setup

1. Push this repo to `https://github.com/your-org/my-platform-gitops.git` (already set).
1. Cluster + ArgoCD - needed before the next steps, since they need
   `kubectl` access:

   ```bash
   kind create cluster --name my-platform

   helm install argocd argo-cd --repo https://argoproj.github.io/argo-helm --namespace argocd --create-namespace
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d   # admin password
   kubectl port-forward -n argocd svc/argocd-server 8080:443   # ArgoCD UI at https://localhost:8080, user "admin"
   ```
1. Create Grafana's admin Secret - it reads credentials from an existing
   Secret instead of a value committed here:

   > [!IMPORTANT]
   > `monitoring` won't sync until this Secret exists.

   ```bash
   kubectl create namespace monitoring
   kubectl create secret generic grafana-admin-credentials \
     --namespace monitoring \
     --from-literal=admin-user=admin \
     --from-literal=admin-password='<your password>'
   ```

1. Install [cert-manager](https://cert-manager.io/) - Kargo's internal
   admission webhooks server needs it, not installed by this tool yet:

   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.1/cert-manager.yaml
   ```

1. Create Kargo's admin Secret - it reads credentials from an existing
   Secret instead of a value committed here:

   > [!IMPORTANT]
   > `kargo` won't sync until this Secret exists.

   ```bash
   hash=$(htpasswd -nbBC 10 "" '<your password>' | tr -d ':\n')   # bcrypt hash - htpasswd is in apache2-utils/httpd-tools
   key=$(openssl rand -base64 48 | tr -d "=+/" | head -c 32)      # random signing key

   kubectl create namespace kargo
   kubectl create secret generic kargo-admin \
     --namespace kargo \
     --from-literal=ADMIN_ACCOUNT_PASSWORD_HASH="$hash" \
     --from-literal=ADMIN_ACCOUNT_TOKEN_SIGNING_KEY="$key"
   ```

1. Create Kargo's git credentials Secret by hand - Kargo discovers it via
   the `kargo.akuity.io/cred-type: git` label, not a name reference
   (creating the `app` namespace here too, since the `Project`
   resource that would otherwise create it hasn't synced yet at this
   point - labeled `kargo.akuity.io/project: "true"` so the `Project`
   accepts it as its own instead of rejecting it as a pre-existing,
   unrelated namespace once it does sync):

   ```bash
   kubectl create namespace app --dry-run=client -o yaml | kubectl label --local -f - kargo.akuity.io/project=true -o yaml | kubectl apply -f -
   kubectl apply -f - <<EOF
   apiVersion: v1
   kind: Secret
   metadata:
     name: app-kargo-git-creds
     namespace: app
     labels:
       kargo.akuity.io/cred-type: git
   stringData:
     repoURL: https://github.com/your-org/my-app.git
     username: git
     password: <a GitHub PAT with repo write access>
   EOF
   ```

1. Apply `root-application.yaml` to point ArgoCD at this repo:

   ```bash
   kubectl apply -n argocd -f root-application.yaml
   ```

## Layout

```text
argocd/
└── applications/
    ├── istio-base.yaml             # Application: Istio CRDs - sync-wave "-1"
    ├── istiod.yaml                # Application: Istio control plane - sync-wave "0"
    ├── monitoring.yaml            # Application: kube-prometheus-stack - sync-wave "0"
    ├── kargo.yaml                  # Application: Kargo controller - sync-wave "0"
    ├── app-dev.yaml     # Application: overlays/dev - sync-wave "1"
    ├── app-staging.yaml # Application: overlays/staging - sync-wave "1"
    ├── app-prod.yaml    # Application: overlays/prod - sync-wave "1"
    └── kiali.yaml                 # Application: mesh visualization - sync-wave "1"

apps/
├── istio/
│   └── peer-authentication.yaml # mesh-wide mTLS: PERMISSIVE - sync-wave "1"
├── monitoring/
│   └── values.yaml          # Helm values, referenced by argocd/applications/monitoring.yaml via "$values"
├── kargo/
│   ├── values.yaml                 # Helm values, referenced by argocd/applications/kargo.yaml via "$values"
│   ├── project.yaml                # Kargo Project for app - sync-wave "1"
│   ├── project-config.yaml         # promotion policy - dev/staging auto, prod manual
│   ├── warehouse.yaml              # watches kargo_image_repo_url for new tags
│   ├── stage-dev.yaml
│   ├── stage-staging.yaml
│   ├── stage-prod.yaml
│   └── promotion-task.yaml         # shared promotion steps, reused by all three Stages
└── kiali/
    └── values.yaml   # Helm values, referenced by argocd/applications/kiali.yaml via "$values"

docs/
└── runbooks/
    ├── monitoring.md          # status checks, secret rotation, troubleshooting
    └── kargo.md               # promotion status, approvals, stuck-promotion recovery, secret rotation
```

## Reaching Grafana
```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Open `http://localhost:3000`.

Day-2 operations (secret rotation, no-data troubleshooting, resizing
storage) are in `docs/runbooks/monitoring.md`.

## Reaching Kargo
```bash
kubectl port-forward -n kargo svc/kargo-api 8082:80
```

Open `http://localhost:8082`.

Log in with the admin account (`kargo_admin_secret_name`'s password);
day-2 operations (stuck promotions, approvals, secret rotation) are in
`docs/runbooks/kargo.md`.

## Reaching Kiali

```bash
kubectl port-forward -n istio-system svc/kiali 20001:20001
```

Open `http://localhost:20001` - no login (`auth.strategy: anonymous`, see
"Not yet in this repo").

## Not yet in this repo
- An ArgoCD `ServiceMonitor` (so Prometheus scrapes ArgoCD's own metrics) -
  deliberate follow-up, added once the monitoring stack above is confirmed
  working.
- Kargo OIDC/Dex login and ECR/GAR/ACR ambient-credential Pod Identity
  support - admin-account login only for now. Only one tracked app's
  promotion pipeline is generated per repo (same limitation as
  `app_repo_url` itself, not new here).
- Mesh-wide mTLS in `STRICT` mode - `PERMISSIVE` (both mTLS and plaintext
  accepted) is the scope for this first pass, so enabling Istio doesn't
  silently break traffic to any namespace that isn't sidecar-injected yet.
  Sidecar injection itself is opt-in per namespace (`istio-injection:
  enabled` label) - nothing in this repo is auto-enrolled.
- Kiali's own login (`auth.strategy: anonymous` for now) and an Ingress
  for its UI - `kubectl port-forward` only, same first-pass scope as
  Kargo's admin-account-only login above.

## Promotion pipeline

`apps/kargo/` includes a Kargo `Project`/`Warehouse`/three `Stage`s/one
shared `PromotionTask` for `app`, watching
`ghcr.io/your-org/my-app` for new image tags and promoting
`./overlays/{dev,staging,prod}` in `https://github.com/your-org/my-app.git`
between them (dev/staging auto, prod needs a human to merge the PR Kargo
opens - see "Not yet in this repo"). `argocd/applications/` includes the
`app-dev`/`-staging`/`-prod` `Application`s, pre-annotated so
Kargo is authorized to update them (see Layout above) - you still need to
build the `overlays/{dev,staging,prod}` directories themselves in
`https://github.com/your-org/my-app.git`; `examples/kustomize/` is a worked example of that
layout. Once you've pushed it, check status:

```bash
kubectl get application app-dev app-staging app-prod -n argocd
kubectl get stage -n app
```

Day-2 operations (checking a stuck promotion, approving prod, rotating
Kargo's secrets) are in `docs/runbooks/kargo.md`.

## How this works

`argocd/` holds the `Application` objects; `apps/<name>/` holds what each
one deploys - Helm values, and for Kargo, the `Project`/`Stage`/`Warehouse`/
`PromotionTask` CRDs it doesn't apply itself. Each addon's Helm values live
in their own `values.yaml`, not inlined in the `Application` - resolved via
ArgoCD's multi-source `$values` pattern, where a second source points back
at this same repo (`repo_url`/`repo_revision`). The root Application's
`directory.recurse: true` (path `.`) picks up both trees automatically;
`values.yaml` files are excluded from that sync since they're plain Helm
values, not k8s manifests.
