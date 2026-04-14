# homelab-backstage Companion Repo Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the `homelab-backstage` companion repo at `~/Documents/workshop/claude/homelab-backstage/` with Kubernetes + ArgoCD plugins, Dockerfile, and GitHub Actions CI/CD for GHCR publishing.

**Architecture:** Standard `npx @backstage/create-app` scaffold with surgical plugin additions. The built image is consumed by `homelab-infra` GitOps repo which owns deployment wiring.

**Tech Stack:** Backstage 1.x, Node 20, TypeScript, Docker multi-stage build, GitHub Actions, GHCR.

---

## File Map

| File | Purpose |
|------|---------|
| `~/Documents/workshop/claude/homelab-backstage/` | Root — created by `npx @backstage/create-app` |
| `packages/app/src/components/catalog/EntityPage.tsx` | Add Kubernetes + ArgoCD entity cards |
| `packages/backend/src/plugins/argocd.ts` | ArgoCD backend plugin |
| `packages/backend/src/index.ts` | Register ArgoCD backend plugin |
| `app-config.production.yaml` | Production overrides (DB, cluster, catalog, ArgoCD) |
| `Dockerfile` | Multi-stage build for GHCR image |
| `.github/workflows/release.yaml` | CI/CD: build + push on tag |
| `catalog-info.yaml` | Backstage's own catalog entry |

---

## Task 1: Scaffold Backstage App

**Files:**
- Create: `~/Documents/workshop/claude/homelab-backstage/` (entire scaffold)

- [ ] **Step 1: Run create-app scaffold**

```bash
cd ~/Documents/workshop/claude
npx @backstage/create-app@latest homelab-backstage
```

When prompted:
- `name`: `homelab-backstage`
- `database`: `PostgreSQL` (we'll override with CNPG in production config)
- `package manager`: `yarn`

Expected: Scaffold completes, creates `homelab-backstage/` directory with monorepo structure.

- [ ] **Step 2: Verify scaffold structure**

```bash
ls ~/Documents/workshop/claude/homelab-backstage/
```

Expected output includes: `packages/`, `app-config.yaml`, `package.json`, `Dockerfile` (scaffold may include a basic one).

- [ ] **Step 3: Initialize git repo**

```bash
cd ~/Documents/workshop/claude/homelab-backstage
git init
git add .
git commit -m "feat: scaffold Backstage app with create-app"
```

- [ ] **Step 4: Verify app builds**

```bash
yarn install
yarn tsc
yarn build
```

Expected: All commands succeed with no errors.

---

## Task 2: Add Kubernetes Plugin

**Files:**
- Modify: `packages/app/package.json` — add `@backstage/plugin-kubernetes`, `@backstage/plugin-kubernetes-react`
- Modify: `packages/app/src/components/catalog/EntityPage.tsx` — add `EntityKubernetesContent`
- Modify: `packages/backend/package.json` — add `@backstage/plugin-kubernetes-backend` (may already be included)

- [ ] **Step 1: Install frontend Kubernetes plugin**

```bash
cd ~/Documents/workshop/claude/homelab-backstage
yarn --cwd packages/app add @backstage/plugin-kubernetes @backstage/plugin-kubernetes-react
```

- [ ] **Step 2: Install backend Kubernetes plugin**

```bash
yarn --cwd packages/backend add @backstage/plugin-kubernetes-backend
```

- [ ] **Step 3: Update EntityPage.tsx**

Read the current file first:

```bash
cat packages/app/src/components/catalog/EntityPage.tsx
```

Then add the Kubernetes content import and tab. The exact edit depends on the scaffold output, but the target state is:

```tsx
// Add import at top
import { EntityKubernetesContent } from '@backstage/plugin-kubernetes';

// Add to the entity page layout (inside the <EntityLayout> or <EntitySwitch>)
<EntityLayout.Route path="/kubernetes" title="Kubernetes">
  <EntityKubernetesContent />
</EntityLayout.Route>
```

- [ ] **Step 4: Commit**

```bash
git add packages/app/package.json packages/backend/package.json packages/app/src/components/catalog/EntityPage.tsx
git commit -m "feat: add Kubernetes plugin to frontend and backend"
```

- [ ] **Step 5: Verify build**

```bash
yarn tsc
yarn build
```

Expected: No TypeScript or build errors.

---

## Task 3: Add ArgoCD Plugin (Roadie)

**Files:**
- Modify: `packages/app/package.json` — add `@roadiehq/backstage-plugin-argo-cd`
- Modify: `packages/backend/package.json` — add `@roadiehq/backstage-plugin-argo-cd-backend`
- Modify: `packages/app/src/components/catalog/EntityPage.tsx` — add `EntityArgoCDContentCard`
- Create: `packages/backend/src/plugins/argocd.ts` — ArgoCD backend plugin router
- Modify: `packages/backend/src/index.ts` — register ArgoCD backend

- [ ] **Step 1: Install ArgoCD frontend plugin**

```bash
cd ~/Documents/workshop/claude/homelab-backstage
yarn --cwd packages/app add @roadiehq/backstage-plugin-argo-cd
```

- [ ] **Step 2: Install ArgoCD backend plugin**

```bash
yarn --cwd packages/backend add @roadiehq/backstage-plugin-argo-cd-backend
```

- [ ] **Step 3: Update EntityPage.tsx with ArgoCD card**

Add to the entity page:

```tsx
// Add import
import { EntityArgoCDContentCard } from '@roadiehq/backstage-plugin-argo-cd';

// Add card to entity page (e.g., in the overview tab or a dedicated ArgoCD tab)
<EntityLayout.Route path="/argocd" title="ArgoCD">
  <EntityArgoCDContentCard />
</EntityLayout.Route>
```

- [ ] **Step 4: Create ArgoCD backend plugin file**

Create `packages/backend/src/plugins/argocd.ts`:

```ts
import {
  createRouter,
  DefaultRouterOptions,
} from '@roadiehq/backstage-plugin-argo-cd-backend';
import { Router } from 'express';
import { PluginEnvironment } from '../types';

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  return createRouter({
    config: env.config,
    logger: env.logger,
    discovery: env.discovery,
  } as DefaultRouterOptions);
}
```

- [ ] **Step 5: Register ArgoCD backend in index.ts**

Read the current file:

```bash
cat packages/backend/src/index.ts
```

Add the ArgoCD plugin registration. The exact syntax depends on scaffold version, but target state:

```ts
// Add import
import argocd from './plugins/argocd';

// In the backend initialization
backend.add(argocd);
```

Or for the older pattern:

```ts
const argocdEnv = useHotMemoize(module, () => createEnv('argocd'));
apiRouter.use('/argocd', await argocd(argocdEnv));
```

- [ ] **Step 6: Commit**

```bash
git add packages/app/package.json packages/backend/package.json packages/app/src/components/catalog/EntityPage.tsx packages/backend/src/plugins/argocd.ts packages/backend/src/index.ts
git commit -m "feat: add ArgoCD plugin (frontend + backend)"
```

- [ ] **Step 7: Verify build**

```bash
yarn tsc
yarn build
```

Expected: No errors.

---

## Task 4: Create app-config.production.yaml

**Files:**
- Create: `app-config.production.yaml`

- [ ] **Step 1: Create production config**

Create `~/Documents/workshop/claude/homelab-backstage/app-config.production.yaml`:

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

- [ ] **Step 2: Commit**

```bash
git add app-config.production.yaml
git commit -m "feat: add production config with CNPG, K8s, ArgoCD, catalog wiring"
```

---

## Task 5: Write Dockerfile

**Files:**
- Modify: `Dockerfile` (replace scaffold version if present)

- [ ] **Step 1: Read current Dockerfile**

```bash
cat Dockerfile
```

- [ ] **Step 2: Replace with multi-stage build**

Write `~/Documents/workshop/claude/homelab-backstage/Dockerfile`:

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

- [ ] **Step 3: Test Docker build**

```bash
docker build -t homelab-backstage:test .
```

Expected: Build succeeds, no errors.

- [ ] **Step 4: Commit**

```bash
git add Dockerfile
git commit -m "feat: add multi-stage Dockerfile for GHCR image"
```

---

## Task 6: Write GitHub Actions Workflow

**Files:**
- Create: `.github/workflows/release.yaml`

- [ ] **Step 1: Create workflow directory**

```bash
mkdir -p ~/Documents/workshop/claude/homelab-backstage/.github/workflows
```

- [ ] **Step 2: Create release.yaml**

Write `~/Documents/workshop/claude/homelab-backstage/.github/workflows/release.yaml`:

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

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/release.yaml
git commit -m "feat: add GitHub Actions workflow for GHCR image publishing"
```

---

## Task 7: Create Backstage's Own Catalog Entry

**Files:**
- Create: `catalog-info.yaml`

- [ ] **Step 1: Create catalog-info.yaml**

Write `~/Documents/workshop/claude/homelab-backstage/catalog-info.yaml`:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: homelab-backstage
  description: Homelab Backstage companion app with Kubernetes and ArgoCD plugins
  annotations:
    github.com/project-slug: RizaSatya/homelab-backstage
spec:
  type: service
  lifecycle: production
  owner: user:default/riza
```

- [ ] **Step 2: Commit**

```bash
git add catalog-info.yaml
git commit -m "feat: add catalog-info.yaml for Backstage self-registration"
```

---

## Task 8: Create GitHub Repository and Push

**Files:**
- None (GitHub operations only)

- [ ] **Step 1: Create GitHub repo**

```bash
gh repo create RizaSatya/homelab-backstage --public --source=. --push
```

Expected: Repo created at `https://github.com/RizaSatya/homelab-backstage` and code pushed.

- [ ] **Step 2: Verify repo exists**

```bash
gh repo view RizaSatya/homelab-backstage
```

Expected: Shows repo details.

---

## Task 9: Tag v0.1.0 and Trigger First Build

**Files:**
- None (git tag + push only)

- [ ] **Step 1: Create and push tag**

```bash
git tag -a v0.1.0 -m "feat: initial release with Kubernetes and ArgoCD plugins"
git push origin v0.1.0
```

- [ ] **Step 2: Verify GitHub Actions workflow triggered**

```bash
gh run list --repo RizaSatya/homelab-backstage --limit 1
```

Expected: Shows a running or completed workflow for tag `v0.1.0`.

- [ ] **Step 3: Wait for workflow to complete, then verify image published**

```bash
gh run watch --repo RizaSatya/homelab-backstage
```

After completion:

```bash
docker manifest inspect ghcr.io/rizasatya/homelab-backstage:v0.1.0
```

Expected: Shows manifest for the published image.

---

## Task 10: Update homelab-infra values.yaml (Follow-Up)

**Files:**
- Modify: `~/Documents/workshop/claude/homelab/homelab-infra/platform/control-plane/backstage/values.yaml`

This task updates the GitOps repo to match the companion repo's PostgreSQL config. It should be done in a separate PR on the `idp-11/backstage` branch.

- [ ] **Step 1: Remove SQLite PVC config from values.yaml**

Remove from `values.yaml`:
```yaml
  extraVolumes:
    - name: backstage-data
      persistentVolumeClaim:
        claimName: backstage-data
  extraVolumeMounts:
    - name: backstage-data
      mountPath: /var/lib/backstage
```

- [ ] **Step 2: Add PostgreSQL environment variables**

Add to `values.yaml`:
```yaml
  envFrom:
    - secretRef:
        name: backstage-db-app
```

- [ ] **Step 3: Add CNPG Cluster manifest**

Create `platform/control-plane/backstage/runtime/backstage-db.yaml`:

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

- [ ] **Step 4: Commit**

```bash
git add platform/control-plane/backstage/values.yaml platform/control-plane/backstage/runtime/backstage-db.yaml
git commit -m "feat: switch Backstage from SQLite to PostgreSQL via CNPG"
```

---

## Spec Coverage Check

| Spec Section | Task(s) |
|--------------|---------|
| Section 1: Repository Structure | Task 1 (scaffold), Task 7 (catalog-info) |
| Section 2: Plugin Wiring | Task 2 (Kubernetes), Task 3 (ArgoCD) |
| Section 3: Configuration | Task 4 (app-config.production.yaml) |
| Section 4: CI/CD | Task 5 (Dockerfile), Task 6 (GitHub Actions) |
| Section 5: GitOps Contract | Task 8 (push), Task 10 (values.yaml update) |
| Section 6: CNPG Follow-Up | Task 10 (CNPG manifest + values update) |
| Section 7: Rollout Order | Tasks 1-9 (build), Task 10 (GitOps integration) |

All spec requirements covered.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-04-13-homelab-backstage-companion-repo-plan.md`. Two execution options:

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
