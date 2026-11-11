# 002 - Helming: Packaging and Deploying

In the first exercise, we used raw `kubectl` commands and static YAML files. That was great to learn the basics, but in real life, deploying by hand file-by-file gets messy quickly.

This is where **Helm** comes in. Helm is like the Homebrew or App Store for Kubernetes. Instead of running `kubectl apply` on three different files, Helm packages everything together so you can install, update, or delete the whole app with one simple command.

---

## What We Need to Deploy

Just like in exercise 001, our goal is to have:
1. **A Namespace (`demo-lab-helm`)**: To keep our app isolated in its own space.
2. **A Deployment (`web-app-helm`)**: To run our NGINX container and make sure it stays alive (named `web-app-helm` to clearly separate it from our raw kubectl lab).
3. **A Service (`web-app-helm-service`)**: To act as the internal door and give our pods a stable network address.

---

## How Helm Organizes Files

A Helm chart is just a simple folder with a few specific files:

```text
web-app-helm/
├── Chart.yaml              # The ID card of the chart (name, version)
└── templates/              # The actual Kubernetes YAML files
    ├── deployment.yaml     # Tells K8s how to run the pods
    └── service.yaml        # Tells K8s how to network the pods
```

### 1. `Chart.yaml` (The ID Card)
Every chart must have this file. It simply tells Helm what this package is:
```yaml
apiVersion: v2
name: web-app-helm
description: A Helm chart for deploying our NGINX web application
type: application
version: 0.1.0
appVersion: "1.25.0"
```
- `version: 0.1.0`: The version of this Helm folder/chart.
- `appVersion: "1.25.0"`: The version of the app running inside (NGINX).

### 2. `templates/deployment.yaml`
This is almost identical to the deployment file from exercise 001. It tells Kubernetes to run 1 copy of `nginx:alpine` and listen on port 80.

### 3. `templates/service.yaml`
This tells Kubernetes to set up an internal `ClusterIP` network bridge that forwards traffic to any pod with the label `app: web-app-helm`.

### 4. The Namespace File

The catch: there is no `namespace.yaml` file to be created. 

The namespace gets specified at deployment time directly in the command. Helm will automatically check if that namespace already exists or if it needs to be created on the fly (via the `--create-namespace` flag).

This makes the chart **fully reusable**. Instead of hardcoding a namespace and locking the chart to a single place, the exact same package can be deployed across different environments (like `dev`, `staging`, or `production`) without changing a single line of code.


## Step-by-Step Commands

### Step 1: Test Before Installing (Dry Run)
Before touching the cluster, this checks if Helm can read and compile the files properly:
```bash
helm template web-app-release ./web-app-helm
```
This renders and prints the final YAML to the terminal so the engineer can inspect it before applying.

**What does this catch?**
- Catches YAML syntax errors, bad indentation, missing colons, or template typos. If something is broken, Helm will immediately throw an error with the exact line number.
- Note: this runs **offline** on your machine, so it won't verify if the specified Docker image actually exists or if the cluster has enough RAM.

**Additional testing - The Live Cluster Dry-Run:**  
As additional test, it is possible to validate the deployment against the live cluster and test if Kubernetes will accept the configuration without deploying it yet:
```bash
helm install web-app-release ./web-app-helm --namespace demo-lab-helm --dry-run
```

### Step 2: Install Everything to the Cluster
Once tested above's commands, I can now run this command to create the namespace and deploy the whole app:
```bash
helm install web-app-release ./web-app-helm --namespace demo-lab-helm --create-namespace
```
- `web-app-release`: The name given to this running installation.
- `./web-app-helm`: The path to the chart folder.
- `--namespace demo-lab-helm`: Puts everything inside the `demo-lab-helm` namespace.
- `--create-namespace`: Tells Helm to make the namespace if it does not exist yet.

Output:
```
NAME: web-app-release
LAST DEPLOYED: Wed Nov 11 18:41:43 2026
NAMESPACE: demo-lab-helm
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
```

### Step 3: Check That Everything is Running
```bash
helm list -n demo-lab-helm
kubectl get all -n demo-lab-helm
```
Can now observe Helm tracking the release, along with the deployment, pod, and service all running inside `demo-lab-helm`.

### Step 4: Test Access via Port-Forward
Just like the deployment via kubectl, i can tunnel traffic from my laptop:
```bash
kubectl port-forward svc/web-app-helm-service 8080:80 -n demo-lab-helm
```

While the tunnel is running, i can verify it in two ways:

1. **Via `curl` in a second terminal tab:**
   ```bash
   curl -I http://localhost:8080
   ```
   *Expected output:* `HTTP/1.1 200 OK` with `Server: nginx/...` headers.

2. **Via Browser:**
   I can open `http://localhost:8080` to see the "Welcome to nginx!" HTML page. 

### Step 5: Updating the Application (Zero-Downtime Upgrades)
In production, uninstall and reinstall an app to update it is not the correct way. That causes outages. Instead, Helm uses `upgrade` to trigger a Kubernetes rolling update (new pods spin up and become healthy before old pods are terminated):
After making some changes, say, to the chart version and the replica number, i can trigger the upgrade and observe:

```bash
helm upgrade web-app-release ./web-app-helm -n demo-lab-helm
```
Helm calculates what changed, applies the diff, and bumps the release to `REVISION: 2`.

### Step 6: Tracking History and Instant Rollbacks
Helm keeps a full revision history of every deployment:

```bash
helm history web-app-release -n demo-lab-helm
```
Output:
```
REVISION	UPDATED                 	STATUS    	CHART             	APP VERSION	DESCRIPTION
1       	Wed Nov 11 18:41:43 2026	superseded	web-app-helm-0.1.0	1.25.0     	Install complete
2       	Wed Nov 11 18:55:40 2026	deployed  	web-app-helm-0.2.0	1.25.0     	Upgrade complete
```

If an update introduces a bug or breaks in production, i can roll back to any previous revision instantly in one command without touching YAML files:

```bash
helm rollback web-app-release 1 -n demo-lab-helm
```
Output:
```
REVISION	UPDATED                 	STATUS    	CHART             	APP VERSION	DESCRIPTION
1       	Wed Nov 11 18:41:43 2026	superseded	web-app-helm-0.1.0	1.25.0     	Install complete
2       	Wed Nov 11 18:55:40 2026	superseded	web-app-helm-0.2.0	1.25.0     	Upgrade complete
3       	Wed Nov 11 18:57:07 2026	deployed  	web-app-helm-0.1.0	1.25.0     	Rollback to 1
```

### Step 7: Clean Teardown
To remove the app while preserving the release audit trail, i can uninstall with the `--keep-history` flag:
```bash
helm uninstall web-app-release -n demo-lab-helm --keep-history
```
This tears down all running pods and services to free cluster resources, while keeping the revision records intact.

Finally, to delete the namespace:
```bash
kubectl delete ns demo-lab-helm
```
