# GitOps-Infra

Kubernetes deployment manifests and ArgoCD configurations for the GitOps demo application. This repository implements the **App of Apps** pattern with Kustomize overlays for multi-environment deployments.

## 🎯 Purpose

This repository contains the **deployment configuration** for the [GitOps-App](https://github.com/ThorntonCloud/GitOps-App) nginx application. It demonstrates GitOps best practices by separating application source code from deployment manifests, enabling:

- **Declarative Infrastructure**: All Kubernetes resources defined in Git
- **Environment Promotion**: Progressive delivery through dev → staging → prod
- **Automated Sync**: ArgoCD continuously reconciles desired state
- **Audit Trail**: Complete history of infrastructure changes via Git

## 🏗️ Repository Structure

```
.
├── argocd/
│   ├── application.yaml              # App of Apps parent application
│   └── appsets/
│       └── applicationset.yaml       # ApplicationSet for multi-environment deployment
│
└── nginx-app/
    ├── base/                         # Base Kubernetes manifests
    │   ├── kustomization.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── configmap.yaml            # HTML content (index, 404, 50x pages)
    │
    └── overlays/                     # Environment-specific configurations
        ├── dev/
        │   ├── kustomization.yaml
        │   ├── namespace.yaml
        │   └── patches/              # Dev-specific customizations
        │       ├── deployment.yaml   # (latest tag, 1 replica, low resources)
        │       ├── service.yaml      # (NodePort)
        │       └── configmap.yaml
        │
        ├── staging/
        │   ├── kustomization.yaml
        │   ├── namespace.yaml
        │   └── patches/              # Staging-specific customizations
        │       ├── deployment.yaml   # (staging tag, 2 replicas, medium resources)
        │       ├── service.yaml      # (LoadBalancer)
        │       └── configmap.yaml
        │
        └── prod/
            ├── kustomization.yaml
            ├── namespace.yaml
            └── patches/              # Production-specific customizations
                ├── deployment.yaml   # (semantic version, 3+ replicas, high resources)
                ├── service.yaml      # (LoadBalancer)
                └── configmap.yaml
```

## 🔄 App of Apps Pattern

This repository implements ArgoCD's **App of Apps** pattern for managing multiple applications and environments:

```
┌─────────────────────────────────────────────────────────────┐
│  ArgoCD Application: applicationset-controller               │
│  (argocd/application.yaml)                                   │
│                                                               │
│  Watches: argocd/appsets/                                    │
│  Auto-sync: Enabled with self-heal                           │
└───────────────┬─────────────────────────────────────────────┘
                │
                │ Deploys
                │
                ▼
┌─────────────────────────────────────────────────────────────┐
│  ArgoCD ApplicationSet: nginx-app                            │
│  (argocd/appsets/applicationset.yaml)                        │
│                                                               │
│  Generates 3 Applications using List Generator:              │
└───────────────┬─────────────────────────────────────────────┘
                │
                ├─────────────────┬─────────────────┬───────────
                │                 │                 │
                ▼                 ▼                 ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ dev-nginx-app    │  │ staging-nginx-app│  │ prod-nginx-app   │
│                  │  │                  │  │                  │
│ Namespace: dev   │  │ Namespace: staging│ │ Namespace: prod  │
│ Path: overlays/  │  │ Path: overlays/  │  │ Path: overlays/  │
│       dev/       │  │       staging/   │  │       prod/      │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

### Why App of Apps?

- **Single Source of Truth**: One ApplicationSet manages all environments
- **DRY Principle**: Environment differences handled via Kustomize overlays
- **Consistent Deployments**: Same base manifests, different configurations
- **Easy Scaling**: Add new environments by updating the list generator

## 🌍 Environment Configurations

| Environment | Namespace | Image Tag | Replicas | Resources | Service Type | Auto-Sync | Use Case |
|-------------|-----------|-----------|----------|-----------|--------------|-----------|----------|
| **dev** | `dev` | `latest` | 1 | Low | NodePort | ✅ Yes | Feature testing, rapid iteration |
| **staging** | `staging` | `staging` | 2 | Medium | LoadBalancer | ✅ Yes | QA validation, pre-prod testing |
| **prod** | `prod` | `v1.0.0` (semantic) | 3+ | High | LoadBalancer | ✅ Yes* | Production workloads |

**\*Note:** In a real production environment, prod would typically have `automated: false` to require manual approval for deployments.

### Environment Details

#### Development (dev)
- **Purpose**: Rapid development and testing
- **Image**: Uses `latest` tag for automatic updates
- **Access**: NodePort service for local/cluster access
- **Resources**: Minimal resource requests/limits
- **Updates**: Automatically syncs on every commit to main

#### Staging (staging)
- **Purpose**: QA validation and integration testing
- **Image**: Uses `staging` tag (promoted via GitOps-App workflow)
- **Access**: LoadBalancer for external testing
- **Resources**: Production-like resource allocation
- **Updates**: Automatically syncs when staging tag is updated

#### Production (prod)
- **Purpose**: Live customer-facing application
- **Image**: Uses semantic versioning tags (e.g., `v1.0.0`, `v1.2.3`)
- **Access**: LoadBalancer with production ingress
- **Resources**: High availability with multiple replicas and resource guarantees
- **Updates**: Requires manual image tag update in kustomization.yaml

### Deployment Topology

**Current Setup** (Single Cluster):
```
Kubernetes Cluster
├── Namespace: dev (NodePort :30080)
├── Namespace: staging (LoadBalancer)
└── Namespace: prod (LoadBalancer)
```

**Production-Ready Setup** (Multi-Cluster):
```
Development Cluster
└── Namespace: dev

Staging Cluster
└── Namespace: staging

Production Cluster
└── Namespace: prod
```

## 🚀 GitOps Workflow

### Complete Deployment Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│  1. Developer commits code to GitOps-App                     │
└───────────────┬─────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────┐
│  2. CI builds and pushes image                               │
│     thorntoncloud/nginx-gitops:abc1234                       │
│     thorntoncloud/nginx-gitops:latest                        │
└───────────────┬─────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Dev environment auto-deploys (latest tag)                │
│     ArgoCD syncs dev overlay → Deployed to dev namespace     │
└───────────────┬─────────────────────────────────────────────┘
                │
                │ Testing and validation
                │
                ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Manual promotion to staging                              │
│     Workflow: promote-image.yaml (GitOps-App)                │
│     Creates: thorntoncloud/nginx-gitops:staging              │
└───────────────┬─────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────┐
│  5. Staging environment auto-deploys                         │
│     ArgoCD syncs staging overlay → Deployed to staging       │
└───────────────┬─────────────────────────────────────────────┘
                │
                │ QA validation
                │
                ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Manual promotion to production                           │
│     Workflow: promote-image.yaml (GitOps-App)                │
│     Creates: thorntoncloud/nginx-gitops:v1.0.0               │
└───────────────┬─────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Manual update to prod kustomization.yaml                 │
│     Edit: nginx-app/overlays/prod/kustomization.yaml         │
│     Change: newTag: v1.0.0                                   │
│     Commit and push to GitOps-Infra                          │
└───────────────┬─────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────┐
│  8. Production deployment                                    │
│     ArgoCD syncs prod overlay → Deployed to prod namespace   │
└─────────────────────────────────────────────────────────────┘
```

## 🔧 Kustomize Structure

### Base Manifests

The `base/` directory contains the core Kubernetes resources:

**Deployment** (`deployment.yaml`):
- Container: `thorntoncloud/nginx-gitops`
- Port: 8080
- Health checks: `/health` endpoint
- Security: Non-root user, read-only root filesystem
- Probes: Liveness and readiness configured

**Service** (`service.yaml`):
- Targets port 8080 on pods
- Service type overridden in overlays

**ConfigMap** (`configmap.yaml`):
- Contains HTML files: `index.html`, `404.html`, `50x.html`
- Mounted into nginx container at `/usr/share/nginx/html`

### Overlays (Environment-Specific Patches)

Each environment uses Kustomize patches to customize the base manifests:

**Common Patch Patterns:**

```yaml
# deployment.yaml patch
- Replica count (dev: 1, staging: 2, prod: 3+)
- Resource requests/limits
- Image tag (latest, staging, v1.0.0)
- Environment-specific labels/annotations

# service.yaml patch
- Service type (NodePort, LoadBalancer)
- Port mappings (NodePort: 30080)

# configmap.yaml patch
- Environment-specific HTML content
- Custom error pages per environment
```

### Example: Promoting an Image to Production

1. **Promote image** using GitOps-App workflow:
```bash
# Via GitHub Actions in GitOps-App repo
Source SHA: abc1234
Target Tag: v1.2.0
Environment: prod
```

2. **Update image tag** in this repo:
```bash
# Edit nginx-app/overlays/prod/kustomization.yaml
git clone https://github.com/ThorntonCloud/GitOps-Infra.git
cd GitOps-Infra

# Update the newTag field
vim nginx-app/overlays/prod/kustomization.yaml
```

```yaml
images:
  - name: thorntoncloud/nginx-gitops
    newTag: v1.2.0  # Update this line
```

3. **Commit and push**:
```bash
git add nginx-app/overlays/prod/kustomization.yaml
git commit -m "chore: update prod to v1.2.0"
git push origin main
```

4. **ArgoCD syncs automatically** (or manually via ArgoCD UI)

## 🔍 ArgoCD Configuration

### Parent Application (App of Apps)

Located at `argocd/application.yaml`:

```yaml
metadata:
  name: applicationset-controller
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/ThorntonCloud/GitOps-Infra.git
    path: argocd/appsets
  syncPolicy:
    automated:
      selfHeal: true  # Auto-corrects drift
      prune: false    # Doesn't delete resources
```

**Purpose**: Deploys the ApplicationSet that manages all environments

### ApplicationSet (Multi-Environment Generator)

Located at `argocd/appsets/applicationset.yaml`:

```yaml
generators:
  - list:
      elements:
        - cluster: dev
        - cluster: staging
        - cluster: prod

template:
  metadata:
    name: '{{cluster}}-nginx-app'
  spec:
    source:
      path: nginx-app/overlays/{{cluster}}
    syncPolicy:
      automated:
        prune: true      # Removes deleted resources
        selfHeal: true   # Auto-corrects drift
```

**Purpose**: Generates three ArgoCD Applications (dev, staging, prod) from a single template

### Sync Policies

| Policy | Description | Current Setting |
|--------|-------------|-----------------|
| **automated** | Auto-sync on Git changes | ✅ Enabled (all environments) |
| **selfHeal** | Auto-correct manual kubectl changes | ✅ Enabled |
| **prune** | Delete resources removed from Git | ✅ Enabled (ApplicationSet apps) |
| **CreateNamespace** | Auto-create namespace if missing | ✅ Enabled |

## 📊 Visibility

### ArgoCD UI

Access application health and sync status:

```bash
# Port-forward ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login credentials
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**Applications visible in UI:**
- `applicationset-controller` (parent)
- `dev-nginx-app`
- `staging-nginx-app`
- `prod-nginx-app`

### Application Health Checks

Each application monitors:
- **Sync Status**: Git commit vs. deployed state
- **Health Status**: Pod readiness and liveness probes
- **Resource Status**: All Kubernetes resources healthy

### Rollback Procedure

**Using ArgoCD:**
```bash
# Via UI: Application → History → Rollback
# Via CLI:
argocd app rollback prod-nginx-app <revision>
```

**Using Git:**
```bash
# Revert the commit that changed the image tag
git revert <commit-hash>
git push origin main
```

## 🔒 Security Considerations

### Current Security Features

1. **Non-Root Containers**: All pods run as non-root user (UID 65532)
2. **Read-Only Root Filesystem**: Container filesystem is immutable
3. **Image Scanning**: Integrated Trivy for runtime security
4. **Resource Limits**: CPU and memory limits prevent resource exhaustion
5. **Network Policies**: (Can be added) Restrict pod-to-pod communication
6. **RBAC**: ArgoCD service account has minimal required permissions

### Production Hardening Recommendations

For production environments, consider:

- **Namespace Isolation**: Deploy environments to separate clusters
- **Manual Sync for Prod**: Disable `automated` sync for production
- **Fail on Critical CVE**: Configure the pipeline to fail if CVEs above the accepted risk are found in the image
- **Secret Management**: Use sealed-secrets or external-secrets-operator
- **Network Policies**: Implement strict ingress/egress rules
- **Pod Security Standards**: Enforce restricted pod security standards

## 🚀 Getting Started

### Prerequisites

- Kubernetes cluster (v1.24+)
- ArgoCD installed in cluster
- kubectl configured with cluster access

### Initial Deployment

1. **Install ArgoCD** (if not already installed):
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. **Deploy the App of Apps**:
```bash
kubectl apply -f argocd/application.yaml
```

3. **Verify ApplicationSet is created**:
```bash
kubectl get applicationset -n argocd
```

4. **Verify all applications are synced**:
```bash
kubectl get applications -n argocd
```

5. **Access the applications**:
```bash
# Dev (NodePort)
kubectl get svc -n dev

# Staging (LoadBalancer)
kubectl get svc -n staging

# Prod (LoadBalancer)
kubectl get svc -n prod
```

## 🔧 Customization

### Adding a New Environment

1. **Create new overlay**:
```bash
mkdir -p nginx-app/overlays/qa
cp -r nginx-app/overlays/staging/* nginx-app/overlays/qa/
```

2. **Update kustomization.yaml**:
```bash
vim nginx-app/overlays/qa/kustomization.yaml
# Update namespace, image tag, and patches
```

3. **Add to ApplicationSet**:
```yaml
# argocd/appsets/applicationset.yaml
generators:
  - list:
      elements:
        - cluster: qa
          url: https://kubernetes.default.svc
```

### Modifying Base Resources

```bash
# Edit base manifests
vim nginx-app/base/deployment.yaml

# Test with kustomize
kubectl kustomize nginx-app/overlays/dev

# Commit and push
git add nginx-app/base/
git commit -m "feat: update deployment configuration"
git push origin main
```

## 🐛 Troubleshooting

### Application Not Syncing

```bash
# Check ArgoCD application status
kubectl get application prod-nginx-app -n argocd -o yaml

# Check for sync errors
argocd app get prod-nginx-app

# Force sync
argocd app sync prod-nginx-app
```

### Pods Not Starting

```bash
# Check pod status
kubectl get pods -n prod

# View pod logs
kubectl logs -n prod <pod-name>

# Describe pod for events
kubectl describe pod -n prod <pod-name>
```

### Image Pull Errors

```bash
# Verify image exists in registry
docker pull thorntoncloud/nginx-gitops:<tag>

# Check image pull secrets (if using private registry)
kubectl get secrets -n prod
```

### Configuration Drift

```bash
# Check for manual changes
argocd app diff prod-nginx-app

# Self-heal will auto-correct, or manually sync
argocd app sync prod-nginx-app
```

## 📚 Related Resources

- **[GitOps-App Repository](https://github.com/ThorntonCloud/GitOps-App)** - Application source code and CI/CD
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/) - GitOps continuous delivery
- [Kustomize Documentation](https://kustomize.io/) - Kubernetes configuration management
- [ArgoCD ApplicationSet](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/) - Multi-environment management
- [GitOps Principles](https://opengitops.dev/) - GitOps standards

## 🤝 Contributing

This is a demonstration repository showing GitOps patterns and best practices.

### Making Changes

1. Fork the repository
2. Create a feature branch
3. Make your changes to overlays or base manifests
4. Test with `kustomize`
5. Submit a pull request

### Testing Changes Locally

```bash
# Preview changes without applying
kustomize build nginx-app/overlays/dev

# Apply to test cluster
kubectl apply -k nginx-app/overlays/dev

# Clean up
kubectl delete -k nginx-app/overlays/dev
```

---

**Part of the ThorntonCloud GitOps Demo** | [Application Repository](https://github.com/ThorntonCloud/GitOps-App)