# Cluster Setup Manual

This manual maps the DevOps case study requirements to the current repository structure.

## 1) What this repo manages

This repository is the **GitOps layer** for cluster state:

- Flux system: `clusters/staging/flux-system/`
- App reconciliation: `clusters/staging/apps.yaml`
- Monitoring reconciliation: `clusters/staging/monitoring.yaml`
- Application manifests: `apps/`
- Monitoring manifests: `monitoring/`

Infrastructure provisioning (VMs, node groups, cloud network, etc.) should be handled by Terraform in a separate infra stack or repo.

## 2) Prerequisites

Install:

- `kubectl`
- `flux`
- `kustomize` (optional, useful for local validation)
- `sops`
- `age`
- `terraform` (for cluster provisioning)

## 3) Provision Kubernetes with Terraform

Use your preferred provider (Kind, EKS, GKE, AKS, etc.).

Minimum baseline:

- Kubernetes API reachable from your workstation/runner
- Default StorageClass available (PVCs are used by `apps/base/linkding/storage.yaml`)
- Ingress controller installed with class `traefik` (used by `apps/staging/linkding/ingress.yaml`)

## 4) Bootstrap Flux

Use the same bootstrap model already reflected in `clusters/staging/flux-system/gotk-sync.yaml`.

Example:

```bash
export GITHUB_TOKEN=<token>
export GITHUB_USER=<user>

flux bootstrap github \
  --owner="${GITHUB_USER}" \
  --repository="home-lab-kubernetes" \
  --branch="main" \
  --path="clusters/staging" \
  --personal
```

## 5) Enable SOPS decryption in Flux

`clusters/staging/apps.yaml` is configured with:

- decryption provider: `sops`
- secret reference: `sops-age` in `flux-system`

Create the key secret in-cluster (see `docs/sops-age-secrets-manual.md` for full steps).

## 6) Reconciliation and validation

After push to `main`, validate:

```bash
flux check
flux get sources git -A
flux get kustomizations -A
kubectl get pods -n flux-system
kubectl get pods -A
```

## 7) Security checklist

Align with case study objectives:

- Keep Flux-generated `flux-system` manifests untouched.
- Keep secrets encrypted with SOPS at rest in Git.
- Add/maintain NetworkPolicies per namespace as workloads grow.
- Apply least-privilege RBAC for app operators.
- Avoid storing plaintext credentials in Helm values/manifests.
