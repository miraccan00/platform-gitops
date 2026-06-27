# Platform GitOps — ArgoCD Multi-Product Management

## What this repo is

This is the **platform team's repo**. It owns two things:

1. **Namespace resources** — creating namespaces, ResourceQuotas, LimitRanges per product per environment
2. **ApplicationSet wiring** — telling ArgoCD which product repos to watch and which Helm charts to deploy

It does **not** own application code, Helm chart templates, or product-specific values. Those belong to each product team in their own repo.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  THIS REPO (platform-gitops)                                     │
│                                                                  │
│  applicationsets/                                                │
│    namespaces-appset.yaml   ← manages namespaces + quotas       │
│    products-appset.yaml     ← deploys apps from product repos   │
│                                                                  │
│  namespaces/                                                     │
│    mirac/dev/resourcequota.yaml                                  │
│    mirac/prod/resourcequota.yaml                                 │
│    product-b/dev/resourcequota.yaml                              │
│    ...                                                           │
└─────────────────────────────────────────────────────────────────┘
          │                              │
          │ watches                      │ watches (per product)
          ▼                              ▼
┌──────────────────┐         ┌──────────────────────────────────┐
│  Helm/OCI Proxy  │         │  mirac repo (Azure DevOps)       │
│  (Nexus/Harbor)  │         │                                  │
│                  │         │  dev/values.yaml                 │
│  chart: my-app   │         │  prod/values.yaml                │
│  version: 1.2.3  │         │                                  │
└──────────────────┘         └──────────────────────────────────┘
          │                              │
          └──────────────┬───────────────┘
                         ▼
              ArgoCD merges chart + values
              and deploys to cluster
              
              Namespace: mirac-dev
              Namespace: mirac-prod
```

### Two ApplicationSets, two sync waves

| ApplicationSet | Sync Wave | What it does |
|---|---|---|
| `namespaces-appset` | `-1` (first) | Creates namespace + applies ResourceQuota |
| `products-appset` | `0` (second) | Deploys Helm release into the namespace |

Wave -1 runs first, so the namespace and quotas always exist before the app pods start.

---

## Repository layout

```
platform-gitops/                        ← THIS REPO
├── applicationsets/
│   ├── namespaces-appset.yaml          ← auto-discovers namespaces/ directory
│   └── products-appset.yaml           ← product registry (one entry per product)
└── namespaces/
    └── <product>/
        ├── dev/
        │   └── resourcequota.yaml
        └── prod/
            └── resourcequota.yaml

product-gitops (separate repo per team)  ← PRODUCT TEAM'S REPO
├── dev/
│   └── values.yaml
└── prod/
    └── values.yaml
```

---

## Does Azure DevOps work?

**Yes.** ArgoCD supports Azure DevOps repos over HTTPS with a Personal Access Token (PAT). The repo URL format is:

```
https://dev.azure.com/<ORG>/<PROJECT>/_git/<REPO>
```

For example, if your mirac product repo is in Azure DevOps:

```
https://dev.azure.com/your-org/mirac/_git/mirac-gitops
```

You register this URL in ArgoCD once during product onboarding (Step 3 below), and ArgoCD polls it for changes automatically.

---

## Day 0 — Initial platform setup

Do this once when setting up the platform for the first time.

### Prerequisites

- ArgoCD installed in the cluster (`argocd` namespace)
- `argocd` CLI installed and logged in
- `kubectl` access to the cluster
- Helm proxy/OCI registry available (Nexus, Harbor, Artifactory, etc.)

### Step 1 — Register this platform repo with ArgoCD

```bash
# If repo is public or uses SSH key already configured, skip --username/--password
argocd repo add https://github.com/YOUR_ORG/platform-gitops.git \
  --username git \
  --password <GITHUB_PAT>
```

### Step 2 — Update repoURL in ApplicationSets

Open both ApplicationSet files and replace `YOUR_ORG` with your actual org:

```bash
# applicationsets/namespaces-appset.yaml  — line with repoURL
# applicationsets/products-appset.yaml    — lines with repoURL
```

### Step 3 — Apply the ApplicationSets to the cluster

```bash
kubectl apply -f applicationsets/namespaces-appset.yaml
kubectl apply -f applicationsets/products-appset.yaml
```

At this point ArgoCD is watching the repo. No products are registered yet so nothing deploys. The platform is ready to onboard products.

---

## Onboarding a new product — example: `mirac`

Do this whenever a new product team needs a namespace and app deployment on the cluster.

### Step 1 — Create namespace resources in this repo

```bash
mkdir -p namespaces/mirac/dev
mkdir -p namespaces/mirac/prod
```

Create `namespaces/mirac/dev/resourcequota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: mirac-dev
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 2Gi
    limits.cpu: "2"
    limits.memory: 4Gi
    pods: "20"
```

Create `namespaces/mirac/prod/resourcequota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: mirac-prod
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "50"
```

Commit and push. ArgoCD's `namespaces-appset` discovers the new `namespaces/mirac/*` directories automatically and creates the namespaces `mirac-dev` and `mirac-prod` on the cluster within 3 minutes (default poll interval).

### Step 2 — Register the mirac Azure DevOps repo with ArgoCD

ArgoCD needs credentials to pull `values.yaml` from the mirac team's Azure DevOps repo. The mirac team provides you a PAT with read access to their repo.

```bash
argocd repo add https://dev.azure.com/YOUR_ORG/mirac/_git/mirac-gitops \
  --username git \
  --password <MIRAC_TEAM_PAT>
```

> The PAT needs only **Code (Read)** permission on the mirac project. No write access required.

Alternatively, use a shared service account PAT scoped to all product repos if your organization allows it.

### Step 3 — Add mirac to the products ApplicationSet

Edit `applicationsets/products-appset.yaml` and add one entry under the `list` generator:

```yaml
- product: mirac
  productRepoURL: https://dev.azure.com/YOUR_ORG/mirac/_git/mirac-gitops
  chart: my-app                          # chart name in your Nexus/Harbor
  chartRepoURL: https://nexus.example.com/helm
  chartVersion: "1.2.3"
```

Commit and push. ArgoCD picks up the change and creates two new Applications:
- `mirac-dev` → deploys `my-app:1.2.3` into namespace `mirac-dev`
- `mirac-prod` → deploys `my-app:1.2.3` into namespace `mirac-prod`

### Step 4 — Tell the mirac team what their repo must contain

The mirac team's Azure DevOps repo (`mirac-gitops`) must have this structure:

```
mirac-gitops/
├── dev/
│   └── values.yaml      ← dev environment overrides
└── prod/
    └── values.yaml      ← prod environment overrides
```

Minimum `values.yaml` the team needs:

```yaml
image:
  tag: "1.0.0"           # the version they want to run

replicaCount: 1

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

That's it. Once these files exist on `main` in their repo, ArgoCD syncs automatically.

---

## Day-to-day workflows

### Platform team: update a quota

```
namespaces/mirac/prod/resourcequota.yaml  ← edit limits here
```

Open a PR to this repo, get review, merge to `main`. ArgoCD syncs the ResourceQuota within 3 minutes. The mirac team is not involved.

### Product team (mirac): update app config or version

```
mirac-gitops/prod/values.yaml  ← edit image.tag, env vars, replicas, etc.
```

They open a PR in their own Azure DevOps repo, get their own team review, merge to `main`. ArgoCD detects the change and syncs the Helm release. Platform team is not involved.

### Platform team: upgrade the chart version for all products

Edit `applicationsets/products-appset.yaml` and update `chartVersion` for the relevant product. Or if all products share the same chart, update all entries. One PR, one merge, all affected products upgrade on next ArgoCD sync.

### Offboarding a product

1. Remove the product entry from `applicationsets/products-appset.yaml` — ArgoCD deletes the Application (and with `prune: true`, removes deployed resources)
2. Remove `namespaces/<product>/` — ArgoCD deletes the namespace resources
3. Remove the repo credentials: `argocd repo rm https://dev.azure.com/YOUR_ORG/<product>/_git/<repo>`

---

## Local testing — verify values render correctly

Before onboarding a product, pull the chart from your proxy and run `helm template` locally to verify the values file produces valid manifests:

```bash
# Add your proxy repo locally (one-time)
helm repo add my-proxy https://nexus.example.com/helm
helm repo update

# dev environment
helm template mirac my-proxy/my-app \
  --version 1.2.3 \
  -f path/to/mirac-gitops/dev/values.yaml \
  --namespace mirac-dev

# prod environment
helm template mirac my-proxy/my-app \
  --version 1.2.3 \
  -f path/to/mirac-gitops/prod/values.yaml \
  --namespace mirac-prod
```

---

## Quick reference

### ArgoCD Application naming

| Product | Environment | ArgoCD Application name | Kubernetes Namespace |
|---|---|---|---|
| mirac | dev | `mirac-dev` | `mirac-dev` |
| mirac | prod | `mirac-prod` | `mirac-prod` |
| product-b | dev | `product-b-dev` | `product-b-dev` |

### Files you edit per action

| Action | File | Repo |
|---|---|---|
| Onboard product | `applicationsets/products-appset.yaml` | platform-gitops |
| Create namespace | `namespaces/<product>/<env>/resourcequota.yaml` | platform-gitops |
| Update quota | `namespaces/<product>/<env>/resourcequota.yaml` | platform-gitops |
| Upgrade chart version | `applicationsets/products-appset.yaml` | platform-gitops |
| Update app config/version | `<env>/values.yaml` | product's own repo |

### Supported Git providers for product repos

ArgoCD supports any of the following for product team repos:

| Provider | repoURL format |
|---|---|
| Azure DevOps | `https://dev.azure.com/<org>/<project>/_git/<repo>` |
| GitHub | `https://github.com/<org>/<repo>.git` |
| GitLab | `https://gitlab.com/<org>/<repo>.git` |
| Bitbucket | `https://bitbucket.org/<org>/<repo>.git` |
| Self-hosted | any HTTPS or SSH URL |
