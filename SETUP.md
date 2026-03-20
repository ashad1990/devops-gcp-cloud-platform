# GCP Environment Setup Guide

> Complete setup guide for a GCP DevOps development environment. Follow these steps once to be ready for all exercises in this training.

---

## Table of Contents

1. [Create a GCP Account](#1-create-a-gcp-account)
2. [Install gcloud SDK](#2-install-gcloud-sdk)
3. [Configure gcloud CLI](#3-configure-gcloud-cli)
4. [Set Default Project and Region](#4-set-default-project-and-region)
5. [Authentication Setup](#5-authentication-setup)
6. [Install kubectl for GKE](#6-install-kubectl-for-gke)
7. [Install Terraform](#7-install-terraform)
8. [Set Up Billing Alerts](#8-set-up-billing-alerts)
9. [Install Optional Tools](#9-install-optional-tools)
10. [Verify Your Setup](#10-verify-your-setup)

---

## 1. Create a GCP Account

### New Account (Free Tier)

1. Navigate to https://cloud.google.com/free
2. Click **Get started for free**
3. Sign in with or create a Google Account
4. Select your country and agree to terms of service
5. Enter billing information (required, but **you will not be charged** until you manually upgrade)
6. You receive **$300 in free credits** valid for 90 days

### Free Tier Highlights

- **$300 credit** for 90 days — applies to any GCP service
- **Always-free tier** — certain services remain free forever (Cloud Run, Cloud Functions, Cloud Storage up to limits)
- **No auto-charge** — when credits expire, services pause unless you manually upgrade to a paid account

### Existing Account

If you already have a GCP account:

```bash
# Verify you can access GCP
gcloud auth list
gcloud projects list
```

### Create a Training Project

Always use a dedicated project for training to:
- Isolate costs
- Easily clean up when done
- Avoid accidentally affecting production

```bash
# Create a project (IDs must be globally unique — add a timestamp or random suffix)
gcloud projects create devops-training-$(date +%s) \
  --name="DevOps Training"

# Note the Project ID (you'll need it)
gcloud projects list --format="table(projectId,name,projectNumber)"
```

---

## 2. Install gcloud SDK

### Linux (Debian/Ubuntu)

```bash
# Method 1: Package manager (recommended for staying up to date)
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates gnupg curl

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg

echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | \
  sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

sudo apt-get update && sudo apt-get install -y google-cloud-cli

# Method 2: Interactive install script
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
```

### Linux (RHEL/CentOS/Fedora)

```bash
sudo tee -a /etc/yum.repos.d/google-cloud-sdk.repo << 'EOF'
[google-cloud-cli]
name=Google Cloud CLI
baseurl=https://packages.cloud.google.com/yum/repos/cloud-sdk-el8-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

sudo dnf install -y google-cloud-cli
```

### macOS

```bash
# Option 1: Homebrew (recommended)
brew install --cask google-cloud-sdk

# Add to your shell profile (if not done automatically)
echo 'source "$(brew --prefix)/share/google-cloud-sdk/path.bash.inc"' >> ~/.bash_profile
echo 'source "$(brew --prefix)/share/google-cloud-sdk/completion.bash.inc"' >> ~/.bash_profile
source ~/.bash_profile

# Option 2: Download archive
# Download from: https://cloud.google.com/sdk/docs/install#mac
# Extract and run: ./google-cloud-sdk/install.sh
```

### Windows

**Option 1: Windows Installer (Recommended)**
1. Download the installer: https://dl.google.com/dl/cloudsdk/channels/rapid/GoogleCloudSDKInstaller.exe
2. Run the installer and follow prompts
3. Restart your terminal (PowerShell or Command Prompt)

**Option 2: Windows Package Manager (winget)**
```powershell
winget install Google.CloudSDK
# Restart terminal after installation
```

**Option 3: WSL2 (Windows Subsystem for Linux)**
```bash
# In WSL2 terminal, follow the Linux instructions above
# WSL2 provides the best Linux-compatible experience on Windows
```

### Verify Installation

```bash
gcloud version
# Expected output: Google Cloud SDK 480.x.x (or newer)
# gcloud core 2024.xx.xx
# gsutil version: X.XX (if installed)
```

### Update gcloud SDK

```bash
# Update all installed components
gcloud components update

# Install specific components
gcloud components install kubectl terraform-tools beta

# List installed components
gcloud components list
```

---

## 3. Configure gcloud CLI

### Initial Configuration

```bash
# Run the interactive initialization wizard
gcloud init

# The wizard will:
# 1. Log you in (opens browser)
# 2. Let you select or create a project
# 3. Set a default region/zone
```

### Manual Configuration

```bash
# Set active project
gcloud config set project YOUR_PROJECT_ID

# Set default region (compute, run, functions, etc.)
gcloud config set compute/region us-central1
gcloud config set run/region us-central1
gcloud config set functions/region us-central1

# Set default zone for Compute Engine
gcloud config set compute/zone us-central1-a

# View current configuration
gcloud config list

# View a specific property
gcloud config get-value project
gcloud config get-value compute/region
```

### Named Configurations (Profiles)

Manage multiple GCP environments (dev, staging, prod) with named configurations:

```bash
# Create a configuration for each environment
gcloud config configurations create dev
gcloud config set project my-project-dev
gcloud config set compute/region us-central1
gcloud config set account dev-account@company.com

gcloud config configurations create prod
gcloud config set project my-project-prod
gcloud config set compute/region us-central1
gcloud config set account prod-account@company.com

# List all configurations
gcloud config configurations list

# Switch between configurations
gcloud config configurations activate dev
gcloud config configurations activate prod

# Show which configuration is active
gcloud config configurations list --filter="is_active=true"
```

### Shell Completion

Enable tab-completion for `gcloud` commands:

```bash
# Bash
echo 'source "$(gcloud info --format="value(installation.sdk_root)")/completion.bash.inc"' >> ~/.bashrc
source ~/.bashrc

# Zsh
echo 'source "$(gcloud info --format="value(installation.sdk_root)")/completion.zsh.inc"' >> ~/.zshrc
source ~/.zshrc
```

---

## 4. Set Default Project and Region

### Best Practice: Use Environment Variables

In CI/CD pipelines and scripts, use environment variables rather than `gcloud config`:

```bash
# Add to ~/.bashrc or ~/.zshrc for interactive use
export GOOGLE_CLOUD_PROJECT="my-devops-training-project"
export CLOUDSDK_CORE_PROJECT="my-devops-training-project"
export CLOUDSDK_COMPUTE_REGION="us-central1"
export CLOUDSDK_COMPUTE_ZONE="us-central1-a"
export CLOUDSDK_RUN_REGION="us-central1"

# Reload shell
source ~/.bashrc
```

### Region Selection Guide

Choose regions based on:
- **Proximity to users**: Lower latency
- **Cost**: `us-central1` and `us-east1` tend to be cheapest
- **Compliance**: Some regulations require data residency in specific regions
- **Services availability**: Not all services are available in all regions

```bash
# List all available regions
gcloud compute regions list

# Check which services are available in a region
gcloud run regions list
gcloud functions regions list
```

**Recommended training regions:**
| Region | Location | Notes |
|--------|----------|-------|
| `us-central1` | Iowa, USA | Cheapest, most services available |
| `us-east1` | South Carolina, USA | Low cost, second most features |
| `europe-west1` | Belgium | Good for EU-based learners |
| `asia-east1` | Taiwan | Good for APAC-based learners |

---

## 5. Authentication Setup

### Interactive Authentication (Human Users)

```bash
# Log in with your Google Account
gcloud auth login

# Verify the active account
gcloud auth list

# Set up Application Default Credentials (ADC)
# Used by GCP SDKs, Terraform, client libraries
gcloud auth application-default login

# Verify ADC
gcloud auth application-default print-access-token | head -c 50

# Print your current access token (useful for manual API calls)
gcloud auth print-access-token
```

### Service Account Authentication

For CI/CD pipelines and automated processes:

```bash
# Create a service account for your application
gcloud iam service-accounts create devops-training-sa \
  --display-name="DevOps Training Service Account" \
  --project=YOUR_PROJECT_ID

# Grant necessary permissions
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:devops-training-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/editor"

# Create a key (for local development only — prefer Workload Identity in CI)
gcloud iam service-accounts keys create ~/devops-training-key.json \
  --iam-account=devops-training-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com

# Activate the service account
gcloud auth activate-service-account \
  --key-file=~/devops-training-key.json

# Set GOOGLE_APPLICATION_CREDENTIALS for SDKs
export GOOGLE_APPLICATION_CREDENTIALS=~/devops-training-key.json
```

### Workload Identity Federation (For CI/CD — Keyless Auth)

For GitHub Actions, GitLab CI, or other OIDC-capable CI systems:

```bash
# Create a Workload Identity Pool
gcloud iam workload-identity-pools create "github-pool" \
  --project=YOUR_PROJECT_ID \
  --location="global" \
  --display-name="GitHub Actions Pool"

# Create a provider (GitHub OIDC)
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --project=YOUR_PROJECT_ID \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub OIDC Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref" \
  --issuer-uri="https://token.actions.githubusercontent.com"

# Allow GitHub repo to authenticate as the service account
gcloud iam service-accounts add-iam-policy-binding \
  devops-training-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com \
  --project=YOUR_PROJECT_ID \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/YOUR_GITHUB_ORG/YOUR_REPO"
```

---

## 6. Install kubectl for GKE

`kubectl` is the Kubernetes CLI, required for interacting with GKE clusters.

### Install via gcloud (Recommended)

```bash
# Install kubectl component
gcloud components install kubectl

# Verify
kubectl version --client
```

### Install via Package Manager

**Linux (apt)**:
```bash
# Add Kubernetes apt repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubectl
```

**macOS (Homebrew)**:
```bash
brew install kubectl
```

**Windows (winget)**:
```powershell
winget install Kubernetes.kubectl
```

### Configure kubectl for GKE

```bash
# Generate kubeconfig for a GKE cluster
gcloud container clusters get-credentials CLUSTER_NAME \
  --region=us-central1 \
  --project=YOUR_PROJECT_ID

# Verify connection
kubectl cluster-info
kubectl get nodes

# View all configured contexts
kubectl config get-contexts

# Switch between clusters
kubectl config use-context CONTEXT_NAME
```

### Install kubectx and kubens (Optional but Recommended)

```bash
# Quick context and namespace switching
# macOS
brew install kubectx

# Linux
sudo apt-get install kubectx  # or download from GitHub

# Usage
kubectx                    # list contexts
kubectx prod-cluster       # switch to prod
kubens kube-system         # switch namespace
```

---

## 7. Install Terraform

Terraform is the industry-standard IaC tool for GCP (and multi-cloud).

### Linux

```bash
# Method 1: Official HashiCorp repository (recommended)
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update && sudo apt-get install -y terraform

# Verify
terraform version
```

### macOS

```bash
# Homebrew
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Verify
terraform version
```

### Windows

```powershell
# winget
winget install HashiCorp.Terraform

# Or download from: https://www.terraform.io/downloads
# Verify
terraform version
```

### Install tfenv (Terraform Version Manager — Recommended)

Managing multiple Terraform versions across projects:

```bash
# macOS
brew install tfenv

# Linux
git clone --depth=1 https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Install specific version
tfenv install 1.7.0
tfenv use 1.7.0

# List installed versions
tfenv list

# Automatically use version from .terraform-version file
echo "1.7.0" > .terraform-version
```

### Configure Terraform for GCP

```bash
# Create a minimal GCP Terraform configuration
mkdir -p ~/terraform-gcp-lab && cd ~/terraform-gcp-lab

cat > main.tf << 'EOF'
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

variable "project_id" {}
variable "region" { default = "us-central1" }
EOF

# Initialize (downloads Google provider)
terraform init

# Verify provider is downloaded
terraform providers
```

---

## 8. Set Up Billing Alerts

**Do this immediately** — billing alerts protect you from unexpected charges.

### Via GCP Console

1. Go to https://console.cloud.google.com/billing
2. Select your billing account
3. Click **Budgets & alerts**
4. Click **Create budget**
5. Set scope to your training project
6. Set budget amount: $10 (enough for labs with the free tier)
7. Set threshold alerts: 50%, 90%, 100%
8. Configure email notifications (your account email)
9. Click **Finish**

### Via gcloud CLI

```bash
# Find your billing account ID
gcloud billing accounts list

# Create a $10 budget with alerts
gcloud billing budgets create \
  --billing-account=YOUR_BILLING_ACCOUNT_ID \
  --display-name="Training Lab Budget" \
  --budget-amount=10USD \
  --threshold-rule=percent=0.50,basis=CURRENT_SPEND \
  --threshold-rule=percent=0.90,basis=CURRENT_SPEND \
  --threshold-rule=percent=1.00,basis=CURRENT_SPEND
```

### Enable Billing Export to BigQuery

Track detailed cost data for analysis:

```bash
# In GCP Console: Billing → Billing export → BigQuery export
# Enable "Standard usage cost" and "Detailed usage cost" exports
# This lets you run SQL queries against your cost data
```

---

## 9. Install Optional Tools

### Docker

Required for building container images (Cloud Run, GKE):

```bash
# Linux (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
newgrp docker

# macOS
brew install --cask docker

# Verify
docker version
```

### Helm (Kubernetes Package Manager)

```bash
# Linux/macOS
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS (Homebrew)
brew install helm

# Verify
helm version
```

### jq (JSON Processor)

Essential for parsing `gcloud` JSON output:

```bash
# Linux
sudo apt-get install -y jq

# macOS
brew install jq

# Usage with gcloud
gcloud compute instances list --format=json | jq '.[].name'
gcloud projects list --format=json | jq '.[].projectId'
```

### Skaffold (For GKE Development)

```bash
# Linux
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
sudo install skaffold /usr/local/bin/

# macOS
brew install skaffold

# Verify
skaffold version
```

### Cloud SQL Auth Proxy

Required for connecting to Cloud SQL instances:

```bash
# Linux (amd64)
curl -o cloud-sql-proxy \
  https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.11.0/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy
sudo mv cloud-sql-proxy /usr/local/bin/

# macOS
brew install cloud-sql-proxy

# Verify
cloud-sql-proxy --version
```

---

## 10. Verify Your Setup

Run this checklist to confirm your environment is properly configured:

```bash
#!/bin/bash
echo "=== GCP DevOps Environment Check ==="
echo ""

# gcloud SDK
echo -n "gcloud SDK: "
gcloud version --format="value(Google Cloud SDK)" 2>/dev/null || echo "NOT FOUND"

# Authentication
echo -n "Active account: "
gcloud auth list --filter="status=ACTIVE" --format="value(account)" 2>/dev/null || echo "NOT AUTHENTICATED"

# Active project
echo -n "Active project: "
gcloud config get-value project 2>/dev/null || echo "NOT SET"

# Active region
echo -n "Default region: "
gcloud config get-value compute/region 2>/dev/null || echo "NOT SET"

# kubectl
echo -n "kubectl: "
kubectl version --client --short 2>/dev/null | head -1 || echo "NOT FOUND"

# Terraform
echo -n "Terraform: "
terraform version 2>/dev/null | head -1 || echo "NOT FOUND"

# Docker
echo -n "Docker: "
docker version --format "{{.Client.Version}}" 2>/dev/null || echo "NOT FOUND"

# jq
echo -n "jq: "
jq --version 2>/dev/null || echo "NOT FOUND"

echo ""
echo "=== API Status ==="
gcloud services list --enabled --filter="name:(run OR container OR compute OR storage)" \
  --format="table(config.name,config.title)" 2>/dev/null

echo ""
echo "=== Setup Complete ==="
```

Save this as `check-setup.sh`, make it executable (`chmod +x check-setup.sh`), and run it:

```bash
./check-setup.sh
```

---

## Troubleshooting Setup Issues

| Problem | Solution |
|---------|----------|
| `gcloud: command not found` | Run `exec -l $SHELL` or restart terminal; check PATH |
| Authentication expired | Run `gcloud auth login` again |
| `PROJECT_ID not set` | Run `gcloud config set project PROJECT_ID` |
| `API not enabled` | Run `gcloud services enable SERVICE_NAME.googleapis.com` |
| `kubectl: command not found` | Run `gcloud components install kubectl` |
| `Terraform init` fails | Check internet access; verify GCP credentials (`gcloud auth application-default login`) |
| Billing not linked | Link billing at https://console.cloud.google.com/billing/linkedaccount |

For more troubleshooting help, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

---

*Once your environment is set up, proceed to [QUICK-START.md](QUICK-START.md) to deploy your first GCP resources.*
