# NTI DevOps Platform - GitOps Deployment Repository

This is the dedicated **Continuous Delivery (GitOps)** repository for the **NTI DevOps Final Project**.

## 🔄 GitOps Workflow
1. **Jenkins CI Pipeline** in the main repository builds and tests code, scans the container image with Trivy, pushes it to AWS ECR, and creates a commit in this repository updating the image tag in `charts/nti-app/values.yaml`.
2. **ArgoCD Controller** running inside the Amazon EKS cluster watches this repository.
3. Upon detecting a new commit, ArgoCD performs an automated, zero-downtime rolling update onto the cluster.

## 📁 Repository Contents
- `charts/nti-app/`: Production Helm Chart with Gateway API, HPA, and Zero-Trust configuration.
