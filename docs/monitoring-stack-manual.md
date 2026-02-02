# Monitoring Stack Manual

This repository deploys monitoring controllers with Flux + Helm.

## 1) Current monitoring layout

- Cluster entrypoint: `clusters/staging/monitoring.yaml`
- Environment kustomization: `monitoring/controllers/staging/kustomization.yaml`
- Stack overlay: `monitoring/controllers/staging/kube-prometheus-stack/kustomization.yaml`
- Base resources: `monitoring/controllers/base/kube-prometheus-stack/`
  - `namespace.yaml`
  - `repository.yaml` (Prometheus Community Helm repo)
  - `release.yaml` (HelmRelease for kube-prometheus-stack)

## 2) What gets installed

From `release.yaml`:

- `HelmRelease` for `kube-prometheus-stack`
- Chart version pinned to `66.2.2`
- CRD lifecycle configured for install/upgrade
- Drift detection enabled

## 3) Operations runbook

Check status:

```bash
flux get kustomizations -A
flux get helmreleases -A
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Investigate failures:

```bash
kubectl describe helmrelease kube-prometheus-stack -n monitoring
kubectl logs deploy/helm-controller -n flux-system --tail=200
```

## 4) Grafana access

Quick local access:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```

Then open `http://localhost:3000`.

## 5) Hardening and organization guidance

- Move sensitive Helm values (for example admin credentials) into encrypted SOPS secrets.
- Keep monitoring "controllers" and "configs" separated; `monitoring-configs` is already scaffolded in `clusters/staging/monitoring.yaml` but commented out.
- Add logging stack as a separate path (for example Fluent Bit + Loki) to satisfy the case-study logging objective.
