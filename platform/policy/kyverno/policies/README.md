# Kyverno Policies

This directory contains the baseline Kyverno `ClusterPolicy` manifests for the homelab platform.

Current policy set:

- `webapp-namespace-match.yaml`
  - enforces that a `WebApp` lives in the namespace matching its name
- `webapp-image-tag.yaml`
  - enforces explicit non-`latest` image tags on `WebApp.spec.image`
- `webapp-deployment-labels.yaml`
  - enforces identity labels on Crossplane-composed `Deployment` resources
- `webapp-deployment-resources.yaml`
  - enforces CPU and memory requests and limits on those Deployments
- `webapp-deployment-probes.yaml`
  - audits missing readiness and liveness probes on those Deployments

Why these are `ClusterPolicy` resources instead of namespaced `Policy` resources:

- they need to validate cluster-wide resource kinds like `Deployment`
- they also need to see `WebApp` custom resources across tenant namespaces

Why the deployment rules are label-scoped:

- Crossplane adds `platform.rizaes.com/webapp=true` to the composed workload objects
- matching on that label lets us protect the golden path without breaking legacy raw manifests

The probe rule stays in `Audit` for now because the current `WebApp` API does not let tenants specify probes yet.
