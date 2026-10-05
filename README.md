# platform-gitops

The platform team's repo. Argo CD reads it; nobody runs `kubectl apply` against the cluster after day 0.

## Articles

This repo is the platform side of the *GitOps in Production* series on [miraccanyilmaz.me](https://miraccanyilmaz.me).
Each article pins its own branch; the runnable lab for each one is a folder in
[blog-wiki](https://github.com/miraccan00/blog-wiki). `main` follows the latest article.

| Branch | Article | Lab |
|---|---|---|
| `blog-04` | Argo CD in HA, Explained by Breaking It · [EN](https://miraccanyilmaz.me/en/blog/argocd-ha-app-of-apps/) · [TR](https://miraccanyilmaz.me/blog/argocd-ha-app-of-apps/) | [argocd-ha-app-of-apps](https://github.com/miraccan00/blog-wiki/tree/main/argocd-ha-app-of-apps) |
| `blog-05` | Argo CD SSO Integration: OIDC and RBAC with ZITADEL · [EN](https://miraccanyilmaz.me/en/blog/argocd-sso-zitadel/) · [TR](https://miraccanyilmaz.me/blog/argocd-sso-zitadel/) | [argocd-sso-zitadel](https://github.com/miraccan00/blog-wiki/tree/main/argocd-sso-zitadel) |
| `blog-05b` = `main` | Moving Secrets into Vault: From base64 in Git to Vault and ESO Without Downtime · [EN](https://miraccanyilmaz.me/en/blog/vault-eso-secret-migration/) · [TR](https://miraccanyilmaz.me/blog/vault-eso-secret-migration/) | [vault-eso-secret-migration](https://github.com/miraccan00/blog-wiki/tree/main/vault-eso-secret-migration) |

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
│   ├── applicationsets.yaml      # the ApplicationSets below, as one Application
│   ├── external-secrets.yaml     # ESO operator and CRDs (wave 0)
│   ├── external-secrets-store.yaml  # the ClusterSecretStore (wave 1)
│   ├── vault.yaml                # Vault, one node, raft (wave 1)
│   ├── zitadel-db.yaml           # ZITADEL's namespace, ExternalSecrets, Postgres (wave 1)
│   ├── zitadel.yaml              # ZITADEL chart (wave 2)
│   └── argocd-secrets.yaml       # ExternalSecrets for Argo CD's own Secrets (wave 2)
├── argocd/
│   ├── values-ha.yaml            # argo/argo-cd chart values: redis-ha, 2x server/repo-server/appset, PDBs, OIDC, RBAC
│   └── secrets/oidc-zitadel.yaml # ZITADEL client ID/secret from Vault
├── external-secrets/             # ESO values + store/cluster-secret-store.yaml
├── vault/values.yaml             # hashicorp/vault chart values
├── zitadel/                      # ZITADEL values; db/: Postgres and the ExternalSecrets for its Secrets
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
├── external-secrets, external-secrets-store (ESO 2.11.0, ClusterSecretStore "vault")
├── vault (hashicorp/vault chart 0.34.1)
├── zitadel-db, zitadel (Postgres, ZITADEL chart 10.1.0; Secrets from Vault)
├── argocd-secrets (ExternalSecret argocd-oidc-zitadel)
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

Vault comes up **sealed**: Argo CD installs it, a human runs `vault operator init` once and unseals it after
every restart. Until then the ExternalSecrets can't sync and ZITADEL waits.

That is the last `helm` and the last `kubectl apply`. From here Argo CD owns its own install
(`apps/argocd.yaml`); upgrading Argo CD = bumping `targetRevision` there in a PR.

A local lab that runs exactly this on kind (1 control plane + 3 workers), Vault init and secret rotation
included, is in [blog-wiki/vault-eso-secret-migration](https://github.com/miraccan00/blog-wiki/tree/main/vault-eso-secret-migration).

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
`apps/external-secrets*.yaml`, `apps/argocd-secrets.yaml`, `vault/`, `external-secrets/`, `argocd/secrets/`;
`zitadel/db/secrets.yaml` replaced by `zitadel/db/externalsecrets.yaml`). `main` is fast-forwarded to the
latest article branch. Branches are never deleted; the articles and their labs read them.
`products-appset.yaml` keeps `blog-04` for `product-helloapi-gitops`, which did not change in 05 or 05b.

On `blog-05`, `zitadel/db/secrets.yaml` holds base64 Secrets **on purpose**: the starting point of the
Vault article. From `blog-05b` on there are no Secret values in this repo, only ExternalSecrets that say
which Vault path to read. The Postgres password from `blog-05` was rotated and no longer works; the
ZITADEL masterkey was moved unchanged (ZITADEL's data is encrypted with it). Lab values only.
