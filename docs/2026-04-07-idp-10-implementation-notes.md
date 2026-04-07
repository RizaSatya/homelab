# IDP-10 Implementation Notes

## Context

`IDP-10` is the first policy-focused step for the homelab IDP.

The goal is not to add every possible guardrail. The goal is to add a small baseline set of Kyverno rules that make the Crossplane-backed `WebApp` golden path safer without making the platform hard to understand.

That means this phase stays intentionally narrow:

- protect the `WebApp` golden path only
- keep the rule set small and readable
- choose `Enforce` only where the tenant can actually fix the problem through the current platform API
- use `Audit` where the current API still has a capability gap

## The Delivery Model

Kyverno was already installed in `IDP-7`, but the repo did not yet have a delivery path for policy files themselves.

This issue adds a second Argo CD child app:

- [clusters/homelab/bootstrap/kyverno-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/kyverno-application.yaml)
  - installs the Kyverno controller from Helm
- [clusters/homelab/bootstrap/kyverno-policies-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/kyverno-policies-application.yaml)
  - syncs plain YAML policies from [platform/policy/kyverno/policies](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/policy/kyverno/policies)

I kept this explicit on purpose.

It matches the current bootstrap style in `clusters/homelab/bootstrap/`, and it means operator delivery and policy content can fail independently in ways that are easy to reason about.

## What The Baseline Policies Cover

### 1. `WebApp` request validation

These rules validate the platform-facing custom resource directly:

- [webapp-namespace-match.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/policy/kyverno/policies/webapp-namespace-match.yaml)
  - enforces namespace/name alignment so the tenant app contract and the Crossplane object identity stay aligned
- [webapp-image-tag.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/policy/kyverno/policies/webapp-image-tag.yaml)
  - enforces explicit non-`latest` image tags

These are good `Enforce` candidates because the tenant controls these fields directly in the current `WebApp` API.

### 2. Crossplane-composed workload validation

These rules validate the rendered `Deployment` objects, but only when they carry the label `platform.rizaes.com/webapp=true`:

- [webapp-deployment-labels.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/policy/kyverno/policies/webapp-deployment-labels.yaml)
  - enforces identity labels for traceability
- [webapp-deployment-resources.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/policy/kyverno/policies/webapp-deployment-resources.yaml)
  - enforces per-container CPU and memory requests and limits
- [webapp-deployment-probes.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/policy/kyverno/policies/webapp-deployment-probes.yaml)
  - audits missing readiness and liveness probes

The label scoping matters a lot.

It is what keeps this issue focused on the Crossplane golden path instead of suddenly applying new rules to every raw `Deployment` already living under `apps/`.

## Why Some Rules Enforce And One Only Audits

The baseline stance for `IDP-10` is mixed:

- enforce when the current platform API lets the tenant fix the problem directly
- audit when the rule describes a good outcome but the API cannot express it yet

That is why probes are different.

The current `WebApp` XRD exposes:

- image
- port
- host
- owner
- resources
- env
- secretRefs
- autoscaling

It does **not** expose:

- readiness probes
- liveness probes

If probe rules were `Enforce` today, the platform would reject Crossplane-composed `Deployment` objects for something the tenant cannot supply through the current API.

That would create confusing failures rather than useful guardrails.

So the deliberate choice in this issue is:

- keep probe validation in `Audit`
- document the gap clearly
- move probe support and any later enforcement into a future API-focused issue

## Practical Outcome

After `IDP-10`, the homelab IDP has a first policy baseline that:

- blocks obviously unsafe `WebApp` requests
- blocks missing resource bounds and identity labels on the rendered workload
- warns about missing health probes without breaking the current golden path
- keeps legacy raw tenant apps out of scope

That is the right level of strictness for the first policy pass: useful, understandable, and still honest about the current API boundaries.
