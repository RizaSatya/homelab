# Crossplane Providers

This directory holds Crossplane `Provider` package manifests and any supporting provider runtime configuration.

Current contents:

- `provider-kubernetes` for composing Kubernetes runtime resources
- a `DeploymentRuntimeConfig` that gives the provider a stable service account name
- a `ClusterRoleBinding` that lets the provider manage tenant namespaces and workloads in-cluster
- a default in-cluster `ProviderConfig`
