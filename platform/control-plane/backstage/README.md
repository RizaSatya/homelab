# Backstage Control Plane

This directory holds the GitOps-managed homelab Backstage v1 rollout.

Scope for this first pass:

- one Backstage instance in the `backstage` namespace
- internal-first access through `kubectl port-forward`
- one homelab Kubernetes cluster
- one Argo CD instance
- tenant app catalog coverage only

This repo owns:

- the Argo CD child application wiring
- the Helm values for the Backstage release
- the Kubernetes service account and read-only cluster RBAC for the Kubernetes plugin
- the SQLite PVC used by the lean single-pod deployment
- the Argo CD local account and RBAC fragment used by the Argo CD plugin
- the catalog root and ownership data that point at tenant app `catalog-info.yaml` files

This repo does not hold the customized Backstage app source itself.

The `examples/` directory is intentionally not referenced by the Argo CD application.
It holds secret templates and other operator-only examples that should be filled in before use, not synced as-is.

The official Backstage chart deploys a vanilla demo image. The homelab needs a custom image for:

- the Kubernetes plugin UI wiring
- the Argo CD plugin frontend and backend packages
- the entity page layout updates for those plugins

Companion Backstage app repo contract:

- build and publish a custom image such as `ghcr.io/rizasatya/homelab-backstage`
- keep the app close to the default Backstage app
- add only:
  - Software Catalog
  - Kubernetes plugin
  - Roadie Argo CD plugin
- make the image read these env vars when present:
  - `BACKEND_SECRET`
  - `ARGOCD_AUTH_TOKEN`
  - `GITHUB_TOKEN`

Catalog contract:

- Backstage ingests the root location file at `platform/control-plane/backstage/catalog/root.yaml`
- each tenant app can expose `apps/<name>/catalog-info.yaml`
- `metadata.name` must match the tenant app name, namespace, and generated Argo CD application name

Argo CD token flow:

1. Let Argo CD sync the manifests under `platform/control-plane/backstage/argocd`.
2. Generate a token for the local `backstage` Argo CD account.
3. Seal that token into the example sealed secret under `platform/control-plane/backstage/examples/`.
4. Update the companion Backstage app repo to read `ARGOCD_AUTH_TOKEN` from the sealed secret-backed env.

Port-forward for v1:

```bash
kubectl -n backstage port-forward svc/backstage 7007:7007
```
