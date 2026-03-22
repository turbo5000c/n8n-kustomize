# n8n-kustomize

Kustomize manifests for deploying [n8n](https://n8n.io) — a fair-code workflow automation platform — on Kubernetes.  
Designed with Home Assistant and FluxCD users in mind, but usable with any Kubernetes cluster.

---

## Table of Contents

1. [What is this?](#what-is-this)
2. [What problem does it solve?](#what-problem-does-it-solve)
3. [Home Assistant integration](#home-assistant-integration)
4. [Architecture overview](#architecture-overview)
5. [Prerequisites](#prerequisites)
6. [Configuration](#configuration)
7. [Installation — FluxCD (GitOps)](#installation--fluxcd-gitops)
8. [Installation — kubectl](#installation--kubectl)
9. [Accessing the n8n UI](#accessing-the-n8n-ui)
10. [Upgrading](#upgrading)
11. [Troubleshooting](#troubleshooting)

---

## What is this?

This repository provides ready-to-use Kubernetes manifests for n8n, managed with [Kustomize](https://kustomize.io).  
It gives you:

- A **Deployment** running the official `n8nio/n8n` container image
- A **PersistentVolumeClaim** for n8n's workflow and credential data
- A **ClusterIP Service** exposing n8n inside the cluster
- Secret **templates** for the PostgreSQL connection and n8n encryption key
- **FluxCD examples** so GitOps users can deploy with a single `kubectl apply`

---

## What problem does it solve?

Installing n8n on Kubernetes from scratch requires writing and coordinating several manifest files.  
This repository provides a working, opinionated baseline so you can be up and running in minutes instead of hours.

Key decisions already made for you:

| Decision | Choice | Reason |
|----------|--------|--------|
| Deployment strategy | `Recreate` | Prevents two pods writing to the same PVC simultaneously |
| Init container | `busybox chown` | Ensures the PVC is writable by the n8n process (UID 1000) |
| Database | PostgreSQL (via secret) | More reliable than SQLite for long-running deployments |
| Service type | `ClusterIP` | Keeps n8n internal; you choose how to expose it (Ingress, port-forward, etc.) |

---

## Home Assistant integration

n8n pairs naturally with [Home Assistant](https://www.home-assistant.io) for advanced home automation:

- **Webhooks** — expose an n8n webhook URL and trigger workflows from Home Assistant automations using the `RESTful Command` or `Webhook` integrations.
- **REST API** — call Home Assistant's REST API from an n8n HTTP Request node to read sensor states, fire events, or control devices.
- **Long-running automations** — build multi-step automations (e.g. send a notification, wait for a response, then act) that are difficult to express in Home Assistant's automation editor.

After deploying n8n, create a workflow with an **HTTP Request** node pointed at your Home Assistant instance (`http://homeassistant.local:8123/api/...`) and use a [long-lived access token](https://developers.home-assistant.io/docs/auth_api/#long-lived-access-token) for authentication.

---

## Architecture overview

```
┌─────────────────────────────────────────┐
│  Kubernetes cluster (namespace: n8n)    │
│                                         │
│  ┌──────────┐    ┌───────────────────┐  │
│  │ Service  │───▶│   Deployment      │  │
│  │ ClusterIP│    │   (n8nio/n8n)     │  │
│  │ :5678    │    │                   │  │
│  └──────────┘    │  initContainer:   │  │
│                  │  chown /data      │  │
│                  └────────┬──────────┘  │
│                           │             │
│                  ┌────────▼──────────┐  │
│                  │  PVC n8n-claim0   │  │
│                  │  (2 Gi)           │  │
│                  └───────────────────┘  │
│                                         │
│  Secrets (managed outside Git):         │
│    postgres-secret   n8n-secret         │
└─────────────────────────────────────────┘
         │
         │ DB_POSTGRESDB_HOST (from secret)
         ▼
  External PostgreSQL instance
```

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| Kubernetes ≥ 1.25 | Any distribution (k3s, k0s, kind, EKS, GKE, AKS, …) |
| `kubectl` configured | `kubectl cluster-info` must succeed |
| Kustomize ≥ 5.0 | Included in `kubectl` ≥ 1.14 via `kubectl apply -k` |
| A PostgreSQL instance | Must be reachable from the cluster (see [Configuration](#configuration)) |
| A StorageClass | Any that supports `ReadWriteOnce` PVCs |
| FluxCD ≥ 2.0 *(optional)* | Required only for the GitOps installation method |

---

## Configuration

Before deploying you must provide two Kubernetes Secrets.  
**Do not commit real values to Git.** Use a secret manager instead (see options below).

### Secret: `postgres-secret`

Created from the template in [`n8n-postgresql.yaml`](n8n-postgresql.yaml):

| Key | Description |
|-----|-------------|
| `POSTGRES_HOST` | Hostname/IP of your PostgreSQL server |
| `POSTGRES_DB` | Database name (default: `n8n`) |
| `POSTGRES_NON_ROOT_USER` | Database user n8n connects as |
| `POSTGRES_NON_ROOT_PASSWORD` | Password for that user |

### Secret: `n8n-secret`

Created from the template in [`n8n-secret.yaml`](n8n-secret.yaml):

| Key | Description |
|-----|-------------|
| `N8N_ENCRYPTION_KEY` | Random ≥ 32-character string used to encrypt stored credentials |

Generate a key with:

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

> **Important:** Changing `N8N_ENCRYPTION_KEY` after first use will invalidate all previously saved credentials in n8n.

### Recommended secret management options

| Tool | When to use |
|------|-------------|
| [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) | Simple GitOps-friendly approach; secrets are encrypted in Git |
| [SOPS](https://github.com/mozilla/sops) | Works with age, PGP, or cloud KMS; integrates with FluxCD |
| [External Secrets Operator](https://external-secrets.io) | Syncs secrets from AWS SM, Vault, 1Password, etc. |

---

## Installation — FluxCD (GitOps)

Use this method if you already have [FluxCD](https://fluxcd.io) running on your cluster.

### Step 1 — Create the secrets

Create the two secrets in the `n8n` namespace **before** Flux applies the manifests.  
Example using `kubectl` (replace placeholder values):

```bash
kubectl create namespace n8n

kubectl create secret generic postgres-secret \
  --namespace n8n \
  --from-literal=POSTGRES_HOST=postgres.database.svc.cluster.local \
  --from-literal=POSTGRES_DB=n8n \
  --from-literal=POSTGRES_NON_ROOT_USER=n8nuser \
  --from-literal=POSTGRES_NON_ROOT_PASSWORD=changeme

kubectl create secret generic n8n-secret \
  --namespace n8n \
  --from-literal=N8N_ENCRYPTION_KEY=$(python3 -c "import secrets; print(secrets.token_hex(32))")
```

Or manage them with Sealed Secrets / SOPS as described in [Configuration](#configuration).

### Step 2 — Apply the FluxCD resources

```bash
# Register this repository as a Flux source
kubectl apply -f https://raw.githubusercontent.com/turbo5000c/n8n-kustomize/main/examples/flux/gitrepository.yaml

# Tell Flux to apply the manifests from this repository
kubectl apply -f https://raw.githubusercontent.com/turbo5000c/n8n-kustomize/main/examples/flux/kustomization.yaml
```

Or copy the files from [`examples/flux/`](examples/flux/) into your own Flux repository and commit them.

### Step 3 — Watch the rollout

```bash
# Watch Flux reconcile the Kustomization
flux get kustomization n8n --watch

# Watch the Deployment
kubectl rollout status deployment/n8n -n n8n
```

### Repository layout expected by Flux

```
n8n-kustomize/
├── kustomization.yaml   ← Flux reads this
├── namespace.yaml
├── deployment.yaml
├── pvc.yaml
├── service.yaml
├── n8n-secret.yaml      ← template only; populate via secret manager
├── n8n-postgresql.yaml  ← template only; populate via secret manager
└── examples/
    └── flux/
        ├── gitrepository.yaml
        └── kustomization.yaml
```

---

## Installation — kubectl

Use this method if you are not using FluxCD.

### Step 1 — Clone the repository

```bash
git clone https://github.com/turbo5000c/n8n-kustomize.git
cd n8n-kustomize
```

### Step 2 — Create the namespace and secrets

```bash
kubectl apply -f namespace.yaml

# Create PostgreSQL connection secret
kubectl create secret generic postgres-secret \
  --namespace n8n \
  --from-literal=POSTGRES_HOST=<your-postgres-host> \
  --from-literal=POSTGRES_DB=n8n \
  --from-literal=POSTGRES_NON_ROOT_USER=<your-db-user> \
  --from-literal=POSTGRES_NON_ROOT_PASSWORD=<your-db-password>

# Create n8n encryption key secret
kubectl create secret generic n8n-secret \
  --namespace n8n \
  --from-literal=N8N_ENCRYPTION_KEY=$(python3 -c "import secrets; print(secrets.token_hex(32))")
```

### Step 3 — Apply the manifests

```bash
kubectl apply -k .
```

This applies `kustomization.yaml`, which deploys the Deployment, PVC, and Service in the `n8n` namespace.

### Step 4 — Verify the deployment

```bash
kubectl get all -n n8n
kubectl rollout status deployment/n8n -n n8n
```

---

## Accessing the n8n UI

n8n is exposed as a `ClusterIP` service on port `5678`. Choose one of the following methods to access it:

### Port-forward (quickest, for testing)

```bash
kubectl port-forward svc/n8n 5678:5678 -n n8n
```

Then open [http://localhost:5678](http://localhost:5678) in your browser.

### Ingress (recommended for permanent access)

Add an Ingress resource for your ingress controller. Example for nginx-ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: n8n
  namespace: n8n
spec:
  rules:
    - host: n8n.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: n8n
                port:
                  number: 5678
```

---

## Upgrading

n8n releases new versions frequently. To upgrade:

1. Check the [n8n releases page](https://github.com/n8n-io/n8n/releases) for the latest version.
2. Update the image tag in `deployment.yaml` (or pin it in your Kustomize overlay):

   ```yaml
   image: n8nio/n8n:1.x.y   # replace with the target version
   ```

3. Apply the change:

   ```bash
   # kubectl
   kubectl apply -k .

   # FluxCD — commit the change to Git; Flux will reconcile automatically
   git add deployment.yaml && git commit -m "chore: upgrade n8n to 1.x.y" && git push
   ```

4. Monitor the rollout:

   ```bash
   kubectl rollout status deployment/n8n -n n8n
   ```

> **Tip:** Pin the image to a specific digest (e.g. `n8nio/n8n@sha256:…`) for fully reproducible deployments.

---

## Troubleshooting

### Pod is stuck in `Init:Error` or `Init:CrashLoopBackOff`

The `volume-permissions` init container failed to `chown` the PVC mount. Check whether the StorageClass allows the `busybox` container to write to the volume:

```bash
kubectl describe pod -n n8n -l service=n8n
kubectl logs -n n8n -l service=n8n -c volume-permissions
```

### Pod is stuck in `CrashLoopBackOff`

n8n failed to start. Check the logs:

```bash
kubectl logs -n n8n -l service=n8n
```

Common causes:
- Missing or incorrect `postgres-secret` / `n8n-secret`
- PostgreSQL is unreachable from the pod (check the `POSTGRES_HOST` value and network policies)

### Secrets not found

Verify both secrets exist in the `n8n` namespace:

```bash
kubectl get secret -n n8n
```

If either is missing, re-run the `kubectl create secret` commands from the installation steps.

### Flux Kustomization is not ready

```bash
flux get kustomization n8n
flux logs --kind=Kustomization --name=n8n --namespace=flux-system
```

Ensure the `GitRepository` is healthy first:

```bash
flux get source git n8n-kustomize
```
