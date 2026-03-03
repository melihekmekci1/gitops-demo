# ArgoCD + Kustomize GitOps Demo

Production-grade GitOps demo using ArgoCD, Kustomize, and the App-of-Apps pattern.

## Repository Structure

```
gitops-demo/
├── README.md
├── argocd/                              # App-of-Apps: ArgoCD Application manifests
│   ├── kustomization.yaml               # Kustomization for child apps (excludes root-app)
│   ├── root-app.yaml                    # Root application (applied manually)
│   ├── dev-app.yaml                     # Dev environment Application
│   ├── staging-app.yaml                 # Staging environment Application
│   └── prod-app.yaml                    # Prod environment Application (no auto-sync)
├── base/                                # Shared base manifests
│   ├── kustomization.yaml
│   ├── deployment.yaml                  # nginx Deployment
│   └── service.yaml                     # ClusterIP Service
└── overlays/
    ├── dev/                             # StrategicMerge patch, auto-sync
    │   ├── kustomization.yaml
    │   ├── deployment-patch.yaml        # StrategicMergePatch
    │   └── config/
    │       └── app.conf                 # ConfigMapGenerator source
    ├── staging/                         # JSON6902 patch, auto-sync
    │   ├── kustomization.yaml
    │   ├── deployment-patch.json        # JSON6902 patch
    │   └── config/
    │       └── app.conf
    └── prod/                            # Both patch types, manual sync, HPA
        ├── kustomization.yaml
        ├── deployment-patch.yaml        # StrategicMergePatch (replicas + image)
        ├── resource-patch.json          # JSON6902 patch (resources)
        ├── hpa.yaml                     # HorizontalPodAutoscaler
        └── config/
            └── app.conf
```

## Environment Configuration Summary

| Environment | Replicas | Image Tag | CPU Req/Limit | Memory Req/Limit | Auto-Sync | Self-Heal | HPA |
|-------------|----------|-----------|---------------|------------------|-----------|-----------|-----|
| Dev         | 1        | 1.25.3    | 50m / 100m    | 64Mi / 128Mi     | Yes       | Yes       | No  |
| Staging     | 2        | 1.25.4    | 100m / 200m   | 128Mi / 256Mi    | Yes       | Yes       | No  |
| Prod        | 3        | 1.25.5    | 200m / 500m   | 256Mi / 512Mi    | No        | No        | Yes |

## Kustomize Features Demonstrated

- **StrategicMergePatch**: Used in `overlays/dev/` (all fields) and `overlays/prod/` (replicas + image)
- **JSON6902 Patch**: Used in `overlays/staging/` (all fields) and `overlays/prod/` (resources)
- **ConfigMapGenerator**: Each overlay generates `nginx-config` from `config/app.conf`
- **SecretGenerator**: Each overlay generates `nginx-secret` with environment-specific literals
- **namePrefix**: Each overlay prefixes resources (`dev-`, `staging-`, `prod-`)
- **namespace**: Each overlay targets its own namespace

---

## Prerequisites

- A Kubernetes cluster (minikube, kind, k3d, EKS, GKE, etc.)
- `kubectl` configured and connected to the cluster
- `git`

### Quick Start with kind (optional)

```bash
# Install kind if you don't have it
brew install kind

# Create a cluster
kind create cluster --name gitops-demo

# Verify
kubectl cluster-info
```

---

## Step 1: Install ArgoCD

```bash
# Create the argocd namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all ArgoCD components to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
kubectl wait --for=condition=available deployment/argocd-repo-server -n argocd --timeout=300s
kubectl wait --for=condition=available deployment/argocd-applicationset-controller -n argocd --timeout=300s

# Get the initial admin password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD admin password: $ARGOCD_PASSWORD"

# Port-forward the ArgoCD UI (run in background or separate terminal)
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```

Access the ArgoCD UI at **https://localhost:8080**
- Username: `admin`
- Password: output from the command above

### Install ArgoCD CLI (optional but recommended)

```bash
# macOS
brew install argocd

# Login to ArgoCD
argocd login localhost:8080 --username admin --password $ARGOCD_PASSWORD --insecure
```

---

## Step 2: Configure the Repository URL

Update all ArgoCD application manifests with your actual Git repository URL:

```bash
# Set your repository URL
REPO_URL="https://github.com/YOUR-USERNAME/gitops-demo.git"

# Replace the placeholder URL in all ArgoCD manifests (macOS)
find argocd/ -name '*.yaml' -exec sed -i '' "s|https://github.com/example-org/gitops-demo.git|${REPO_URL}|g" {} +

# Replace the placeholder URL in all ArgoCD manifests (Linux)
# find argocd/ -name '*.yaml' -exec sed -i "s|https://github.com/example-org/gitops-demo.git|${REPO_URL}|g" {} +
```

### Push to Git

```bash
git init
git add .
git commit -m "Initial GitOps demo repository"
git remote add origin $REPO_URL
git push -u origin main
```

---

## Step 3: Deploy the Root Application (App-of-Apps)

```bash
# Apply the root application - this creates and manages all child applications
kubectl apply -f argocd/root-app.yaml
```

The root application watches the `argocd/` directory. Through the App-of-Apps pattern, it automatically creates three child ArgoCD Applications:
- `nginx-dev` → `overlays/dev/`
- `nginx-staging` → `overlays/staging/`
- `nginx-prod` → `overlays/prod/`

### Verify Deployment

```bash
# List all ArgoCD applications
argocd app list

# Or with kubectl
kubectl get applications -n argocd

# Check that namespaces were created
kubectl get namespaces | grep nginx

# Verify resources in each environment
kubectl get all -n nginx-dev
kubectl get all -n nginx-staging
kubectl get all -n nginx-prod
```

> **Note**: Dev and staging will sync automatically within ~3 minutes. Prod requires manual sync (see Step 6).

### Sync Prod for the First Time

Since prod has no auto-sync, you must manually trigger the initial sync:

```bash
argocd app sync nginx-prod
```

---

## Step 4: Simulate Drift (Dev Environment)

Drift occurs when someone manually changes a resource in the cluster, causing it to diverge from the Git-declared state.

```bash
# Manually scale the dev deployment to 5 replicas (this is NOT in Git)
kubectl scale deployment dev-nginx -n nginx-dev --replicas=5

# Immediately check - you'll see 5 replicas briefly
kubectl get deployment dev-nginx -n nginx-dev

# Check ArgoCD status - it will detect the drift
argocd app get nginx-dev

# The app will briefly show "OutOfSync", then ArgoCD will auto-correct it
# Watch it happen in real time:
kubectl get deployment dev-nginx -n nginx-dev -w
```

Within the sync interval (~3 minutes, or force it):

```bash
# Force immediate sync to see self-healing instantly
argocd app sync nginx-dev

# Verify replicas reverted to 1 (as declared in Git)
kubectl get deployment dev-nginx -n nginx-dev -o jsonpath='{.spec.replicas}'
# Output: 1
```

---

## Step 5: Demonstrate Self-Healing (Dev and Staging)

Both dev and staging have `selfHeal: true`, meaning ArgoCD automatically reverts any manual changes.

### Test 1: Image Drift

```bash
# Manually change the dev image to an old version
kubectl set image deployment/dev-nginx nginx=nginx:1.24.0 -n nginx-dev

# Watch ArgoCD revert it (wait up to 3 minutes or force sync)
argocd app sync nginx-dev

# Verify the image was restored
kubectl get deployment dev-nginx -n nginx-dev -o jsonpath='{.spec.template.spec.containers[0].image}'
# Output: nginx:1.25.3
```

### Test 2: Resource Drift

```bash
# Manually remove resource limits in staging
kubectl patch deployment staging-nginx -n nginx-staging --type=json \
  -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/resources"}]'

# Force sync
argocd app sync nginx-staging

# Verify resources were restored
kubectl get deployment staging-nginx -n nginx-staging -o jsonpath='{.spec.template.spec.containers[0].resources}'
# Output shows the original resource requests/limits
```

### Test 3: Delete a Resource

```bash
# Delete the dev service
kubectl delete service dev-nginx -n nginx-dev

# ArgoCD will recreate it (force sync or wait)
argocd app sync nginx-dev

# Verify service was recreated
kubectl get service dev-nginx -n nginx-dev
```

---

## Step 6: Demonstrate Manual Prod Sync

Prod has **no auto-sync and no self-heal**. All changes require explicit human approval.

### 6a: Simulate Drift in Prod (Drift Persists)

```bash
# Manually scale prod to 1 replica
kubectl scale deployment prod-nginx -n nginx-prod --replicas=1

# Check ArgoCD status - it shows OutOfSync but does NOT auto-correct
argocd app get nginx-prod

# Verify the drift persists (unlike dev/staging)
kubectl get deployment prod-nginx -n nginx-prod -o jsonpath='{.spec.replicas}'
# Output: 1 (drift persists!)

# Review what changed
argocd app diff nginx-prod

# Manually approve and sync to restore desired state
argocd app sync nginx-prod

# Verify replicas restored to 3
kubectl get deployment prod-nginx -n nginx-prod -o jsonpath='{.spec.replicas}'
# Output: 3
```

### 6b: Git-Driven Change (Requires Manual Sync)

```bash
# 1. Edit the prod image tag in Git
sed -i '' 's/nginx:1.25.5/nginx:1.26.0/' overlays/prod/deployment-patch.yaml
# On Linux: sed -i 's/nginx:1.25.5/nginx:1.26.0/' overlays/prod/deployment-patch.yaml

# 2. Commit and push
git add overlays/prod/deployment-patch.yaml
git commit -m "Bump prod nginx to 1.26.0"
git push

# 3. Wait for ArgoCD to detect the change (or refresh manually)
argocd app get nginx-prod --refresh

# 4. Status shows OutOfSync
argocd app get nginx-prod
# Health: Healthy | Sync: OutOfSync

# 5. Review the diff before approving
argocd app diff nginx-prod

# 6. Approve and sync
argocd app sync nginx-prod

# 7. Verify the new image is deployed
kubectl get deployment prod-nginx -n nginx-prod -o jsonpath='{.spec.template.spec.containers[0].image}'
# Output: nginx:1.26.0
```

---

## Step 7: Verify All Kustomize Features

### ConfigMaps (via ConfigMapGenerator)

```bash
# Each environment has a unique configmap with content-hash suffix
kubectl get configmaps -n nginx-dev
kubectl get configmaps -n nginx-staging
kubectl get configmaps -n nginx-prod

# Inspect the dev configmap
kubectl get configmap -n nginx-dev -l env=dev -o yaml
```

### Secrets (via SecretGenerator)

```bash
# Each environment has a unique secret with content-hash suffix
kubectl get secrets -n nginx-dev -l env=dev
kubectl get secrets -n nginx-staging -l env=staging
kubectl get secrets -n nginx-prod -l env=prod
```

### HPA (Prod Only)

```bash
# Verify HPA exists only in prod
kubectl get hpa -n nginx-prod
# NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS
# prod-nginx   Deployment/prod-nginx   ...       3         10        3

kubectl get hpa -n nginx-dev
# No resources found in nginx-dev namespace.

kubectl get hpa -n nginx-staging
# No resources found in nginx-staging namespace.
```

### Verify Environment-Specific Values

```bash
echo "=== Dev ==="
echo "Replicas: $(kubectl get deployment dev-nginx -n nginx-dev -o jsonpath='{.spec.replicas}')"
echo "Image:    $(kubectl get deployment dev-nginx -n nginx-dev -o jsonpath='{.spec.template.spec.containers[0].image}')"
echo "CPU Req:  $(kubectl get deployment dev-nginx -n nginx-dev -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')"
echo "Mem Req:  $(kubectl get deployment dev-nginx -n nginx-dev -o jsonpath='{.spec.template.spec.containers[0].resources.requests.memory}')"

echo ""
echo "=== Staging ==="
echo "Replicas: $(kubectl get deployment staging-nginx -n nginx-staging -o jsonpath='{.spec.replicas}')"
echo "Image:    $(kubectl get deployment staging-nginx -n nginx-staging -o jsonpath='{.spec.template.spec.containers[0].image}')"
echo "CPU Req:  $(kubectl get deployment staging-nginx -n nginx-staging -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')"
echo "Mem Req:  $(kubectl get deployment staging-nginx -n nginx-staging -o jsonpath='{.spec.template.spec.containers[0].resources.requests.memory}')"

echo ""
echo "=== Prod ==="
echo "Replicas: $(kubectl get deployment prod-nginx -n nginx-prod -o jsonpath='{.spec.replicas}')"
echo "Image:    $(kubectl get deployment prod-nginx -n nginx-prod -o jsonpath='{.spec.template.spec.containers[0].image}')"
echo "CPU Req:  $(kubectl get deployment prod-nginx -n nginx-prod -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')"
echo "Mem Req:  $(kubectl get deployment prod-nginx -n nginx-prod -o jsonpath='{.spec.template.spec.containers[0].resources.requests.memory}')"
```

Expected output:
```
=== Dev ===
Replicas: 1
Image:    nginx:1.25.3
CPU Req:  50m
Mem Req:  64Mi

=== Staging ===
Replicas: 2
Image:    nginx:1.25.4
CPU Req:  100m
Mem Req:  128Mi

=== Prod ===
Replicas: 3
Image:    nginx:1.25.5
CPU Req:  200m
Mem Req:  256Mi
```

---

## Cleanup

```bash
# Delete all applications (cascading delete removes all managed resources)
argocd app delete gitops-demo-root --cascade -y

# Or with kubectl
kubectl delete application gitops-demo-root -n argocd

# Wait for child apps to be cleaned up
sleep 10

# Delete namespaces (if not already removed by cascade)
kubectl delete namespace nginx-dev nginx-staging nginx-prod --ignore-not-found

# Uninstall ArgoCD
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd

# Delete kind cluster (if using kind)
kind delete cluster --name gitops-demo
```

---

## Architecture

### App-of-Apps Pattern

```
gitops-demo-root (auto-sync, manages argocd/)
├── nginx-dev       (auto-sync + selfHeal, manages overlays/dev/)
├── nginx-staging   (auto-sync + selfHeal, manages overlays/staging/)
└── nginx-prod      (manual sync only, manages overlays/prod/)
```

The root application monitors the `argocd/` directory. When a new environment Application YAML is added there, ArgoCD automatically picks it up and deploys it. Removing an Application YAML triggers pruning.

### Patch Strategy

- **Dev** (`overlays/dev/deployment-patch.yaml`): **StrategicMergePatch** — a partial Kubernetes resource YAML that is merged with the base. Simple and readable; ideal for straightforward overrides.
- **Staging** (`overlays/staging/deployment-patch.json`): **JSON6902** — RFC 6902 JSON Patch operations (`add`, `replace`, `remove`). Precise and surgical; useful when you need to target specific array elements or paths.
- **Prod**: Uses **both** — StrategicMerge for replicas/image (`deployment-patch.yaml`) and JSON6902 for resources (`resource-patch.json`), demonstrating that both patch types can coexist in a single overlay.

### Sync Policies

- **Dev / Staging**: `automated: { prune: true, selfHeal: true }` — ArgoCD automatically syncs Git changes and reverts manual drift.
- **Prod**: No `automated` block — requires explicit `argocd app sync nginx-prod` to apply changes. Drift is detected but not corrected automatically.
# gitops-demo
