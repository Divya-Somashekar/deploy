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
│   ├── data.yaml             two Applications, wave 0: data-catalog, data-search
│   └── services.yaml         ApplicationSet -> services/*/overlays/local, wave 1
├── platform/data-catalog/    Postgres (CloudNativePG) -> ns catalog-service
├── platform/data-search/     Elasticsearch + Kibana (ECK) -> ns search-service
└── services/
    ├── catalog-service/{base,overlays/local}
    ├── search-service/{base,overlays/local}
    └── chakra/{base,overlays/local}
```

## Rules that matter

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
kustomize build platform/data-catalog
kustomize build platform/data-search
kubectl apply --dry-run=client -f apps/ -f bootstrap/
kubectl -n argocd get applications
kubectl -n catalog-service port-forward svc/catalog-service 8080:80
kubectl -n search-service  port-forward svc/search-service  8081:80
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
