# 003 - GitHub Actions: Automated CI & DevSecOps Workflows

In exercises 001 and 002, I tested my Kubernetes and Helm configurations manually by running `helm template` and `kubectl` commands from my local terminal.

While manual testing was essential to learn the core mechanics, in a real engineering environment, manual checks do not scale. Anyone can forget to run a linter or miss a syntax error before merging code.

To solve this, I introduced **Continuous Integration (CI)** using **GitHub Actions** to automate quality checks, security scans, and build validation in the cloud.

---

## 1. What is GitHub Actions?
GitHub Actions is an automation platform built directly into GitHub. Every time I push code or open a Pull Request, GitHub boots up a clean, ephemeral Linux virtual machine in the cloud (called a **Runner**) and executes the exact test instructions I have defined.

Instead of relying on tests on my local machine, GitHub validates my code independently in the cloud before anything can be merged into `main`.

---

## 2. The 4 Core Building Blocks of a Workflow

Every workflow I designed is built with these 4 concepts:

```text
Event (Trigger)  -->  Runner (Virtual Machine)  -->  Job  -->  Steps (Commands)
```

1. **Event (`on`):**  
   The trigger that wakes up the automation (such as a `push` to `main` or opening a `pull_request`).
2. **Runner (`runs-on`):**  
   The virtual machine executing the work (I use `ubuntu-latest`).
3. **Job (`jobs`):**  
   A set of tasks executed on that runner.
4. **Steps (`steps`):**  
   The sequential tasks I defined:
   - Cloning my repository code (`actions/checkout`)
   - Setting up runtimes (like `helm` or `docker`)
   - Executing commands (such as `docker build`, `curl`, or `helm lint`)

---

## 3. Directory Layout & Sample Microservice

Workflows must live at the repository root in `.github/workflows/`, while this folder (`003-github-actions/`) contains my documentation and a custom containerized NGINX web app I built to test Docker CI pipelines:

```text
k8s/
├── .github/
│   └── workflows/
│       ├── 01-helm-lint.yaml       # Syntax validation & compilation checks
│       ├── 02-trivy-scan.yaml      # Security vulnerability & configuration scanner
│       └── 03-docker-build.yaml    # Docker build, container health check & image CVE scan
│
└── 003-github-actions/
    ├── README.md                   # This documentation hub
    └── app/                        # Custom containerized web application
        ├── index.html              # Custom landing page
        └── Dockerfile              # Lightweight NGINX Alpine container build
```

---

## 4. The Three Automated Pipelines in This Project

I split my automation into three dedicated, modular workflows:

### Pipeline 1: Helm Lint and Test (`01-helm-lint.yaml`)
- **Trigger:** Runs on PRs touching `002-helming/**` or the workflow itself (Path Filtering).
- **Execution:**
  1. Boots a clean `ubuntu-latest` VM.
  2. Clones the repository using `actions/checkout@v4`.
  3. Installs Helm v3 using the `azure/setup-helm@v4` action (note: `@v4` refers to the version of the GitHub Action runner tool, while it installs Helm v3.*.*).
  4. Runs `helm lint` on both static and variabilized charts.
  5. Runs `helm template` dry-runs to guarantee all `values.yaml` parameters compile cleanly.

### Pipeline 2: Trivy Helm Security Scan (`02-trivy-scan.yaml`)
- **Trigger:** Runs simultaneously alongside the lint pipeline.
- **Execution:**
  1. Checks out the code.
  2. Uses Aqua Security's `aquasecurity/trivy-action` to scan the Helm manifests for security misconfigurations (such as missing resource limits or privilege escalation risks).
  3. Outputs a vulnerability report table directly in the GitHub Actions summary.

### Pipeline 3: Docker Build & Image Security Scan (`03-docker-build.yaml`)
- **Trigger:** Runs when files inside `003-github-actions/app/**` are modified.
- **Execution:**
  1. Clones the repository code.
  2. Executes `docker build` to create `custom-nginx:latest` using the Alpine base image.
  3. Launches a temporary container in background and tests HTTP responses via `curl`.
  4. Uses Trivy to scan the newly built container image for operating system vulnerabilities (CVEs).

---

## 5. Architectural Note: Artifact Lifecycle & Container Registries

In this exercise, Pipeline 3 intentionally functions as a **pre-merge CI validation gate**—it builds the image on the runner's ephemeral virtual machine, executes smoke tests, scans for security flaws, and discards the image when the job finishes.

In a complete enterprise production pipeline, I would extend this workflow to include a **publishing step** once code is merged into `main`:
- Authenticating to an external container registry (such as GitHub Container Registry `ghcr.io`, Docker Hub, AWS ECR, or Azure ACR).
- Tagging the image with the Git commit SHA or a SemVer release tag (e.g. `custom-nginx:1.0.0`).
- Running `docker push` so that the validated artifact is permanently stored and available for Kubernetes clusters or GitOps controllers (such as ArgoCD) to deploy.
