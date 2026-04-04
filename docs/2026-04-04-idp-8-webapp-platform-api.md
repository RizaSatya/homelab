# IDP-8 WebApp Platform API

## What this adds

`IDP-8` introduces the first real Crossplane-backed platform API for tenant web apps in the homelab cluster.

This step keeps the existing GitOps onboarding contract intact:

- tenant apps still live at `apps/<app-name>/app/`
- Argo CD still auto-discovers those folders
- the tenant folder can now hold a `WebApp` claim instead of raw runtime manifests

The new Crossplane API is intentionally narrow. It is designed for the first internal web app golden path, not for every workload type the homelab may ever run.

## Delivery model

Crossplane now has two separate Argo CD child applications:

- `crossplane`: installs the Crossplane operator itself from the Helm chart
- `crossplane-platform`: syncs the platform assets that live in this repo

That split keeps the operator install boring while making the XRDs, Compositions, Providers, and Functions first-class Git-managed platform assets.

## Current platform assets

The platform assets introduced for this first API are:

- `provider-kubernetes` to let Crossplane manage Kubernetes objects in-cluster
- `function-go-templating` to render the composed objects from the `WebApp` claim
- `function-auto-ready` to report readiness cleanly for the composed resources
- one `XWebApp` / `WebApp` API
- one composition that renders a namespace, deployment, service, and optional HPA

## `WebApp` v1 contract

The claim kind is:

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

Secrets stay out of the claim body itself. The claim references an existing secret by name and key, which keeps the current app-owned `SealedSecret` flow compatible with the new API.

## What the composition creates

For each `WebApp` claim, Crossplane composes:

- a `Namespace` named after the claim
- a `Deployment` named after the claim
- a `Service` named after the claim
- an `HorizontalPodAutoscaler` when `spec.autoscaling` is provided

The composition also injects a small set of platform-managed OTEL environment defaults so the first migration path keeps the current observability wiring.

`spec.host` is present in the API now for onboarding and future Backstage compatibility, but the v1 composition does not create ingress or Cloudflare routing objects. The host value is carried as metadata only.

## Example claim

```yaml
apiVersion: platform.rizaes.com/v1alpha1
kind: WebApp
metadata:
  name: web-riza
  namespace: web-riza
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

That claim is the shape future manual Git PR onboarding and later Backstage forms should target.

## Intentional v1 limits

This first API does not try to cover:

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

The best next step after `IDP-8` is `IDP-9`: migrate `web-riza` onto the new `WebApp` claim.

That is the fastest way to prove the Crossplane setup is real, because at this point the platform API exists, but no tenant app is using it yet.

A practical order from here is:

1. Replace the raw `web-riza` runtime manifests with a `WebApp` claim.
2. Keep the app-owned `SealedSecret` in place.
3. Validate that Argo syncs the claim and Crossplane reconciles it into the expected runtime resources.
4. Use that first migration to discover any missing fields, RBAC gaps, or reconciliation issues.
5. Move on to policy guardrails like `IDP-10` once the golden path is actually in use.

In repo terms, that means the likely next change is:

- remove the raw `Deployment` and `Service` manifests under `apps/web-riza/app/`
- add one `webapp.yaml` claim under the same folder
- keep `sealedsecret.yaml`

That is the moment the repo fully crosses from:

- "Crossplane platform API exists"

to:

- "A real tenant app is onboarded through the platform API"
