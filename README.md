# atlas-gitops

GitOps source of truth (same role as `k8s-management`). ArgoCD ApplicationSets watch **`main`**.

Dev only: **`aks-dev/`**.

```
_global.yaml
aks-dev/
  _global.yaml
  _admin/
    _global.yaml      # repo URLs + chart pin
    gitops.yaml
  atlas-shop-dev1/
    _admin.yaml       # namespace.enabled + charts[]
    app.yaml          # catalog-api overlay
```

## Overlay contract

- `_admin.yaml` `charts[].name` must match the values file name: `app` → `app.yaml`
- `charts[].chart` is the folder under `atlas-charts-apps/charts/` (`atlas-shop`)
- Internal LB annotation is on the catalog-api Service (no Istio)

## After you push Git remotes

1. Confirm Git remotes are `https://github.com/Zachery2008/atlas-*.git` (already set in `_global.yaml` and `aks-dev/_admin/_global.yaml`)
2. Build/push `atlas-catalog-api:0.1.0` to `atlas.azurecr.io` (or change `image.registry`)
3. Helm-install `platform-gitops` from `atlas-charts-common` (see that README)
4. Argo creates `ns-atlas-shop-dev1` and `ns-atlas-shop-dev1-app-app`

## Demo flow

Change `catalog-api.image.tag` or `ROOT_LOG_LEVEL` in `app.yaml` and merge to `main`. Argo syncs. No `kubectl apply`.
