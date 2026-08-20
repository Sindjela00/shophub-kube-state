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

`DISCORD_GUILD_ID` here is a *default* guild, not the only one — it's what shophub-app's own
platform alert channel (`shop-operator/values.yaml`'s `discord.platformChannelName`) uses, and
what any shop that hasn't attached its own server yet falls back to. A shop owner can invite the
bot to their own server and attach it there instead (shophub-app's per-shop Discord onboarding
flow) without touching this secret. For the invite-link flow to actually produce a usable link,
`shophub/values.yaml`'s `discord.clientId` also needs to be set to that same bot application's
real (public, not secret) client id.

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

Separately, `clusters/local/shop-operator/values.yaml` already sets `shopImage:
shophub-shop:local` — that's the image the operator puts on every *Shop* Deployment it
reconciles (not its own image, see above), and it needs a matching locally built
`shophub-shop:local` image (`docker build` in that repo, same `imagePullPolicy: Never`
reasoning) or every Shop sits in `ImagePullBackOff` too.

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

## 5. Scoped Grafana access per user

Only needed if you installed with the observability stack on (dropped
`--set kube-prometheus-stack.enabled=false` from step 4). Grafana's `[auth.proxy]` (see
`clusters/local/shophub/values.yaml`'s comments) needs an org dedicated to ShopHub end users
and a service account scoped to it — neither can be created declaratively (Grafana has no API
for "provision this at install time" the way a chart value can), so this is a one-time manual
step per cluster, same spirit as the database/JWT secrets in step 2.

This org's datasources (Prometheus + Alertmanager, so per-shop dashboard panels can actually
query something) are provisioned here too, via the same API calls, rather than as a static
`extraManifests` ConfigMap — see `clusters/local/shophub/values.yaml`'s `extraManifests`
comment for why a ConfigMap live from Grafana's first boot, targeting an org that doesn't
exist until this very step runs, deadlocks Grafana entirely.

```bash
# The chart sets a random admin password unless overridden — read the real one out.
GRAFANA_ADMIN_PASSWORD=$(kubectl get secret shophub-grafana -n shophub \
  -o jsonpath='{.data.admin-password}' | base64 -d)

kubectl port-forward -n shophub svc/shophub-grafana 3000:80 &

# Create the org kube-prometheus-stack.grafana.grafana.ini's [auth.proxy] users land in
# (must match grafana.usersOrgName in clusters/local/shophub/values.yaml, default
# "ShopHub Users") and a service account scoped to it.
ORG_ID=$(curl -s -u "admin:$GRAFANA_ADMIN_PASSWORD" -X POST http://localhost:3000/api/orgs \
  -H "Content-Type: application/json" -d '{"name":"ShopHub Users"}' | jq -r .orgId)
curl -s -u "admin:$GRAFANA_ADMIN_PASSWORD" -X POST "http://localhost:3000/api/user/using/$ORG_ID"

# Datasources for this org — same uids/URLs shophub-app's per-shop dashboard JSON already
# expects (datasource uid: "prometheus" / "alertmanager"), just scoped to org 2 instead of
# the default org kube-prometheus-stack's own ConfigMap already provisions into.
curl -s -u "admin:$GRAFANA_ADMIN_PASSWORD" -X POST http://localhost:3000/api/datasources \
  -H "Content-Type: application/json" -d '{
    "name": "Prometheus", "type": "prometheus", "uid": "prometheus",
    "url": "http://shophub-kube-prometheus-st-prometheus.shophub:9090/",
    "access": "proxy", "isDefault": true,
    "jsonData": {"httpMethod": "POST", "timeInterval": "30s"}
  }'
curl -s -u "admin:$GRAFANA_ADMIN_PASSWORD" -X POST http://localhost:3000/api/datasources \
  -H "Content-Type: application/json" -d '{
    "name": "Alertmanager", "type": "alertmanager", "uid": "alertmanager",
    "url": "http://shophub-kube-prometheus-st-alertmanager.shophub:9093/",
    "access": "proxy",
    "jsonData": {"handleGrafanaManagedAlerts": false, "implementation": "prometheus"}
  }'

SA_ID=$(curl -s -u "admin:$GRAFANA_ADMIN_PASSWORD" -X POST http://localhost:3000/api/serviceaccounts \
  -H "Content-Type: application/json" \
  -d '{"name":"shophub-backend","role":"Admin"}' | jq -r .id)
# Admin org role, not Editor: GrafanaProvisioningService's folder permission grants replace a
# folder's whole ACL down to just the shop's owner, which also revokes the *creating*
# service account's own implicit access to it — Editor can no longer delete what it created
# once that happens. Org Admin bypasses per-folder ACLs entirely, so deprovisioning still
# works. (Confirmed by hitting exactly this as a real 403 with an Editor-role SA.)
SA_TOKEN=$(curl -s -u "admin:$GRAFANA_ADMIN_PASSWORD" \
  -X POST "http://localhost:3000/api/serviceaccounts/$SA_ID/tokens" \
  -H "Content-Type: application/json" -d '{"name":"shophub-backend-token"}' | jq -r .key)

kubectl create secret generic shophub-grafana-provisioning -n shophub \
  --from-literal=AdminUser=admin \
  --from-literal=AdminPassword="$GRAFANA_ADMIN_PASSWORD" \
  --from-literal=ServiceAccountToken="$SA_TOKEN"

# shophub-app's pod needed this secret to exist before it could start (Grafana__* env vars
# are wired via secretKeyRef) — it's likely sitting in CreateContainerConfigError until now.
kubectl rollout restart deployment/shophub -n shophub
```

Verify: register/log in through the app, create a shop site, then `GET
/api/shop-sites/{id}/dashboard-link` (authenticated) and open the URL it returns — expect a
real Grafana dashboard for that shop, scoped to just its own folder. See
`clusters/local/shophub/values.yaml`'s `grafana.ini` comments for what was actually checked
end-to-end doing this for real (including two real bugs it caught).

## 6. Tear down

```bash
helm uninstall shophub -n shophub
helm uninstall shop-operator -n shops
# CRDs survive helm uninstall on purpose (helm.sh/resource-policy: keep) — remove explicitly
# if you actually want them gone:
kubectl delete crd shops.apps.shophub.io wallets.apps.shophub.io discordchannels.apps.shophub.io
kubectl delete namespace shops shophub
```
