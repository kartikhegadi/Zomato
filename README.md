# Zomato DevOps Pipeline — Complete Setup Guide

> **Repository:** [https://github.com/kartikhegadi/Zomato.git](https://github.com/kartikhegadi/Zomato.git)

This guide walks you through setting up a full CI/CD pipeline for the Zomato application on AWS, including Jenkins, Docker, SonarQube, Prometheus, Grafana, EKS, and ArgoCD.

---

## Table of Contents

1. [Infrastructure Setup](#1-infrastructure-setup)
2. [Connect & Update the Instance](#2-connect--update-the-instance)
3. [Install AWS CLI](#3-install-aws-cli)
4. [Install Jenkins](#4-install-jenkins)
5. [Install Docker](#5-install-docker)
6. [Install Docker Scout](#6-install-docker-scout)
7. [Install SonarQube](#7-install-sonarqube)
8. [Jenkins Plugin Installation](#8-jenkins-plugin-installation)
9. [Jenkins Configuration](#9-jenkins-configuration)
10. [Create the Pipeline Job](#10-create-the-pipeline-job)
11. [Monitoring Server Setup](#11-monitoring-server-setup)
12. [EKS Cluster Setup](#12-eks-cluster-setup)
13. [ArgoCD Installation](#13-argocd-installation)
14. [Monitor Kubernetes with Prometheus](#14-monitor-kubernetes-with-prometheus)
15. [Cleanup](#15-cleanup)

---

## 1. Infrastructure Setup

### 1.1 Launch an EC2 Instance

| Setting | Value |
|---|---|
| **OS** | Ubuntu 24.04 |
| **Instance Type** | t2.large |
| **Storage** | 30 GB EBS |

### 1.2 Security Group — Open the Following Ports

| Protocol | Port | Source | Purpose |
|---|---|---|---|
| HTTPS | 443 | 0.0.0.0/0 | Secure web traffic |
| HTTP | 80 | 0.0.0.0/0 | Web traffic |
| SSH | 22 | 0.0.0.0/0 | Remote access |
| Custom TCP | 8080 | 0.0.0.0/0 | Jenkins |
| Custom TCP | 9090 | 0.0.0.0/0 | Prometheus |
| Custom TCP | 9000 | 0.0.0.0/0 | SonarQube |
| Custom TCP | 9100 | 0.0.0.0/0 | Node Exporter |
| Custom TCP | 3000 | 0.0.0.0/0 | Application & Grafana |

> **Note:** Port 9100 is required for Node Exporter to push CPU, memory, and system metrics to Prometheus.

---

## 2. Connect & Update the Instance

Once the instance is running, SSH into it and update all packages:

```bash
# Switch to root user
sudo su

# Update all packages
sudo apt update -y
```

---

## 3. Install AWS CLI

The AWS CLI is required for interacting with AWS services programmatically.

```bash
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify the installation:

```bash
aws --version
```

---

## 4. Install Jenkins

Jenkins is the core CI/CD automation server for this pipeline.

> **Reference:** [Jenkins Installation on Ubuntu](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)

Create and run the installation script:

```bash
vi jenkins.sh
```

Paste the following content:

```bash
#!/bin/bash
sudo apt update -y
sudo apt install fontconfig openjdk-21-jre -y
java -version
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/rpm-stable/jenkins.repo
sudo dnf upgrade
sudo dnf install fontconfig java-21-openjdk -y
sudo dnf install jenkins -y
sudo systemctl daemon-reload
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

Run the script:

```bash
chmod +x jenkins.sh
./jenkins.sh
```

### 4.1 Access Jenkins

- Open port **8080** in your security group (already done in Step 1).
- Access Jenkins at: `http://<your-ec2-public-ip>:8080`
- Follow the on-screen setup wizard to complete the initial configuration.

---

## 5. Install Docker

Docker is used to containerize the application and run SonarQube.

```bash
vi docker.sh
```

Paste the following content:

```bash
#!/bin/bash
sudo apt update -y

# Remove old/conflicting Docker packages
sudo apt remove -y docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc

# Install prerequisites
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings

# Add Docker's official GPG key
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Allow current user to run Docker without sudo
sudo usermod -aG docker ubuntu
sudo chmod 777 /var/run/docker.sock
newgrp docker

# Verify Docker installation
sudo docker run hello-world
```

Run the script:

```bash
chmod +x docker.sh
./docker.sh
```

---

## 6. Install Docker Scout

Docker Scout is used for vulnerability scanning of container images.

```bash
curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
sh install-scout.sh
```

---

## 7. Install SonarQube

SonarQube performs static code analysis to detect bugs, vulnerabilities, and code smells. It is run as a Docker container.

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Verify the container is running:

```bash
docker ps
```

Access SonarQube at: `http://<your-ec2-public-ip>:9000`

> Default credentials: **admin / admin** (you will be prompted to change on first login)

---

## 8. Jenkins Plugin Installation

Navigate to **Manage Jenkins → Plugins → Available Plugins** and install the following:

| Plugin | Purpose |
|---|---|
| Pipeline: Stage View | Visualize pipeline stages |
| Eclipse Temurin installer | Manage JDK installations |
| SonarQube Scanner | Integrate SonarQube analysis |
| Docker | Core Docker support |
| Docker Commons | Shared Docker utilities |
| Docker Pipeline | Use Docker in pipeline scripts |
| Docker API | Docker API access |
| docker-build-step | Docker build step in freestyle jobs |
| Email Extension Template | Advanced email notifications |
| OWASP Dependency-Check | Scan for vulnerable dependencies |
| NodeJS | Manage Node.js installations |
| Prometheus metrics | Expose Jenkins metrics for Prometheus |

Restart Jenkins after installation to activate the plugins.

---

## 9. Jenkins Configuration

### 9.1 Tools Configuration

Navigate to **Manage Jenkins → Tools** and configure:
- **JDK** — Add JDK using Eclipse Temurin installer
- **NodeJS** — Add the required Node.js version
- **SonarQube Scanner** — Add the scanner installation
- **Docker** — Add Docker installation

### 9.2 SonarQube Token Configuration

1. In SonarQube, generate an authentication token under **My Account → Security**.
2. In Jenkins, navigate to **Manage Jenkins → Credentials → System → Global credentials**.
3. Add a new **Secret Text** credential with the SonarQube token.
4. In **Manage Jenkins → System**, configure the SonarQube server URL and link the token credential.

### 9.3 DockerHub Credentials

1. In Jenkins, navigate to **Manage Jenkins → Credentials → System → Global credentials**.
2. Add a **Username with Password** credential using your DockerHub username and password.
3. This credential will be used by the pipeline to push Docker images to DockerHub automatically after a successful build.

### 9.4 Email Notification Configuration

Navigate to **Manage Jenkins → System → Extended E-mail Notification** and configure your SMTP server settings (e.g., Gmail SMTP). This allows Jenkins to send build status notifications.

---

## 10. Create the Pipeline Job

### 10.1 Pre-requisites Before Creating the Job

Before pasting the pipeline script, make the following changes in the Jenkinsfile:

1. In the **Tag and Push to DockerHub** stage — replace the placeholder with your DockerHub username.
2. Apply the same DockerHub username change in the **DockerScoutImage** and **Deploy to container** stages.
3. In the **post actions** section — replace the placeholder email with the email address configured in Jenkins.

### 10.2 SonarQube Webhook

In SonarQube, navigate to **Administration → Configuration → Webhooks** and create a new webhook pointing to your Jenkins URL:

```
http://<your-jenkins-ip>:8080/sonarqube-webhook/
```

This allows SonarQube to notify Jenkins when analysis is complete, so the pipeline can continue automatically.

### 10.3 Create the Pipeline

1. In Jenkins, click **New Item → Pipeline**.
2. Configure the pipeline to pull the `Jenkinsfile` from your Git repository.
3. Save and run the pipeline.

---

## 11. Monitoring Server Setup

A dedicated monitoring server is used to run Prometheus, Grafana, and Node Exporter — keeping monitoring concerns separate from the application server.

### 11.1 Launch the Monitoring Server

| Setting | Value |
|---|---|
| **Name** | Monitoring Server |
| **OS** | Ubuntu 24.04 |
| **Instance Type** | t2.large |
| **Storage** | 30 GB EBS |
| **Security Group** | Use the same security group created in Step 1 |

Connect to the new instance via SSH.

---

### 11.2 Install Prometheus

Prometheus is an open-source monitoring and alerting toolkit that scrapes and stores time-series metrics.

**Create a dedicated system user for Prometheus:**

```bash
sudo useradd --system --no-create-home --shell /bin/false prometheus
```

**Download and extract Prometheus:**

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/
```

**Move binaries and set up directories:**

```bash
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```

**Set correct ownership:**

```bash
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

**Create the systemd service file:**

```bash
sudo vi /etc/systemd/system/prometheus.service
```

Paste the following:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

> **Key configuration notes:**
> - `User` and `Group` run Prometheus with least privilege.
> - `--web.listen-address=0.0.0.0:9090` makes Prometheus accessible on all interfaces.
> - `--web.enable-lifecycle` enables reloading configuration via API without a restart.

**Enable and start Prometheus:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

**Access Prometheus:**

```
http://<monitoring-server-ip>:9090
```

Navigate to **Status → Targets** to verify Prometheus is up.

---

### 11.3 Install Node Exporter

Node Exporter collects hardware and OS-level metrics (CPU, memory, disk, network) from the server and exposes them to Prometheus.

```bash
cd ~
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
```

**Create the systemd service file:**

```bash
sudo vi /etc/systemd/system/node_exporter.service
```

Paste the following:

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

**Enable and start Node Exporter:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

---

### 11.4 Configure Prometheus to Scrape Jenkins and Node Exporter

By default, Prometheus only scrapes itself. You need to add scrape jobs for Node Exporter and Jenkins.

```bash
sudo vi /etc/prometheus/prometheus.yml
```

Append the following jobs at the end of the file:

```yaml
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['<MonitoringServerIP>:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<JenkinsServerIP>:8080']
```

> Replace `<MonitoringServerIP>` and `<JenkinsServerIP>` with the actual IP addresses.

**Validate the configuration:**

```bash
promtool check config /etc/prometheus/prometheus.yml
```

You should see `SUCCESS` if the configuration is valid.

**Reload Prometheus without restarting:**

```bash
curl -X POST http://localhost:9090/-/reload
```

**Verify in Prometheus UI:**

Navigate to `http://<monitoring-server-ip>:9090/targets` — you should see three targets all showing `UP`:
- `prometheus (1/1 up)`
- `node_exporter (1/1 up)`
- `jenkins (1/1 up)`

> **Note:** Ensure port **9100** is open in the Security Group for the Monitoring Server.

---

### 11.5 Install Grafana

Grafana provides rich, interactive dashboards for visualizing the metrics collected by Prometheus.

**Step 1 — Install dependencies:**

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common
```

**Step 2 — Add the Grafana GPG key:**

```bash
cd ~
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://packages.grafana.com/gpg.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
```

**Step 3 — Add the Grafana repository:**

```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main" | \
sudo tee /etc/apt/sources.list.d/grafana.list
```

**Step 4 — Install Grafana:**

```bash
sudo apt-get update
sudo apt-get -y install grafana
```

**Step 5 — Enable and start Grafana:**

```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

**Step 6 — Access Grafana:**

```
http://<monitoring-server-ip>:3000
```

- Default credentials: **admin / admin**
- You will be prompted to set a new password on first login.

**Step 7 — Add Prometheus as a Data Source:**

1. In Grafana, go to **Configuration → Data Sources → Add data source**.
2. Select **Prometheus**.
3. Set the URL to `http://localhost:9090`.
4. Click **Save & Test**.

**Step 8 — Import Dashboards:**

Import pre-built dashboards from [grafana.com/dashboards](https://grafana.com/grafana/dashboards/) for Node Exporter and Jenkins metrics. Navigate to **Dashboards → Import** and enter the dashboard IDs.

---

## 12. EKS Cluster Setup

The Zomato application is also deployed on a Kubernetes cluster managed by AWS EKS.

> **Prerequisites:**
> - `eksctl` installed and configured
> - `kubectl` installed
> - AWS CLI configured with appropriate IAM permissions
> - Run your terminal/VS Code as Administrator on Windows

### Step 1 — Create the EKS Cluster

```bash
eksctl create cluster --name=kartik-cluster \
                      --region=ap-northeast-1 \
                      --zones=ap-northeast-1a,ap-northeast-1c \
                      --without-nodegroup
```

> Cluster creation takes approximately **20–25 minutes**. You can track progress in the **AWS CloudFormation** console — a stack named `kartik-cluster` will appear.

**PowerShell alternative (use backtick for line continuation):**

```powershell
eksctl create cluster --name=kartik-cluster `
                      --region=ap-northeast-1 `
                      --zones=ap-northeast-1a,ap-northeast-1c `
                      --without-nodegroup
```

**Verify cluster creation:**

```bash
eksctl get cluster
```

---

### Step 2 — Associate IAM OIDC Provider

This is required to enable IAM roles for Kubernetes service accounts.

```bash
eksctl utils associate-iam-oidc-provider \
    --region ap-northeast-1 \
    --cluster kartik-cluster \
    --approve
```

---

### Step 3 — Create the Node Group

```bash
eksctl create nodegroup --cluster=kartik-cluster \
                       --region=ap-northeast-1 \
                       --name=kartik-ng-public1 \
                       --node-type=t3.medium \
                       --nodes=2 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=<your-key-pair-name> \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access
```

> Replace `<your-key-pair-name>` with the name of your AWS EC2 key pair.

**Verify nodes:**

```bash
kubectl get nodes
```

---

## 13. ArgoCD Installation

ArgoCD is a declarative GitOps continuous delivery tool for Kubernetes. It automatically syncs your Kubernetes cluster with the desired state defined in your Git repository.

### Step 1 — Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
```

Wait for all pods to reach a `Running` state:

```bash
kubectl get pods -n argocd
```

### Step 2 — Expose the ArgoCD Server

By default, the ArgoCD server is not publicly accessible. Expose it via a LoadBalancer:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

**PowerShell alternative:**

```powershell
kubectl patch svc argocd-server -n argocd -p "{\"spec\": {\"type\": \"LoadBalancer\"}}"
```

Wait approximately 5 minutes for the AWS Load Balancer to be provisioned.

### Step 3 — Get the ArgoCD Server URL

```bash
export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'`
echo $ARGOCD_SERVER
```

**PowerShell alternative:**

```powershell
$env:ARGOCD_SERVER = $(kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname')
echo $env:ARGOCD_SERVER
```

Open the URL in a browser to access the ArgoCD UI.

### Step 4 — Retrieve the Admin Password

```bash
export ARGO_PWD=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
echo $ARGO_PWD
```

**PowerShell alternative:**

```powershell
$env:ARGO_PWD = (kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | % { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) })
echo $env:ARGO_PWD
```

Login with username `admin` and the retrieved password.

### Step 5 — Configure Application in ArgoCD

1. Click **New App** in the ArgoCD UI.
2. Set the **repository URL** to your Git repository.
3. Set the **path** to the `Kubernetes` folder.
4. Set the **destination** to your EKS cluster.
5. Click **Create** and then **Sync**.

> **Important:** In the `Kubernetes/deployment.yml` file, update the Docker image reference to use your DockerHub username.

---

## 14. Monitor Kubernetes with Prometheus

### 14.1 Install Node Exporter on the EKS Cluster

Use Helm to deploy the Prometheus Node Exporter as a DaemonSet across all cluster nodes.

**Add the Prometheus Helm repository:**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

**Create a namespace:**

```bash
kubectl create namespace prometheus-node-exporter
```

**Install Node Exporter:**

```bash
helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter \
  --namespace prometheus-node-exporter
```

### 14.2 Add the Kubernetes Scrape Job to Prometheus

On the Monitoring Server, edit the Prometheus configuration:

```bash
sudo vi /etc/prometheus/prometheus.yml
```

Append the following at the end of the file:

```yaml
  - job_name: 'k8s'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['<NodeIP>:9100']
```

**To find the `<NodeIP>`:**
1. In AWS Console, navigate to **EKS → Clusters → kartik-cluster**.
2. Click on the **Compute** tab → **Nodes**.
3. Click on any node → Click the **Instance ID** link.
4. Copy the **Public IP** address.

**Validate and reload Prometheus:**

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

### 14.3 Access the Application

Once the pipeline is complete and ArgoCD has synced the deployment:

1. Copy the **NodeIP** obtained above.
2. Open in browser: `http://<NodeIP>:30001`
3. Ensure port **30001** is open in the Security Group for the EKS node instances.

> **Note:** If you see errors in Prometheus under the `k8s` job, open port **9100** on the EC2 instances that serve as EKS nodes.

---

## 15. Cleanup

When you are done with the demo, clean up all resources to avoid ongoing AWS charges.

### Delete the Node Group

```bash
# List clusters
eksctl get clusters

# List node groups
eksctl get nodegroup --cluster=kartik-cluster

# Delete node group
eksctl delete nodegroup --cluster=kartik-cluster --name=kartik-ng-public1
```

### Delete the EKS Cluster

```bash
eksctl delete cluster kartik-cluster
```

### Delete CloudFormation Stacks

Go to **AWS Console → CloudFormation** and delete any remaining stacks associated with this project.

> Also terminate the EC2 instances (Application Server and Monitoring Server) and remove any unused Elastic IPs, Load Balancers, and Security Groups.

---

## Architecture Overview

```
Git Repository
      │
      ▼
   Jenkins (CI)
   ├── SonarQube Analysis
   ├── OWASP Dependency Check
   ├── Docker Build & Scout Scan
   └── Push to DockerHub
          │
          ▼
      ArgoCD (CD)
          │
          ▼
    EKS Cluster (Kubernetes)
          │
          ▼
    Application (Port 30001)

Monitoring Stack (Separate VM):
  Prometheus ← Node Exporter (App Server)
  Prometheus ← Jenkins Metrics
  Prometheus ← Node Exporter (EKS Nodes)
  Grafana    ← Prometheus (Dashboards)
```

---

*Guide prepared for the Zomato DevOps CI/CD Pipeline project.*
