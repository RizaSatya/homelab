# Backstage Deployment Guide

> Step-by-step guide for deploying Backstage as an internal developer portal (IDP) on Kubernetes with ArgoCD and PostgreSQL.

## Overview

We deployed Backstage with:
- **Companion repo** (`homelab-backstage`) — Builds the custom Backstage image with Kubernetes and ArgoCD plugins
- **GitOps repo** (`homelab-infra`) — Manages deployment via ArgoCD with Helm charts and Kubernetes manifests

---

## Step 1: Scaffold the Backstage App

**Why:** The companion repo provides a custom Backstage image. We start with the official scaffold.

```bash
npx @backstage/create-app@latest
```

**Key files created:**
- `packages/app/` — Frontend (React)
- `packages/backend/` — Backend (Node.js)
- `app-config.yaml` — Dev config
- `app-config.production.yaml` — Production config
- `packages/backend/Dockerfile` — Docker build file

---

## Step 2: Add Kubernetes Plugin

**Why:** Lets Backstage display Kubernetes resources (pods, deployments, services) for each entity in the catalog.

```bash
cd ~/Documents/workshop/claude/homelab-backstage
yarn --cwd packages/app add @backstage/plugin-kubernetes @backstage/plugin-kubernetes-react
yarn --cwd packages/backend add @backstage/plugin-kubernetes-backend
```

**Note:** The scaffold already included the Kubernetes plugin. Just verify it's in `packages/backend/src/index.ts`:
```ts
backend.add(import('@backstage/plugin-kubernetes-backend'));
```

---

## Step 3: Add ArgoCD Plugin

**Why:** Shows ArgoCD application status, sync history, and health directly in Backstage's UI.

```bash
yarn --cwd packages/app add @roadiehq/backstage-plugin-argo-cd
yarn --cwd packages/backend add @roadiehq/backstage-plugin-argo-cd-backend
```

---

## Step 4: Configure Production Config

**Why:** The production config tells Backstage how to connect to PostgreSQL, Kubernetes, ArgoCD, and the catalog.

Create `app-config.production.yaml` in the companion repo:

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
      user: ${username}        # CNPG secret key
      password: ${password}    # CNPG secret key
      database: ${dbname}      # CNPG secret key
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

integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}  # Optional, for rate limits

catalog:
  rules:
    - allow: [Component, Group, Location, User]
  locations:
    - type: url
      target: https://raw.githubusercontent.com/.../root.yaml

argocd:
  username: backstage
  password: ${ARGOCD_AUTH_TOKEN}
  baseUrl: https://argocd-server.argocd.svc.cluster.local
```

---

## Step 5: Set Up GitHub Actions CI/CD

**Why:** Automatically builds and pushes the Backstage image to GHCR when you tag a release.

Create `.github/workflows/release.yaml`:

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
          node-version: 24
          cache: yarn
      - run: yarn install --immutable
      - run: yarn tsc
      - run: yarn build:backend
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          file: packages/backend/Dockerfile
          push: true
          tags: ghcr.io/username/homelab-backstage:${{ github.ref_name }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Build and push:**
```bash
git tag -a v0.1.0 -m "initial release"
git push origin v0.1.0
```

---

## Step 6: Create ArgoCD Application

**Why:** ArgoCD watches the GitOps repo and deploys Backstage to the cluster automatically.

Create `clusters/homelab/bootstrap/backstage-application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backstage
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://backstage.github.io/charts
      chart: backstage
      targetRevision: 2.6.3
      helm:
        releaseName: backstage
        valueFiles:
          - $values/platform/control-plane/backstage/values.yaml
    - repoURL: https://github.com/username/homelab.git
      targetRevision: main
      ref: values
    - repoURL: https://github.com/username/homelab.git
      targetRevision: main
      path: platform/control-plane/backstage/runtime
      directory:
        recurse: true
    - repoURL: https://github.com/username/homelab.git
      targetRevision: main
      path: platform/control-plane/backstage/argocd
      directory:
        recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: backstage
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

---

## Step 7: Create CNPG Cluster

**Why:** CloudNativePG provides a managed PostgreSQL database in Kubernetes. It automatically creates secrets with connection credentials.

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
    storageClass: local-path    # Use storageClass, NOT storageClassName
  bootstrap:
    initdb:
      database: backstage
      owner: backstage
```

**Important:** CNPG uses `storageClass` (not `storageClassName`). Using the wrong field name will cause sync errors.

---

## Step 8: Configure ArgoCD RBAC

**Why:** The ArgoCD plugin needs a local account to read application status from the ArgoCD API.

**Create account** (`platform/control-plane/backstage/argocd/argocd-cm-backstage-account.yaml`):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  accounts.backstage: apiKey
  accounts.backstage.enabled: "true"
```

**Create RBAC policy** (`platform/control-plane/backstage/argocd/argocd-rbac-cm-backstage-policy.yaml`):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, backstage, applications, get, */*, allow
    p, backstage, applications, action/*, */*, allow
```

---

## Step 9: Create Kubernetes RBAC

**Why:** The Kubernetes plugin needs permissions to read cluster resources (pods, deployments, etc.).

Create `platform/control-plane/backstage/rbac/backstage-kubernetes-reader.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backstage
  namespace: backstage
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backstage-kubernetes-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "deployments", "replicasets", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["postgresql.cnpg.io"]
    resources: ["clusters"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["platform.rizaes.com"]
    resources: ["webapps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: backstage-kubernetes-reader
subjects:
  - kind: ServiceAccount
    name: backstage
    namespace: backstage
roleRef:
  kind: ClusterRole
  name: backstage-kubernetes-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Step 10: Create SealedSecret

**Why:** Sensitive credentials (ArgoCD token, backend secret) need to be encrypted before committing to Git.

**Generate the ArgoCD token:**
```bash
# From within the cluster
kubectl run argocd-client --rm -i --restart=Never --image=quay.io/argoproj/argocd:v3.3.4 -- \
  sh -c 'argocd login argocd-server.argocd.svc.cluster.local --username admin --password <admin-password> --plaintext --insecure \
  && argocd account generate-token --account backstage'
```

**Generate the backend secret:**
```bash
openssl rand -hex 32
```

**Create the SealedSecret:**
```bash
kubectl create secret generic backstage-secrets \
  --namespace backstage \
  --from-literal=ARGOCD_AUTH_TOKEN="<token>" \
  --from-literal=BACKEND_SECRET="<secret>" \
  --dry-run=client -o yaml | kubeseal -o yaml
```

Save the output to `platform/control-plane/backstage/runtime/backstage-sealedsecret.yaml`.

---

## Step 11: Configure Helm Values

**Why:** The `values.yaml` configures the Backstage Helm chart, including image, database, and secrets.

Create `platform/control-plane/backstage/values.yaml`:

```yaml
fullnameOverride: backstage

serviceAccount:
  create: false
  name: backstage
  automountServiceAccountToken: true

backstage:
  replicas: 1
  image:
    repository: ghcr.io/username/homelab-backstage
    tag: v0.1.0
    pullPolicy: IfNotPresent
  appConfig:
    # ... (database, kubernetes, catalog, argocd config)
  extraEnvVarsSecrets:
    - backstage-db-app      # CNPG-generated secret
    - backstage-secrets     # SealedSecret with ArgoCD token, backend secret
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

service:
  type: ClusterIP

ingress:
  enabled: false

postgresql:
  enabled: false
```

**Important:** Use `extraEnvVarsSecrets` (Helm chart format), NOT `envFrom` (raw Kubernetes format). The Helm chart doesn't support `envFrom`.

---

## Common Issues & Solutions

### 1. Helm chart schema error: `envFrom` not allowed

**Problem:** Using `envFrom` under `backstage` in values.yaml.

**Solution:** Use `extraEnvVarsSecrets` instead:
```yaml
backstage:
  extraEnvVarsSecrets:
    - backstage-db-app
    - backstage-secrets
```

### 2. CNPG field error: `storageClassName` not declared in schema

**Problem:** Using `storageClassName` in CNPG Cluster spec.

**Solution:** Use `storageClass` (no "Name" suffix):
```yaml
spec:
  storage:
    storageClass: local-path    # Correct
    # storageClassName: local-path  # Wrong!
```

### 3. Database connection error: `no PostgreSQL user name specified`

**Problem:** Environment variable names don't match CNPG secret keys.

**Solution:** CNPG secret uses `username`, `password`, `dbname` (not `POSTGRES_USER`, etc.):
```yaml
database:
  connection:
    user: ${username}        # Matches CNPG secret key
    password: ${password}
    database: ${dbname}
```

### 4. Database permission error: `permission denied to create database`

**Problem:** Backstage tries to create separate databases for each plugin (`backstage_plugin_app`, `backstage_plugin_catalog`, etc.) but the PostgreSQL user doesn't have `CREATEDB` permission.

**Error:**
```
Failed to connect to the database to make sure that 'backstage_plugin_app' exists, error: CREATE DATABASE "backstage_plugin_app" - permission denied to create database
```

**Solution:** Grant `CREATEDB` to the database owner via `postInitApplicationSQL` in the CNPG Cluster manifest. This only runs during initial bootstrap, so you must delete and recreate the cluster:

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
    storageClass: local-path
  bootstrap:
    initdb:
      database: backstage
      owner: backstage
      postInitApplicationSQL:
        - ALTER USER backstage CREATEDB;
```

Apply the change:
```bash
kubectl delete cluster backstage-db -n backstage
kubectl delete pvc backstage-db-1 -n backstage
```

ArgoCD will recreate the cluster with the new configuration. Then restart the Backstage deployment:
```bash
kubectl rollout restart deployment/backstage -n backstage
```

Verify the permission:
```bash
kubectl exec -n backstage backstage-db-1 -- psql -U postgres -d backstage -c "SELECT rolname, rolcreatedb FROM pg_roles WHERE rolname = 'backstage';"
```

Expected output: `rolcreatedb = t`

### 5. Readiness probe failing (503)

**Problem:** Backstage health check returns 503 when backend fails to start.

**Solution:** Check pod logs for database or configuration errors:
```bash
kubectl logs -n backstage <pod-name> | head -50
```

### 5. ArgoCD can't access private GitHub repo

**Problem:** ArgoCD fails with `failed to get git client for repo`.

**Solution:** Either make the repo public, or add repository credentials:
```bash
argocd repo add https://github.com/username/repo.git --username <user> --password <token>
```

### 6. Catalog entries not appearing

**Problem:** Entities defined in `catalog-info.yaml` files aren't showing up.

**Solution:** Make sure the file is listed in the catalog locations:
```yaml
catalog:
  locations:
    - type: url
      target: https://raw.githubusercontent.com/.../root.yaml
```

And `root.yaml` references the `catalog-info.yaml` files:
```yaml
spec:
  type: url
  targets:
    - https://raw.githubusercontent.com/.../catalog-info.yaml
```

---

## File Structure Summary

### Companion Repo (`homelab-backstage/`)
```
├── packages/
│   ├── app/
│   │   └── src/App.tsx              # Register frontend plugins
│   └── backend/
│       ├── src/index.ts             # Register backend plugins
│       └── Dockerfile               # Image build
├── app-config.yaml                  # Dev config
├── app-config.production.yaml       # Production config (baked into image)
└── .github/workflows/release.yaml   # CI/CD
```

### GitOps Repo (`homelab-infra/`)
```
platform/control-plane/backstage/
├── argocd/
│   ├── argocd-cm-backstage-account.yaml   # ArgoCD local account
│   └── argocd-rbac-cm-backstage-policy.yaml # ArgoCD RBAC
├── rbac/
│   └── backstage-kubernetes-reader.yaml   # K8s RBAC
├── runtime/
│   ├── backstage-db.yaml                  # CNPG Cluster
│   └── backstage-sealedsecret.yaml        # Encrypted secrets
├── catalog/
│   ├── root.yaml                          # Catalog root location
│   └── org.yaml                           # User/team definitions
└── values.yaml                            # Helm values
```

---

## Environment Variables Reference

| Variable | Source | Purpose |
|----------|--------|---------|
| `username` | CNPG secret `backstage-db-app` | PostgreSQL username |
| `password` | CNPG secret `backstage-db-app` | PostgreSQL password |
| `dbname` | CNPG secret `backstage-db-app` | PostgreSQL database name |
| `BACKEND_SECRET` | SealedSecret `backstage-secrets` | Backend session signing |
| `ARGOCD_AUTH_TOKEN` | SealedSecret `backstage-secrets` | ArgoCD API authentication |
| `GITHUB_TOKEN` | Optional SealedSecret | GitHub API rate limits |

---

## Verification

After deployment, verify Backstage is running:

```bash
kubectl get pods -n backstage
kubectl logs -n backstage <pod-name> | head -20
```

Port-forward to access the UI:

```bash
kubectl port-forward svc/backstage -n backstage 7007:7007
```

Open http://localhost:7007 in your browser.
