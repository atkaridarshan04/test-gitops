# my-platform-GitOps

The **GitOps repo**: ArgoCD watches it and applies everything under `apps/`
automatically (`prune` + `selfHeal` on - direct `kubectl` edits get
reverted).

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
   Secret instead of a value committed here, and won't sync without it:

   ```bash
   kubectl create namespace monitoring
   kubectl create secret generic grafana-admin-credentials \
     --namespace monitoring \
     --from-literal=admin-user=admin \
     --from-literal=admin-password='<your password>'
   ```

1. Apply `root-application.yaml` to point ArgoCD at this repo:

   ```bash
   kubectl apply -n argocd -f root-application.yaml
   ```

## Layout

```
apps/
├── monitoring/
│   ├── application.yaml     # kube-prometheus-stack (Prometheus, Grafana, Alertmanager) - sync-wave "0"
│   └── values.yaml          # its Helm values, referenced by application.yaml via "$values"
└── app/
    └── application.yaml     # points at https://github.com/your-org/my-app.git - sync-wave "1", after the platform's ready
```

## Reaching Grafana
```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Open `http://localhost:3000`.

## Not yet in this repo

An ArgoCD `ServiceMonitor` (so Prometheus scrapes ArgoCD's own metrics) is a
deliberate follow-up, added once the monitoring stack above is confirmed
working - not bundled into the same untested change.

## Onboarded app

`apps/app/application.yaml` points ArgoCD at
`https://github.com/your-org/my-app.git` (path `.`, revision
`main`) - a separate repo this tool doesn't generate.
Nothing extra to run for the app itself: it's already in this repo's
`apps/` tree, so the root Application picks it up the same automatic way
it picks up everything else here, once you've pushed and applied
`root-application.yaml` (Setup above). That other repo's own
docs cover deploying into it and reaching it once synced - check sync
status here:

```bash
kubectl get application app -n argocd
```

## How this works

Each addon's Helm values live in their own `values.yaml`, not inlined in
the `Application` - resolved via ArgoCD's multi-source `$values` pattern,
where a second source points back at this same repo (`repo_url`/
`repo_revision`). The root Application's `directory.recurse: true` lets
`apps/<name>/` use subfolders; `values.yaml` files are excluded from that
sync since they're plain Helm values, not k8s manifests.
