# IDP-5 Implementation Notes

## Context

This note captures what I did for `IDP-5`, the first infrastructure step toward turning this homelab into an internal developer platform.

The specific goal of `IDP-5` was not to add Backstage, Crossplane, or Kyverno yet. It was narrower:

- create a real GitOps root for the cluster
- stop relying on manually applied child Argo CD `Application` objects
- make tenant app onboarding discoverable from Git

In other words, this phase was about fixing the control plane shape before adding more platform features on top.

## The problem I wanted to solve

Before this change, the repo stored Argo CD child `Application` manifests under `argocd/`, but Argo CD was not watching that directory through a parent app or `ApplicationSet`.

That created an awkward GitOps gap:

- updating an existing app mostly worked through Git
- creating a brand-new app entry in Git did not automatically create the Argo CD `Application`
- I still had to run a one-time manual `kubectl apply` against the new child app manifest

That is fine during early experiments, but it is not a good base for an internal platform. If the platform story is "create something in Git and Argo picks it up", then that needs to be true for new app onboarding too.

## Design decision

I chose a hybrid model instead of going fully dynamic all at once:

- one manually bootstrapped root `Application`
- explicit child `Application` manifests for platform components
- one `ApplicationSet` for tenant app auto-discovery

This felt like the right tradeoff for the current repo.

Why not a full app-of-apps tree:

- I wanted something smaller and easier to debug
- platform apps still have different chart sources, namespaces, and sync behavior
- I did not want to introduce more abstraction than the homelab needs yet

Why an `ApplicationSet` for tenant apps:

- that is the part that needs to scale later
- it matches the future Backstage onboarding path
- it removes the old "new app in Git is invisible" problem

## What changed in the repo

### 1. Added a real cluster bootstrap entrypoint

I introduced:

- [clusters/homelab/root-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/root-application.yaml)
- [clusters/homelab/bootstrap](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap)

The root app is now the only manual bootstrap object after Argo CD itself exists.

Its job is simple:

- watch `clusters/homelab/bootstrap`
- recurse through that directory
- let Git define the rest of the Argo CD control plane

### 2. Moved platform control entries under the bootstrap path

Platform control apps now live under:

- [clusters/homelab/bootstrap/cloudflared-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/cloudflared-application.yaml)
- [clusters/homelab/bootstrap/cloudnative-pg-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/cloudnative-pg-application.yaml)
- [clusters/homelab/bootstrap/kube-prometheus-stack-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/kube-prometheus-stack-application.yaml)
- [clusters/homelab/bootstrap/opentelemetry-collector-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/opentelemetry-collector-application.yaml)
- [clusters/homelab/bootstrap/pgweb-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/pgweb-application.yaml)

I kept these explicit on purpose. They are still easier to reason about as individual Argo apps than as generated entries.

### 3. Added tenant app auto-discovery

I introduced:

- [clusters/homelab/bootstrap/apps-applicationset.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/apps-applicationset.yaml)

The discovery contract is:

```text
apps/<app-name>/app/
```

If that directory exists, the `ApplicationSet` generates:

- app name: `<app-name>`
- source path: `apps/<app-name>/app`
- destination namespace: `<app-name>`

This is the first stable onboarding contract for tenant workloads in the repo.

### 4. Added transitional app entrypoints

To make that contract work without doing the full repo cleanup yet, I added self-contained `app/` directories for the first tenant apps:

- [apps/web-riza/app](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/apps/web-riza/app)
- [apps/garmin-scraper/app](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/apps/garmin-scraper/app)

These are transitional rather than final design.

I originally tried using a `kustomization.yaml` wrapper that referenced the parent manifests, but that ran into Kustomize load restrictions because the generated app path lives one directory deeper. The fix was to make `app/` self-contained for now.

That is not the final repo shape I want long term, but it gave me a stable discovery path for `IDP-5` without expanding the scope into a full reorganization.

### 5. Deprecated the old Argo entrypoint location

The old `argocd/*.yaml` control entrypoint path is no longer the source of truth.

I replaced that with:

- [argocd/README.md](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/argocd/README.md)

That leaves a clear breadcrumb for future me instead of silently removing the old path.

## What changed in the cluster

After rolling out the new root app, the cluster moved to the new control model:

- `homelab-root` became the parent GitOps entrypoint
- `homelab-apps` started generating tenant applications
- `web-riza` became owned by the `ApplicationSet`
- `garmin-scraper` also showed up through the new path

The important proof point was `web-riza`.

Its Argo CD `Application` now shows:

- owner reference to `ApplicationSet/homelab-apps`
- source path `apps/web-riza/app`
- healthy and synced status

That means the new app onboarding contract is not just present in Git; it is actually controlling live state.

## What worked well

### 1. The bootstrap shape is much clearer now

There is now one obvious answer to "what do I apply to bootstrap Argo-managed state?":

```bash
kubectl apply -n argocd -f clusters/homelab/root-application.yaml
```

That is much easier to explain than "there are several child apps under `argocd/`, but some of them still need to be applied manually once."

### 2. Tenant discovery now matches the future platform direction

The `apps/<name>/app` contract is simple enough that later tools can target it:

- manual PRs
- Backstage scaffolder
- future platform APIs that still emit Git-managed app definitions

That makes `IDP-5` a real foundation task instead of a one-off cleanup.

### 3. The rollout for `web-riza` succeeded

`web-riza` is still serving traffic, and its Argo ownership moved to the generated app.

That is important because it proves this was not just a directory refactor. The new control path actually works against a live application.

## What was awkward or surprising

### 1. Generated app paths make Kustomize wrappers trickier

The first attempt used `app/kustomization.yaml` with `../deployment.yaml` style references. That looked tidy, but Kustomize rejected it under normal load restrictions.

That forced a practical choice:

- either depend on looser Kustomize behavior
- or duplicate the app manifests in the new `app/` directory for now

I chose the second option because the Argo discovery contract matters more than avoiding a little duplication in this phase.

### 2. Existing live Argo apps need deliberate cutover

For apps like `web-riza`, there was already an `Application` with the same name in the cluster.

That means the new `ApplicationSet` cannot just "take over by magic". The ownership transition has to be intentional so I do not accidentally prune a running workload.

The good news is that once the cutover is done, the new model is much simpler.

## Practical rules from this phase

- A GitOps repo is not really GitOps for new apps until Argo can discover new app definitions on its own.
- Platform components and tenant apps do not need to be modeled the same way on day one.
- A small root app plus one targeted `ApplicationSet` is enough to unlock the next platform steps.
- Transitional duplication is acceptable if it creates a stable path contract and keeps scope under control.
- Cutover of existing live apps should be treated as an ownership migration, not just a file move.

## What I would emphasize in a later blog post

If I turn this into a blog post later, I think the strongest story is:

1. early homelab GitOps often looks correct but still hides manual bootstrap gaps
2. the first platform milestone is not Backstage or Crossplane, it is making app discovery true
3. a boring Argo control-plane design is a feature, not a weakness
4. once the discovery contract is real, the rest of the IDP roadmap becomes much easier to layer on

I would probably frame `IDP-5` as:

"Before self-service, make the control plane honest."

## What this enables next

With `IDP-5` in place, the next steps are more straightforward:

- `IDP-6`: clean up the repo into a more durable platform-vs-app layout
- `IDP-7`: add Crossplane and Kyverno into the new structure
- `IDP-8`: define the first `WebApp` platform API
- later: let Backstage target the same onboarding contract through PRs

That is why this phase mattered even though it did not add any flashy end-user feature yet. It created the control-plane contract the later platform layers will depend on.
