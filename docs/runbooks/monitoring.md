# Runbook: monitoring

Day-2 operations for kube-prometheus-stack. Initial setup (creating the
admin Secret) is covered in the root README's "Setup" section - this is
what you reach for once it's already synced.

## Check sync/health status

```bash
kubectl get application monitoring -n argocd
kubectl get pods -n monitoring
```

## Reach Grafana

No ALB/DNS on `deployment_target: local` - always port-forward:

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

## Rotate the Grafana GitHub OAuth credentials

```bash
kubectl -n monitoring patch secret grafana-admin-credentials -p '{"stringData": {
  "github-client-id": "<new OAuth App client ID>",
  "github-client-secret": "<new OAuth App client secret>"
}}'
```

Restart the Grafana pod to pick it up:

```bash
kubectl rollout restart deploy/monitoring-grafana -n monitoring
```

To change the allowed org
(`grafana_github_sso_org: atkaridarshan04`), update the
Copier answer and re-render this repo instead - it's baked into
`apps/monitoring/values.yaml`'s `grafana.ini`, not read from the Secret.

## Rotate the Grafana admin password

```bash
kubectl create secret generic grafana-admin-credentials \
  --namespace monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='<new password>' \
  --from-literal=github-client-id='<current OAuth App client ID>' \
  --from-literal=github-client-secret='<current OAuth App client secret>' \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deploy/monitoring-grafana -n monitoring
```

## No data in dashboards

```bash
kubectl get pods -n monitoring   # confirm prometheus/grafana/alertmanager are all Running
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus --tail=100
```

Most common cause on a fresh cluster: PVC still `Pending` - check
`kubectl get pvc -n monitoring` against `storage_class_name` actually
existing (`kubectl get storageclass`).

## No ArgoCD metrics

ArgoCD's `server`/`repoServer`/`controller` components expose `/metrics`
unconditionally (set in the infra repo's Terraform, not here), and this
repo's `apps/monitoring/servicemonitor-argocd-*.yaml` `ServiceMonitor`s
pick them up. The `notifications` `ServiceMonitor` is also always present
but has no targets - `notifications.enabled` stays `false` in the infra
repo by default.

```bash
kubectl get servicemonitor -n argocd
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090   # then check Status > Targets in the UI
```

A `ServiceMonitor` with no matching targets usually means the `release:
monitoring` label (which Prometheus's `serviceMonitorSelector` requires)
is missing, or the ArgoCD Helm release isn't actually named `argocd` -
`app.kubernetes.io/instance` on the target Service must match.

## Resize storage

`prometheus_storage_size`/`grafana_storage_size` only apply on first
create - resizing an existing PVC needs a direct edit (only if
`storage_class_name` has `allowVolumeExpansion: true` -
`enable_volume_expansion` on the infra side):

```bash
kubectl patch pvc <pvc-name> -n monitoring -p '{"spec":{"resources":{"requests":{"storage":"<new size>"}}}}'
```
