# 005 - Prometheus & Grafana: Cloud-Native Observability

In exercises 001 through 004, I deployed Kubernetes workloads, packaged them with Helm, automated CI gates with GitHub Actions, and implemented GitOps Continuous Delivery with ArgoCD.

However, deploying applications is only half of operating Kubernetes in production. Once workloads are live, I need real-time visibility into cluster health, node capacity, pod restarts, and automated alerts when thresholds are breached.

In this module, I implement **Cloud-Native Observability (Metrics, Dashboards, and Alerting)** using **`kube-prometheus-stack`** (Prometheus Operator, Prometheus, Alertmanager, exporters, and Grafana).

---

## 1. Architecture & Components (`kube-prometheus-stack`)

Instead of manually wiring standalone monitoring containers and hand-editing static `prometheus.yml` files, production Kubernetes clusters use the **Prometheus Operator** pattern via the `kube-prometheus-stack` Helm chart:

1. **Prometheus Operator:** Watches Kubernetes Custom Resources (`ServiceMonitor`, `PodMonitor`, `PrometheusRule`) so scraping targets and alert thresholds are managed declaratively as YAML.
2. **Prometheus Server (Pull Model & TSDB):** Periodically scrapes HTTP `/metrics` endpoints (`scrape_interval`), discovers Kubernetes pods dynamically via Service Discovery, stores labeled time-series data in its Time-Series Database (TSDB), and evaluates PromQL queries and alert rules.
3. **Core Metric Exporters:**
   - **Node Exporter (DaemonSet):** Collects host-level hardware and OS metrics per node (CPU, RAM, disk, network I/O).
   - **cAdvisor (built into Kubelet):** Tracks per-container CPU, memory, and filesystem consumption against resource limits.
   - **`kube-state-metrics`:** Queries the Kubernetes API server to expose Kubernetes object states as metrics (desired vs. available replicas, `Pending` pods, `CrashLoopBackOff`, restart counts).
4. **Alertmanager:** Receives firing alerts from Prometheus, groups/deduplicates related alerts, applies silences, and routes notifications (Slack, PagerDuty, Email, webhooks).
5. **Grafana:** Connects to Prometheus as a data source and renders PromQL queries into live cluster, node, namespace, and pod dashboards.

### Metric Types & PromQL Basics
- **Gauge:** Point-in-time value that goes up or down (memory usage, running pods).
- **Counter:** Cumulative value that only increases (e.g., `container_restarts_total`, `http_requests_total`), normally queried with `rate(...[5m])`.
- **Histogram:** Buckets latencies or request sizes to calculate p95/p99 percentiles.

---

## 2. Directory Layout

```text
005-prometheus-grafana/
├── README.md                           # Architecture notes, Helm deployment, PromQL, and alerting lab
└── web-app-monitoring/                 # Target NGINX Helm chart deployed to generate workload metrics
    ├── Chart.yaml                      # Chart metadata
    ├── values.yaml                     # Configurable defaults (replicas, image, port)
    └── templates/
        ├── deployment.yaml             # NGINX Deployment
        └── service.yaml                # ClusterIP Service
```

---

## 3. Step-by-Step Implementation Plan

### Step 1: Deploy the Target Application
Deploy the target NGINX workload into `demo-lab-monitoring` so there is a live application to monitor:
```bash
helm install web-app-monitoring ./005-prometheus-grafana/web-app-monitoring \
  --namespace demo-lab-monitoring \
  --create-namespace
```
Verify it is running:
```bash
kubectl get all -n demo-lab-monitoring
```

---

### Step 2: Install `kube-prometheus-stack` via Helm
Add (or refresh) the official `prometheus-community` Helm repository:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install the monitoring stack into a dedicated `monitoring` namespace:
```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set kubeControllerManager.enabled=false \
  --set kubeScheduler.enabled=false \
  --set kubeEtcd.enabled=false
```

*Note for local KinD / Docker Desktop clusters:* By default, `kube-controller-manager`, `kube-scheduler`, and `etcd` bind their metrics ports to `127.0.0.1` inside the control-plane container, which prevents in-cluster Prometheus pods from reaching them and triggers false `TargetDown` alerts. Disabling those three control-plane scrapers keeps local KinD labs clean (whereas in custom KinD clusters with `0.0.0.0` patches or managed cloud clusters, teams configure those targets explicitly).

---

### Step 3: Verify Pods, Services, and Operator CRDs
Confirm all monitoring pods reach `Running` (give the stack 1–2 minutes to finish pulling images and initializing):
```bash
kubectl get pods -n monitoring
```
Inspect the Prometheus Operator Custom Resources installed automatically by the chart:
```bash
kubectl get servicemonitors,prometheusrules -n monitoring
```

---

### Step 4: Explore Prometheus Targets and Run PromQL Queries
Port-forward the Prometheus UI:
```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring
```
Open `http://localhost:9090`:
1. Navigate to **Status -> Target health** (or **Status -> Targets**) to verify scrape endpoints are `UP`.
2. Open the **Graph** view and test PromQL queries:
   - Check target health: `up`
   - Count pods per namespace: `sum by (namespace) (kube_pod_info)`
   - Inspect container restarts: `kube_pod_container_status_restarts_total{namespace="demo-lab-monitoring"}`

---

### Step 5: Access Grafana and Inspect Pre-Built Dashboards
Retrieve the auto-generated `admin` password from the `monitoring-grafana` Secret:
```bash
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d; echo
```
- **Username:** `admin`
- **Password:** Output from the command above.

Port-forward the Grafana dashboard service:
```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```
Open `http://localhost:3000`, log in, and explore the pre-provisioned **Dashboards** (such as **Kubernetes / Compute Resources / Cluster**, **Kubernetes / Compute Resources / Namespace (Pods)** for `demo-lab-monitoring`, and **Node Exporter / Nodes**).

---

### Step 6: Test Alerting Thresholds (`PrometheusRule` + Alertmanager)
1. Create a custom `PrometheusRule` threshold (or trigger a failing pod in `demo-lab-monitoring`, such as an intentional `ImagePullBackOff` / `CrashLoopBackOff` test workload) and watch Prometheus transition the alert from **Inactive** -> **Pending** -> **Firing** in the **Alerts** tab (`http://localhost:9090/alerts`).
2. Port-forward **Alertmanager** to see how Prometheus forwards and groups the firing alert:
   ```bash
   kubectl port-forward svc/monitoring-kube-prometheus-alertmanager 9093:9093 -n monitoring
   ```
   Open `http://localhost:9093` and verify the active alert is listed and grouped cleanly.

---

## 4. Cleanup
Remove both Helm releases and namespaces when finished:
```bash
helm uninstall monitoring -n monitoring
helm uninstall web-app-monitoring -n demo-lab-monitoring
kubectl delete ns monitoring demo-lab-monitoring
```
*(Optional: Helm deliberately leaves CustomResourceDefinitions in place to protect cluster data. To remove the Prometheus Operator CRDs completely as well, delete CRDs matching `monitoring.coreos.com`.)*
