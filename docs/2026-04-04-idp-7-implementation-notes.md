# IDP-7 Implementation Notes

## Context

This note captures what I did for `IDP-7`, the next infrastructure step in the homelab IDP sequence.

The specific goal of `IDP-7` was not to define the first Crossplane `WebApp` API yet. It was narrower:

- bring Crossplane under Argo-managed platform control in the new repo structure
- bring Kyverno under the same platform control path
- reserve explicit repo homes for the Crossplane and Kyverno assets that `IDP-8` will need

In other words, this phase was about laying the platform foundation cleanly before the first platform API lands on top.

## The problem I wanted to solve

`IDP-6` cleaned up the repo shape, but the platform control plane still had a gap:

- Crossplane was not yet installed through the new bootstrap path
- Kyverno was not yet installed through the new bootstrap path
- there was no stable, documented place in the repo for providers, functions, XRDs, compositions, or policies

That mattered because `IDP-8` depends on those paths existing already. If the first golden-path API has to reshape the repo again, the foundation was too loose.

## Design decision

I chose to keep this phase intentionally small and explicit:

- one Argo-managed child app for Crossplane
- one Argo-managed child app for Kyverno
- plain repository directories for the future Crossplane and Kyverno assets

Why keep the operators as explicit child apps:

- it matches the current `clusters/homelab/bootstrap/` pattern
- it keeps the control plane easy to reason about
- it avoids introducing a new hierarchy or another layer of automation too early

Why reserve the repo homes now:

- `IDP-8` needs a stable place to put the first `WebApp` XRD and composition
- future providers and functions should not force another directory reshuffle
- Kyverno policies need a clear home even before the first policy exists

## What changed in the repo

### 1. Added Crossplane and Kyverno bootstrap apps

I introduced:

- [clusters/homelab/bootstrap/crossplane-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/crossplane-application.yaml)
- [clusters/homelab/bootstrap/kyverno-application.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/clusters/homelab/bootstrap/kyverno-application.yaml)

Both are explicit child `Application` manifests under the existing root app discovery path.

The Crossplane app uses the official Crossplane chart and points at:

- [platform/control-plane/crossplane/values.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/control-plane/crossplane/values.yaml)

The Kyverno app uses the official Kyverno chart and points at:

- [platform/policy/kyverno/values.yaml](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/policy/kyverno/values.yaml)

### 2. Reserved the Crossplane control-plane layout

I added a durable Crossplane home under:

- [platform/control-plane/crossplane/README.md](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/control-plane/crossplane/README.md)

That area now explicitly reserves:

- `providers/`
- `functions/`
- `xrds/`
- `compositions/`

The important part is not the empty directories themselves; it is that the repo now has a stable contract for where Crossplane assets will live once `IDP-8` starts adding them.

### 3. Reserved the Kyverno policy layout

I added a durable Kyverno home under:

- [platform/policy/kyverno/README.md](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/policy/kyverno/README.md)

That area now explicitly reserves:

- `policies/`

### 4. Documented the new platform areas

I updated:

- [platform/README.md](/Users/riza.satyabudhi/Documents/workshop/claude/homelab/homelab-infra/platform/README.md)

That keeps the new `control-plane/` and `policy/` areas visible alongside the existing platform categories.

## What I validated

The repo-level checks I used were simple but useful:

- the staged diff matched the `IDP-7` scope exactly
- the new manifests follow the same explicit child-app model already used by the cluster bootstrap path
- `git diff --check` passed before commit
- the branch was pushed and a draft PR was opened

Live cluster validation is still pending, which is expected for a foundation-only step like this.

## What worked well

### 1. The bootstrap model stayed boring

There is now one clear answer to "where do Crossplane and Kyverno come from?":

- Argo manages them from `clusters/homelab/bootstrap/`

That is a good fit for this repo because it keeps platform installation visible and debuggable.

### 2. The repo now has stable landing zones for `IDP-8`

The first `WebApp` API can now target existing paths instead of inventing new ones.

That is a bigger win than it looks like at first glance because it keeps the next issue focused on API design rather than repo shape.

## What was awkward or surprising

### 1. Foundation work can look empty

This issue does not add real providers, functions, XRDs, compositions, or policies yet.

That can feel thin in isolation, but it is intentional. The value here is in making the control plane honest and the repo contract durable before the platform API arrives.

### 2. The PR workflow depends on GitHub auth and network access

I had to re-enable `gh` before publishing the PR, and the final push still depends on GitHub access being available in the session.

That is not a repo problem, but it is worth remembering when I do the next publish flow.

## Practical rules from this phase

- Installation and API design do not need to happen at the same time.
- Crossplane control-plane assets should have one clear home before the first XRD exists.
- Kyverno policy placement should be explicit before the first policy lands.
- If a platform issue is only about foundation, it should stop at foundation.

## What this enables next

With `IDP-7` in place, the next step is straightforward:

- `IDP-8` can define the first `WebApp` platform API and composition
- providers, functions, XRDs, compositions, and policies can land in the reserved paths
- the repo does not need another layout rewrite before the first real platform claim exists

That is why this phase mattered even though it did not add the first user-facing platform API yet. It created the control-plane contract that the later layers depend on.
