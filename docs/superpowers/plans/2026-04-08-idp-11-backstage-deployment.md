# Backstage Deployment Implementation Plan

**Phase 1: Base deployment with catalog**

This plan covers Phase 1: deploying Backstage with PostgreSQL and catalog autodiscovery. 

Phase 2 (separate future plan) will add Kubernetes and ArgoCD plugins.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy Backstage with PostgreSQL backend and catalog autodiscovery for Crossplane-provisioned apps.

**Architecture:** Backstage runs as a platform component in the `backstage` namespace, backed by PostgreSQL for persistence. The catalog auto-discovers apps from `apps/*/catalog-info.yaml` files in the homelab Git repository. Internal-only access via port-forward (no ingress in this phase).

**Tech Stack:** Backstage Helm chart (official), Bitnami PostgreSQL subchart, ArgoCD for GitOps.

**Scope:** Phase 1 - base deployment with catalog. Phase 2 (separate plan) - Kubernetes and ArgoCD plugins.

---

## File Structure

**Create:**
- `platform/backstage/values.yaml` - Helm values for Backstage chart
- `platform/backstage/rbac.yaml` - ServiceAccount and ClusterRoleBinding
- `platform/backstage/app-config.yaml` - Backstage application configuration (ConfigMap)
- `clusters/homelab/bootstrap/backstage-application.yaml` - ArgoCD Application manifest
- `apps/web-riza-crossplane/catalog-info.yaml` - Example catalog entity for Crossplane app
- `catalog-info.yaml` - Root catalog entities (System and Group)

---

## Task 1: Create Backstage Platform Directory

**Files:**
- Create: `platform/backstage/`

- [ ] **Step 1: Create directory**

```bash
mkdir -p platform/backstage
```

---

## Task 2: Create RBAC Manifests

**Files:**
- Create: `platform/backstage/rbac.yaml`

- [ ] **Step 1: Create ServiceAccount and ClusterRoleBinding**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backstage
  namespace: backstage
  labels:
    app.kubernetes.io/name: backstage
    app.kubernetes.io/component: backstage
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: backstage-read-only
  labels:
    app.kubernetes.io/name: backstage
    app.kubernetes.io/component: backstage
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: backstage
    namespace: backstage
```

- [ ] **Step 2: Commit RBAC**

```bash
git add platform/backstage/rbac.yaml
git commit -m "feat(backstage): add RBAC manifests for backstage service account"
```

---

## Task 3: Create App Config ConfigMap

**Files:**
- Create: `platform/backstage/app-config.yaml`

- [ ] **Step 1: Create ConfigMap with Backstage app configuration**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backstage-app-config
  namespace: backstage
  labels:
    app.kubernetes.io/name: backstage
    app.kubernetes.io/component: config
data:
  app-config.yaml: |
    organization:
      name: homelab

    backend:
      baseUrl: http://backstage.backstage.svc.cluster.local:7007
      listen:
        port: 7007

    catalog:
      locations:
        - type: url
          target: https://github.com/RizaSatya/homelab/blob/main/catalog-info.yaml
        
        - type: github-discovery
          target: https://github.com/RizaSatya/homelab
          rules:
            - pattern: apps/*/catalog-info.yaml
      
      rules:
        - allow: [Component, System, Group, Location]

    permission:
      enabled: false

    techdocs:
      builder: local
      generator:
        runIn: local
      publisher:
        type: local
```

- [ ] **Step 2: Commit app config**

```bash
git add platform/backstage/app-config.yaml
git commit -m "feat(backstage): add app config configmap"
```

---

## Task 4: Create Helm Values

**Files:**
- Create: `platform/backstage/values.yaml`

- [ ] **Step 1: Create Helm values file**

```yaml
backstage:
  image:
    repository: backstage
    pullPolicy: IfNotPresent
    tag: latest

  replicas: 1

  service:
    type: ClusterIP
    port: 7007

  ingress:
    enabled: false

  postgresql:
    enabled: true
    primary:
      persistence:
        enabled: true
        size: 8Gi
    auth:
      database: backstage

  serviceAccount:
    create: false
    name: backstage

  extraAppConfig:
    - configMapName: backstage-app-config
      filename: app-config.yaml

  livenessProbe:
    httpGet:
      path: /.backstage/health/v1/liveness
      port: 7007
    initialDelaySeconds: 60
    periodSeconds: 10
    failureThreshold: 3

  readinessProbe:
    httpGet:
      path: /.backstage/health/v1/readiness
      port: 7007
    initialDelaySeconds: 30
    periodSeconds: 10
    failureThreshold: 3

  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

- [ ] **Step 2: Commit values file**

```bash
git add platform/backstage/values.yaml
git commit -m "feat(backstage): add helm values for backstage deployment"
```

---

## Task 5: Create ArgoCD Application Manifest

**Files:**
- Create: `clusters/homelab/bootstrap/backstage-application.yaml`

- [ ] **Step 1: Create ArgoCD Application**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backstage
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "10"
spec:
  project: default
  sources:
    - repoURL: https://backstage.github.io/charts
      chart: backstage
      targetRevision: "2.0.0"
      helm:
        releaseName: backstage
        valueFiles:
          - $values/platform/backstage/values.yaml
    - repoURL: https://github.com/RizaSatya/homelab.git
      targetRevision: main
      ref: values
    - repoURL: https://github.com/RizaSatya/homelab.git
      targetRevision: main
      path: platform/backstage
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

- [ ] **Step 2: Commit ArgoCD Application**

```bash
git add clusters/homelab/bootstrap/backstage-application.yaml
git commit -m "feat(backstage): add argocd application manifest"
```

---

## Task 6: Create Root Catalog Entities

**Files:**
- Create: `catalog-info.yaml` (repository root)

- [ ] **Step 1: Create root catalog-info.yaml with System and Group**

```yaml
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: homelab
  description: Homelab Kubernetes Platform
  annotations:
    github.com/project-slug: RizaSatya/homelab
spec:
  owner: homelab-team
---
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: homelab-team
  description: Homelab Platform Team
spec:
  type: team
  children: []
---
apiVersion: backstage.io/v1alpha1
kind: Location
metadata:
  name: homelab-catalog
  description: Root location for homelab catalog
spec:
  targets:
    - ./apps/*/catalog-info.yaml
```

- [ ] **Step 2: Commit root catalog**

```bash
git add catalog-info.yaml
git commit -m "feat(backstage): add root catalog entities (system and group)"
```

---

## Task 7: Create Example Catalog Entity for Crossplane App

**Files:**
- Create: `apps/web-riza-crossplane/catalog-info.yaml`

- [ ] **Step 1: Create catalog entity for web-riza-crossplane**

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: web-riza-crossplane
  description: Crossplane-managed web application
  annotations:
    backstage.io/techdocs-ref: dir:.
    github.com/project-slug: RizaSatya/homelab
spec:
  type: website
  lifecycle: production
  owner: homelab-team
  system: homelab
```

- [ ] **Step 2: Commit catalog entity**

```bash
git add apps/web-riza-crossplane/catalog-info.yaml
git commit -m "feat(backstage): add catalog entity for web-riza-crossplane"
```

---

## Task 8: Push and Deploy

- [ ] **Step 1: Push all commits to Git**

```bash
git push origin main
```

- [ ] **Step 2: Wait for ArgoCD to sync**

Run: `argocd app get backstage --refresh`

Wait for `Health: Healthy` and `Status: Synced`

Expected: ArgoCD creates namespace `backstage`, Deployments for Backstage and PostgreSQL

- [ ] **Step 3: Verify Backstage pod is running**

```bash
kubectl get pods -n backstage
```

Expected output:
```
NAME                                  READY   STATUS    RESTARTS   AGE
backstage-<hash>                      1/1     Running   0          <time>
backstage-postgresql-0                 1/1     Running   0          <time>
```

- [ ] **Step 4: Check Backstage startup logs**

```bash
kubectl logs -n backstage -l app.kubernetes.io/name=backstage --tail=100
```

Expected: Backstage started successfully, listening on port 7007, catalog in sync

- [ ] **Step 5: Port-forward to access Backstage**

```bash
kubectl port-forward -n backstage svc/backstage 7007:7007
```

Access: `http://localhost:7007`

Expected: Backstage UI loads with default page

- [ ] **Step 6: Verify catalog entities**

Access: `http://localhost:7007/catalog`

Expected:
- System `homelab` appears in catalog
- Group `homelab-team` appears in catalog
- Component `web-riza-crossplane` appears in catalog

---

## Task 9: Verify Git Discovery

- [ ] **Step 1: Check Backstage logs for catalog discovery**

```bash
kubectl logs -n backstage -l app.kubernetes.io/name=backstage | grep -i catalog
```

Expected: Logs showing catalog location processing and entity discovery

- [ ] **Step 2: Navigate to web-riza-crossplane component**

Access: `http://localhost:7007/catalog/component/default/component/web-riza-crossplane`

Expected: Component page shows metadata (owner: homelab-team, system: homelab, type: website, lifecycle: production)

---

## Success Criteria

- [ ] Backstage pod running in `backstage` namespace
- [ ] PostgreSQL pod running in `backstage` namespace
- [ ] Catalog shows `homelab` system
- [ ] Catalog shows `homelab-team` group
- [ ] Catalog shows `web-riza-crossplane` component
- [ ] git discovery configured for `apps/*/catalog-info.yaml`
- [ ] Port-forward works (`kubectl port-forward -n backstage svc/backstage 7007:7007`)

---

## Future Work (Part 2)

**Kubernetes Plugin:**
- Build custom Backstage image with `@backstage/plugin-kubernetes-backend`
- Configure kubernetes cluster locator in app-config
- RBAC already configured (ClusterRoleBinding for view)

**ArgoCD Plugin:**
- Build custom Backstage image with `@backstage/plugin-argo-cd`
- Create Secret for ArgoCD admin credentials
- Configure argocd instance in app-config
- Update values.yaml to mount credentials

**Ingress:**
- Add Ingress with TLS for external access
- Configure hostname `backstage.homelab.rizaes.com`

**Scaffolder:**
- Create templates for common patterns
- Integrate with Crossplane WebApp creation

**TechDocs:**
- Enable TechDocs for component documentation

---

## Notes

**Why no custom image in Part 1:**
The official `backstage` image includes the core catalog functionality. Kubernetes and ArgoCD plugins require building a custom image with additional dependencies. This is deferred to Part 2 to keep the first rollout focused.

**Why manual catalog-info.yaml:**
Crossplane compositions generate Kubernetes resources, not Git files. catalog-info.yaml files are created manually when provisioning WebApps. This keeps the first rollout simple and GitOps-native.

**Why no ingress:**
First rollout is internal-only via port-forward. Ingress with TLS is deferred toPart 2.

**PostgreSQL persistence:**
Uses Bitnami PostgreSQL subchart with 8Gi PVC. Credentials auto-generated and stored in Secret.