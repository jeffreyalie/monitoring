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
---
## Troubleshooting commands

```
microk8s kubectl get pvc -n monitoring

# See which pods are stuck
microk8s kubectl get pods -n monitoring

# For any pod NOT in Running state:
microk8s kubectl describe pod <pod-name> -n monitoring

# Most likely culprit — check events at the bottom of describe output
microk8s kubectl get events -n monitoring --sort-by='.lastTimestamp'

# Delete the PVC so it gets a fresh volume without the bad ownership
microk8s kubectl delete pvc -n monitoring \
  $(microk8s kubectl get pvc -n monitoring | grep grafana | awk '{print $1}')

# Delete the pod so it respawns after the workflow redeploys
microk8s kubectl delete pod -n monitoring \
  kube-prometheus-stack-grafana-646b796bf5-kj7j7
```

## Kube promethius stack diagram

---
```
                                [ USER / ADMIN ]
                                        |
                         1. Apply YAML (ServiceMonitors, Rules) <----------------- Via gitea repo
                                        |
                                        v
[--------------------------- KUBERNETES API SERVER ---------------------------]
                                        |
                                        | (Watches for changes)
                                        v
                         [ PROMETHEUS OPERATOR POD ]
                         "The Manager / Foreman"
                                        |
           -------------------------------------------------------------
           | (Configures)               | (Configures)                 | (Configures)
           v                            v                              v
   [ PROMETHEUS POD ] <---------- [ ALERTMANAGER ]                [ GRAFANA POD ]
   "The Database"      (Alerts)    "The Notifier"                 "The Visualizer"
           |                                                           |
           | 2. PULL / SCRAPE                                          | 3. QUERY
           | (Every 15-30s)                                            | (On Demand)
           |                                                           |
           |                                                           v
           |                                                     (User Web Browser)
           |                                                     "lxd-dashboard.local"
|          | 
           |
           |
           |                           
           |---> [ NODE EXPORTER ] ----> (Hardware/OS Metrics)
           |
           |---> [ YOUR GO APP ] 
                    |
                    |-- (:8080/metrics) <--- (Exposed via Go Client Library) (API for Prometheus to pull)
                    |
                    |-- [ INTERNAL APP LOGIC ]
                            |
                            |-- (LXD SDK) ----> [ LXD SERVER ]
                            |-- (Vault SDK) ---> [ OPENBAO ]
```

---
## Kube-promethius-grafana stack components

```
Component	          Pod Name (Typical)	                  Purpose
Prometheus	        prometheus-kube-prometheus-0	        The database that "scrapes" and stores metrics.
Grafana	            grafana-xxxxxxx-xxxx	                The web UI where you build dashboards.
Alertmanager	      alertmanager-kube-prometheus-0	      Handles the logic for sending emails/Slack pings when things break.
Operator	          prometheus-operator-xxxx-xxxx	        The manager that coordinates the pods above.
Node Exporter	      prometheus-node-exporter-xxxx	        One pod per node in your cluster to monitor hardware.
                                                          label them via ServiceMonitors yaml and operator auto picks.
Go App              Go app pod                            (:8080/metrics) <--- (Exposed via Go Client Library) (API for Prometheus to pull)
                                                          label them via ServiceMonitors yaml and operator auto picks.
```
---

## Node exporter connects via

```
Method	                What the Exporter uses to "Login"	    Where you put the Creds
Node Exporter	          Local System Permissions	            Linux User/Group (root/prometheus)
SNMP	                  Community String / SNMPv3 Auth	      Exporter snmp.yml config
HTTP API	              API Key / Basic Auth	                Exporter config file or Env Vars
Metrics Endpoints	      Bearer Tokens / TLS Certs	            Exporter config file
Logs	                  OS Read Permissions	                  chmod / chown on the log files
SDKs (Custom)           Whatever the SDK requires             Inside your Go code logic

```
---

## Note
Go app / exporter dont need authentication to prometheus: We label them via ServiceMonitors yaml and operator auto picks.
Go app/exporter only need authentication for the hardware/VM/system they are collecting data from
