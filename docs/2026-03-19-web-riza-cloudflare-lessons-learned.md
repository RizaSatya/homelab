# Web Riza and Cloudflare Tunnel Lessons Learned

## Context

These notes capture the main problems I hit while exposing `web-riza` from the k3s homelab with Argo CD, Cloudflare Tunnel, and Sealed Secrets, plus the fixes that worked.

## What Broke

- `web-riza` needed a secret-backed `OPENROUTER_API_KEY`, but the cluster initially used a plain Kubernetes `Secret`.
- Adding a new Argo CD app manifest for `cloudflared` in Git did not make Argo CD automatically detect the app.
- It was not obvious what Cloudflare Tunnel hostname and service URL should point to for a Kubernetes `Service`.
- After switching `web-riza` to a `SealedSecret`, Argo CD showed the app as `Degraded` even though the pod was healthy.
- Deleting the `whoami` Argo CD `Application` did not automatically remove the already-created workload objects in namespace `demo`.

## Root Causes

### 1. New Argo CD `Application` manifests are not auto-discovered by magic

This repo stores child `Application` manifests under `argocd/`, but there is no parent app-of-apps or `ApplicationSet` in Git that watches that directory.

That meant:

- pushing `argocd/cloudflared-application.yaml` to Git was not enough
- Argo CD did not know it should create a new `cloudflared` `Application`
- the manifest had to be applied to the cluster once

What fixed it:

- apply the new Argo CD `Application` directly

```bash
kubectl apply -n argocd -f argocd/cloudflared-application.yaml
```

Practical rule:

- updating an existing Argo CD app can happen through Git
- creating a brand-new Argo CD app needs either a parent app, an `ApplicationSet`, or a one-time `kubectl apply`

### 2. Cloudflare Tunnel should target the Kubernetes `Service`, not a pod or localhost

The public route in Cloudflare Tunnel needed to point at the in-cluster service DNS name for `web-riza`.

The correct service URL was:

```text
http://web-riza.web-riza.svc.cluster.local:3000
```

What made this easier:

- `web-riza` already had a `ClusterIP` `Service`
- Cloudflare Tunnel only needed outbound connectivity from `cloudflared`
- no ingress controller or public load balancer was required

Practical rule:

- for Kubernetes workloads behind Cloudflare Tunnel, route to the internal `Service` DNS name whenever possible

### 3. Sealed Secrets cannot take over an existing unmanaged `Secret`

After creating a `SealedSecret` named `web-riza-secrets`, Argo CD marked the app as `Degraded` even though the `Deployment` and pod were healthy.

The key error was:

```text
failed update: Resource "web-riza-secrets" already exists and is not managed by SealedSecret
```

That meant:

- the old plain `Secret` still existed
- the Sealed Secrets controller refused to overwrite it
- Argo CD health reflected the failing `SealedSecret`, not just the healthy pod

What fixed it:

- delete the old plain `Secret`
- let the Sealed Secrets controller recreate the managed `Secret`

```bash
kubectl -n web-riza delete secret web-riza-secrets
```

### 4. Deleting the old `Secret` was not enough because the `SealedSecret` status stayed stale

After the plain `Secret` was deleted, the `SealedSecret` still showed the old failed condition:

```text
failed update: Resource "web-riza-secrets" already exists and is not managed by SealedSecret
```

The controller logs showed it had already given up after a short retry loop and had not reconciled the object again.

What fixed it:

- delete the live `SealedSecret`
- refresh and sync the Argo CD app so the object is recreated and reconciled from scratch

```bash
kubectl -n web-riza delete sealedsecret web-riza-secrets
kubectl -n argocd annotate application web-riza argocd.argoproj.io/refresh=hard --overwrite
```

Then re-sync the Argo CD app.

Practical rule:

- if a `SealedSecret` has a stale failed condition after the underlying conflict is gone, recreate the `SealedSecret` object to force a fresh reconcile

### 5. Deleting an Argo CD `Application` does not always delete the workloads you expect

Removing `whoami` from Git and deleting the live `Application` did not automatically clean up the `Deployment`, `Service`, pods, and `ReplicaSet` in namespace `demo`.

That meant:

- Argo CD stopped tracking the app
- the workload objects could remain in the cluster
- cleanup still had to happen separately

What fixed it:

- delete both the app manifests in Git
- manually delete the workload resources if they were left behind

```bash
kubectl delete -f apps/whoami/
kubectl delete namespace demo
```

Practical rule:

- deleting an Argo CD `Application` object and deleting the workload it used to manage are related, but not identical actions

## Good Debugging Sequence Next Time

When a GitOps-managed app behaves strangely, debug in this order:

1. Check whether Argo CD is failing on the app itself or on one child resource.
2. Check whether the app is truly managed by Git, or whether the `Application` manifest was only added locally.
3. If secrets are involved, check whether a plain `Secret` already exists before introducing a `SealedSecret` with the same name.
4. If the app is `Synced` but `Degraded`, inspect the health of each child resource instead of only checking pods.
5. If a controller-based resource still shows an old error after the underlying cause is gone, inspect controller logs and force a fresh reconcile.

## Commands That Were Most Useful

### Check Argo CD app health and resource state

```bash
kubectl -n argocd get application web-riza -o yaml
kubectl -n argocd describe application web-riza
```

### Check workload health separately from Argo health

```bash
kubectl -n web-riza get deployment,pods,replicasets,service -o wide
```

### Inspect `SealedSecret` status

```bash
kubectl -n web-riza get sealedsecret web-riza-secrets -o yaml
kubectl -n web-riza describe sealedsecret web-riza-secrets
```

### Inspect Sealed Secrets controller logs

```bash
kubectl -n kube-system logs deployment/sealed-secrets-controller --tail=200
```

### Confirm whether the real `Secret` exists

```bash
kubectl -n web-riza get secret web-riza-secrets -o yaml
```

### Remove a manually managed Argo CD app

```bash
kubectl delete -n argocd -f argocd/whoami-application.yaml
kubectl delete -f apps/whoami/
```

## Practical Rules To Remember

- A new child `Application` manifest in Git is not enough unless something is already managing the `argocd/` directory.
- Cloudflare Tunnel should usually point to the Kubernetes `Service` DNS name, not `localhost`.
- `Synced` does not mean `Healthy`; Argo CD can be synced while a child resource is degraded.
- A healthy pod does not guarantee a healthy Argo CD app if another managed resource is failing.
- A `SealedSecret` cannot take over an existing unmanaged `Secret` with the same name.
- When removing apps, think separately about Git cleanup, Argo CD `Application` cleanup, and workload cleanup.

## Final Mental Model

For this setup, there are really four layers working together:

1. Argo CD `Application` objects that tell the cluster what to sync
2. plain Kubernetes manifests for workloads and services
3. secret-management controllers like Sealed Secrets
4. public exposure through Cloudflare Tunnel

If layer 1 is missing, Git changes are invisible to the cluster.
If layer 3 is unhealthy, Argo CD can report the app as degraded even when the workload is running.
If layer 4 is correct, a private in-cluster service can be exposed publicly without opening inbound ports on the router.
