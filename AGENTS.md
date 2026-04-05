# AGENTS.md

This repository manages a homelab Kubernetes platform with Argo CD and Crossplane.

## Repo map

- `clusters/homelab/`: cluster entrypoints and Argo CD bootstrap objects
- `clusters/homelab/bootstrap/`: child Argo CD `Application` manifests and the tenant app `ApplicationSet`
- `apps/<app-name>/app/`: tenant workload manifests discovered automatically by `homelab-apps`
- `platform/`: shared platform capabilities such as operators, networking, observability, policy, and utilities
- `docs/`: design notes, implementation notes, and lessons learned

## Source of truth

- Treat `clusters/homelab/root-application.yaml` as the manual bootstrap entrypoint.
- Treat `clusters/homelab/bootstrap/apps-applicationset.yaml` as the discovery contract for tenant apps.
- New tenant workloads should be added under `apps/<name>/app/`.
- Shared services and operator-facing tools belong under `platform/`, not `apps/`.
- `argocd/` is legacy documentation only.
- Use Linear for issue tracking and work management when a task needs a ticket, project, or related planning artifact.

## Editing rules

- Keep manifests plain YAML unless a directory already establishes a stronger pattern.
- Preserve the current directory contract; avoid creating new top-level buckets unless the repo structure is being intentionally redesigned.
- For Crossplane work in this repo, always target Crossplane version `2.2`.
- When adding a tenant app, make sure the namespace and app name line up with the `ApplicationSet` convention.
- When changing shared platform behavior, prefer updating the matching `platform/<area>/` subtree and the corresponding bootstrap `Application` only if wiring changes are required.
- Do not treat empty legacy directories under `apps/` as active sources of truth.

## Validation

- There is no detected top-level build, lint, or task runner in this repo.
- After edits, validate YAML carefully and, when possible, check that file placement still matches the GitOps discovery rules.
- Be especially careful with:
  - Argo CD `Application` and `ApplicationSet` paths
  - Crossplane XRD and Composition schema alignment
  - namespace consistency across tenant app manifests

## Useful context

- `apps/README.md` explains the tenant app contract.
- `platform/README.md` explains what belongs under shared platform management.
- `docs/2026-04-04-idp-6-repo-structure-notes.md` documents the current repo shape and why `apps/` vs `platform/` is split this way.
