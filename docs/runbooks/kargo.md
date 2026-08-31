# Runbook: kargo

Day-2 operations for the `app` promotion pipeline. Initial
setup (secrets, creating/annotating the `Application`s) is covered in the
root README's "Setup" section - this is what you reach for once a
pipeline is already running.

## Check promotion status

```bash
kubectl get warehouse -n app
kubectl get stage -n app
kubectl get freight -n app
kubectl get application app-dev app-staging app-prod -n argocd
```

`Stage.status.currentFreight` shows what's actually deployed in that
environment; a `Warehouse` with no new `Freight` past your last release
means nothing new has matched `kargo_image_constraint` yet.

## Approve a prod promotion

Staging's `git-open-pr` step opens a PR against `https://github.com/atkaridarshan04/test-app.git` -
merging it is the approval. Find it:

```bash
kubectl get stage app-prod -n app -o jsonpath='{.status.currentPromotion}'
```

or check `https://github.com/atkaridarshan04/test-app.git`'s open PRs directly. `git-wait-for-pr` picks
up the merge automatically - no Kargo-side action needed after merging.

## A promotion is stuck

```bash
kubectl get promotion -n app
kubectl describe promotion <name> -n app
```

Common causes: the PR was closed without merging (re-run by refreshing
the `Stage`, `kubectl annotate stage <name> -n app
kargo.akuity.io/refresh=$(date +%s) --overwrite`), or `argocd-update`
found no `Application` annotated `kargo.akuity.io/authorized-stage:
app:<env>` - confirm the annotation on
`app-<env>` matches exactly (see `examples/kustomize/` in this
template's own source repo).

## Force a Warehouse refresh

```bash
kubectl annotate warehouse app -n app \
  kargo.akuity.io/refresh=$(date +%s) --overwrite
```

## Rotate Kargo's admin credentials

```bash
kubectl create secret generic kargo-admin \
  --namespace kargo \
  --from-literal=ADMIN_ACCOUNT_PASSWORD_HASH='<new bcrypt hash>' \
  --from-literal=ADMIN_ACCOUNT_TOKEN_SIGNING_KEY='<new signing key>' \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deploy/kargo-api -n kargo
```

## Rotate the git credentials PAT

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-kargo-git-creds
  namespace: app
  labels:
    kargo.akuity.io/cred-type: git
stringData:
  repoURL: https://github.com/atkaridarshan04/test-app.git
  username: git
  password: <new PAT>
EOF
```

## Reach the Kargo UI

```bash
kubectl port-forward -n kargo svc/kargo-api 8081:443
```

No Ingress in this first pass (see root README's "Not yet in this repo") -
port-forward only.
