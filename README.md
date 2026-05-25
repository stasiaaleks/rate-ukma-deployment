# RateUKMA — Kubernetes Infrastructure

GitOps infrastructure for [RateUKMA](https://github.com/ukma-cs-ssdm-2025/rate-ukma) — a course and professor rating application for NaUKMA students.

## Stack

| Component | Technology |
|---|---|
| Application | Django (Gunicorn) + React SPA (Vite) |
| Database | PostgreSQL — managed by **CloudNativePG** operator |
| Cache / Sessions | Redis-compatible — managed by **Dragonfly** operator |
| GitOps | FluxCD v2 |
| Secrets | Sealed Secrets (Bitnami) |
| Ingress | NGINX Ingress Controller |
| Packaging | Helm chart (`./charts/rate-ukma`) |

## Repository Structure

```
.
├── clusters/
│   ├── staging/        # Flux bootstrap path for staging cluster
│   └── production/     # Flux bootstrap path for production cluster
├── infrastructure/
│   ├── controllers/    # Operators: CNPG, Dragonfly, nginx-ingress, Sealed Secrets
│   └── image-automation/  # Flux image reflection & automation
├── charts/
│   └── rate-ukma/     # Helm chart (backend + frontend + DB CRs)
└── environments/
    ├── staging/        # staging HelmRelease + SealedSecrets
    └── production/     # production HelmRelease + SealedSecrets
```

## Environments

| Environment | Namespace | Domain | Replicas | DB |
|---|---|---|---|---|
| Staging | `staging` | `app.staging.local` | 1 backend | CNPG single instance |
| Production | `production` | `k8s-beta.rateukma.com` | 2–5 (HPA) | CNPG primary + 1 replica |

## VM Requirements

| Env | Size | Spec |
|---|---|---|
| Staging | Hetzner CX22 | 2 vCPU · 4 GB RAM · 40 GB SSD |
| Production | Hetzner CX32 | 4 vCPU · 8 GB RAM · 80 GB SSD |

k3s (without Traefik) is installed on each VM. The Flux bootstrap points each cluster to its own path in this repo.

---

## Bootstrap Instructions

### 1. Provision VMs & install k3s

```bash
# Run on each VM
curl -sfL https://get.k3s.io | sh -s - --disable traefik

# Copy kubeconfig to local machine
scp root@<staging-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/staging.yaml
scp root@<prod-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/prod.yaml

# Update server addresses in kubeconfigs
sed -i 's/127.0.0.1/<staging-ip>/' ~/.kube/staging.yaml
sed -i 's/127.0.0.1/<prod-ip>/' ~/.kube/prod.yaml
```

### 2. Install Flux CLI

```bash
curl -s https://fluxcd.io/install.sh | bash
```

### 3. Bootstrap Flux on each cluster

```bash
export GITHUB_TOKEN=<your-pat>   # needs repo write access

# Staging
KUBECONFIG=~/.kube/staging.yaml flux bootstrap github \
  --owner=stasiaaleks \
  --repository=rate-ukma-deployment \
  --branch=main \
  --path=clusters/staging \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller

# Production
KUBECONFIG=~/.kube/prod.yaml flux bootstrap github \
  --owner=stasiaaleks \
  --repository=rate-ukma-deployment \
  --branch=main \
  --path=clusters/production \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller
```

Flux will commit `gotk-components.yaml`, `gotk-sync.yaml`, and `kustomization.yaml` into `clusters/staging/flux-system/` and `clusters/production/flux-system/`. Pull those commits locally before proceeding.

### 4. Create Sealed Secrets

Each cluster has its own Sealed Secrets controller with its own encryption key. Secrets sealed for staging **cannot** be decrypted by the production cluster.

```bash
# Install kubeseal CLI (run on your local machine, not the VM)
# macOS
brew install kubeseal
# Linux (Debian/Ubuntu)
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest | grep tag_name | cut -d '"' -f4 | sed 's/^v//')
curl -Lo kubeseal.tar.gz "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xzf kubeseal.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# Fetch public certs (wait for sealed-secrets-controller to be Ready first)
KUBECONFIG=~/.kube/staging.yaml kubeseal --fetch-cert \
  --controller-namespace kube-system \
  --controller-name sealed-secrets-controller \
  > staging-cert.pem

KUBECONFIG=~/.kube/prod.yaml kubeseal --fetch-cert \
  --controller-namespace kube-system \
  --controller-name sealed-secrets-controller \
  > prod-cert.pem
```

#### Django app secret (both environments)

Create `secret-django.yaml` (do NOT commit this file):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rate-ukma-django
  namespace: staging   # change to "production" for prod
type: Opaque
stringData:
  DJANGO_SECRET_KEY: "<generate with: python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'>"
  MICROSOFT_CLIENT_ID: "<azure-app-client-id>"
  MICROSOFT_CLIENT_SECRET: "<azure-app-client-secret>"
  MICROSOFT_TENANT_ID: "<azure-tenant-id>"
  SENTRY_DSN_BACKEND: ""
  VITE_SENTRY_DSN_FRONTEND: ""
  CORPORATE_EMAIL: ""
  CORPORATE_PASSWORD: ""
```

Seal and commit:
```bash
kubeseal --cert staging-cert.pem --format yaml \
  < secret-django.yaml \
  > environments/staging/sealed-secrets/django-sealed.yaml

# Repeat with prod-cert.pem and namespace: production
kubeseal --cert prod-cert.pem --format yaml \
  < secret-django-prod.yaml \
  > environments/production/sealed-secrets/django-sealed.yaml
```

#### PostgreSQL credentials (both environments)

```yaml
# secret-postgres.yaml — do NOT commit
apiVersion: v1
kind: Secret
metadata:
  name: rate-ukma-postgres
  namespace: staging
type: Opaque
stringData:
  username: rateukma
  password: "<strong-password>"
```

```bash
kubeseal --cert staging-cert.pem --format yaml \
  < secret-postgres.yaml \
  > environments/staging/sealed-secrets/postgres-sealed.yaml
```

After sealing, add the files to the kustomization:
```yaml
# environments/staging/sealed-secrets/kustomization.yaml
resources:
  - django-sealed.yaml
  - postgres-sealed.yaml
```

Commit and push. Flux will pick up the sealed secrets and the Sealed Secrets controller will decrypt them into regular K8s Secrets.

### 5. Update /etc/hosts (local access)

```
<staging-vm-ip>  app.staging.local
<prod-vm-ip>     k8s-beta.rateukma.com
```

### 6. Update Azure App Registration

Add redirect URIs in the Azure portal for your Microsoft OAuth app:
- `http://app.staging.local/accounts/microsoft/login/callback/`
- `http://k8s-beta.rateukma.com/accounts/microsoft/login/callback/`

---

## Verification

```bash
# All HelmReleases should be Ready
kubectl get helmreleases -A

# All Kustomizations should be Applied
flux get kustomizations -A

# Pods in both namespaces
kubectl get pods -A

# Ingresses
kubectl get ingress -A

# Test self-healing (Flux restores within ~1 minute)
kubectl delete deployment -n production rate-ukma-backend
kubectl get pods -n production -w

# HPA in production
kubectl get hpa -n production
```

---

## Image Automation

When CI pushes a new image to GHCR, Flux Image Reflector detects it. Image Automation commits the new tag back to this repo (in `environments/*/helmrelease.yaml`). Flux then performs a rolling update.

Tags tracked:
- `backend`: semver (`>=0.0.0`)
- `webapp:staging-*`: staging environment
- `webapp:live-*`: production environment

Note: `flux bootstrap` must be run with `--components-extra=image-reflector-controller,image-automation-controller` for image automation to work.
