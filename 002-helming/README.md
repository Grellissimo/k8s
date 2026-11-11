# 002 - Helming: Packaging & Release Management

While managing workloads using raw `kubectl apply` and static YAML manifests is excellent for foundational learning and quick experiments, it becomes brittle and unmaintainable in production environments. Static manifests lead to hardcoded values, code duplication across environments (Dev, Staging, Prod), and lack release lifecycle management. To resolve this, we transition to **Helm**—the Kubernetes package manager. Helm allows us to parameterize configurations using `values.yaml`, treat the entire stack (Namespace, Deployment, and Service) as a single versioned release, and perform atomic installs, upgrades, and rollbacks.

---

## Requirements
- **macOS** with Docker Desktop (Kubernetes / KinD enabled)
- **`kubectl`** CLI configured to interact with the local cluster
- **`helm`** CLI (v3+) installed via Homebrew (`brew install helm`)

---

## 1. What is a Helm Chart?
In Kubernetes, if the cluster is considered the operating system, **Helm is its package manager** (analogous to `brew`, `apt`, or `yum`). 

- A **Chart** is a deployable package (similar to a `.deb` or `.rpm`).
- It packages together all necessary Kubernetes primitives (Deployments, Services, Ingresses, ConfigMaps) into a single, version-controlled unit.
- A minimal Helm chart requires:
  ```text
  web-app-helm/
  ├── Chart.yaml             # Metadata (Package name, version, app version)
  ├── values.yaml            # Default configuration variables (optional in static mode)
  └── templates/             # Kubernetes manifests (supercharged with Go-templating)
      ├── deployment.yaml
      └── service.yaml
  ```

---

## 2. Package Metadata (`Chart.yaml`)
Every chart must have a `Chart.yaml` at its root acting as its identity card:

```yaml
apiVersion: v2
name: web-app-helm
description: A Helm chart for deploying our NGINX web application
type: application
version: 0.1.0
appVersion: "1.25.0"
```

### Key Field Breakdown:
- **`apiVersion: v2`**: Identifies this chart as **Helm 3** compliant. (Helm 2 used `v1` and relied on a deprecated in-cluster component called *Tiller*).
- **`name: web-app-helm`**: The unique identifier of the package.
- **`type: application`**: Defines this as a deployable chart (as opposed to a `library` chart used only to share helper logic).
- **`version: 0.1.0` vs. `appVersion: "1.25.0"` (Critical Architectural Distinction)**:
  - **`version` (0.1.0)**: The SemVer version of the **Helm chart packaging code**. If templates, ports, or default configs change, bump this number.
  - **`appVersion` ("1.25.0")**: The version of the **underlying software application** running inside the container (e.g., NGINX version). Bumping the container image tag updates `appVersion`.

---

## 3. The `templates/` Directory
The `templates/` folder is a mandatory Helm convention:
- Helm scans this directory for Kubernetes YAML manifests.
- While it supports dynamic Go-templating (`{{ .Values... }}`), **templates can also be standard static YAML manifests**.
- When deploying, Helm parses all files in `templates/`, injects any variables, and submits the rendered output to the Kubernetes API.

---

## 4. Architectural Best Practice: Namespace Portability
Unlike raw `kubectl` manifests where `metadata.namespace` is often hardcoded:
- **Helm manifests deliberately omit hardcoded namespaces.**
- **Why?** Hardcoding a namespace locks the chart to a single environment. By omitting it, the exact same chart becomes completely portable across environments using Helm's `--namespace <env> --create-namespace` CLI flag:
  - `helm install web-app ./web-app-helm -n development --create-namespace`
  - `helm install web-app ./web-app-helm -n staging --create-namespace`
  - `helm install web-app ./web-app-helm -n production --create-namespace`

---

## 5. Learning Roadmap: Static First, Then Variabilized
1. **Phase 1 (Static Manifests):** Place working static YAML directly inside `templates/` to demonstrate that Helm deploys standard Kubernetes resources out of the box.
2. **Phase 2 (Parameterization / `values.yaml`):** Extract hardcoded strings (replicas, image tags, ports) into `values.yaml` and reference them dynamically using template syntax.

---

## 6. Pre-flight Validation: `helm template` (Dry Run)
Before submitting manifests to the cluster, Helm provides a built-in compiler command to preview and validate the rendered output locally:

```bash
helm template my-release ./web-app-helm
```
*(Renders and prints the full Kubernetes YAML to the terminal without making changes to the cluster).*
