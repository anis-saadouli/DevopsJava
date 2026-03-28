# DevOps Exam — Complete Implementation Guide
## Angular CI/CD · Azure DevOps · Docker · Kubernetes (kubeadm)

---

## Architecture

```
Developer
    │
    ▼ git push (dev branch)
GitHub Repository  (professor repo: hamzasarraj/DevopsJava)
    │
    ▼  GitHub service connection triggers Azure Pipelines
Azure DevOps Pipeline
    │
    ├── Stage 1 · Source Checkout      (git clone)
    ├── Stage 2 · Static Code Analysis (SonarQube + quality gate)
    ├── Stage 3 · Build                (npm install + ng build --prod)
    ├── Stage 4 · Containerization     (docker build — multi-stage)
    ├── Stage 5 · Security Scan        (Trivy — fails on HIGH/CRITICAL)
    ├── Stage 6 · Image Registry       (docker push → DockerHub)
    └── Stage 7 · Kubernetes Deploy    (kubectl apply — main branch only)
                                              │
                                              ▼
                              kubeadm Kubernetes Cluster
                              ┌──────────────────────────┐
                              │  master-node  (control)  │
                              │  worker-node-1           │
                              │  worker-node-2           │
                              └──────────────────────────┘
                                              │
                              Namespace: devops-frontend
                              Deployment: angular-frontend (2 replicas)
                              Service:    NodePort 30080
                                              │
                                              ▼
                              http://<WorkerNodeIP>:30080
```

---

## Part 1 — Azure DevOps Initial Setup

### 1.1 Create Organization and Project

1. Go to [https://dev.azure.com](https://dev.azure.com) → **New organization**
2. Choose a region close to you (e.g. West Europe)
3. Inside the organization → **New project**
   - Name: `DevopsExam`
   - Visibility: Private
   - Version control: Git
   - Work item process: Scrum

### 1.2 Connect Pipeline to GitHub

1. **Pipelines → New Pipeline**
2. Select **GitHub** as source
3. Authenticate with your GitHub account
4. Select repository: `hamzasarraj/DevopsJava`
5. Choose **Existing Azure Pipelines YAML file**
6. Branch: `main` / Path: `angular-app-kubernetes/azure-pipelines.yml`
7. Save (do not run yet — configure service connections first)

---

## Part 2 — Service Connections

Navigate to **Project Settings → Service connections → New service connection**.

| # | Name in YAML | Type | Details |
|---|---|---|---|
| 1 | `sc-dockerhub` | Docker Registry | DockerHub username + password/token |
| 2 | `sc-sonarqube` | SonarQube Server | Server URL + user token |
| 3 | `sc-github` | GitHub | OAuth or PAT (auto-created when linking pipeline) |

> **AKS/ACR optional** (exam theory alternative only):
> `sc-acr` → Azure Container Registry  
> `sc-aks` → Kubernetes / Azure  
> `sc-azure-subscription` → Azure Resource Manager

### 2.1 Create DockerHub Service Connection

1. Type: **Docker Registry → DockerHub**
2. Docker ID: `<your-dockerhub-username>`
3. Password: use a DockerHub **Access Token** (not your password)
   - DockerHub → Account Settings → Security → New Access Token
4. Service connection name: **`sc-dockerhub`** (must match YAML exactly)
5. ✅ Grant access permission to all pipelines

### 2.2 Create SonarQube Service Connection

1. Type: **SonarQube**
2. Server URL: `http://<your-sonarqube-host>:9000`
3. Token: generate on SonarQube → **My Account → Security → Generate Token**
4. Service connection name: **`sc-sonarqube`**

---

## Part 3 — Variable Group (Pipeline Secrets)

**Pipelines → Library → + Variable group**

Group name: **`New variable group 18-Mar`** ← must match the name in the YAML

| Variable | Value | Secret? |
|---|---|---|
| `DOCKERHUB_USERNAME` | your DockerHub username | No |
| `IMAGE_NAME` | `angular-app` (or your image name) | No |
| `SONAR_HOST_URL` | `http://<sonarqube-host>:9000` | No |
| `SONAR_LOGIN` | SonarQube user token | **Yes** |
| `KUBE_CONFIG` | base64-encoded kubeconfig (see §5.4) | **Yes** |

> **How to encode the kubeconfig** (run on master node after cluster setup):
> ```bash
> cat ~/.kube/config | base64 -w 0
> ```
> Paste the output as the `KUBE_CONFIG` secret variable.

---

## Part 4 — Branch Strategy

### 4.1 Branches

| Branch | Purpose | Pipeline behavior |
|---|---|---|
| `dev` | Daily development | Stages 1–6 (CI only — no deploy) |
| `main` | Stable releases | Stages 1–7 (full CI + CD) |

### 4.2 Workflow

```
feature work → commit to dev → CI pipeline runs (Sonar + build + docker + trivy + push)
dev → Pull Request → review → merge to main → CD stage runs (kubectl deploy)
```

### 4.3 Branch Protection (Optional but recommended)

**Repos → Branches → main → Branch policies**:
- ✅ Require a minimum 1 reviewer
- ✅ Check for linked work items
- ✅ Build validation: link your pipeline (triggers on PR)

---

## Part 5 — Kubernetes Cluster Setup (kubeadm — 1 master + 2 workers)

### 5.1 VM Prerequisites (repeat on all 3 VMs)

OS: Ubuntu 22.04 LTS · 2 vCPU · 2 GB RAM minimum

```bash
# Disable swap (Kubernetes requirement)
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Enable IPv4 forwarding
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

### 5.2 Install Container Runtime — containerd (all 3 VMs)

```bash
# Install containerd
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 5.3 Install kubeadm, kubelet, kubectl (all 3 VMs)

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 5.4 Initialize the Master Node (master only)

```bash
# Replace <MASTER_IP> with the master VM's IP address
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<MASTER_IP>

# Set up kubeconfig for the current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Deploy Flannel CNI network plugin
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Generate the base64 kubeconfig for the Azure DevOps variable:
```bash
cat ~/.kube/config | base64 -w 0
# Copy the output → Azure DevOps Library → KUBE_CONFIG secret variable
```

### 5.5 Join Worker Nodes

On the master, get the join command:
```bash
kubeadm token create --print-join-command
```

Run the printed command on **both worker VMs**:
```bash
# Example (your tokens will differ)
sudo kubeadm join <MASTER_IP>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

Verify from master:
```bash
kubectl get nodes
# Expected output:
# NAME           STATUS   ROLES           AGE
# master-node    Ready    control-plane   5m
# worker-node-1  Ready    <none>          2m
# worker-node-2  Ready    <none>          2m
```

---

## Part 6 — SonarQube Setup

### 6.1 Run SonarQube (Docker — fastest for exam)

On any accessible server:
```bash
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:community

# Wait ~60 seconds then open http://<server-ip>:9000
# Default login: admin / admin  (change password on first login)
```

### 6.2 Create SonarQube Project

1. SonarQube UI → **Create Project → Manually**
2. Project key: `devopsjava-angular-frontend`  ← matches `sonar-project.properties`
3. Display name: `devopsjava-angular-frontend`
4. **My Account → Security → Generate Token** → copy for Azure DevOps variable

### 6.3 Quality Gate (SonarQube)

The pipeline uses `-Dsonar.qualitygate.wait=true` which causes sonar-scanner to
poll SonarQube until the quality gate result is available and **exits with code 1**
if the gate fails — blocking all downstream stages (build, docker, deploy).

Default "Sonar Way" gate conditions: 0 new bugs, 0 new vulnerabilities, coverage > 80%.
Adjust thresholds in SonarQube → **Quality Gates** if needed for the exam app.

---

## Part 7 — Pipeline Stages Explained

### Stage 1 · Source Checkout
- Agent checks out the repository from GitHub using the GitHub service connection
- Equivalent to `git clone` from a CI perspective

### Stage 2 · Static Code Analysis
- Installs `sonarqube-scanner` globally via npm
- Analyzes `src/` (excludes spec files, node_modules, dist, css)
- `-Dsonar.qualitygate.wait=true` blocks the pipeline until the quality gate completes

### Stage 3 · Build
- Uses Node.js 16
- `npm ci --legacy-peer-deps` for reproducible install
- `npm run build -- --configuration production` compiles Angular with AOT + minification
- Publishes `dist/angular-app/` as a pipeline artifact `angular-dist`

### Stage 4 · Containerization
- Multi-stage Dockerfile:
  - **Stage 1 (build):** `node:16-alpine` → `npm ci` → `ng build --prod`
  - **Stage 2 (runtime):** `nginx:1.27-alpine` → copy dist → custom nginx.conf
- `nginx.conf` handles Angular SPA routing (`try_files $uri /index.html`)
- Image tagged: `username/image-name:<buildId>` and `username/image-name:latest`
- Saved as `.tar` artifact for the next stage

### Stage 5 · Security Scan
- Downloads the image `.tar` artifact
- Loads into local Docker daemon
- Runs Trivy via Docker container scan:
  - `--severity HIGH,CRITICAL` — fails on critical vulnerabilities
  - `--ignore-unfixed` — skips issues with no available fix
  - `--exit-code 1` — pipeline fails if issues found

### Stage 6 · Image Registry
- Downloads the image `.tar` artifact
- Logs in to DockerHub using the `sc-dockerhub` service connection
- Pushes both `:buildId` and `:latest` tags

### Stage 7 · Kubernetes Deployment
- **Condition:** only executes when branch is `main`
- Configures `kubectl` using the base64-decoded `KUBE_CONFIG` secret variable
- Substitutes `__IMAGE__` placeholder in `deployment.yaml` with the actual pushed tag
- Applies manifests in order: namespace → deployment → service
- Verifies rollout completes within 180 seconds via `kubectl rollout status`

---

## Part 8 — Kubernetes Manifests

### namespace.yaml
Creates the isolated workspace for the application:
```yaml
kubectl apply -f k8s/namespace.yaml
# Creates: namespace/devops-frontend
```

### deployment.yaml
- 2 replicas for HA
- `__IMAGE__` placeholder replaced by pipeline `sed` command
- `imagePullPolicy: Always` ensures the latest DockerHub image is pulled
- Liveness probe: restarts pod if `/` returns no response after 10s init delay
- Readiness probe: removes pod from service endpoints until it responds to `/`

### service.yaml
- Type: `NodePort` — appropriate for on-premise kubeadm clusters
- Exposes the application on port `30080` of any worker node
- Access: `http://<worker-node-ip>:30080`

> **LoadBalancer alternative** (for AKS/cloud):  
> Change `type: NodePort` → `type: LoadBalancer` and remove `nodePort: 30080`  
> Azure automatically provisions a public IP and external load balancer

---

## Part 9 — Self-Hosted Agent (Reference)

Although this implementation uses hosted `ubuntu-latest` agents, the exam requires
understanding self-hosted agents. Here is the setup:

### 9.1 Create Agent Pool

**Organization Settings → Agent pools → Add pool**
- Type: Self-hosted
- Name: `SelfHostedLinux`
- ✅ Grant access permission to all pipelines

### 9.2 Install Agent on a VM

```bash
mkdir ~/myagent && cd ~/myagent
curl -O https://vstsagentpackage.blob.core.windows.net/agent/3.x.x/vsts-agent-linux-x64-3.x.x.tar.gz
tar zxvf vsts-agent-linux-x64-*.tar.gz
./config.sh
# Enter: Azure DevOps URL, PAT token, pool name: SelfHostedLinux
./run.sh        # interactive
# OR
sudo ./svc.sh install && sudo ./svc.sh start  # as a service
```

### 9.3 Required Tools on Self-Hosted Agent

```bash
# Docker
curl -fsSL https://get.docker.com | bash
sudo usermod -aG docker $USER

# Node.js 16
curl -fsSL https://deb.nodesource.com/setup_16.x | bash
sudo apt-get install -y nodejs

# kubectl
sudo apt-get install -y kubectl

# Trivy
sudo apt-get install -y wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb generic main | \
  sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy
```

### 9.4 Switch Pipeline to Self-Hosted

In `azure-pipelines.yml`, replace every occurrence of:
```yaml
pool:
  vmImage: 'ubuntu-latest'
```
with:
```yaml
pool:
  name: 'SelfHostedLinux'
```

---

## Part 10 — AKS / ACR Alternative Architecture

For the cloud variant (Option 1 from the exam spec):

### ACR Service Connection
```
Project Settings → Service Connections → Docker Registry → Azure Container Registry
Name: sc-acr
```

### AKS Service Connection
```
Project Settings → Service Connections → Kubernetes → Azure Subscription
Name: sc-aks
Select your AKS cluster and namespace: devops-frontend
```

### Pipeline changes for ACR push (replace ImageRegistry stage script)
```yaml
- task: Docker@2
  displayName: Build and push to ACR
  inputs:
    command: buildAndPush
    containerRegistry: sc-acr
    repository: angular-app
    tags: |
      $(Build.BuildId)
      latest
```

### Pipeline changes for AKS deploy (replace Deploy stage script)
```yaml
- task: KubernetesManifest@1
  displayName: Deploy to AKS
  inputs:
    action: deploy
    connectionType: azureResourceManager
    azureSubscriptionConnection: sc-azure-subscription
    azureResourceGroup: <your-rg>
    kubernetesCluster: <your-aks-name>
    namespace: devops-frontend
    manifests: |
      k8s/namespace.yaml
      k8s/deployment.yaml
      k8s/service.yaml
    containers: $(acrLoginServer)/angular-app:$(Build.BuildId)
```

---

## Part 11 — Verification Checklist

### Azure DevOps
- [ ] Organization and project created
- [ ] Pipeline created from GitHub YAML
- [ ] All service connections green (test connection)
- [ ] Variable group bound to pipeline
- [ ] dev and main branches exist in repository

### kubeadm Cluster
- [ ] All 3 nodes show `Ready`: `kubectl get nodes`
- [ ] Flannel pods running: `kubectl get pods -n kube-flannel`
- [ ] KUBE_CONFIG secret variable set (base64 encoded)

### CI Pipeline (dev branch)
- [ ] Push to dev triggers pipeline
- [ ] SonarQube shows analysis result in dashboard
- [ ] Quality gate blocks intentional code issue
- [ ] Docker image built and tagged with BuildId
- [ ] Trivy scan completes (HIGH/CRITICAL fails, others pass)
- [ ] Image appears on DockerHub

### CD Pipeline (main branch)
- [ ] Merge PR dev → main triggers full pipeline
- [ ] Deploy stage runs (not skipped)
- [ ] `kubectl rollout status` passes
- [ ] App accessible: `curl http://<worker-ip>:30080`
- [ ] SPA routing works: refresh on `/some/path` returns 200 (not 404)

---

## Part 12 — Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ImagePullBackOff` in pods | Wrong image name or DockerHub auth | Check `imageRepository` variable and DockerHub credentials |
| Deploy stage skipped | Branch is not `main` | Ensure you merged PR to main |
| `kubectl: command not found` | kubectl not on agent path | Hosted agents have kubectl; for self-hosted, install it |
| `cluster-info` fails in pipeline | Wrong or expired KUBE_CONFIG | Re-generate: `cat ~/.kube/config \| base64 -w 0` |
| SonarQube: quality gate ERROR | Code issues above gate threshold | Fix code issues OR relax the gate in SonarQube UI |
| Trivy exits 1 | HIGH/CRITICAL CVEs in image | Base image `nginx:1.27-alpine` auto-updated by `apk upgrade`; rebuild |
| `404` on Angular deep-link | nginx not serving `index.html` fallback | Verify `nginx.conf` was copied into the image; `try_files` must be present |
| Worker node `NotReady` | CNI not started | Check Flannel pods: `kubectl get pods -n kube-flannel` |
