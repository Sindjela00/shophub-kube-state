# shophub-kube-state

Desired-state configuration for the Kubernetes cluster(s) running ShopHub. Tracks which
Helm chart versions (from `shophub-helm-charts`) are installed, and with what value overrides.

## Structure

```
shophub-kube-state
├── README.md
├── BOOTSTRAP.md              # step-by-step: stand up the local cluster from this state
└── clusters/
    └── local/                # Docker Desktop Kubernetes (minikube/kind/k3s work identically)
        ├── cluster.yaml      # cluster metadata
        ├── shop-operator/
        │   ├── helm.yaml     # chart reference + version
        │   └── values.yaml   # value overrides
        └── shophub/
            ├── helm.yaml
            └── values.yaml
```

Additional clusters (e.g. `staging`) would get their own directory under `clusters/`.

There's no separate `shophub-discord` release — Discord alert-channel provisioning is a
capability of the `shop-operator` chart itself (its `DiscordChannel` controller, configured via
that release's own `discord.*` values), not a standalone service. An earlier version of this
repo's structure assumed otherwise; that placeholder has been removed.

**Chart source**: `shophub-helm-charts` doesn't publish its charts anywhere yet (no OCI
registry or classic Helm repo has been set up — see that repo's own governance work, which
explicitly left this undecided). Until it does, every `helm.yaml` here points at a relative
local path to a `shophub-helm-charts` checkout next to this repo — see `BOOTSTRAP.md`.

**Optional**: this repo could later be wired up to a GitOps tool (ArgoCD or Flux) to
reconcile the cluster automatically from this state.

## Related repositories

- `shophub-helm-charts` — source of the chart versions referenced by `helm.yaml` files here
- `shophub-app`, `shophub-shop`, `shophub-shop-operator` — application/operator source
