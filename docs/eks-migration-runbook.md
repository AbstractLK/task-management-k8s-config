# EKS Migration Runbook

## What Stays the Same

- The `task-management-app` source repository
- Docker images
- DockerHub registry
- Helm charts
- ArgoCD deployment model
- MongoDB Atlas database

## What Changes

- Minikube becomes EKS.
- Local hostname becomes Route53 DNS.
- API ingress uses a production domain such as `api.example.com`.
- Frontend delivery moves to S3 + CloudFront.
- TLS is managed with ACM and/or cert-manager.

## Migration Steps

1. Create EKS cluster and node group.
2. Install NGINX Ingress Controller or AWS Load Balancer Controller.
3. Install ArgoCD in EKS.
4. Create `task-management-secrets` in the EKS cluster.
5. Replace placeholders in `environments/eks`.
6. Apply `argocd/eks` applications.
7. Build frontend and upload it to S3.
8. Configure CloudFront, Route53, and TLS.
