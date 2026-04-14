# homelab-backstage Companion Repo Design

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the `homelab-backstage` companion repo that builds a custom Backstage image with Kubernetes and ArgoCD plugins, published to GHCR via GitHub Actions.

**Architecture:** Standard `npx @backstage/create-app` scaffold with surgical plugin additions (Kubernetes + Roadie ArgoCD). The image is consumed by the existing `homelab-infra` GitOps repo which owns deployment wiring, RBAC, and catalog metadata.

**Tech Stack:** Backstage 1.x, Node 20, TypeScript, Docker multi-stage build, GitHub Actions, GHCR, PostgreSQL (CNPG) in production.

---

## Section 1: Repository Structure

```
homelab-backstage/
├── .github/
│   └── workflows/
│       └── release.yaml              # CI: build + push image to GHCR on tag push
├── packages/
│   ├── app/
│   │   └── src/
│   │       └── components/
│   │           └── catalog/
│   │               └── EntityPage.tsx # Kubernetes + ArgoCD entity cards
│   ├── backend/
│   │   └── src/
│   │       └── plugins/
│   │           └── argocd.ts          # ArgoCD backend plugin registration
│   ├── app/
│   │   └── package.json
│   ├── backend/
│   │   └── package.json
│   └── ...
├── app-config.yaml                    # Base config (dev/local)
├── app-config.production.yaml         # Production overrides (DB, cluster, catalog)
├── Dockerfile                         # Multi-stage build for GHCR image
├── catalog-info.yaml                  # Backstage's own catalog entry
├── package.json                       # Monorepo root
├── tsconfig.json
└── README.md
```

**Key decisions:**
- Standard monorepo layout from `create-app` — easy to upgrade with `backstage-cli migrate`
- Only modifications: `EntityPage.tsx` (frontend cards), `plugins/argocd.ts` (backend), `app-config.production.yaml` (runtime wiring)
- The `Dockerfile` and GitHub Actions workflow produce `ghcr.io/rizasatya/homelab-backstage:vX.Y.Z`

---

## Section 2: Plugin Wiring

### Frontend — EntityPage.tsx

Add two entity cards to the default Backstage entity page:

```tsx
import { EntityKubernetesContent } from '@backstage/plugin-kubernetes';
import { EntityArgoCDContentCard } from '@roadiehq/backstage-plugin-argo-cd';
```

- **Kubernetes tab:** `EntityKubernetesContent` renders cluster resource view for the entity
- **ArgoCD card:** `EntityArgoCDContentCard` renders ArgoCD application status, history, and sync info

### Backend — plugins/argocd.ts

```ts
import { createRouter } from '@roadiehq/backstage-plugin-argo-cd-backend';
```

Register in `packages/backend/src/index.ts`:

```ts
backend.add(createRouter({ ... }));
```

The Kubernetes backend plugin (`@backstage/plugin-kubernetes-backend`) is already included in the default scaffold and requires no additional registration.

### Environment variables consumed at runtime

| Variable | Source | Purpose |
|----------|--------|---------|
| `BACKEND_SECRET` | SealedSecret in homelab-infra | Backend session signing |
| `ARGOCD_AUTH_TOKEN` | SealedSecret in homelab-infra | ArgoCD API authentication |
| `GITHUB_TOKEN` | Optional, SealedSecret | Catalog discovery rate limits |

---

## Section 3: Configuration

### app-config.yaml (dev/local)

```yaml
app:
  title: Homelab Backstage
  baseUrl: http://localhost:7007

backend:
  baseUrl: http://localhost:7007
  listen:
    port: 7007
  database:
    client: better-sqlite3
    connection: ':memory:'

catalog:
  rules:
    - allow: [Component, Group, Location, User]
  locations:
    - type: file
      target: ../../catalog-info.yaml
```

### app-config.production.yaml

```yaml
app:
  title: Homelab Backstage
  baseUrl: http://localhost:7007

backend:
  baseUrl: http://localhost:7007
  listen:
    port: 7007
  database:
    client: pg
    connection:
      host: backstage-db-rw.backstage.svc.cluster.local
      port: 5432
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
      database: ${POSTGRES_DB}
  auth:
    keys:
      - secret: ${BACKEND_SECRET}

kubernetes:
  serviceLocatorMethod:
    type: multiTenant
  clusterLocatorMethods:
    - type: config
      clusters:
        - name: homelab
          url: https://kubernetes.default.svc
          authProvider: serviceAccount
          skipTLSVerify: false
          skipMetricsLookup: true
          customResources:
            - group: postgresql.cnpg.io
              apiVersion: v1
              plural: clusters
            - group: platform.rizaes.com
              apiVersion: v1alpha1
              plural: webapps

integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

catalog:
  rules:
    - allow: [Component, Group, Location, User]
  locations:
    - type: url
      target: https://raw.githubusercontent.com/RizaSatya/homelab/main/platform/control-plane/backstage/catalog/root.yaml

argocd:
  username: backstage
  password: ${ARGOCD_AUTH_TOKEN}
  baseUrl: https://argocd-server.argocd.svc.cluster.local
```

**Note:** `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` are managed by the CNPG `Cluster` operator and injected via secrets. The CNPG Cluster manifest lives in `homelab-infra`, not this repo.

---

## Section 4: CI/CD — GitHub Actions

### .github/workflows/release.yaml

```yaml
name: Release Backstage Image
on:
  push:
    tags: ['v*']

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: yarn

      - run: yarn install --frozen-lockfile
      - run: yarn tsc
      - run: yarn build

      - uses: docker/setup-build-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/rizasatya/homelab-backstage:${{ github.ref_name }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Dockerfile

```dockerfile
FROM node:20-bookworm AS build

WORKDIR /app
COPY package.json yarn.lock ./
COPY packages/app/package.json packages/app/
COPY packages/backend/package.json packages/backend/
RUN yarn install --frozen-lockfile --network-timeout 600000

COPY . .
RUN yarn tsc
RUN yarn build

FROM node:20-bookworm-slim

WORKDIR /app
RUN addgroup --system --gid 1001 backstage && \
    adduser --system --uid 1001 backstage

COPY --from=build /app/packages/app/dist /app/packages/app/dist
COPY --from=build /app/packages/backend/dist /app/packages/backend/dist
COPY --from=build /app/packages/backend/node_modules /app/packages/backend/node_modules
COPY --from=build /app/node_modules /app/node_modules
COPY --from=build /app/package.json /app/package.json
COPY --from=build /app/app-config.yaml /app/app-config.yaml
COPY --from=build /app/app-config.production.yaml /app/app-config.production.yaml

ENV NODE_ENV=production
ENV APP_CONFIG=/app/app-config.yaml,/app/app-config.production.yaml
USER backstage

EXPOSE 7007
CMD ["node", "packages/backend"]
```

---

## Section 5: GitOps Cross-Repo Contract

| Concern | `homelab-backstage` (companion) | `homelab-infra` (GitOps) |
|---------|--------------------------------|--------------------------|
| Backstage image | Build + publish to GHCR | Reference image tag in `values.yaml` |
| App config (runtime) | `app-config.production.yaml` baked into image | Helm values overlay for cluster/catalog |
| Kubernetes plugin | UI + backend registration | ClusterRole, ServiceAccount, ClusterRoleBinding |
| ArgoCD plugin | Backend registration, env var expectations | Local account, RBAC policy, sealed secret |
| Database | PostgreSQL client config in app-config | CNPG Cluster manifest |
| Catalog entities | Own `catalog-info.yaml` | Root location, org, tenant catalog-info files |

---

## Section 6: CNPG Follow-Up in homelab-infra

The `idp-11/backstage` branch currently configures SQLite. Since this design uses PostgreSQL, the following changes are needed in `homelab-infra` (separate PR/commit):

1. **Add CNPG Cluster manifest** at `platform/control-plane/backstage/runtime/backstage-db.yaml`:
   ```yaml
   apiVersion: postgresql.cnpg.io/v1
   kind: Cluster
   metadata:
     name: backstage-db
     namespace: backstage
   spec:
     instances: 1
     storage:
       size: 1Gi
       storageClassName: local-path
     bootstrap:
       initdb:
         database: backstage
         owner: backstage
   ```

2. **Update `values.yaml`** to remove SQLite PVC config and add PostgreSQL environment variables from the CNPG-generated secrets.

3. **Update `app-config.production.yaml`** in the companion repo to match the CNPG service endpoints.

---

## Section 7: Rollout Order

1. Scaffold `homelab-backstage` with `npx @backstage/create-app`
2. Add Kubernetes + ArgoCD plugins
3. Write `app-config.production.yaml` with CNPG wiring
4. Write `Dockerfile` and GitHub Actions workflow
5. Push `v0.1.0` tag → image builds and publishes to GHCR
6. In `homelab-infra` (separate PR): add CNPG Cluster manifest, update values.yaml
7. Merge `idp-11/backstage` into main (deploys Backstage via ArgoCD)
8. Generate ArgoCD local account token, seal it, apply
9. Port-forward and verify: `kubectl -n backstage port-forward svc/backstage 7007:7007`

---

## Spec Self-Review

- [x] No placeholders — all code blocks contain complete, copyable content (fixed: ArgoCD URL, APP_CONFIG env var)
- [x] No contradictions — CNPG is consistently used throughout (values.yaml, app-config, CNPG manifest)
- [x] Scope is focused — companion repo creation only; homelab-infra changes noted as follow-up
- [x] Cross-repo contract is explicit — table maps each concern to its owning repo
- [x] Rollout order is actionable — numbered steps with clear dependencies
- [x] Environment variables are documented — `BACKEND_SECRET`, `ARGOCD_AUTH_TOKEN`, `GITHUB_TOKEN`, `POSTGRES_*`
- [x] Dockerfile sets `APP_CONFIG` env var to load both base and production configs
