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
├── application.yaml                    # Declarative ArgoCD Application manifest
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
- Points to the Git repository: `https://github.com/adrianofarina/devops-portfolio`
- Points to the path: `004-argocd/web-app-argocd-project`
- Points to the destination namespace: `demo-lab-argocd`

Apply the application manifest so ArgoCD takes over the existing release:
```bash
kubectl apply -f 004-argocd/application.yaml
```
Because the workload is already running from Step 1, ArgoCD initially shows the application as **`Healthy`** and **`OutOfSync`** (missing ArgoCD tracking metadata). Clicking **Sync** -> **Synchronize** in the UI adopts the existing `Deployment` and `Service` under GitOps management with zero downtime.

---

### Step 5: Test GitOps Capabilities

1. **Manual Drift Detection (Manual Sync Mode):**
   Instead of enabling `selfHeal`, I kept ArgoCD in manual sync mode so it detects drift immediately while leaving final sync approval to an operator:
   ```bash
   kubectl scale deployment/web-app-argocd-project --replicas=3 -n demo-lab-argocd
   ```
   ArgoCD kept the pods **`Healthy`**, flipped the status to **`OutOfSync`**, highlighted the `replicas` mismatch in **App Diff**, and scaled back down once I clicked **Sync** -> **Synchronize**.

2. **Deploying from Immutable Git Tags (`v0.4.0` and `v0.4.1`):**
   I created annotated Git tags on commits with `2` replicas (`v0.4.0`) and `3` replicas (`v0.4.1`) in `values.yaml` and pushed each tag explicitly to GitHub:
   ```bash
   git tag -a v0.4.0 -m "Release v0.4.0 with 2 replicas"
   git push origin v0.4.0

   git tag -a v0.4.1 -m "Release v0.4.1 with 3 replicas"
   git push origin v0.4.1
   ```
   By editing the application's **Target Revision** in ArgoCD (**Details -> Summary -> Edit**), I switched cleanly from `v0.4.0` (`2` pods) to `v0.4.1` (`3` pods).

3. **Rollbacks:**
   Using **History and Rollback** in the ArgoCD UI, I reverted the live deployment from `v0.4.1` (`3` pods) back to `v0.4.0` (`2` pods). ArgoCD stores the exact commit SHA of each past sync, so rolling back restores the cluster immediately while marking the app **`OutOfSync`** until **Target Revision** (or Git via `git revert`) is updated to match.

#### How Branches and Tags Work Together in GitOps
A branch (such as `main` or `feature/argocd`) is a moving target that advances with every new commit, whereas an annotated Git tag (`v0.4.0`, `v0.4.1`) is a frozen snapshot anchored permanently to one exact commit SHA—even if the feature branch is later merged or deleted. In a clean release workflow, developers work and test on feature branches, merge verified changes into `main` via Pull Request, and then create a release tag directly on `main`. Pointing ArgoCD at a release tag instead of a moving branch ensures deployments and rollbacks always target an immutable, verified version of the manifests.

---

## 5. Additional Integration: GitHub OAuth SSO & RBAC

Instead of relying only on the local break-glass `admin` account, I wired ArgoCD's built-in **Dex** identity broker (`argocd-dex-server`) to authenticate via **GitHub OAuth** and map my GitHub identity to `role:admin` in RBAC (following the official [ArgoCD User Management & Dex SSO docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/)).

1. **Create a GitHub OAuth App** (GitHub Settings -> Developer settings -> OAuth Apps -> New OAuth App, or directly via `https://github.com/settings/developers`):
   - **Homepage URL:** `https://localhost:8080`
   - **Redirect URI (or Authorization callback URL):** `https://localhost:8080/api/dex/callback`

2. **Store the Client Secret locally in `argocd-secret`** (never commit OAuth secrets to Git):
   ```bash
   kubectl patch secret argocd-secret -n argocd --type merge -p '{"stringData":{"dex.github.clientSecret":"<YOUR_CLIENT_SECRET>"}}'
   ```

3. **Configure the Dex GitHub connector in `argocd-cm`** (`--type merge` keeps existing keys intact):
   ```bash
   kubectl patch configmap argocd-cm -n argocd --type merge -p '
   data:
     url: https://localhost:8080
     dex.config: |
       connectors:
         - type: github
           id: github
           name: GitHub
           config:
             clientID: <GITHUB_CLIENT_ID>
             clientSecret: $dex.github.clientSecret
             redirectURI: https://localhost:8080/api/dex/callback
   '
   ```

4. **Restrict RBAC access in `argocd-rbac-cm`** (deny unmapped users by default and grant `role:admin` to my GitHub email):
   ```bash
   kubectl patch configmap argocd-rbac-cm -n argocd --type merge -p '
   data:
     policy.default: ""
     scopes: "[groups, email]"
     policy.csv: |
       g, <YOUR_GITHUB_EMAIL>, role:admin
   '
   ```

5. **Restart Dex and the ArgoCD server, re-open `port-forward`, and click "LOG IN VIA GITHUB":**
   ```bash
   kubectl rollout restart deployment argocd-dex-server argocd-server -n argocd
   kubectl port-forward svc/argocd-server 8080:443 -n argocd
   ```

---

## 6. Cleanup
To cleanly remove the lab resources:
```bash
kubectl delete -f 004-argocd/application.yaml
kubectl delete ns demo-lab-argocd
kubectl delete ns argocd
```
