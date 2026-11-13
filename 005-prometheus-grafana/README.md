# 005 - Prometheus & Grafana: Cloud-Native Observability

In exercises 001 through 004, I deployed Kubernetes workloads, packaged them with Helm, automated CI gates with GitHub Actions, and implemented GitOps Continuous Delivery with ArgoCD.

However, deploying applications is only half of operating Kubernetes in production. Once workloads are live, I need real-time visibility into cluster health, node capacity, pod restarts, and automated alerts when thresholds are breached.

In this module, I implement **Cloud-Native Observability (Metrics, Dashboards, and Alerting)** using **`kube-prometheus-stack`** (Prometheus Operator, Prometheus, Alertmanager, exporters, and Grafana).

---

## 1. Overview: Prometheus & Grafana

**Prometheus** is a time-series monitoring and alerting system built for dynamic environments like Kubernetes. Using a pull (scrape) model, it periodically collects metrics from cluster nodes, pods, and applications via HTTP `/metrics` endpoints, stores them with labels in its time-series database, and evaluates PromQL queries and alert rules when thresholds are breached.

**Grafana** is the visualization and dashboarding layer that sits on top of Prometheus. It connects to Prometheus as a data source, runs PromQL queries behind the scenes, and turns raw metrics into live graphs, tables, and Kubernetes dashboards for cluster health, resource usage, and pod restarts.

---

## 2. Directory Layout

```text
005-prometheus-grafana/
├── README.md                           # Architecture notes, Helm deployment, PromQL, and alerting lab
├── prometheus-rule.yaml                # Custom PrometheusRule alert for stuck containers
├── values.yaml                         # Helm overrides for kube-prometheus-stack (KinD lab settings)
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
Add the official `prometheus-community` Helm repository (per the [kube-prometheus-stack chart documentation](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/README.md)):
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install the monitoring stack into a dedicated `monitoring` namespace using [values.yaml](./values.yaml):
```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f ./005-prometheus-grafana/values.yaml
```

*Note for local KinD / Docker Desktop clusters:* By default, `kube-controller-manager`, `kube-scheduler`, and `etcd` bind their metrics ports to `127.0.0.1` inside the control-plane container, which prevents in-cluster Prometheus pods from reaching them and triggers false `TargetDown` alerts. Disabling those three control-plane scrapers keeps local KinD labs clean (whereas in custom KinD clusters with `0.0.0.0` patches or managed cloud clusters, teams configure those targets explicitly).

#### Post-Install Helm Output & Reference Commands
After running `helm install`, Helm prints the release notes below for quick status checks, Grafana credentials, and Operator configuration (`https://github.com/prometheus-operator/kube-prometheus`):

```text
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=monitoring"

Get Grafana 'admin' user password by running:

  kubectl --namespace monitoring get secrets monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

Access Grafana local instance:

  export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitoring" -oname)
  kubectl --namespace monitoring port-forward $POD_NAME 3000

Get your grafana admin user password by running:

  kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```

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
2. From the **Graph** view and test PromQL queries:
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

### Step 6: Test Custom Alerting Thresholds (`PrometheusRule` + Alertmanager)
Prometheus is not limited to built-in default rules—alerting is completely customizable using declarative **`PrometheusRule`** YAML manifests. Teams write custom rules tailored to specific failure modes they care about (such as containers stuck in `ImagePullBackOff` or `CrashLoopBackOff`) with custom wait durations (`for:`) and severity levels (`warning`, `critical`).

1. **Create and apply a custom `PrometheusRule` (`prometheus-rule.yaml`):**
   Define an alert watching `kube_pod_container_status_waiting_reason{namespace="demo-lab-monitoring", reason=~"ImagePullBackOff|ErrImagePull|CrashLoopBackOff"} > 0` (tagged with `release: monitoring` so Prometheus Operator picks it up automatically):
   ```bash
   kubectl apply -f 005-prometheus-grafana/prometheus-rule.yaml
   ```
2. **Simulate a failing pod in `demo-lab-monitoring`:**
   Trigger a temporary `ImagePullBackOff` pod (`kubectl run broken-pod --image=nginx:does-not-exist -n demo-lab-monitoring`) and watch Prometheus transition the custom alert from **Inactive** -> **Pending** -> **Firing** in the **Alerts** tab (`http://localhost:9090/alerts`).
3. **Inspect the alert in Alertmanager:**
   Port-forward **Alertmanager** to see how Prometheus forwards and groups the firing alert:
   ```bash
   kubectl port-forward svc/monitoring-kube-prometheus-alertmanager 9093:9093 -n monitoring
   ```
   Open `http://localhost:9093` and verify the active alert is listed and grouped cleanly, then remove the test pod (`kubectl delete pod broken-pod -n demo-lab-monitoring`).

---

## 4. Cleanup
Remove both Helm releases and namespaces when finished:
```bash
helm uninstall monitoring -n monitoring
helm uninstall web-app-monitoring -n demo-lab-monitoring
kubectl delete ns monitoring demo-lab-monitoring
```
*(Optional: Helm deliberately leaves CustomResourceDefinitions in place to protect cluster data. To remove the Prometheus Operator CRDs completely as well, delete CRDs matching `monitoring.coreos.com`.)*
