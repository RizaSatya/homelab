# Garmin Scraper CloudNativePG Design

## Context

This design adds a PostgreSQL database for the `garmin-scraper` application using CloudNativePG.

The repository already follows a simple GitOps pattern:

- each application lives under `apps/<name>`
- Argo CD `Application` manifests live under `argocd/`
- application secrets are currently managed with Bitnami `SealedSecret`

There is already an Argo CD application manifest for `garmin-scraper` pointing at `apps/garmin-scraper`, but that application folder does not exist yet. This design fills in that missing application folder and introduces a per-app PostgreSQL cluster managed by CloudNativePG.

The workload itself is a `CronJob` that runs once per day in the `Asia/Jakarta` time zone and expects a single `DATABASE_URL` environment variable from the `garmin-sync-secrets` secret.

Naming decisions for this design:

- app folder and namespace: `garmin-scraper`
- Argo CD application name: `garmin-scraper`
- database cluster name: `garmin-scraper-db`
- CronJob name: `garmin-sync`
- runtime secret name: `garmin-sync-secrets`

## Goals

- deploy a PostgreSQL database for `garmin-scraper` using CloudNativePG
- keep the database scoped to the `garmin-scraper` namespace
- preserve the current repo pattern of app-owned manifests in `apps/garmin-scraper`
- keep the application contract simple by continuing to provide `DATABASE_URL`
- use CloudNativePG-generated credentials as the bootstrap source of truth
- manually copy the generated connection details into a Git-managed `SealedSecret`
- document a clear onboarding flow from first sync to first successful job run

## Non-Goals

- introduce shared Postgres infrastructure for multiple apps
- automate secret mirroring from CloudNativePG into `SealedSecret`
- add high-availability Postgres replicas in the first version
- add backup or point-in-time recovery in the first version
- redesign the application to consume separate `PGHOST`, `PGUSER`, or `PGPASSWORD` variables

## Recommended Approach

Deploy a dedicated single-instance CloudNativePG cluster inside the `garmin-scraper` namespace and bootstrap one database owner for the application. CloudNativePG will generate the application user secret during bootstrap. That generated secret will be used only as the initial source for the connection details. The long-lived application contract in Git will remain a manually maintained `SealedSecret` named `garmin-sync-secrets` that contains `DATABASE_URL` together with the other application secrets.

This approach is recommended because it matches the existing GitOps style in the repository, avoids coupling `garmin-scraper` directly to CloudNativePG secret formats, and keeps the workload manifest simple. It also keeps future credential rotation possible: the operator can generate or manage the real database credentials, while the GitOps secret can be refreshed intentionally after rotation.

## Architecture

The runtime flow is:

`Argo CD -> namespace + CNPG cluster + CronJob + SealedSecret -> CloudNativePG bootstraps Postgres -> app reads DATABASE_URL -> CronJob writes to Postgres`

The credential flow is:

`CloudNativePG generated app secret -> manual extraction -> DATABASE_URL assembly -> kubeseal -> garmin-sync-secrets SealedSecret`

Component responsibilities:

- Argo CD syncs the `garmin-scraper` application manifests from Git
- the `garmin-scraper` namespace isolates the workload and database
- the CloudNativePG operator reconciles the `Cluster` resource and manages Postgres pods, services, and PVCs
- the `Cluster` manifest defines the database name, owner user, storage, and instance count
- the CNPG-generated secret provides bootstrap credentials for the application owner
- the app `SealedSecret` provides the stable runtime secret contract consumed by the `CronJob`
- the `CronJob` runs the `garmin-sync` container on schedule and connects to Postgres using `DATABASE_URL`

## Platform Components

### 1. Argo CD Application

The existing Argo CD application manifest should remain the top-level entry point for this app.

Responsibilities:

- point to `apps/garmin-scraper`
- target the `garmin-scraper` namespace
- automate sync and self-healing
- create the namespace if it does not exist

Existing file:

- `argocd/garmin-scraper-application.yaml`

### 2. Namespace

Create a namespace manifest for the application.

Responsibilities:

- define the `garmin-scraper` namespace explicitly in Git
- provide a stable home for both the database and the workload

Expected file:

- `apps/garmin-scraper/namespace.yaml`

### 3. CloudNativePG Cluster

Create a dedicated Postgres cluster manifest for the application.

Responsibilities:

- run PostgreSQL 16 as a single instance initially
- create one writable primary database pod
- allocate persistent storage using the existing `local-path` storage class
- bootstrap database `garmin`
- bootstrap owner user `garmin_app`
- expose the standard CloudNativePG read-write service for the application

Expected file:

- `apps/garmin-scraper/cluster.yaml`

Resource name:

- `Cluster.metadata.name: garmin-scraper-db`

### 4. Application SealedSecret

Create a `SealedSecret` for application runtime secrets.

Responsibilities:

- store `GARMIN_EMAIL`
- store `GARMIN_PASSWORD`
- store `FERNET_KEY`
- store `DATABASE_URL`
- remain the only secret directly referenced by the `CronJob`

Expected file:

- `apps/garmin-scraper/sealedsecret.yaml`

Resource name:

- `SealedSecret.metadata.name: garmin-sync-secrets`

### 5. CronJob

Create a `CronJob` manifest for the scraper workload.

Responsibilities:

- run daily at `23:59` in `Asia/Jakarta`
- prevent overlapping runs with `concurrencyPolicy: Forbid`
- read credentials and application configuration from `garmin-sync-secrets`
- run with `restartPolicy: Never`

Expected file:

- `apps/garmin-scraper/cronjob.yaml`

Resource name:

- `CronJob.metadata.name: garmin-sync`

## CloudNativePG Configuration

The first database rollout should stay intentionally small and explicit.

Recommended cluster settings:

- `instances: 1`
- `imageName`: use the default operator-managed PostgreSQL image unless the cluster requires pinning
- `storage.size: 5Gi`
- `storage.storageClass: local-path`
- database name: `garmin`
- owner user: `garmin_app`

This is sufficient for a homelab cron workload and keeps failure modes simple. The database can be scaled or backed up later once the ingestion path is proven useful.

## Service And Connection Design

CloudNativePG will create a read-write service for the cluster. For a cluster named `garmin-scraper-db`, the expected writable service DNS name will be:

- `garmin-scraper-db-rw.garmin-scraper.svc.cluster.local`

The application connection string should therefore take the form:

- `postgresql://garmin_app:<password>@garmin-scraper-db-rw.garmin-scraper.svc.cluster.local:5432/garmin`

The `CronJob` should continue consuming this through the existing `DATABASE_URL` environment variable rather than being coupled directly to multiple CNPG secret keys.

## Secret Handoff Model

The secret handoff is intentionally manual in this first version.

Bootstrap sequence:

1. Argo CD syncs the namespace, CNPG cluster, and CronJob manifests.
2. CloudNativePG creates the Postgres cluster and generates the owner secret for `garmin_app`.
3. The operator also creates the `-rw` service used by applications.
4. An operator user retrieves the generated username and password from the CNPG secret.
5. The operator user assembles `DATABASE_URL` using the `-rw` service DNS name.
6. The operator user creates or updates the plaintext secret manifest locally and seals it with `kubeseal`.
7. The resulting `SealedSecret` is committed to Git and synced by Argo CD.
8. The `CronJob` can then read `DATABASE_URL` from `garmin-sync-secrets`.

This keeps CloudNativePG in charge of the live database bootstrap while preserving the current GitOps secret management workflow used in the repository.

## GitOps Layout

The new application files should be organized as:

- `apps/garmin-scraper/namespace.yaml`
- `apps/garmin-scraper/cluster.yaml`
- `apps/garmin-scraper/cronjob.yaml`
- `apps/garmin-scraper/sealedsecret.yaml`

The top-level Argo CD application stays at:

- `argocd/garmin-scraper-application.yaml`

This layout matches the repository's existing convention where app-specific resources are owned together in one folder and Argo CD points at that folder as the deployment unit.

The CloudNativePG operator itself is a cluster prerequisite for this design. Installing the operator is part of onboarding, but the first implementation pass does not require adding a platform-level operator installation manifest to this repository unless the user explicitly wants the operator GitOps-managed here as well.

## Resource And Availability Strategy

Initial database characteristics:

- one Postgres instance
- one PVC
- `5Gi` of storage on `local-path`
- no synchronous replicas
- no backups in the first version

Initial workload characteristics:

- one scheduled run per day
- one job at a time
- failed jobs retained for debugging, but limited history

This is the simplest possible architecture that still uses a real operator-managed database and a stable GitOps workflow.

## Error Handling And Debugging Model

Expected verification order:

1. confirm the CloudNativePG operator is installed and healthy
2. confirm Argo CD syncs `garmin-scraper` successfully
3. confirm the namespace exists
4. confirm the CNPG `Cluster` resource reaches healthy status
5. confirm the Postgres pod is ready
6. confirm the `garmin-scraper-db-rw` service exists
7. confirm the generated CNPG secret exists
8. build and seal the final `garmin-sync-secrets` manifest
9. confirm the `CronJob` references the expected secret keys
10. create a one-off job from the `CronJob` and inspect logs for database connectivity

Common failure modes:

- the CloudNativePG operator is not installed, so the `Cluster` resource never reconciles
- the chosen storage class is unavailable in the cluster
- the application secret is applied before `DATABASE_URL` is assembled correctly
- the `CronJob` image tag is still a placeholder
- the app expects SSL or connection parameters not included in the first `DATABASE_URL`
- a future CNPG credential rotation happens without updating the Git-managed `SealedSecret`

## Security And Scope

This first version keeps everything internal to the cluster:

- Postgres is not exposed externally
- the application connects through the cluster-local CNPG service
- database credentials are still present in a GitOps-managed secret, but only in sealed form
- credential copying is a deliberate operator action, which makes bootstrap and rotation explicit

The main trade-off is that the Git-managed sealed secret can drift from the live CNPG-generated credentials if they are rotated without updating Git. That is acceptable for the initial homelab workflow, but it should be documented operationally.

## Onboarding Runbook

Recommended onboarding sequence:

1. install the CloudNativePG operator in the cluster if it is not already present
2. commit the `garmin-scraper` application manifests except the final sealed secret payload
3. sync the Argo CD application
4. wait for the CNPG cluster to become healthy
5. retrieve the CNPG-generated app credentials
6. construct `DATABASE_URL` using the CNPG `-rw` service DNS name
7. generate the final `SealedSecret` for `garmin-sync-secrets`
8. commit and sync the sealed secret
9. trigger a one-off `Job` from the `CronJob` for first verification
10. confirm the application can connect and write to Postgres

Operational notes:

- if credentials are rotated later, rebuild and reseal `DATABASE_URL`
- if the database service name changes, rebuild and reseal `DATABASE_URL`
- if the app later needs migrations, those should be designed as a separate job or init workflow rather than folded into this first cron deployment

## Testing And Verification

Definition of done for implementation:

- `apps/garmin-scraper` exists and contains namespace, CNPG cluster, CronJob, and `SealedSecret` manifests
- Argo CD syncs the `garmin-scraper` application successfully
- the CNPG cluster becomes ready in the `garmin-scraper` namespace
- the CNPG writable service is reachable
- `garmin-sync-secrets` exists with the expected keys
- a manually triggered one-off run of the CronJob can connect to the database successfully

Suggested manual verification commands after implementation:

1. `kubectl get applications -n argocd`
2. `kubectl get cluster -n garmin-scraper`
3. `kubectl get pods -n garmin-scraper`
4. `kubectl get svc -n garmin-scraper`
5. `kubectl get secret -n garmin-scraper`
6. `kubectl create job --from=cronjob/garmin-sync garmin-sync-manual-$(date +%s) -n garmin-scraper`
7. `kubectl logs job/<manual-job-name> -n garmin-scraper`

## Future Expansion

This design intentionally leaves room for later work:

- add backups with CNPG-managed backup configuration
- add a second database instance for higher availability
- automate secret propagation into the app secret contract
- add a migration job if schema management becomes necessary
- move from `DATABASE_URL` to structured database environment variables if the application grows more complex

## Decision Summary

`garmin-scraper` will use a dedicated CloudNativePG-managed PostgreSQL cluster in its own namespace. CloudNativePG will bootstrap the database and generate the initial application credentials. Those credentials will then be manually converted into the Git-managed `garmin-sync-secrets` `SealedSecret`, which remains the only secret consumed by the scheduled scraper `CronJob`. This design fits the current repository structure, keeps the first rollout small, and provides a clear operational onboarding path.
