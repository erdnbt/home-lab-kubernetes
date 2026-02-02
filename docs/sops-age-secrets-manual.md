# SOPS + age Secrets Manual

This repo already uses SOPS encryption for Kubernetes Secrets.

## 1) Current repo configuration

- SOPS rules file: `clusters/staging/.sops.yaml`
- Encryption rule:
  - `path_regex: .*.yaml`
  - `encrypted_regex: ^(data|stringData)$`
  - age recipient configured
- Encrypted examples:
  - `apps/staging/linkding/cloudflare-secret.yaml`
  - `apps/staging/linkding/linkding-container-env-secret.yaml`

## 2) Generate age key pair (one time)

```bash
brew install sops age
age-keygen -o age.agekey
```

Show recipient public key:

```bash
age-keygen -y age.agekey
export AGE_PUBLIC=$(age-keygen -y age.agekey)
```

Use that public key in `.sops.yaml` `age:` recipients.

## 3) Create Flux decryption secret

Flux needs the private key in `flux-system` namespace:

```bash
cat age.agekey |
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

## 4) Create a new encrypted Kubernetes Secret

### Example workflow (API_KEY):

```bash
kubectl create secret generic my-app-secret \
  --from-literal=API_KEY='replace-me' \
  --dry-run=client \
  -o yaml > apps/staging/my-app/my-app-secret.yaml

sops --age=$AGE_PUBLIC \
  --encrypt --encrypted-regex '^(data|stringData)$' \
  --in-place apps/staging/my-app/my-app-secret.yaml
```

### Example workflow (username/password):

```bash
kubectl create secret generic test-secret \
  --from-literal=user=USERNAME \
  --from-literal=password=PASSWORD \
  --dry-run=client \
  -o yaml > apps/staging/my-app/my-app-user-secret.yaml

sops --age=$AGE_PUBLIC \
  --encrypt --encrypted-regex '^(data|stringData)$' \
  --in-place apps/staging/my-app/my-app-user-secret.yaml
```

Optional edit later:

```bash
sops apps/staging/my-app/my-app-secret.yaml
```

### Example workflow (TLS):

```bash
# Generate the private key and certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ./tls.key \
  -out ./tls.crt \
  -subj "/C=DE/ST=Hesse/L=Darmstadt/O=Sakura Ops Inc./OU=Department of Monitoring/CN=grafana.sakuraops.dev" \
  -addext "subjectAltName=DNS:grafana.sakuraops.dev"

kubectl create secret tls grafana-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  --namespace=monitoring \
  --dry-run=client \
  -o yaml > grafana-tls-secret.yaml

sops --age=$AGE_PUBLIC \
  --encrypt --encrypted-regex '^(data|stringData)$' \
  --in-place grafana-tls-secret.yaml
```


## 5) Validate before commit

```bash
sops --decrypt apps/staging/my-app/my-app-secret.yaml | head -n 40
kubectl kustomize apps/staging/my-app
```

## 6) Key rotation

When rotating age recipients:

1. Update recipient list in `.sops.yaml`.
2. Re-encrypt affected files:

```bash
sops updatekeys -y apps/staging/**/*.yaml
```

3. Update `sops-age` secret in `flux-system` with the new private key.

## 7) Guardrails

- Never commit plaintext Secrets.
- Keep metadata minimal when generating Secret YAML (avoid runtime-only fields).
- Encrypt only `data`/`stringData` to preserve manifest readability.
- Store private age key securely; treat it as production secret material.
