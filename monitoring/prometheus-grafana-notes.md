# Prometheus and Grafana

Install the kube-prometheus-stack chart into the `monitoring` namespace:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

Access Grafana locally:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Default dashboards to review:

- Kubernetes cluster overview
- Node exporter dashboard
- Kubernetes workload dashboard

Add `/metrics` endpoints later with `prom-client` in the Node.js services if you want service-level request metrics.
