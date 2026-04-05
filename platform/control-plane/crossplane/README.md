# Crossplane Control Plane

This directory holds the Argo-managed Crossplane install and the Crossplane platform assets for the homelab platform.

Current contents:

- `values.yaml` for the Crossplane Helm release
- `providers/` reserved for future Crossplane `Provider` packages
- `functions/` for Crossplane `Function` packages
- `rbac/` for aggregated RBAC that grants Crossplane access to compose non-Crossplane Kubernetes resources
- `xrds/` for CompositeResourceDefinitions
- `compositions/` for Compositions

The current platform API delivered from this directory is the first `WebApp` golden path introduced by `IDP-8`, implemented in the Crossplane v2 style with a namespaced XR and direct composition of Kubernetes resources.
