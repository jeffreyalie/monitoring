# infra/monitoring

Deploys `kube-prometheus-stack` to MicroK8s via Gitea Actions + OpenBao.

## Stack

| Component | Role | Port |
|---|---|---|
| Prometheus Operator | Manages Prometheus/Alertmanager via CRDs | — |
| Prometheus | Metrics scraper & TSDB | NodePort **30090** |
| Alertmanager | Alert routing | NodePort **30093** |
| Grafana | Dashboards | NodePort **30300** |
| kube-state-metrics | K8s object metrics | ClusterIP |
| node-exporter | Host metrics | DaemonSet |

## Prerequisites

### 1. OpenBao secrets

Two paths must exist in OpenBao before the workflow runs:

```bash
# kubeconfig for MicroK8s
bao kv put homelab/microk8s \
  kubeconfig="$(microk8s config)"

# Grafana admin password
bao kv put homelab/grafana \
  admin_password="<your-password>"
```

### 2. MicroK8s addons

```bash
microk8s enable dns storage     # dns + hostpath storage class
```

### 3. Gitea org secrets (already set)

```
VAULT_ADDR / VAULT_ROLE_ID / VAULT_SECRET_ID
```

## Trigger the deployment

Any push to `main` that touches `helm/kube-prometheus-stack/**` fires the workflow.
You can also trigger manually: Gitea → Actions → "Deploy kube-prometheus-stack" → Run workflow.

## Access after deployment

```
http://<microk8s-node-ip>:30090   → Prometheus
http://<microk8s-node-ip>:30093   → Alertmanager
http://<microk8s-node-ip>:30300   → Grafana  (admin / <vault password>)
```

If you're on the same machine: use `microk8s kubectl get node -o wide` to find the internal IP.

## Adding your Go app later

When you're ready to wire in the Go app's `/metrics` endpoint, create:
- `helm/kube-prometheus-stack/servicemonitor-goapp.yaml` — a `ServiceMonitor` CRD pointing at your app's service on port 8080
- Grafana dashboard JSON in `helm/kube-prometheus-stack/dashboards/`

The chart's `serviceMonitorSelector: {}` already picks up ServiceMonitors from any namespace.

## Repository layout

```
monitoring/
├── .gitea/
│   └── workflows/
│       └── deploy-monitoring.yml
└── helm/
    └── kube-prometheus-stack/
        └── values.yaml
```
