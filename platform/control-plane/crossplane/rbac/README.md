# Crossplane RBAC

This directory holds additional RBAC that aggregates to Crossplane's primary cluster role.

Crossplane v2 can compose native Kubernetes resources directly, but it still needs permission to create and manage those resources.

Current contents:

- the aggregated ClusterRole that allows the `WebApp` composition to manage `Deployments`, `Services`, and `HorizontalPodAutoscalers`
