# CLAUDE.md

## Project Overview

GitOps configuration for all of Divya's projects, split out of the `product-search` repo so several
application repos can share one deployment pipeline. ArgoCD runs in minikube, watches `main` of this
repo, and applies everything. **No application code and no builds happen here.**

Read `README.md` first — it is the operator guide (bootstrap, adding a service, the release flow,
the token).

## Layout

```text
deploy/
├── bootstrap/root-app.yaml   app-of-apps; the only thing applied by hand
├── apps/
│   ├── eck-operator.yaml     Helm, external repo, wave -1
│   ├── cnpg-operator.yaml    Helm, external repo, wave -1
│   └── services.yaml         ApplicationSet, wave 1; reads each service's overlay app.yaml
└── services/
    ├── catalog-service/{base,overlays/local}   + postgres.yaml      -> ns product-search
    ├── search-service/{base,overlays/local}    + elasticsearch.yaml, kibana.yaml
    └── chakra/{base,overlays/local}            + redis.yaml         -> ns chakra
```

There is no `platform/` any more. A datastore used by exactly one service lives in that service's
`base/`, which is where chakra's Redis already was. Promote one to `platform/<name>/` with its own
Application only when a second service genuinely needs it.

```text
```

## Rules that matter

- **A namespace is per product, not per service.** It is an RBAC, quota and NetworkPolicy
  boundary, so it belongs to whoever owns the thing. `catalog-service` and `search-service` are two
  services of one product and share `product-search`; chakra is its own and keeps `chakra`. Each
  service declares its name and namespace in `overlays/local/app.yaml`, which the ApplicationSet
  reads — deriving it from the directory name would force a namespace per service, and with it a
  separate copy of every datastore, since Secrets cannot cross namespaces.
- **An Application that owns stateful resources needs
  `finalizers: [resources-finalizer.argocd.argoproj.io]`.** `prune: true` only removes resources
  that disappear from a *live* Application's manifests. Delete or rename the Application itself and
  its resources are orphaned — still running, owned by nobody, and invisible to ArgoCD. Renaming
  `data` into `data-catalog`/`data-search` did exactly that: a second Postgres, Elasticsearch and
  Kibana kept running in `product-search` and took the node to 94% of its memory requests, which
  is what made the new datastores restart and hang in `ApplyingChanges`.
- **Every git `repoURL` must point at this repo**, never at an application repo. CI fails the build
  if a reference to `github.com/Divya-Somashekar/product-search` survives — that is the classic
  split-repo regression.
- **The `services` ApplicationSet derives both the Application name and the namespace from the
  directory name** (`index .path.segments 1`). Renaming a directory renames and re-creates the
  Application; with `prune: true` that deletes the old one's resources. Rename deliberately.
- **An overlay's `kustomization.yaml` must set `namespace:` to its own directory name**, matching the
  generated destination. If they diverge ArgoCD reports a conflict rather than applying to the wrong
  namespace, but it is still a broken sync.
- **`images[].name` in an overlay is a contract with the releasing repo's workflow.** It is the
  placeholder `kustomize edit set image <name>=…` targets. Changing it silently breaks that repo's
  deploys — nothing fails, the tag simply stops being updated.
- **CI is `on: pull_request` only, deliberately.** App repos push tag bumps to `main` here with a
  PAT or App token, and unlike `GITHUB_TOKEN` those pushes *do* trigger workflows. An `on: push` job
  that wrote back to the repo would loop.
- **Operators need `ServerSideApply=true`** (CRDs too large for client-side apply), and `data` plus
  the generated Applications need `SkipDryRunOnMissingResource=true` and a `retry` block, because
  sync waves order apps but do not wait for CRDs to be established.
- Credentials come only from operator-created Secrets: `products-db-app` (CloudNativePG) and
  `search-es-elastic-user` (ECK). Service names `products-db-rw:5432`, `search-es-http:9200`.

## Commands

```bash
kustomize build services/catalog-service/overlays/local   # what ArgoCD's DESIRED tab shows
kustomize build services/search-service/overlays/local
kubectl apply --dry-run=client -f apps/ -f bootstrap/
kubectl -n argocd get applications

# The ArgoCD UI. Not 8080 — that collides with a service port-forward, and with anything else
# already on it; the password is in argocd-initial-admin-secret.
kubectl -n argocd port-forward svc/argocd-server 8443:443        # https://localhost:8443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d

kubectl -n product-search port-forward svc/catalog-service 18080:80
kubectl -n product-search port-forward svc/search-service  18081:80
```

**Always render after editing any YAML here.** Several manifests have been broken by pasted
indentation and stray terminal text. Quote URLs inside `{ … }` flow mappings.

## Known PoC shortcuts

- Elasticsearch runs with TLS disabled and the app uses the `elastic` superuser.
- No NetworkPolicy (minikube's default CNI does not enforce one anyway).
- One environment (`overlays/local`); stg/prod overlays would sit beside it.
- Each datastore is a single instance sized for a demo, and `search-service` keeps its change-log
  cursor in memory, so a restart replays the catalog's change log from the beginning.
- The catalog's `/internal/*` feed is unauthenticated and is now a cross-namespace call, so without
  a NetworkPolicy anything in the cluster can read the whole catalogue.
- Demo data is loaded by Flyway under the `demo` profile — never enable it anywhere real.

## Conventions

- Conventional commits. `chore(deploy): <service> <sha>` is the release bot from an app repo.
- A service directory name is its Application name, its namespace, and its kustomize image key.
  Keep the three identical.
