# Crossplane Control Plane

This directory holds the Argo-managed Crossplane install and the future Crossplane assets for the homelab platform.

Current contents:

- `values.yaml` for the Crossplane Helm release
- `providers/` for future Crossplane `Provider` packages
- `functions/` for future Crossplane `Function` packages
- `xrds/` for future CompositeResourceDefinitions
- `compositions/` for future Compositions

`IDP-7` intentionally stops at the operator install and durable repo layout. Future issues can populate these directories without another repo reorganization.
