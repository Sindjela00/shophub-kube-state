# shophub-kube-state

Desired-state configuration for the Kubernetes cluster(s) running ShopHub. Tracks which
Helm chart versions (from `shophub-helm-charts`) are installed, and with what value overrides.

## Structure

```
shophub-kube-state
├── README.md
└── clusters/
    └── local/                    # e.g. minikube / kind / k3s
        ├── cluster.yaml          # cluster metadata
        ├── shop-operator/
        │   ├── helm.yaml         # OCI chart reference + version
        │   └── values.yaml       # value overrides
        ├── shophub/
        │   ├── helm.yaml
        │   └── values.yaml
        └── shophub-discord/
            ├── helm.yaml
            └── values.yaml
```

Additional clusters (e.g. `staging`) would get their own directory under `clusters/`.

**Optional**: this repo could later be wired up to a GitOps tool (ArgoCD or Flux) to
reconcile the cluster automatically from this state.

## Related repositories

- `shophub-helm-charts` — source of the chart versions referenced by `helm.yaml` files here
- `shophub-app`, `shophub-shop`, `shophub-shop-operator` — application/operator source
