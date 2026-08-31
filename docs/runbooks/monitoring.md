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

## Rotate the Grafana admin password

```bash
kubectl create secret generic grafana-admin-credentials \
  --namespace monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='<new password>' \
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

## Resize storage

`prometheus_storage_size`/`grafana_storage_size` only apply on first
create - resizing an existing PVC needs a direct edit (only if
`storage_class_name` has `allowVolumeExpansion: true` -
`enable_volume_expansion` on the infra side):

```bash
kubectl patch pvc <pvc-name> -n monitoring -p '{"spec":{"resources":{"requests":{"storage":"<new size>"}}}}'
```
