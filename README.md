# GitOps for the Home Lab Kubernetes Cluster

This repository is managed using GitOps with Flux. The cluster continuously reconciles the state declared in Git with what is running in Kubernetes. You make changes by committing YAML to this repo; Flux applies and keeps the cluster in sync.

## How GitOps Works Here

At a high level, the flow is:

1. You declare desired state as Kubernetes manifests in this repository.
2. Flux runs in the cluster and watches this repo.
3. Flux reconciles the cluster to match what is in Git.
4. Any drift is corrected by Flux, and all changes are auditable in Git history.

Think of Git as the single source of truth, and Flux as the controller that enforces it.

## Repo Layout

This repo follows a Flux monorepo layout, with cluster configuration under `clusters/` and application manifests grouped by environment under `clusters/apps/`:

```
clusters/
  staging/
    flux-system/
      gotk-components.yaml
      gotk-sync.yaml
      kustomization.yaml
    app.yaml
  apps/
    base/
      linkding/
        deployment.yaml
        kustomization.yaml
        namespace.yaml
    staging/
      linkding/
        kustomization.yaml
```

The `flux-system` manifests are generated and managed by Flux. Do not edit them by hand.

## Bootstrapping Flux (Summary)

Flux is installed into the cluster using `flux bootstrap`. This does two things:

- Installs the Flux controllers in the `flux-system` namespace.
- Commits the Flux system manifests back into this repo so Flux can manage itself.

Example flow:

1. Create a GitHub repo (this repo).
2. Create a GitHub personal access token (classic) with `repo` scope.
3. Export credentials for Flux:

```
export GITHUB_TOKEN=...your-token...
export GITHUB_USER=your-github-username
```

4. Bootstrap Flux:

```
flux bootstrap github \
  --owner="${GITHUB_USER}" \
  --repository="home-lab-kubernetes" \
  --branch="main" \
  --path="clusters/staging" \
  --personal
```

Flux will commit the `clusters/staging/flux-system` files and begin reconciling.

## Day-to-Day Workflow

- Add or update manifests under `clusters/<cluster>/` or `clusters/<cluster>/apps/`.
- Commit and push changes to Git.
- Flux detects the change and reconciles the cluster.

If you need to verify:

```
flux check
flux get kustomizations
kubectl get pods -n flux-system
```

## Important Notes

- Do not manually edit `flux-system` manifests; Flux owns them.
- Use Git history as your audit log.
- Keep changes small and frequent; reconciliation is continuous.
