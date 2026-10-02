# deploy

GitOps configuration for every project. ArgoCD runs inside minikube, watches `main` of **this**
repo, and applies what it finds. Application code lives elsewhere; nothing here is built.

```text
deploy/
├── bootstrap/root-app.yaml   the only manifest applied by hand
├── apps/                     ArgoCD Applications + the services ApplicationSet
└── services/
    ├── catalog-service/   app + Postgres (CloudNativePG)      -> ns product-search
    ├── search-service/    app + Elasticsearch, Kibana (ECK)   -> ns product-search
    └── chakra/            app + Redis                         -> ns chakra
```

## Bootstrap a fresh cluster

```bash
minikube start --driver=docker --cpus 4 --memory 6g
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
kubectl apply -f bootstrap/root-app.yaml
kubectl -n argocd get applications          # wait for Synced / Healthy
```

`kubectl apply` on `root-app.yaml` updates it in place. Deleting `root` does **not** cascade to its
children (no finalizer), so delete those explicitly when starting over.

## What ArgoCD creates

| Application | Wave | Source | Namespace |
|---|---|---|---|
| `eck-operator` | -1 | Helm `helm.elastic.co` `eck-operator` 3.5.0 | `elastic-system` |
| `cnpg-operator` | -1 | Helm `cloudnative-pg.github.io/charts` 0.29.1 | `cnpg-system` |
| `services` (ApplicationSet) | 1 | one Application per `services/*/overlays/local/app.yaml` | from that file |

Operators use `ServerSideApply=true` — their CRDs are too large for client-side apply. `data` and the
generated service Applications use `SkipDryRunOnMissingResource=true` plus a `retry` block, because
sync waves order the apps but do not wait for CRDs to be established, so a first sync can fail and
succeed on retry.

## Adding a service

The `services` ApplicationSet uses a git directory generator, so onboarding is a directory, not a new
Application file:

```text
services/<name>/
├── base/            Deployment, Service, PodDisruptionBudget, kustomization.yaml
└── overlays/local/  kustomization.yaml — namespace: <name>, plus the image to deploy
```

Conventions the generator depends on:

- the directory name **is** the Application name and the target namespace
- the overlay must live at `overlays/local` and render with `kustomize build`
- the overlay's `kustomization.yaml` must set `namespace: <name>` to match the generated destination
- `images[].name` must be a stable placeholder the release workflow can target, e.g.
  `kustomize edit set image <name>=ghcr.io/…`

A directory with no renderable overlay is simply not generated, so a project can sit here
half-finished without breaking a sync.

### Status of the other projects

| Project | Ready? | Missing |
|---|---|---|
| `catalog-service` | manifests ready | its first release, which creates the GHCR package. New packages are private by default and the cluster has no pull secret, so make it public or the pod sits in `ImagePullBackOff` |
| `search-service` | manifests ready | same as above |
| `chakra` | manifests ready | the app side: a Dockerfile, a release workflow, `DEPLOY_REPO_TOKEN` in that repo, and `spring-boot-starter-actuator` (the probes below need it). Until its first release lands a real tag, the overlay points at `:placeholder` and the pod sits in `ImagePullBackOff` |
| `redis-lab` | not intended | it is a Testcontainers learning repo whose value is `./gradlew test`. Deploying it costs memory on an already-tight node and buys nothing. Leave it test-only. |

## How a release reaches the cluster

```text
push to main in an app repo (app/**)
  └─ its release.yaml: test → build multi-arch image → push to ghcr.io
       └─ checks out THIS repo with DEPLOY_REPO_TOKEN
            └─ kustomize edit set image  →  commit  →  push to main here
                 └─ ArgoCD syncs
```

A deploy is now **two commits in two repos**. `git log` in an application repo no longer tells you
what is deployed — that history lives here.

### The token

Each app repo needs a secret named `DEPLOY_REPO_TOKEN` with write access to this repo:

- **GitHub App installation token** (preferred) — scoped to this repo only, short-lived
- a **fine-grained PAT** with `Contents: read/write` on this repo is simpler and acceptable for a PoC

Do not reach for a classic PAT: it carries write access to everything you own, into three repos.

Note that unlike `GITHUB_TOKEN`, pushes made with these tokens **do** trigger workflows here. That is
why `ci.yaml` is `on: pull_request` only — an `on: push` job that wrote back would loop.

## Per-service dependencies

Each datastore belongs to the one service that owns it, so there is nothing shared to serve.
chakra needs **Redis**, which nothing in `platform/`
provides, so it ships its own in `services/chakra/base/redis.yaml` — a plain Deployment plus Service
in chakra's namespace, with `emptyDir` storage because chakra can repopulate it.

That is the right call while exactly one service needs Redis. The moment a second one does, promote it
to `platform/redis/` with its own Application and point both at it; sharing a datastore across
namespaces is a deliberate decision, not a default.

## A namespace is per product, not per service

`catalog-service` and `search-service` are two services of one product and share the
**`product-search`** namespace. chakra is a different product and keeps its own. A namespace is an
RBAC, quota and NetworkPolicy boundary, so it belongs to whoever owns the thing rather than to each
individual process; one namespace per service is a choice for organisations that need hard
isolation between them, and this is not one.

Each service states its name and namespace in `overlays/local/app.yaml`, which the `services`
ApplicationSet reads. That value and the `namespace:` in the overlay's `kustomization.yaml` are set
by hand and must agree; if they diverge ArgoCD reports it rather than silently applying somewhere
else.

Deriving the namespace from the directory name instead would force one namespace per service, and
with it a separate copy of every datastore — a Secret cannot cross namespaces, so `products-db-app`
and `search-es-elastic-user` have to exist wherever the pod that reads them runs.

The consequence to watch: the full stack needs ~5.5 GB. Elasticsearch exiting 137 means it ran out
of memory — `colima stop && colima start --cpu 4 --memory 8`, then recreate minikube.

## Validating a change

CI renders every overlay and validates it, but do it locally before pushing — several manifests have
been broken by pasted indentation:

```bash
kustomize build services/catalog-service/overlays/local
kustomize build services/search-service/overlays/local
kubectl apply --dry-run=client -f apps/ -f bootstrap/
```

Quote URLs inside `{ … }` flow mappings.
