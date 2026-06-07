# Task Management Kubernetes Config

GitOps configuration for the task management DevOps portfolio project.

This repository is one of two project repositories:

- `task-management-app`: application source monorepo
- `task-management-k8s-config`: Kubernetes, Helm, ArgoCD, environment, secret, and monitoring configuration

## EKS Setup

Create namespaces:

```bash
kubectl create namespace task-management
kubectl create namespace argocd
kubectl create namespace monitoring
```

Create the application secret from the template in `secrets/secret-template.yaml`.

Install ArgoCD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Apply EKS ArgoCD applications after replacing `YOUR_GITHUB_USERNAME`:

```bash
kubectl apply -f argocd/eks/
```

## Environment Values

The EKS environment uses the same Helm charts with values from `environments/eks`. The frontend can be disabled in Kubernetes when production delivery moves to S3 + CloudFront.
