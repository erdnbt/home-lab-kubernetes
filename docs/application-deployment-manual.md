# Application Deployment Organization Manual

This repository uses a base/overlay model with Flux + Kustomize.

## 1) Current pattern in this repo

Example app (`linkding`):

- Base manifests: `apps/base/linkding/`
  - `namespace.yaml`, `deployment.yaml`, `service.yaml`, `storage.yaml`
- Environment overlay: `apps/staging/linkding/`
  - `kustomization.yaml`
  - `ingress.yaml`
  - `cloudflare.yaml`
  - encrypted secrets

Cluster entrypoint for apps:

- `clusters/staging/apps.yaml` points Flux to `./apps/staging`

## 2) Recommended directory convention

For each app:

```text
apps/
  base/<app>/
    kustomization.yaml
    namespace.yaml
    deployment.yaml
    service.yaml
    [storage/config/networkpolicy]
  staging/<app>/
    kustomization.yaml
    ingress.yaml
    [externalsecret or sops secret]
```

Then aggregate under an environment root kustomization (for example `apps/staging/kustomization.yaml`) so Flux can build all app overlays from one path.

## 3) New app workflow

1. Create `apps/base/<app>/` with reusable manifests.
2. Create `apps/staging/<app>/` for environment-specific settings.
3. Reference base from overlay via `../../base/<app>`.
4. Keep secrets encrypted (SOPS) in overlay folder.
5. Push to Git and let Flux reconcile.

## 4) Deployment operational checks

Use:

```bash
flux get kustomizations -A
kubectl get ns
kubectl get deploy,svc,ing -n <app-namespace>
kubectl describe deploy <app> -n <app-namespace>
```

## 5) Rollback and upgrade strategy

Case-study goal includes safe upgrades/rollback:

- Pin images to explicit versions in base manifests.
- Add readiness/liveness probes and resource requests/limits.
- Use progressive rollout options (HelmRelease/flagger/strategy) for safer upgrades.
- Roll back quickly with Git revert; Flux reconciles to last good commit.
