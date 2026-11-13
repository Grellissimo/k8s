# 004 - ArgoCD: GitOps Continuous Delivery & Workload Adoption

In exercises 001 and 002, I managed deployments imperatively from my terminal using `kubectl` and `helm`. In exercise 003, I implemented Continuous Integration (CI) with GitHub Actions to automatically test, lint, and validate my code.

However, deployments were still triggered by manual terminal commands.

In this module, I implement **GitOps Continuous Delivery (CD)** using **ArgoCD**.

---

## 1. The Real-World Customer Scenario: Helm Workload Adoption

In enterprise consulting, teams rarely start from scratch. A typical scenario looks like this:
1. A customer already runs containerized workloads deployed via Helm charts.
2. The team relies on manual `helm upgrade` commands, leading to configuration drift, missed updates, and lack of visibility.
3. The customer wants to adopt **ArgoCD** without causing downtime to the live production services, allowing ArgoCD to take over ownership and continuous synchronization directly from Git.

To simulate this exact scenario:
- I created a dedicated Helm chart: `web-app-argocd-project`.
- I deploy it first to simulate the customer's existing live workload.
- I then install ArgoCD on the cluster and declare an ArgoCD `Application` that connects Git to the running workload and assumes management.

---

## 2. Directory Layout

```text
004-argocd/
├── README.md                           # Documentation & architecture notes
└── web-app-argocd-project/             # Helm chart for the target workload
    ├── Chart.yaml                      # Chart metadata
    ├── values.yaml                     # Configurable defaults (replicas, image, port)
    └── templates/
        ├── deployment.yaml             # NGINX Deployment using {{ .Values... }}
        └── service.yaml                # ClusterIP Service using {{ .Values... }}
```

---

## 3. What is ArgoCD & GitOps?

### GitOps Principles:
1. **Declarative State:** Everything running in the cluster is described in Git.
2. **Git is the Single Source of Truth:** Git holds the definitive version of what should be running.
3. **Automated Synchronization (The Pull Model):** ArgoCD runs inside the Kubernetes cluster. It polls Git, compares the running cluster state to the Git repository, and syncs automatically or alerts on drift.
4. **Self-Healing:** If someone imperatively modifies or deletes a pod or service with `kubectl`, ArgoCD immediately notices the mismatch and forcefully reconciles the cluster back to match Git.

---

## 4. Step-by-Step Implementation Plan

### Step 1: Deploy the Existing Baseline Workload (Simulate Customer State)
Deploy the Helm chart into a dedicated namespace (`demo-lab-argocd`):
```bash
helm install web-app-argocd-release ./004-argocd/web-app-argocd-project --namespace demo-lab-argocd --create-namespace
```
Verify it is running:
```bash
kubectl get all -n demo-lab-argocd
```

---

### Step 2: Install ArgoCD on the Cluster
ArgoCD runs as an in-cluster control plane inside its own namespace (`argocd`):
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Verify that the ArgoCD pods are running:
```bash
kubectl get pods -n argocd
```

---

### Step 3: Access the ArgoCD Web Dashboard
Create a secure tunnel from the local machine into the ArgoCD API server:
```bash
kubectl port-forward svc/argocd-server 8080:443 -n argocd
```
Open `https://localhost:8080` in the browser.

Retrieve the initial auto-generated admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
- **Username:** `admin`
- **Password:** Output from the command above.

---

### Step 4: Define the Declarative ArgoCD Application Manifest
Instead of clicking buttons in the UI, true GitOps declares the ArgoCD Application as code (`application.yaml`):
- Points to the Git repository: `https://github.com/Grellissimo/k8s`
- Points to the path: `004-argocd/web-app-argocd-project`
- Points to the destination namespace: `demo-lab-argocd`

Apply the application manifest so ArgoCD takes over the existing release.

---

### Step 5: Test GitOps Capabilities
1. **Sync Modes:** Practice manual "Click to Deploy" sync vs automated sync.
2. **Drift Detection & Self-Healing:** Manually scale or edit the deployment with `kubectl`, observe ArgoCD flag the application as `OutOfSync`, and watch it self-heal.
3. **Rollbacks:** Test reverting changes via Git and observing immediate cluster reconciliation.

---

## 5. Cleanup
To cleanly remove the lab resources:
```bash
kubectl delete -f 004-argocd/application.yaml
kubectl delete ns demo-lab-argocd
kubectl delete ns argocd
```
