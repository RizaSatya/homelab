# IDP-6 Repo Structure Notes

## Context

`IDP-5` introduced a working GitOps root and tenant app discovery contract, but the repository layout was still transitional.

The main problems left after `IDP-5` were:

- tenant manifests existed twice, once in `apps/<name>/` and again in `apps/<name>/app/`
- some platform-owned utilities were still living under `apps/`
- the repo shape was functional, but not yet a clean base for later Backstage and Crossplane work

`IDP-6` cleans that up without changing the live GitOps model again.

## Repo rules after IDP-6

### 1. `clusters/` is only for cluster entrypoints

`clusters/homelab/` now remains the place for:

- the root Argo CD bootstrap object
- child bootstrap entries such as explicit platform `Application` manifests
- shared cluster-level entrypoints

It should not become a dumping ground for workload manifests.

### 2. `apps/` is only for tenant workloads

`apps/` now means onboarded tenant workloads that fit the tenant app contract:

```text
apps/<app-name>/app/
```

That path is what the tenant `ApplicationSet` discovers.

Current tenant apps:

- [apps/web-riza/app](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/apps/web-riza/app)
- [apps/garmin-scraper/app](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/apps/garmin-scraper/app)

The parent app directory can hold future metadata such as:

- Backstage catalog files
- app-specific docs
- ownership notes

But the live runtime manifests now belong only under `app/`.

### 3. `platform/` is for shared capabilities and operator tools

Shared platform capabilities live under `platform/`.

That includes:

- operators and shared services like `cloudnative-pg`
- observability components
- networking/connectivity tooling like `cloudflared`
- operator-facing tools like `pgweb`

For `IDP-6`, two components moved out of `apps/` and into `platform/`:

- [platform/networking/cloudflared](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/networking/cloudflared)
- [platform/utilities/pgweb](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/utilities/pgweb)

## Why `pgweb` is treated as platform

`pgweb` is not being modeled as a tenant app because it behaves more like an operator utility:

- it lives in a shared `database-tools` namespace
- it is used to inspect databases across workloads
- it is not the product workload being onboarded through the future golden path

That makes it a better fit for explicit platform management than tenant app auto-discovery.

## Why this matters

This cleanup gives later phases a more honest target:

- Backstage onboarding can aim at `apps/<name>/app`
- Crossplane platform APIs can later decide what to generate into app folders
- platform services no longer compete conceptually with tenant apps

This is not the flashy part of the IDP, but it is the part that makes later self-service workflows less confusing.
