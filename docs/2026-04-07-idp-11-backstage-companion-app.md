# IDP-11 Backstage Companion App Contract

This repo now contains the GitOps side of the Backstage rollout, but not the Backstage app source itself.

That split is intentional.

The official Backstage Helm chart can deploy a Backstage container, but plugin-based UI and backend behavior still live in the application source and the image built from it.

For `IDP-11`, the companion Backstage app repo should stay close to the default Backstage app and add only the minimum needed for visibility:

- Software Catalog
- Kubernetes plugin
- Roadie Argo CD plugin

## Why a separate repo makes sense here

This repo is the homelab GitOps source of truth.

It already has a clear split between:

- `clusters/` for Argo bootstrap wiring
- `platform/` for shared platform services
- `apps/` for tenant workloads

Adding a full Node.js Backstage app here would mix application source and image-build concerns into the infrastructure repo. The platform contract is clearer if this repo says:

- here is how Backstage is deployed
- here is the catalog metadata it ingests
- here are the credentials and runtime contracts it expects

and the companion repo says:

- here is the customized Backstage app and image

## Minimal app changes for v1

The companion repo should:

1. Start from the default Backstage app layout.
2. Add the Kubernetes plugin to the frontend entity pages.
3. Add the Roadie Argo CD plugin to both frontend and backend.
4. Keep the entity page focused on:
   - overview
   - Kubernetes content
   - Argo CD overview/history
5. Avoid adding TechDocs, Scaffolder, auth providers, or deeper UI customizations in this phase.

## Runtime config contract from this repo

The deployment values in `platform/control-plane/backstage/values.yaml` assume the image can work with:

- `BACKEND_SECRET`
- `ARGOCD_AUTH_TOKEN`
- `GITHUB_TOKEN`

They also assume the app is prepared to use:

- one Kubernetes cluster named `homelab`
- one Argo CD instance
- catalog ingestion rooted at `platform/control-plane/backstage/catalog/root.yaml`

## Suggested implementation shape in the companion repo

- `packages/app`
  - add Kubernetes and Argo CD cards to the entity page
- `packages/backend`
  - add the Argo CD backend plugin
- `app-config.production.yaml`
  - load Argo CD token from env
  - optionally load a GitHub token from env
  - keep the rest of the cluster and catalog wiring aligned with this repo

## Practical rollout order

1. Build and push the custom Backstage image.
2. Update the image tag in `platform/control-plane/backstage/values.yaml`.
3. Generate the Argo CD local user token after the Backstage Argo manifests sync.
4. Seal the required secrets and apply them through Git.
5. Sync the Backstage Argo application.
6. Port-forward `svc/backstage` and verify catalog, Kubernetes, and Argo CD views.
