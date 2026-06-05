# Local Minikube Runbook

## 1. Start Minikube

```bash
minikube start --driver=docker --cpus=3 --memory=6144
minikube addons enable ingress
```

## 2. Create Secret

Copy `secrets/secret-template.yaml`, replace the MongoDB Atlas URI and JWT secret, then apply it:

```bash
kubectl apply -f secrets/secret-template.yaml
```

## 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 4. Deploy Applications

Replace placeholders in the ArgoCD manifests, then run:

```bash
kubectl apply -f argocd/minikube/
```

## 5. Configure Local DNS

Add this line to `C:\Windows\System32\drivers\etc\hosts`:

```text
<MINIKUBE_IP> taskmanager.local
```

## 6. Test

Open:

```text
http://taskmanager.local
```

Health endpoints:

```text
http://taskmanager.local/api/auth/health
http://taskmanager.local/api/tasks/health
```
