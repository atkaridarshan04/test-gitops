# Runbook: kiali

Day-2 operations for Kiali. Initial setup (installing the chart) is
covered in the root README's "Setup" section - this is what you reach for
once it's already synced.

## Reach the Kiali UI

```bash
kubectl port-forward -n istio-system svc/kiali 20001:20001
```

Open `http://localhost:20001` - no login (`auth.strategy: anonymous`, see
the root README's "Not yet in this repo").

## No mesh data / empty topology

Kiali points at Prometheus/Grafana explicitly (`external_services.*`),
not same-namespace autodetection, since both live in `monitoring`, not
`istio-system` - confirm they're actually up first:

```bash
kubectl get application monitoring -n argocd
kubectl get pods -n monitoring
```

Empty topology with monitoring healthy usually means no sidecar-injected
traffic yet - confirm at least one namespace is labeled
`istio-injection: enabled` and has running pods with an `istio-proxy`
container (see `docs/runbooks/istio.md`).
