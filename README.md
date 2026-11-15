# NTI DevOps Platform - GitOps Delivery Repository

[![GitOps](https://img.shields.io/badge/GitOps-ArgoCD-orange?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)
[![Helm](https://img.shields.io/badge/Helm-v3-0F1689?logo=helm&logoColor=white)](https://helm.sh/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.31-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![AWS ECR](https://img.shields.io/badge/AWS-ECR-FF9900?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/ecr/)
[![Status](https://img.shields.io/badge/Status-Synced%20%26%20Healthy-34d399)](#)

This is the dedicated **Continuous Delivery (GitOps)** repository for the **NTI DevOps Final Project**. It serves as the single source of truth for the desired state of workloads running inside the Kubernetes cluster.

---

## 🎯 Architectural Purpose

In accordance with enterprise DevOps best practices, deployment manifests are strictly decoupled from application source code:

- **Source Code Repository (`nti-project`)**: Hosts the FastAPI backend application, tests, Dockerfile, and CI automation pipelines (Jenkins / GitHub Actions).
- **GitOps Repository (`nti-devops-gitops`)**: Hosts purely declarative Helm charts and cluster configuration. Developers have read access; only automated CI bots and cluster administrators possess write permissions.

### Why Multi-Repo GitOps?
1. **Eliminates CI Build Loops**: When CI bumps an image tag, it commits to this repository, avoiding recursive pipeline triggers.
2. **Principle of Least Privilege**: Application developers do not need direct `kubectl` access or cluster admin credentials.
3. **Audit Trail & Instant Rollbacks**: Every commit in this repository represents an exact release. Reverting a bad deployment is as simple as running `git revert <commit-sha>`.

---

## 🔄 End-to-End GitOps Workflow

```
[ Developer ]
      │ git push
      ▼
[ App Repository: nti-project ]
      │ Webhook trigger
      ▼
[ CI Automation (Jenkins / GitHub Actions) ]
      ├── 1. Code Linting & Pytest Unit Tests
      ├── 2. Trivy Container Vulnerability Scan
      ├── 3. Docker Multi-Stage Build
      ├── 4. Push Image to AWS ECR: `nti-devops-app:<build-id>-<sha>`
      └── 5. Commit updated tag to this GitOps Repo (charts/nti-app/values.yaml)
                                     │
                                     ▼
                   [ GitOps Repo: nti-devops-gitops ]
                                     │
                                     ▼ (Automated Poll / Webhook)
                           [ ArgoCD Controller ]
                                     │
                                     ▼ (Zero-Downtime Rolling Update)
                    [ Kubernetes Cluster (Kubeadm / EKS) ]
                         └── Pods pulling from AWS ECR
```

---

## 📁 Repository Structure

```
nti-devops-gitops/
├── README.md                          # Repository documentation & operations runbook
└── charts/
    └── nti-app/                       # Production Helm chart for nti-app
        ├── Chart.yaml                 # Chart metadata, version, and dependencies
        ├── values.yaml                # Default values (image tag, replicas, ports, env)
        └── templates/                 # Kubernetes manifest templates
            ├── _helpers.tpl           # Template helper functions and labels
            ├── deployment.yaml        # Deployment spec (anti-affinity, probes, security)
            ├── service.yaml           # Service (NodePort 31213 / ClusterIP 80)
            ├── hpa.yaml               # HorizontalPodAutoscaler (HPA v2 scaling 2-6 pods)
            ├── serviceaccount.yaml    # Pod ServiceAccount & IAM bindings
            ├── servicemonitor.yaml    # Prometheus ServiceMonitor metrics scraping
            ├── gateway.yaml           # Kubernetes Gateway API Gateway resource
            ├── httproute.yaml         # Kubernetes Gateway API HTTPRoute routing
            └── externalsecret.yaml    # AWS Secrets Manager ExternalSecret integration
```

---

## ⚙️ Helm Chart Configuration (`charts/nti-app`)

Key configuration parameters defined in [`charts/nti-app/values.yaml`](charts/nti-app/values.yaml):

| Parameter | Default Value | Description |
| :--- | :--- | :--- |
| `replicaCount` | `2` | Initial baseline pod replicas |
| `image.repository` | `270485099284.dkr.ecr.us-east-1.amazonaws.com/nti-devops-app` | Target AWS ECR image repository |
| `image.tag` | `"5-48b62ee"` | Current active image tag (bumped by CI) |
| `image.pullPolicy` | `Always` | Pull policy for fresh container rollouts |
| `service.type` | `NodePort` | Service exposure type (`NodePort` / `ClusterIP`) |
| `service.port` | `80` | Internal Service port |
| `service.targetPort`| `8000` | Application container port |
| `service.nodePort` | `31213` | External NodePort accessible across cluster IPs |
| `resources.requests.cpu` | `100m` | Minimum guaranteed CPU per pod |
| `resources.requests.memory`| `128Mi` | Minimum guaranteed RAM per pod |
| `resources.limits.cpu` | `500m` | Maximum CPU burst ceiling |
| `resources.limits.memory` | `512Mi` | Maximum RAM ceiling before OOM protection |
| `autoscaling.enabled` | `true` | Enables Kubernetes HPA v2 |
| `autoscaling.minReplicas` | `2` | Minimum active pod count |
| `autoscaling.maxReplicas` | `6` | Maximum burst pod count during traffic spikes |
| `autoscaling.targetCPUUtilizationPercentage` | `75` | Target CPU load triggering scale-out |
| `podSecurityContext.runAsNonRoot` | `true` | Restricts container execution to non-root UID |
| `probes.liveness.path` | `/healthz` | Liveness health check probe endpoint |
| `probes.readiness.path` | `/healthz` | Readiness health check probe endpoint |

---

## 🐙 ArgoCD Application Manifest

ArgoCD synchronizes this repository continuously using the following declarative application spec:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nti-app
  namespace: argocd
  labels:
    app.kubernetes.io/name: nti-app
    app.kubernetes.io/part-of: nti-devops-platform
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/abdalrahmanx9/nti-devops-gitops.git
    targetRevision: main
    path: charts/nti-app
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## 🛠️ Operations Runbook

### 1. Triggering an Automated Deployment
To release a new image, CI executes:
```bash
# Update image tag in values.yaml
sed -i 's/tag: .*/tag: "6-a1b2c3d"/' charts/nti-app/values.yaml

# Commit and push to main
git add charts/nti-app/values.yaml
git commit -m "chore(release): bump nti-app to 6-a1b2c3d [skip ci]"
git push origin main
```
ArgoCD detects the commit within 3 minutes (or immediately via GitHub webhook) and triggers a rolling update.

### 2. Performing an Instant Rollback
To roll back a faulty production deployment:
```bash
# Revert the latest tag bump commit
git revert HEAD --no-edit
git push origin main
```
ArgoCD immediately rolls the cluster back to the previous stable container image.

### 3. Checking Sync & Health Status via CLI
```bash
# Query ArgoCD application state
kubectl get application nti-app -n argocd

# Expected Output:
# NAME      SYNC STATUS   HEALTH STATUS
# nti-app   Synced        Healthy
```

### 4. Forcing an Immediate Hard Refresh
```bash
# Force ArgoCD to bypass git cache and sync immediately
kubectl annotate application nti-app -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

---

## 🔒 Security & Best Practices

- **Zero-Trust Hardening**: Containers run under UID `1000` with `allowPrivilegeEscalation: false` and all Linux capabilities dropped (`drop: [ALL]`).
- **High Availability Anti-Affinity**: Pod anti-affinity is enforced across AZs (`topology.kubernetes.io/zone`) to ensure resilient scheduling.
- **Credential Safety**: No AWS IAM secrets, passwords, or tokens are stored in plain text in this repository. Sensitive database credentials are dynamically injected via Kubernetes Secrets and AWS Secrets Manager.
