# GitOps Root and App Discovery

## Why this exists

This repo now has one explicit GitOps bootstrap path for the homelab cluster:

- [clusters/homelab/root-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/root-application.yaml)
- [clusters/homelab/bootstrap](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap)

The immediate goal is to remove the old failure mode where adding a new Argo CD child `Application` to Git did nothing until it was also manually applied to the cluster.

## New bootstrap contract

After Argo CD itself exists in the cluster, the only manual bootstrap object is:

- [clusters/homelab/root-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/root-application.yaml)

That root app watches only:

- [clusters/homelab/bootstrap](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap)

Inside that bootstrap path:

- platform control entries stay as explicit child `Application` manifests
- tenant app discovery is handled by [clusters/homelab/bootstrap/apps-applicationset.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/apps-applicationset.yaml)

## Tenant app discovery contract

An app is discoverable when this path exists:

```text
apps/<app-name>/app/
```

The generated Argo CD `Application` uses:

- application name: `<app-name>`
- source path: `apps/<app-name>/app`
- destination namespace: `<app-name>`

For the current repo, the first apps aligned to that contract are:

- [apps/web-riza/app](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/apps/web-riza/app)
- [apps/garmin-scraper/app](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/apps/garmin-scraper/app)

These `app/` directories are transitional Argo entrypoints for auto-discovery. They intentionally keep the parent app folders in place so the broader repo reorganization can happen in a later issue.

## Platform control entries

Platform components are still explicit child apps because they have different namespaces, chart sources, and sync behavior:

- [clusters/homelab/bootstrap/cloudflared-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/cloudflared-application.yaml)
- [clusters/homelab/bootstrap/cloudnative-pg-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/cloudnative-pg-application.yaml)
- [clusters/homelab/bootstrap/kube-prometheus-stack-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/kube-prometheus-stack-application.yaml)
- [clusters/homelab/bootstrap/opentelemetry-collector-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/opentelemetry-collector-application.yaml)
- [clusters/homelab/bootstrap/pgweb-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/pgweb-application.yaml)

## Bootstrap and migration notes

Bootstrap a fresh cluster with:

```bash
kubectl apply -n argocd -f clusters/homelab/root-application.yaml
```

Important migration note for an existing cluster:

- this repo no longer treats `argocd/*.yaml` as the source of truth for child applications
- if legacy tenant `Application` objects such as `web-riza` or `garmin-scraper` already exist in the cluster, plan the cutover before enabling the new `ApplicationSet`
- the repo structure is ready for auto-discovery, but live cluster adoption should be done deliberately so old manually managed `Application` objects do not conflict with generated ones

## What this issue does not do

- move all workloads into final `clusters/` and `platform/` target layouts
- introduce Crossplane, Kyverno, or Backstage
- convert app manifests to platform claims
- add Argo CD projects or tenant isolation
