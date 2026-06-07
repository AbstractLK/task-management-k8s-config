# EKS Deployment Runbook

This guide deploys the Task Management project to Amazon EKS while keeping the same two-repository model:

- `task-management-app`: React frontend and Node.js microservices
- `task-management-k8s-config`: Helm charts, ArgoCD apps, Kubernetes values, secrets template, and monitoring notes

EKS is the cloud runtime for the backend services and GitOps deployment.

## Target Cloud Architecture

```text
User
  -> Route53
  -> CloudFront
  -> S3 frontend

Frontend
  -> https://api.example.com
  -> Route53
  -> EKS NGINX Ingress LoadBalancer
  -> auth-service / task-service
  -> MongoDB Atlas
```

## Official References

- EKS cluster creation with `eksctl`: https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html
- AWS CLI `update-kubeconfig`: https://docs.aws.amazon.com/cli/latest/reference/eks/update-kubeconfig.html
- ingress-nginx installation: https://kubernetes.github.io/ingress-nginx/deploy/
- Argo CD installation: https://argo-cd.readthedocs.io/en/latest/operator-manual/installation/
- cert-manager Helm install: https://cert-manager.io/docs/installation/helm/
- CloudFront private S3 origin with OAC: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html
- Route53 with CloudFront: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-cloudfront-distribution.html

## 1. Prerequisites

Install and configure these tools locally:

- AWS CLI v2
- `eksctl`
- `kubectl`
- Helm
- Git
- Docker Desktop
- Node.js 20

Verify AWS access:

```bash
aws sts get-caller-identity
```

Set working variables:

```bash
export AWS_REGION=ap-southeast-1
export CLUSTER_NAME=task-management-eks
export DOMAIN_NAME=example.com
export APP_DOMAIN=app.example.com
export API_DOMAIN=api.example.com
export GITHUB_USERNAME=AbstractLK
export DOCKERHUB_NAMESPACE=abstraxlk
export FRONTEND_BUCKET=task-management-frontend-example
```

For Windows PowerShell:

```powershell
$env:AWS_REGION="ap-southeast-1"
$env:CLUSTER_NAME="task-management-eks"
$env:DOMAIN_NAME="example.com"
$env:APP_DOMAIN="app.example.com"
$env:API_DOMAIN="api.example.com"
$env:GITHUB_USERNAME="AbstractLK"
$env:DOCKERHUB_NAMESPACE="abstraxlk"
$env:FRONTEND_BUCKET="task-management-frontend-example"
```

Use your real domain and bucket name. S3 bucket names must be globally unique.

## 2. Prepare MongoDB Atlas

1. Create or reuse a MongoDB Atlas cluster.
2. Create two databases:
   - `task_management_auth`
   - `task_management_tasks`
3. Create a database user with read/write access.
4. Add EKS node outbound access to Atlas Network Access.

For an initial portfolio demo, you may temporarily allow broader access while testing, but tighten it before publishing the project.

## 3. Update EKS Values in the Config Repo

Edit these files in `task-management-k8s-config/environments/eks`:

- `auth-service-values.yaml`
- `task-service-values.yaml`
- `frontend-values.yaml`
- `ingress-values.yaml`

Set backend domains:

```yaml
env:
  corsOrigin: https://app.example.com

ingress:
  host: api.example.com
```

Set Docker image repositories:

```yaml
image:
  repository: abstraxlk/task-management-auth-service
  tag: <latest-successful-jenkins-build-number>
```

For the frontend EKS values, keep Kubernetes frontend delivery disabled if you are using S3 + CloudFront:

```yaml
replicaCount: 0
ingress:
  enabled: false
```

Commit and push the config changes:

```bash
git add environments/eks
git commit -m "Configure EKS environment values"
git push origin main
```

## 4. Create the EKS Cluster

Create a managed-node EKS cluster:

```bash
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed \
  --with-oidc
```

PowerShell version:

```powershell
eksctl create cluster `
  --name $env:CLUSTER_NAME `
  --region $env:AWS_REGION `
  --nodegroup-name standard-workers `
  --node-type t3.medium `
  --nodes 2 `
  --nodes-min 1 `
  --nodes-max 3 `
  --managed `
  --with-oidc
```

Connect `kubectl` to the cluster:

```bash
aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
kubectl get nodes
```

## 5. Create Namespaces

```bash
kubectl create namespace task-management
kubectl create namespace argocd
kubectl create namespace ingress-nginx
kubectl create namespace monitoring
```

If a namespace already exists, the command may fail harmlessly. Continue after confirming:

```bash
kubectl get namespaces
```

## 6. Create Kubernetes Secrets

Create a real secret file from `secrets/secret-template.yaml`.

Do not commit real secrets.

Recommended local filename:

```text
secrets/secret.yaml
```

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: task-management-secrets
  namespace: task-management
type: Opaque
stringData:
  auth-mongodb-uri: mongodb+srv://USER:PASSWORD@cluster.example.mongodb.net/task_management_auth
  task-mongodb-uri: mongodb+srv://USER:PASSWORD@cluster.example.mongodb.net/task_management_tasks
  jwt-secret: replace-with-a-long-random-secret
```

Apply it:

```bash
kubectl apply -f secrets/secret.yaml
kubectl get secret task-management-secrets -n task-management
```

## 7. Install NGINX Ingress Controller

Install ingress-nginx with Helm:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.ingressClassResource.name=nginx \
  --set controller.ingressClass=nginx \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb
```

Wait for the AWS load balancer:

```bash
kubectl get svc -n ingress-nginx
```

Copy the `EXTERNAL-IP` or hostname from `ingress-nginx-controller`. It will look like an AWS load balancer DNS name.

## 8. Configure API DNS in Route53

Create a Route53 record:

```text
api.example.com -> ingress-nginx-controller external load balancer DNS name
```

Use either:

- CNAME record to the load balancer DNS name
- Alias record if Route53 offers the load balancer target

Validate DNS:

```bash
nslookup api.example.com
```

## 9. Optional TLS for API

For a simple first migration, you may test the API over HTTP.

For HTTPS, install cert-manager:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Create a ClusterIssuer for Let's Encrypt after you confirm DNS works.

Then update the backend Helm charts or values to add TLS to the Ingress resources.

## 10. Install ArgoCD in EKS

Install ArgoCD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for ArgoCD:

```bash
kubectl get pods -n argocd
```

Access ArgoCD locally:

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

Open:

```text
https://localhost:8081
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

On PowerShell:

```powershell
$password = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($password))
```

## 11. Deploy Apps with ArgoCD

Before applying the EKS ArgoCD applications, confirm each file in `argocd/eks` points to your GitHub config repo:

```yaml
repoURL: https://github.com/AbstractLK/task-management-k8s-config.git
targetRevision: main
```

Apply the EKS ArgoCD apps:

```bash
kubectl apply -f argocd/eks/
```

Watch sync status:

```bash
kubectl get applications -n argocd
kubectl get pods -n task-management
kubectl get ingress -n task-management
```

If ArgoCD cannot access your config repo, configure repo credentials in the ArgoCD UI.

## 12. Validate Backend APIs

After pods become ready, test health endpoints:

```bash
curl http://api.example.com/api/auth/health
curl http://api.example.com/api/tasks/health
```

Expected responses:

```json
{"status":"ok","service":"auth-service"}
```

```json
{"status":"ok","service":"task-service"}
```

If DNS is not ready yet, test through the ingress load balancer hostname with the host header:

```bash
curl -H "Host: api.example.com" http://<INGRESS_LOAD_BALANCER_DNS>/api/auth/health
```

## 13. Deploy Frontend to S3

Build the frontend from `task-management-app/frontend`:

```bash
cd task-management-app/frontend
npm install
VITE_API_BASE_URL=https://api.example.com npm run build
```

PowerShell:

```powershell
cd task-management-app/frontend
npm install
$env:VITE_API_BASE_URL="https://api.example.com"
npm run build
```

Create the S3 bucket:

```bash
aws s3 mb s3://$FRONTEND_BUCKET --region $AWS_REGION
```

Upload the build:

```bash
aws s3 sync dist s3://$FRONTEND_BUCKET --delete
```

Keep S3 public access blocked. CloudFront should access S3 using Origin Access Control.

## 14. Create CloudFront Distribution

Create a CloudFront distribution with:

- Origin: the S3 bucket REST endpoint, not the S3 static website endpoint
- Origin access: Origin Access Control
- Default root object: `index.html`
- Alternate domain name: `app.example.com`
- Viewer certificate: ACM certificate in `us-east-1`
- SPA fallback/error behavior: return `index.html` for 403/404 if needed

After CloudFront creates the OAC policy, apply the generated S3 bucket policy so CloudFront can read objects.

Then create a Route53 record:

```text
app.example.com -> CloudFront distribution
```

## 15. Update Jenkins for EKS Deployment

The GitOps update stage should update:

```text
config/environments/eks/auth-service-values.yaml
config/environments/eks/task-service-values.yaml
```

For production frontend delivery, Jenkins uploads `frontend/dist` to S3 and invalidates CloudFront.

CloudFront invalidation command:

```bash
aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths "/*"
```

## 16. Install Monitoring

Install Prometheus and Grafana:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

Access Grafana locally:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Open:

```text
http://localhost:3000
```

## 17. Final End-to-End Test

Verify:

- `https://app.example.com` loads the frontend.
- Frontend calls `https://api.example.com`.
- User registration works.
- User login returns a JWT.
- Task create/list/update/delete works.
- ArgoCD apps are healthy and synced.
- Grafana shows EKS node and pod metrics.

Useful commands:

```bash
kubectl get pods -n task-management
kubectl get svc -n task-management
kubectl get ingress -n task-management
kubectl describe ingress -n task-management
kubectl logs -n task-management deploy/auth-service
kubectl logs -n task-management deploy/task-service
```

## 18. Common Issues

### ArgoCD app is OutOfSync

Check:

```bash
kubectl describe application -n argocd task-management-auth-service
```

Common causes:

- wrong `repoURL`
- wrong value file path
- GitHub repo is private and ArgoCD has no credentials
- image tag does not exist in DockerHub

### Pods are CrashLoopBackOff

Check logs:

```bash
kubectl logs -n task-management deploy/auth-service
kubectl logs -n task-management deploy/task-service
```

Common causes:

- missing `task-management-secrets`
- wrong MongoDB Atlas URI
- Atlas network access blocks EKS traffic
- `JWT_SECRET` missing or mismatched

### API DNS does not work

Check:

```bash
kubectl get svc -n ingress-nginx
kubectl get ingress -n task-management
nslookup api.example.com
```

Confirm Route53 points to the ingress controller load balancer.

### Frontend cannot call API

Check:

- `VITE_API_BASE_URL` is set to `https://api.example.com`
- backend `corsOrigin` is `https://app.example.com`
- CloudFront cache was invalidated after deploying frontend changes

## 19. Cost Control and Cleanup

EKS has ongoing cost. Stop charges when you are not using the environment.

Delete the EKS cluster:

```bash
eksctl delete cluster --name $CLUSTER_NAME --region $AWS_REGION
```

Delete optional frontend infrastructure if no longer needed:

```bash
aws s3 rm s3://$FRONTEND_BUCKET --recursive
aws s3 rb s3://$FRONTEND_BUCKET
```

Also remove or disable:

- CloudFront distribution
- Route53 records
- ACM certificates you no longer need
- MongoDB Atlas cluster if it was created only for this demo

## Migration Checklist

- [ ] AWS CLI, kubectl, eksctl, Helm installed
- [ ] MongoDB Atlas database and user ready
- [ ] EKS values updated and pushed
- [ ] EKS cluster created
- [ ] kubeconfig connected to EKS
- [ ] namespaces created
- [ ] Kubernetes secrets applied
- [ ] NGINX Ingress installed
- [ ] API Route53 record created
- [ ] ArgoCD installed
- [ ] EKS ArgoCD apps applied
- [ ] backend health endpoints pass
- [ ] frontend built with EKS API URL
- [ ] S3 bucket created
- [ ] CloudFront distribution created
- [ ] frontend Route53 record created
- [ ] end-to-end register/login/task flow passes
- [ ] monitoring installed
