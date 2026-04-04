# Platform Components

This directory holds shared platform capabilities that are managed explicitly by Argo CD.

Guideline:

- shared controllers, ingress/connectivity, observability, and operator tools belong under `platform/`
- tenant workloads that represent onboarded applications belong under `apps/`

Current platform areas:

- `control-plane/`
- `databases/`
- `networking/`
- `observability/`
- `policy/`
- `utilities/`
