# platform-gitops

The platform team's repo. Argo CD reads it; nobody runs `kubectl apply` against the cluster after day 0.

It owns:

1. **Argo CD itself**: its Helm values (`argocd/values-ha.yaml`) and the Application that keeps it in sync.
2. **The app-of-apps root**: `bootstrap/root.yaml` → `apps/`.
3. **Namespaces and quotas** per product per environment (`namespaces/`).
4. **The product registry**: which product repos Argo CD deploys, to which environments (`applicationsets/`).

It does not own application code or Helm charts. Those live with the product team:

| Repo | Owner | Contents |
|---|---|---|
| [hellofiber](https://github.com/miraccan00/hellofiber) | product team | Go code, Dockerfile, CI → `ghcr.io/miraccan00/hellofiber:sha-<7>` |
| [product-helloapi-gitops](https://github.com/miraccan00/product-helloapi-gitops) | product team | Helm chart + `overlays/{dev,prod}/values.yaml` (image tag, replicas, env) |
| platform-gitops (this repo) | platform team | everything below |

## Layout

```
platform-gitops/
├── bootstrap/
│   └── root.yaml                 # applied once by hand; points at apps/
├── apps/                         # everything root manages (app-of-apps)
│   ├── projects.yaml             # AppProjects: platform, products (wave -2)
│   ├── argocd.yaml               # Argo CD manages its own Helm install (wave -1)
│   └── applicationsets.yaml      # the ApplicationSets below, as one Application
├── argocd/
│   └── values-ha.yaml            # argo/argo-cd chart values: redis-ha, 2x server/repo-server/appset, PDBs
├── applicationsets/
│   ├── namespaces-appset.yaml    # namespaces/<product>/<env> → namespace + ResourceQuota
│   └── products-appset.yaml      # product registry × environments → one Application each
└── namespaces/
    └── product-helloapi/{base,dev,prod}/
```

What runs, top to bottom:

```
root (Application, applied by hand)
├── platform, products (AppProject)
├── argocd (Application → argo-cd chart 10.9.2 + argocd/values-ha.yaml)
└── applicationsets (Application → applicationsets/)
    ├── namespaces (ApplicationSet) → ns-product-helloapi-dev, ns-product-helloapi-prod
    └── products   (ApplicationSet) → product-helloapi-dev, product-helloapi-prod
```

## Day 0

Needs a cluster with at least 3 schedulable nodes (redis-ha uses required pod anti-affinity).

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --version 10.9.2 -n argocd --create-namespace \
  -f argocd/values-ha.yaml --wait
kubectl apply -f bootstrap/root.yaml
```

That is the last `helm` and the last `kubectl apply`. From here Argo CD owns its own install
(`apps/argocd.yaml`); upgrading Argo CD = bumping `targetRevision` there in a PR.

A local lab that runs exactly this on kind (1 control plane + 3 workers) is in
[blog-wiki/argocd-ha-app-of-apps](https://github.com/miraccan00/blog-wiki/tree/main/argocd-ha-app-of-apps).

## Onboarding a product

1. `namespaces/<product>/{base,dev,prod}/`: copy `product-helloapi`, change the names and quotas.
2. `applicationsets/products-appset.yaml`: one list element:
   ```yaml
   - product: <product>
     productRepoURL: https://github.com/miraccan00/<product>-gitops.git
   ```
3. The product repo needs `overlays/dev` and `overlays/prod`, each a kustomization Argo CD can render
   (see product-helloapi-gitops). The `products` AppProject only allows repos named
   `product-*-gitops` and namespaces named `product-*`.

One PR to this repo; after merge Argo CD creates `<product>-dev` and `<product>-prod`.

## Day to day

| Change | Where | Who reviews |
|---|---|---|
| Ship a new build | `overlays/<env>/values.yaml` → `image.tag` (product repo) | product team |
| Replicas, env vars, resources | `overlays/<env>/values.yaml` (product repo) | product team |
| Quota | `namespaces/<product>/<env>/` (this repo) | platform team |
| Onboard / offboard a product | `applicationsets/products-appset.yaml` (this repo) | platform team |
| Upgrade Argo CD | `apps/argocd.yaml` → `targetRevision` (this repo) | platform team |

Offboarding: remove the list element (with `prune: true` the Applications and their resources go),
then remove `namespaces/<product>/`.

## Branch note

Each article pins this repo to its own branch: `blog-04` (Argo CD HA, app-of-apps), `blog-05` (+ ZITADEL
SSO: `apps/zitadel-db.yaml`, `apps/zitadel.yaml`, `zitadel/`, OIDC and RBAC in `argocd/values-ha.yaml`), `blog-05b` (+ Vault and ESO: `apps/vault.yaml`,
`apps/external-secrets*.yaml`, `vault/`, `external-secrets/`, shadow ExternalSecrets in `zitadel/db/`).
Branches are never deleted; the articles and their labs read them. `products-appset.yaml` keeps
`blog-04` for `product-helloapi-gitops`, which did not change in 05.

`zitadel/db/secrets.yaml` holds base64 Secrets **on purpose** (the starting point of the next article,
which moves them into Vault and rotates them). Lab values only.
