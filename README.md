# Atlas GitOps demo

This repository is the **GitOps source of truth**. Argo CD watches `main` here and
reconciles a single **Dev** cluster (`aks-dev`). There is **no Istio**. The app is
exposed with an **Azure internal LoadBalancer**.

The full walkthrough lives in **this file only**. The other three repos are
implementation pieces; they point back here.

| Repo | GitHub | Role |
|------|--------|------|
| **atlas-gitops** (this repo) | https://github.com/Zachery2008/atlas-gitops | Cluster overlays: what to deploy, image tags, config |
| **atlas-charts-common** | https://github.com/Zachery2008/atlas-charts-common | Platform Helm: `_common`, `java-api`, `platform-namespace`, `platform-gitops` (Argo CD) |
| **atlas-charts-apps** | https://github.com/Zachery2008/atlas-charts-apps | Umbrella chart `atlas-shop` (aliases `java-api` as `catalog-api`) |
| **atlas-catalog-api** | https://github.com/Zachery2008/atlas-catalog-api | Java 21 Spring Boot app that becomes the container image |

---

## How the four repos fit together

```text
atlas-catalog-api          docker build / push
        |                    image tag e.g. 0.1.0
        v
atlas-gitops  ---------->  Argo CD ApplicationSets
  aks-dev/.../app.yaml       (running in platform-gitops ns)
        ^
        |  Helm values ($values/...)
        |
atlas-charts-apps          umbrella atlas-shop
  charts/atlas-shop/         vendored java-api subchart
        ^
        |
atlas-charts-common        java-api + platform-gitops
```

**Rule:** CI builds images. Git (this repo) decides *which* tag and *which* chart
run on the cluster. Nobody runs `kubectl apply` for the app after Argo CD is up.

### What each layer owns

1. **App repo** (`atlas-catalog-api`)  
   Java 21 / Spring Boot 3.4. Endpoints:
   - `GET /api/catalog`
   - `GET /actuator/health/liveness`
   - `GET /actuator/health/readiness`  
   Build a container and push it to a registry the cluster can pull
   (sample overlay uses `atlas.azurecr.io/atlas-catalog-api:0.1.0`).

2. **Common charts** (`atlas-charts-common`)  
   Reusable Helm:
   - `_common` — labels, Deployment, Service, ConfigMap helpers
   - `java-api` — Spring Boot workload using those helpers
   - `platform-namespace` — creates the tenant Namespace
   - `platform-gitops` — installs **Argo CD** (upstream Helm chart) plus
     AppProjects and ApplicationSets

3. **App umbrellas** (`atlas-charts-apps`)  
   One umbrella per product. `atlas-shop` depends on `java-api` with
   **alias** `catalog-api`. The subchart is **vendored** under
   `charts/atlas-shop/charts/` so Argo can render from this Git repo
   without a Helm OCI registry.

4. **GitOps overlays** (this repo)  
   Per-cluster folders. Dev only: `aks-dev/`. Each namespace folder has:
   - `_admin.yaml` — enable the namespace + list charts to deploy
   - `app.yaml` — `global:` for the namespace, plus values for chart entry `name: app`

---

## Cluster layout (this repo)

```text
_global.yaml                          # org-wide only (owner tag; k8s-management: tenant-id)
aks-dev/
  _global.yaml                        # cluster identity, registry, resources, internal LB
  _admin/
    _global.yaml                      # Git URLs Argo uses to find the other repos
    gitops.yaml                       # optional extra values for platform-gitops
  atlas-shop-dev1/
    _admin.yaml                       # namespace.enabled + charts[]
    app.yaml                          # global: (namespace) + catalog-api: (service)
```

### `_admin.yaml` (discovery file)

Argo ApplicationSets scan `aks-dev/*/_admin.yaml` on `main`.

```yaml
namespace:
  enabled: true          # false = ApplicationSets ignore this folder
  version: "0.1.0"

charts:
  - name: app            # MUST match overlay filename: app.yaml
    chart: atlas-shop    # folder under atlas-charts-apps/charts/
    version: "0.1.0"
```

If `namespace.enabled` is not `true`, nothing is generated for that folder.

### Configurations: `global` vs overrides

Helm **deep-merges** value files (nested maps merge; **lists replace**). Argo applies
them in this order. **Later files win** on the same key.

| Order | File | Typical contents |
|------:|------|------------------|
| 1 | `_global.yaml` | Org-wide only: `global.tags.owner` (k8s-management keeps Azure tenant-id here) |
| 2 | `aks-dev/_global.yaml` | Cluster: name/host, env tags, registry, default resources, internal LB, `SPRING_PROFILES_ACTIVE` |
| 3 | `aks-dev/atlas-shop-dev1/app.yaml` | `global:` for this namespace, then `catalog-api:` for this service |

There is **no** `_global.yaml` in the namespace folder. Argo still lists
`$values/aks-dev/atlas-shop-dev1/_global.yaml` with `ignoreMissingValueFiles`, so you
can add one later; this demo does not use it.

`global:` is a Helm special key: every subchart sees it as `.Values.global`.
`java-api` **merges** `global.configurations` with `catalog-api.configurations`
(service keys win). Same for `global.resources` vs `catalog-api.resources`.

#### This demo’s resolved ConfigMap

From the committed overlay files:

| Key | Source file | Value |
|-----|-------------|--------|
| `JAVA_TOOL_OPTIONS` | cluster `_global.yaml` | `-XX:+UseG1GC` |
| `SPRING_PROFILES_ACTIVE` | cluster `_global.yaml` | `aks` |
| `ROOT_LOG_LEVEL` | `app.yaml` `global.configurations` | `DEBUG` (was `INFO`) |
| `APP_LOG_LEVEL` | `app.yaml` `global.configurations` | `DEBUG` (was `INFO`) |
| `CATALOG_FEATURE_FLAG` | `app.yaml` `catalog-api.configurations` | `true` (new key) |

Resolved image: `atlas.azurecr.io/atlas-catalog-api:0.1.0`  
(registry from `global.image`, tag from `catalog-api.image.tag`).

Resolved resources: cluster requests/limits, except **memory limit `1Gi`** from `app.yaml` `global.resources`.

Resolved replicas: **1** from the `java-api` chart default.

#### What to put where

| Put in root `_global.yaml` | Put in `aks-dev/_global.yaml` | Put in `app.yaml` `global:` | Put in `catalog-api:` |
|---------------------------|--------------------------------|-----------------------------|----------------------------|
| Org-wide tags only | Cluster name/host, env tags | Namespace resource bumps | `image.tag` (every release) |
| | Registry, default resources, internal LB | Shared log / env for every alias | Feature flags, service-only env |
| | Cluster-wide ConfigMap keys (`SPRING_PROFILES_ACTIVE`) | | `enabled: false` to skip this alias |

Do **not** repeat the full `configurations:` block in `app.yaml` unless you are
overriding a key. Helm merges maps; you only need the keys that change.

#### `app.yaml` (workload values — last file)

```yaml
global:
  resources:
    limits:
      memory: 1Gi
  configurations:
    ROOT_LOG_LEVEL: DEBUG
    APP_LOG_LEVEL: DEBUG

catalog-api:
  enabled: true
  image:
    tag: "0.1.0"
  configurations:
    CATALOG_FEATURE_FLAG: "true"
```

`global:` in this file overlays cluster `_global.yaml` for every alias in the
umbrella. `catalog-api` is the Helm **alias**; keys under it override `java-api`
defaults and matching `global.*` fields.

---

## What Argo CD creates

Bootstrap chart `platform-gitops` installs Argo CD in namespace `platform-gitops`,
then these objects:

| Kind | Name | Purpose |
|------|------|---------|
| AppProject | `admin` | Argo CD / platform apps |
| AppProject | `namespaces` | May create Namespace objects |
| AppProject | `applications` | Tenant workloads (no cluster-scoped resources) |
| Application | `admin-gitops` | Argo manages **itself** from Git after the first helm install |
| ApplicationSet | `namespaces` | One Application per enabled `_admin.yaml` |
| ApplicationSet | `applications` | One Application per `charts[]` entry |

Generated names for this demo:

| Application | Deploys |
|-------------|---------|
| `ns-atlas-shop-dev1` | `platform-namespace` into namespace `atlas-shop-dev1` |
| `ns-atlas-shop-dev1-app-app` | umbrella `atlas-shop` (release `app`) into `atlas-shop-dev1` |

The ApplicationSet **git files** generator reads `_admin.yaml`. A **matrix** +
**list** generator expands `charts[]`. Chart path is
`charts/{{ .chart }}` on `atlas-charts-apps` (here `charts/atlas-shop`).

Sync policy is automated (`prune` + `selfHeal`). Edits that are not in Git
are overwritten.

---

## Ingress (internal LB, no Istio)

The catalog Service is `type: LoadBalancer` with:

```yaml
service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

On **AKS** this gets a private IP on the VNet. There is no Ingress / Gateway /
HTTPRoute.

On Kind or a cluster without a cloud LB controller, the Service stays `Pending`.
That is expected.

Argo CD UI uses the same pattern (internal LB on `argocd-server`) from
`platform-gitops` values.

---

## One-time bootstrap

You need: `kubectl` context on the Dev AKS cluster, `helm` 3, and Git remotes
already pointing at Zachery2008 (they are set in `aks-dev/_admin/_global.yaml`).

### 1. Push the image

```bash
cd atlas-catalog-api
docker build -t atlas.azurecr.io/atlas-catalog-api:0.1.0 .
az acr login --name atlas   # or your registry
docker push atlas.azurecr.io/atlas-catalog-api:0.1.0
```

If the registry name is different, change `global.image.registry` in
`aks-dev/_global.yaml` (this repo) and commit.

### 2. Install Argo CD once

```bash
cd atlas-charts-common/charts/platform-gitops
helm dependency update
helm upgrade --install platform-gitops . \
  --namespace platform-gitops \
  --create-namespace \
  --set cluster.name=aks-dev \
  --set repo.gitops=https://github.com/Zachery2008/atlas-gitops.git \
  --set repo.chartsApps=https://github.com/Zachery2008/atlas-charts-apps.git \
  --set repo.chartsCommon=https://github.com/Zachery2008/atlas-charts-common.git
```

If the four GitHub repos are **private**, create a repo-creds Secret in
`platform-gitops` so Argo can clone:

```bash
kubectl -n platform-gitops create secret generic atlas-git \
  --from-literal=type=git \
  --from-literal=url=https://github.com/Zachery2008 \
  --from-literal=username=Zachery2008 \
  --from-literal=password=<github-pat>
kubectl -n platform-gitops label secret atlas-git \
  argocd.argoproj.io/secret-type=repo
```

Public repos do not need this.

### 3. Wait for sync

```bash
kubectl -n platform-gitops get svc argocd-server
kubectl -n platform-gitops get applications,applicationsets
kubectl -n atlas-shop-dev1 get deploy,svc
```

Get the Argo CD admin password (bcrypt secret created by the chart):

```bash
kubectl -n platform-gitops get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Log in at the internal LB IP of `argocd-server` (user `admin`).  
`server.insecure: true` is intentional: TLS is not terminated on the pod.

When `catalog-api` is Ready, the internal LB IP of Service `catalog-api` serves
`/api/catalog` on port 80.

---

## Day-2 operations (all via Git)

| Change | Where to edit | Then |
|--------|----------------|------|
| New app version | Build/push image, set `catalog-api.image.tag` in `app.yaml` | Merge to `main` |
| Log level / ConfigMap | `app.yaml` `global.configurations` (all aliases) or `catalog-api.configurations` (this service) | Merge to `main` |
| Turn the app off | `catalog-api.enabled: false` or `namespace.enabled: false` | Merge to `main` |
| Chart defaults (probes, resources) | `java-api` in `atlas-charts-common`, then **vendor** into `atlas-charts-apps` | Merge both, then Argo |
| Argo CD / ApplicationSets | `atlas-charts-common/charts/platform-gitops` | Merge; `admin-gitops` self-syncs |

Do **not** `kubectl apply` or `helm upgrade` the app after bootstrap. Argo will
revert drift.

### Vendor `java-api` into the umbrella

After you change `atlas-charts-common` workload templates:

```bash
cd atlas-charts-apps
rm -rf charts/atlas-shop/charts/java-api charts/atlas-shop/charts/_common
cp -R ../atlas-charts-common/charts/java-api charts/atlas-shop/charts/java-api
cp -R ../atlas-charts-common/charts/_common charts/atlas-shop/charts/_common
```

Commit in `atlas-charts-apps`. Argo reads the umbrella from that repo.

---

## Local Helm check (no cluster)

```bash
helm template shop atlas-charts-apps/charts/atlas-shop \
  -f atlas-gitops/_global.yaml \
  -f atlas-gitops/aks-dev/_global.yaml \
  -f atlas-gitops/aks-dev/atlas-shop-dev1/app.yaml
```

Confirm the ConfigMap contains merged keys (`SPRING_PROFILES_ACTIVE=aks`,
`ROOT_LOG_LEVEL=DEBUG`, `APP_LOG_LEVEL=DEBUG`, `CATALOG_FEATURE_FLAG=true`)
and memory limit **1Gi**.

App only (no cluster):

```bash
cd atlas-catalog-api
mvn spring-boot:run
curl http://localhost:8080/api/catalog
```

---

## Troubleshooting

| Symptom | Likely cause |
|---------|----------------|
| ApplicationSet has 0 apps | `_admin.yaml` not on `main`, or `namespace.enabled` is not `true`, or `cluster.name` is not `aks-dev` |
| App OutOfSync / unknown chart | `charts[].chart` does not match a folder under `atlas-charts-apps/charts/` |
| ImagePullBackOff | Tag not in registry, or `image.registry` / AKS pull secret mismatch |
| Service `Pending` | Not AKS (no Azure LB), or no permission to create internal LBs |
| Argo cannot clone | Private GitHub repos without repo-creds Secret |
| Health probes fail | App not listening on 8080, or Actuator probes disabled |

---

## What this demo intentionally omits

No Istio, Ingress nginx, Azure Key Vault CSI, Splunk, Reloader, chart OCI
registry, or extra environments (test/stg/prod). One app, one Dev cluster,
Git as the Helm chart source.
