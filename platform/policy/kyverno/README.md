# Kyverno Policy Platform

This directory holds the Argo-managed Kyverno install and the policy definitions for the homelab platform.

Current contents:

- `values.yaml` for the Kyverno Helm release
- `policies/` for Kyverno `ClusterPolicy` manifests managed from Git

## What Kyverno Is Doing Here

Kyverno is a Kubernetes-native policy engine.

It watches resources as they are created or updated and can:

- validate them and block unsafe changes
- validate them in audit mode and report warnings without blocking
- mutate or generate resources

For `IDP-10` we are using only the first two modes: validation in `Enforce` and `Audit`.

## How This Repo Delivers Policies

There are two Argo CD applications involved:

- `clusters/homelab/bootstrap/kyverno-application.yaml`
  - installs the Kyverno controller itself from the Helm chart
- `clusters/homelab/bootstrap/kyverno-policies-application.yaml`
  - syncs the plain YAML policy files from `platform/policy/kyverno/policies/`

This split keeps the operator install separate from the policy content, which makes the system easier to debug:

- if Kyverno is unhealthy, look at the Helm-backed `kyverno` app
- if a policy file is wrong, look at the Git-backed `kyverno-policies` app

## How To Read A Kyverno Policy

Each `ClusterPolicy` in this repo follows the same basic shape:

- `match`
  - decides which Kubernetes objects the rule should inspect
- `validate`
  - defines what must be true about those objects
- `validate.failureAction`
  - decides whether failures are `Enforce` or `Audit`
- `emitWarning`
  - optional helper for `Audit` rules to surface violations in admission warnings

In practical terms:

- `Enforce`
  - rejects the non-compliant resource
- `Audit`
  - allows the resource but records the violation

## Why The Policies Focus On Crossplane-Composed Workloads

`IDP-10` is intentionally scoped to the Crossplane-backed `WebApp` golden path, not every tenant app in the repo.

That means:

- `WebApp` resources are validated directly for app-facing rules like namespace alignment and image tag hygiene
- rendered `Deployment` resources are validated only when they carry the Crossplane label `platform.rizaes.com/webapp=true`
- legacy raw manifests such as `apps/web-riza/` stay out of scope for this first baseline

This keeps the first policy layer understandable while the new golden path is still taking shape.
