# Bootstrapping the local cluster

Step-by-step procedure to stand up the `clusters/local` state on a real cluster. Every step
below has actually been run against Docker Desktop's built-in Kubernetes.

## Prerequisites

- A running Kubernetes cluster and a matching `kubectl` context (Docker Desktop: Settings ->
  Kubernetes -> Enable Kubernetes; minikube/kind/k3s work the same way, just with a different
  context name — update `clusters/local/cluster.yaml` to match whichever you use).
- `kubectl` and `helm` (v3) on your PATH.
- This repo and a checkout of `shophub-helm-charts` as sibling directories — the `helm.yaml`
  files under `clusters/local/` reference the chart source by relative path
  (`../../../shophub-helm-charts/...`), since no chart registry is published yet (see the
  README). If your layout differs, adjust those paths or `cd` so they resolve.

```
workspace/
├── shophub-kube-state/       (this repo)
└── shophub-helm-charts/
```

## 1. Verify cluster access

```bash
kubectl config current-context
# should print whatever's in clusters/local/cluster.yaml's kubeContext
kubectl get nodes
```

## 2. Create the secrets the releases reference

Neither release's real secret values are committed to this repo (see each `values.yaml`'s
`existingSecret` comments) — create them first, in the namespaces each release will land in:

```bash
# `kubectl create namespace` (unlike `create secret`) has no built-in --dry-run|apply
# idempotency shortcut people remember, so: create, ignore "already exists".
kubectl create namespace shops 2>/dev/null || true
kubectl create secret generic shop-operator-discord -n shops \
  --from-literal=DISCORD_BOT_TOKEN=<your Discord bot token> \
  --from-literal=DISCORD_GUILD_ID=<your Discord guild/server ID>

kubectl create namespace shophub 2>/dev/null || true
kubectl create secret generic shophub-database -n shophub \
  --from-literal=ConnectionString="Host=<postgres-host>;Port=5432;Database=shophub;Username=<user>;Password=<password>"
kubectl create secret generic shophub-jwt -n shophub \
  --from-literal=SigningKey="$(openssl rand -base64 48)"
```

(Skip the `shop-operator-discord` secret if you don't need Discord alert channels yet — the
operator runs fine without it, `DiscordChannel` reconciliation just won't succeed until it
exists.)

`shophub-database` needs a real reachable Postgres — nothing in this repo stands one up. For
local testing, `docker compose up -d` in `shophub-app` and use
`Host=host.docker.internal;Port=5433;...` so the in-cluster pod can reach a container running
on the host.

## 3. Install shop-operator

Run from this repo's root, alongside a `shophub-helm-charts` checkout (see Prerequisites):

```bash
helm install shop-operator ../shophub-helm-charts/charts/shop-operator \
  --namespace shops \
  -f clusters/local/shop-operator/values.yaml
```

This also installs the `Shop`, `Wallet`, and `DiscordChannel` CRDs (they ship inside the
chart, `helm install` creates them automatically — no separate step).

Verify:

```bash
kubectl get deployment -n shops shop-operator
kubectl get crd shops.apps.shophub.io wallets.apps.shophub.io discordchannels.apps.shophub.io
kubectl logs -n shops -l app.kubernetes.io/name=shop-operator
```

As of this writing, `ghcr.io/sindjela00/shophub-shop-operator` isn't published yet (nothing's
been merged to that repo's `master`, so its `publish-image` CI job has never run) — expect
`ImagePullBackOff` on the Deployment until it is. Everything else in this step (CRDs, RBAC,
the Deployment/Service objects themselves) is real and already verified; only the image pull
is a known, separate gap. Point `--set image.repository=... --set image.tag=...` at a locally
built image (`docker build` in that repo, `imagePullPolicy: Never`) to verify the rest for
real in the meantime — see that repo's own chart PR for exactly this being done.

## 4. Install shophub

```bash
helm install shophub ../shophub-helm-charts/charts/shophub \
  --namespace shophub \
  -f clusters/local/shophub/values.yaml \
  --set kube-prometheus-stack.enabled=false
```

(`kube-prometheus-stack.enabled=false` above keeps a first bootstrap fast — drop that flag
once you also want the observability stack; see the "Observability: deploy metrics stack"
work for what it needs.)

Verify:

```bash
kubectl get deployment,svc -n shophub shophub
kubectl port-forward -n shophub svc/shophub 8080:80
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"bootstrap-check@example.com","password":"TestPassword123"}'
# expect HTTP 201 with a JWT
```

## 5. Tear down

```bash
helm uninstall shophub -n shophub
helm uninstall shop-operator -n shops
# CRDs survive helm uninstall on purpose (helm.sh/resource-policy: keep) — remove explicitly
# if you actually want them gone:
kubectl delete crd shops.apps.shophub.io wallets.apps.shophub.io discordchannels.apps.shophub.io
kubectl delete namespace shops shophub
```
