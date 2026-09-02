# Runbook: istio

Day-2 operations for the Istio service mesh. Initial setup (installing
`istio-base`/`istiod`) is covered in the root README's "Setup" section -
this is what you reach for once the mesh is already synced.

## Check mesh health

```bash
kubectl get application istio-base istiod -n argocd
kubectl get pods -n istio-system
```

## Check sidecar injection status

Injection is opt-in per namespace (`istio-injection: enabled` label) -
nothing in this repo is auto-enrolled:

```bash
kubectl get namespace -L istio-injection
kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.containers[*].name}'
```

Confirm `istio-proxy` appears alongside the app's own container(s). A
namespace labeled after its pods were already running still needs a
rollout restart - injection only happens at admission time:

```bash
kubectl rollout restart deployment -n <namespace>
```

## Move to STRICT mTLS

`PERMISSIVE` is the default here - both plaintext and mTLS traffic are
accepted, so enabling Istio doesn't silently break traffic to any namespace
that isn't sidecar-injected yet. Once every relevant namespace is confirmed
injected (see above), either regenerate this repo with
`istio_mtls_mode=STRICT`, or edit `apps/istio/peer-authentication.yaml`'s
`spec.mtls.mode` from `PERMISSIVE` to `STRICT` directly and let ArgoCD sync
it - no other resource changes are needed. See
`docs/adr/0005-istio-strict-mtls.md` in the platform-generator repo this was
generated from for the full trade-off and migration path.

## Sync/self-heal failing on istio-base

Confirm `RespectIgnoreDifferences=true` is still set alongside
`ignoreDifferences` on the `istio-base` and `istiod` `Application`s -
without it, sync re-applies the webhook fields `istiod` owns at runtime,
and every self-heal retry fails outright:

```bash
kubectl get application istio-base istiod -n argocd -o jsonpath='{.spec.syncPolicy.syncOptions}'
```
