# nti-devops-gitops

GitOps deployment repository for `nti-app`. ArgoCD watches the `main` branch of this repository and syncs the Helm chart to the Kubernetes cluster.

## Workflow

1. Push code to the application repo (`nti-project`).
2. CI builds the Docker image, runs tests/scans, and pushes to ECR (`270485099284.dkr.ecr.us-east-1.amazonaws.com/nti-devops-app`).
3. CI updates `image.tag` in `charts/nti-app/values.yaml` and pushes with `[skip ci]`.
4. ArgoCD pulls the update and rolls out the new pods.

```
nti-project (app code) ──► CI Build & Push to ECR ──► Bump image.tag here ──► ArgoCD Sync ──► K8s
```

## Structure

```
charts/nti-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── hpa.yaml
    ├── serviceaccount.yaml
    ├── servicemonitor.yaml
    ├── gateway.yaml
    ├── httproute.yaml
    └── externalsecret.yaml
```

## ArgoCD Setup

Apply the application manifest:

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nti-app
  namespace: argocd
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
EOF
```

## Core Helm Values (`charts/nti-app/values.yaml`)

- `image.repository`: `270485099284.dkr.ecr.us-east-1.amazonaws.com/nti-devops-app`
- `image.tag`: Active image tag (updated by CI)
- `service.type`: `NodePort`
- `service.port`: `80` (container port `8000`, nodePort `31213`)
- `autoscaling`: HPA enabled, 2-6 pods, 75% CPU target
- `resources`: 100m / 128Mi requests, 500m / 512Mi limits
- `probes`: `/healthz` on port `8000` for liveness and readiness

## Operations

### Manual image bump
```bash
sed -i 's/tag: .*/tag: "NEW_TAG"/' charts/nti-app/values.yaml
git commit -am "chore: bump image to NEW_TAG [skip ci]"
git push origin main
```

### Rollback
```bash
git revert HEAD --no-edit
git push origin main
```

### Force immediate sync
```bash
kubectl annotate application nti-app -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

### Check status
```bash
kubectl get application nti-app -n argocd
kubectl get pods -l app.kubernetes.io/name=nti-app
```
