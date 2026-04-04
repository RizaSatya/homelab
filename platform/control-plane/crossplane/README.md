# Crossplane Control Plane

This directory holds the Argo-managed Crossplane install and the Crossplane platform assets for the homelab platform.

Current contents:

- `values.yaml` for the Crossplane Helm release
- `providers/` for Crossplane `Provider` packages and provider runtime config
- `functions/` for Crossplane `Function` packages
- `xrds/` for CompositeResourceDefinitions
- `compositions/` for Compositions

The current platform API delivered from this directory is the first `WebApp` golden path introduced by `IDP-8`.
