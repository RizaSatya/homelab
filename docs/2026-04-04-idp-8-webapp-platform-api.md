# IDP-8 WebApp Platform API

## What this adds

`IDP-8` introduces the first real Crossplane-backed platform API for tenant web apps in the homelab cluster.

This step keeps the existing GitOps onboarding contract intact:

- tenant apps still live at `apps/<app-name>/app/`
- Argo CD still auto-discovers those folders
- the tenant folder can now hold a namespaced `WebApp` resource instead of raw runtime manifests

The new Crossplane API is intentionally narrow. It is designed for the first internal web app golden path, not for every workload type the homelab may ever run.

## Delivery model

Crossplane now has two separate Argo CD child applications:

- `crossplane`: installs the Crossplane operator itself from the Helm chart
- `crossplane-platform`: syncs the platform assets that live in this repo

That split keeps the operator install boring while making the XRDs, Compositions, Functions, and supporting RBAC first-class Git-managed platform assets.

## Crossplane v2 shape

This implementation follows the Crossplane v2 model:

- the XRD uses `apiextensions.crossplane.io/v2`
- the `WebApp` custom resource is a namespaced XR, not a claim
- the composition renders native Kubernetes resources directly
- Crossplane gets permission to compose those resources through aggregated RBAC

Important API version note:

- `CompositeResourceDefinition` uses `apiextensions.crossplane.io/v2`
- `Composition` still uses `apiextensions.crossplane.io/v1`

That version split is intentional and matches the official Crossplane v2 documentation.

## Current platform assets

The platform assets introduced for this first API are:

- `function-go-templating` to render the composed objects from the `WebApp` resource
- `function-auto-ready` to report readiness cleanly for the composed resources
- one aggregated `ClusterRole` that grants Crossplane access to manage `Deployments`, `Services`, and `HorizontalPodAutoscalers`
- one `WebApp` API
- one composition that renders a deployment, service, and optional HPA directly in the XR namespace

## `WebApp` v1 contract

The resource kind is:

```yaml
apiVersion: platform.rizaes.com/v1alpha1
kind: WebApp
```

The v1 API accepts:

- `spec.image`
- `spec.port`
- `spec.host`
- `spec.owner`
- `spec.resources.requests.cpu`
- `spec.resources.requests.memory`
- `spec.resources.limits.cpu`
- `spec.resources.limits.memory`
- optional `spec.secretRefs[]`
- optional `spec.env[]`
- optional `spec.autoscaling`

Secrets stay out of the resource body itself. The `WebApp` resource references an existing secret by name and key, which keeps the current app-owned `SealedSecret` flow compatible with the new API.

Crossplane’s own machinery for composition and resource references lives under `spec.crossplane` in the v2 model, not in the user-facing part of the API.

## What the composition creates

For each namespaced `WebApp` resource, Crossplane composes:

- a `Deployment`
- a `Service`
- a `HorizontalPodAutoscaler` when `spec.autoscaling` is provided

The composed resources live in the same namespace as the `WebApp` resource itself.

This means:

- Argo CD still creates the app namespace through `CreateNamespace=true`
- the `WebApp` resource lives in that namespace
- Crossplane composes the workload resources into that same namespace

The composition also injects a small set of platform-managed OTEL environment defaults so the first migration path keeps the current observability wiring.

`spec.host` is present in the API now for onboarding and future Backstage compatibility, but the v1 composition does not create ingress or Cloudflare routing objects. The host value is carried as metadata only.

## Example resource

```yaml
apiVersion: platform.rizaes.com/v1alpha1
kind: WebApp
metadata:
  name: web-riza
spec:
  image: rizasatyabudhi/web-riza:v1.9
  port: 3000
  host: web-riza.rizaes.com
  owner: riza
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  env:
    - name: PRIVATE_VIEW
      value: "true"
  secretRefs:
    - envName: OPENROUTER_API_KEY
      secretName: web-riza-secrets
      secretKey: OPENROUTER_API_KEY
```

When this file is synced by the tenant app Argo `Application`, it will live in that tenant namespace and Crossplane will compose the runtime resources there.

That resource is the shape future manual Git PR onboarding and later Backstage forms should target.

## Intentional v1 limits

This first API does not try to cover:

- namespace composition
- ingress or gateway resources
- Cloudflare hostname automation
- external secret operator flows
- multiple containers
- probes
- volumes
- sidecars
- service type customization
- arbitrary pod-level escape hatches

Those can come later if the platform needs them, but the first value here is a clean and opinionated golden path.

## What Comes Next

The best next step after `IDP-8` is `IDP-9`: migrate `web-riza` onto the new namespaced `WebApp` resource.

That is the fastest way to prove the Crossplane setup is real, because at this point the platform API exists, but no tenant app is using it yet.

A practical order from here is:

1. Replace the raw `web-riza` runtime manifests with a `WebApp` resource.
2. Keep the app-owned `SealedSecret` in place.
3. Let Argo CD create the app namespace through its existing destination settings.
4. Validate that Argo syncs the `WebApp` resource and Crossplane reconciles it into the expected runtime resources.
5. Use that first migration to discover any missing fields, RBAC gaps, or reconciliation issues.
6. Move on to policy guardrails like `IDP-10` once the golden path is actually in use.

In repo terms, that means the likely next change is:

- remove the raw `Deployment`, `Service`, and likely `Namespace` manifests under `apps/web-riza/app/`
- add one `webapp.yaml` resource under the same folder
- keep `sealedsecret.yaml`

That is the moment the repo fully crosses from:

- "Crossplane platform API exists"

to:

- "A real tenant app is onboarded through the platform API"
