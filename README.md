# Node.js API Deployment Setup (Production-Ready)

## Career Objective
To work as a DevOps Engineer where I can build dependable CI/CD pipelines, manage scalable Kubernetes clusters, and automate infrastructure to help engineering teams ship secure code faster.

---

## Technical Decisions & Project Assumptions

### 1. Dockerfile Optimization & Security
- **Moving away from `node:latest`:** The original Dockerfile used `node:latest`, which creates a massive image (~1GB) and introduces unnecessary security vulnerabilities. [cite_start]I switched to a multi-stage build using `node:20-alpine`[cite: 10]. This brought the final image size down to under 150MB.
- **Non-Root User Access:** By default, Node containers run as root. I added a dedicated `nodeuser` and `nodegroup` inside the runner stage to ensure the app executes with minimal privileges, matching the cluster's `runAsNonRoot: true` security enforcement.
- **Production Dependencies Only:** The build stage handles the full compilation, but before copying files over to the final runner stage, I ran `npm prune --omit=dev` so development tools are left out of production.

### 2. Kubernetes Layout & Networking (AWS EKS)
- [cite_start]**Why Helm over Static Manifests?** I chose to package the infrastructure as a Helm v3 chart instead of raw YAML manifests to make configuration values dynamic, clean, and easily reusable across staging or production environments[cite: 21, 26].
- [cite_start]**Strict Port Naming Requirement:** Per the project requirements, the core container port has been explicitly named `api-web` (running on port `3000`) instead of using generic names like `http` or `web`[cite: 28].
- **Traffic Control (NetworkPolicy):** The included NetworkPolicy ensures a secure baseline by using matching pod selectors (`app: {{ .Release.Name }}`). This blocks unintended lateral pod-to-pod communication within the cluster namespace.

### 3. Monitoring & Observability Setup
- [cite_start]**Metrics Scraping (Prometheus & Grafana):** I added standard Prometheus scraping annotations directly to the Deployment pod template (`prometheus.io/scrape: "true"` on port `3000`)[cite: 19]. This allows the cluster's Prometheus server to auto-discover the API endpoints seamlessly.
- [cite_start]**Centralized Logs (ELK Stack):** The application is configured to output structured logs straight to `stdout` and `stderr`[cite: 20]. This allows host-level log collectors (like Filebeat) to gather logs cleanly without bloating the application with complex sidecar containers.
- **Auto-Scaling (HPA):** I configured the Horizontal Pod Autoscaler to monitor CPU usage. If the average CPU load crosses 70%, it will automatically scale the pods up from a baseline of 2 replicas up to a maximum of 5 to keep the service stable under load.

---

## How to Test and Verify the Configurations

### Prerequisites
Make sure you have `helm` and `kubectl` installed on your local machine.

### Local Validation Steps
To ensure everything renders cleanly without deploying to a live cluster, run these commands from the repository root:

```bash
# Move into the Helm chart directory
cd helm/node-api

# Run the built-in linter to check for syntax or spacing errors
helm lint .

# Render the final template manifests to verify values are mapping correctly
helm template test-release .