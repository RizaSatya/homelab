# Prometheus Stack Lessons Learned

## Context

These notes capture the main problems I hit while installing `kube-prometheus-stack` on the k3s homelab through Argo CD, plus the fixes that worked.

## What Broke

- Argo CD failed to apply several Prometheus Operator CRDs with `metadata.annotations: Too long`.
- Because those CRDs were missing, the `Prometheus` and `Alertmanager` custom resources could not be created.
- Grafana failed its `init-chown-data` init container because the persistent volume denied `chown`.
- Grafana's admin password secret did not match the password stored in Grafana's persisted database.
- Grafana dashboards could not query Prometheus because the Prometheus service existed before the Prometheus workload was actually running.
- The Git-managed node dashboard did not appear until both the Argo CD `Application` spec and the remote Git repo were updated.

## Root Causes

### 1. Large Prometheus Operator CRDs need special Argo CD sync behavior

Prometheus Operator CRDs are large enough that Argo CD's default client-side apply can exceed the Kubernetes annotation size limit.

Key symptom:

```text
CustomResourceDefinition ... is invalid: metadata.annotations: Too long
```

What fixed it:

- `ServerSideApply=true`
- `DisableClientSideApplyMigration=true`

Those sync options were added to:

- [argocd/kube-prometheus-stack-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/argocd/kube-prometheus-stack-application.yaml)

### 2. Operators can start before CRDs are ready

The Prometheus Operator started successfully, but its logs showed that key resources like `prometheuses.monitoring.coreos.com` and `alertmanagers.monitoring.coreos.com` were not installed yet.

That meant:

- the operator pod was healthy
- the CRDs were not fully available when it started
- no Prometheus StatefulSet was created
- Grafana could not reach Prometheus because the service had no endpoints

What fixed it:

- restart the operator after the CRDs were finally present

```bash
kubectl -n observability rollout restart deploy/kube-prometheus-stack-operator
```

### 3. Persistent storage changes Grafana behavior

Grafana was not stateless in practice because persistence was enabled.

Two consequences showed up:

- the `init-chown-data` init container failed on the PVC
- the admin password in the Kubernetes secret no longer matched the real login password stored in Grafana's database

What fixed the init container problem:

- disable the ownership-changing init container
- use `Recreate` rollout strategy for a single-replica Grafana with one persistent volume

These settings live in:

- [platform/observability/kube-prometheus-stack/values.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/observability/kube-prometheus-stack/values.yaml)

What fixed the password mismatch:

- reset the admin password from inside the Grafana container so it matched the current secret again

### 4. In GitOps, local edits are not enough

Argo CD only applies what exists in the cluster `Application` object and what is available in the remote Git repository it watches.

That means:

- editing files locally does not change live cluster state
- changing an `Application` manifest requires re-applying that manifest
- adding new manifests to a watched path requires committing and pushing them

This was especially visible when the Git-managed dashboard did not appear until:

- the updated `Application` was applied
- the new dashboard file was pushed to GitHub

## Good Debugging Sequence Next Time

When the observability stack is not working, debug in this order:

1. Check the Argo CD app status.
2. Check whether Prometheus Operator CRDs exist.
3. Check whether `Prometheus` and `Alertmanager` custom resources exist.
4. Check the operator logs for missing-CRD or reconcile errors.
5. Check whether the Prometheus service has endpoints.
6. Only debug Grafana dashboards after Prometheus is actually reachable.

## Commands That Were Most Useful

### Argo CD app status

```bash
kubectl -n argocd get application kube-prometheus-stack
kubectl -n argocd describe application kube-prometheus-stack
```

### Confirm CRDs are present

```bash
kubectl get crd | grep monitoring.coreos.com
```

### Check operator-managed resources

```bash
kubectl -n observability get prometheus,alertmanager
kubectl -n observability logs deploy/kube-prometheus-stack-operator --tail=200
```

### Check whether Prometheus is actually serving traffic

```bash
kubectl -n observability get svc kube-prometheus-stack-prometheus
kubectl -n observability get endpoints kube-prometheus-stack-prometheus
kubectl -n observability get sts,pods
```

### Check Grafana persistence-related issues

```bash
kubectl -n observability describe pod <grafana-pod>
kubectl -n observability logs <grafana-pod> -c init-chown-data
kubectl -n observability logs deploy/kube-prometheus-stack-grafana --tail=100
```

## Practical Rules To Remember

- If you see `metadata.annotations: Too long`, think CRDs plus client-side apply.
- If a service exists but has `<none>` endpoints, debug the backing workload before touching dashboards.
- If Grafana password from the secret does not work, persistence may be preserving an older password.
- If Grafana is using one PVC and one replica, `Recreate` is safer than overlapping rollouts.
- If an operator started before CRDs were ready, a rollout restart can be the quickest recovery.
- In GitOps, always separate "what I changed locally" from "what Argo CD can actually see in Git".

## Final Mental Model

`kube-prometheus-stack` is really three layers working together:

1. CRDs and operator machinery
2. operator-managed workloads like Prometheus and Alertmanager
3. Grafana and dashboards on top

If layer 1 is broken, layer 2 will not appear.
If layer 2 is missing, layer 3 will look broken even when Grafana itself is healthy.
