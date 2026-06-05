# Task Management Kubernetes Config

GitOps configuration for the task management DevOps portfolio project.

This repository is one of two project repositories:

- `task-management-app`: application source monorepo
- `task-management-k8s-config`: Kubernetes, Helm, ArgoCD, environment, secret, and monitoring configuration

## Local Minikube Setup

Start Minikube:

```bash
minikube start --driver=docker --cpus=3 --memory=6144
minikube addons enable ingress
```

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

Apply local ArgoCD applications after replacing `YOUR_GITHUB_USERNAME`:

```bash
kubectl apply -f argocd/minikube/
```

Map the Minikube IP to a local hostname in the Windows hosts file:

```text
<MINIKUBE_IP> taskmanager.local
```

Get the Minikube IP:

```bash
minikube ip
```

## Future EKS Setup

The EKS environment uses the same Helm charts with values from `environments/eks`. The frontend can be disabled in Kubernetes when production delivery moves to S3 + CloudFront.
