# LAB-01: GCP Account, Projects, and gcloud CLI

**Topic:** GCP Foundations  
**Estimated Time:** 45 minutes  
**Difficulty:** 🟢 Beginner  
**Cost:** Free (Cloud Shell) or minimal

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Create and configure a GCP project via console and CLI
2. Install and authenticate the gcloud SDK on your local machine
3. Navigate the GCP Console and use Cloud Shell effectively
4. Configure gcloud CLI defaults and named configurations
5. Enable GCP service APIs programmatically using gcloud
6. Understand the GCP resource hierarchy (Organization → Folder → Project → Resource)

---

## Prerequisites

- A Google account (Gmail or Google Workspace)
- A credit card for billing activation (new accounts receive **$300 free credit** — you will not be charged during the free trial)
- A modern web browser for the GCP Console

---

## Lab Overview

This foundational lab walks you through setting up your GCP environment from scratch. You will create a project, install and configure the gcloud CLI, explore Cloud Shell, enable APIs, and familiarize yourself with the GCP Console. Everything in this lab is either free or uses the free tier, making it a safe starting point.

---

## Part 1: GCP Project Setup

### 1.1 Create a Project via the GCP Console

1. Navigate to [https://console.cloud.google.com](https://console.cloud.google.com)
2. Click the project dropdown at the top → **New Project**
3. Enter a project name: `DevOps Lab`
4. Note the auto-generated **Project ID** (e.g., `devops-lab-123456`) — you cannot change this later
5. Select a billing account or create one
6. Click **Create**

### 1.2 Create a Project via CLI

```bash
# Replace with a globally unique project ID (lowercase letters, numbers, hyphens only)
export PROJECT_ID="devops-lab-$(date +%s | tail -c 7)"
echo "Using Project ID: $PROJECT_ID"

# Create the project
gcloud projects create $PROJECT_ID \
  --name="DevOps Lab" \
  --labels=env=lab,owner=devops-student

# Set this project as the active project
gcloud config set project $PROJECT_ID

# List all your projects
gcloud projects list
```

**Expected output:**
```
PROJECT_ID              NAME         PROJECT_NUMBER
devops-lab-3456789      DevOps Lab   123456789012
```

### 1.3 Enable Billing

```bash
# List available billing accounts
gcloud billing accounts list

# Link billing account to project (replace BILLING_ACCOUNT_ID)
# gcloud billing projects link $PROJECT_ID \
#   --billing-account=XXXXXX-XXXXXX-XXXXXX

# Verify billing is enabled
gcloud billing projects describe $PROJECT_ID
```

---

## Part 2: Installing and Configuring gcloud SDK

### 2.1 Install gcloud SDK

**Linux (Debian/Ubuntu):**
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates gnupg

echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] \
  https://packages.cloud.google.com/apt cloud-sdk main" | \
  sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | \
  sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -

sudo apt-get update && sudo apt-get install -y google-cloud-cli
```

**macOS (Homebrew):**
```bash
brew install --cask google-cloud-sdk
```

**Windows:**
Download the installer from: https://cloud.google.com/sdk/docs/install-sdk#windows

### 2.2 Initialize and Authenticate

```bash
# Initialize gcloud (interactive wizard)
gcloud init

# Authenticate your user account
gcloud auth login

# Application Default Credentials (for local development)
gcloud auth application-default login

# Verify authentication
gcloud auth list
```

**Expected output for `gcloud auth list`:**
```
                    Credentialed Accounts
ACTIVE  ACCOUNT
*       your-email@gmail.com

To set the active account, run:
    $ gcloud config set account `ACCOUNT`
```

### 2.3 View Configuration

```bash
# View current configuration
gcloud config list

# Get detailed environment info
gcloud info

# Show current project
gcloud config get-value project

# Show current account
gcloud config get-value account
```

**Expected output for `gcloud config list`:**
```
[core]
account = your-email@gmail.com
project = devops-lab-3456789

[compute]
region = us-central1
zone = us-central1-a
```

---

## Part 3: Cloud Shell

### 3.1 Opening Cloud Shell

1. In the GCP Console, click the **Cloud Shell** icon (terminal icon) in the top-right toolbar
2. A terminal session opens at the bottom of the screen
3. Cloud Shell provides a pre-configured Linux environment with gcloud, kubectl, git, docker, terraform, and more

### 3.2 Cloud Shell Features

- **5 GB persistent home directory** (`/home/$USER`) — persists between sessions
- **Pre-installed tools**: gcloud, kubectl, git, python3, go, java, docker, terraform
- **Web Preview**: Expose a local port (e.g., 8080) and preview in your browser
- **Code Editor**: Click the pencil icon for a browser-based VS Code editor
- **Boost Mode**: Up to 6 vCPU and 12 GB RAM for intensive workloads (enabled in settings)

### 3.3 Configure Default Region and Zone

```bash
# Set default compute region and zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Set default Cloud Run region
gcloud config set run/region us-central1

# Verify settings
gcloud config list --format='table(section,name,value)'
```

### 3.4 Named Configurations

```bash
# Create a named configuration for this lab
gcloud config configurations create devops-lab

# Set values in the named config
gcloud config set project $PROJECT_ID
gcloud config set account your-email@gmail.com
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# List all configurations
gcloud config configurations list

# Switch between configurations
gcloud config configurations activate devops-lab
gcloud config configurations activate default
```

---

## Part 4: Enabling APIs

GCP services must be enabled before use. Enable the APIs you'll need throughout the lab series.

### 4.1 Browse Available APIs

```bash
# List all available APIs (first 20)
gcloud services list --available | head -20

# Search for a specific API
gcloud services list --available --filter="name:container"
```

### 4.2 Enable Required APIs

```bash
export PROJECT_ID=$(gcloud config get-value project)

# Enable individual APIs
gcloud services enable compute.googleapis.com
gcloud services enable container.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable run.googleapis.com
gcloud services enable cloudfunctions.googleapis.com
gcloud services enable sqladmin.googleapis.com
gcloud services enable monitoring.googleapis.com
gcloud services enable logging.googleapis.com
gcloud services enable artifactregistry.googleapis.com
gcloud services enable secretmanager.googleapis.com
gcloud services enable pubsub.googleapis.com
gcloud services enable dns.googleapis.com

# Or enable all at once
gcloud services enable \
  compute.googleapis.com \
  container.googleapis.com \
  cloudbuild.googleapis.com \
  run.googleapis.com \
  cloudfunctions.googleapis.com \
  sqladmin.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com \
  artifactregistry.googleapis.com \
  secretmanager.googleapis.com \
  pubsub.googleapis.com \
  dns.googleapis.com

echo "✅ APIs enabled successfully"
```

### 4.3 Verify Enabled APIs

```bash
# List enabled APIs
gcloud services list --enabled

# Check if a specific API is enabled
gcloud services list --enabled --filter="name:compute.googleapis.com"
```

**Expected output:**
```
NAME                          TITLE
bigquery.googleapis.com       BigQuery API
cloudbuild.googleapis.com     Cloud Build API
compute.googleapis.com        Compute Engine API
container.googleapis.com      Kubernetes Engine API
logging.googleapis.com        Cloud Logging API
monitoring.googleapis.com     Cloud Monitoring API
run.googleapis.com            Cloud Run Admin API
...
```

### 4.4 Disable an API (reference)

```bash
# To disable an API (not needed in this lab)
# gcloud services disable SERVICE_NAME.googleapis.com --force
```

---

## Part 5: GCP Console Navigation

### 5.1 Key Console Sections

Familiarize yourself with these areas of the GCP Console:

| Section | Purpose | Navigation |
|---------|---------|------------|
| **Home Dashboard** | Project overview, resource status | Console home |
| **IAM & Admin** | Users, roles, service accounts, audit logs | `IAM & Admin` menu |
| **Billing** | Cost tracking, budgets, invoices | `Billing` menu |
| **APIs & Services** | Enable/disable APIs, view usage quotas | `APIs & Services` menu |
| **Cloud Shell** | Browser-based terminal | Top-right toolbar icon |
| **Activity Log** | Recent project activities | `Activity` panel on home |
| **Notifications** | System alerts and notifications | Bell icon, top-right |

### 5.2 IAM & Admin

```bash
# View project IAM policy from CLI
gcloud projects get-iam-policy $PROJECT_ID \
  --format='table(bindings.role,bindings.members)'

# List all IAM roles
gcloud iam roles list --filter="name:roles/viewer"
```

### 5.3 Resource Hierarchy

GCP organizes resources in a hierarchy:

```
Organization (e.g., yourdomain.com)
└── Folders (e.g., Production, Development)
    └── Projects (e.g., devops-lab-123456)
        └── Resources (VMs, Buckets, Clusters, etc.)
```

Policies set at a higher level are inherited by lower levels. This allows centralized governance across teams and environments.

---

## Part 6: gcloud Command Patterns

### 6.1 Common Patterns

```bash
# Get help for any command
gcloud help
gcloud compute instances --help
gcloud compute instances create --help

# Format output as JSON, YAML, or table
gcloud projects list --format=json
gcloud projects list --format=yaml
gcloud projects list --format='table(projectId,name,projectNumber)'

# Filter output
gcloud compute instances list --filter="zone:us-central1-a"
gcloud compute instances list --filter="status:RUNNING"

# Sort results
gcloud compute instances list --sort-by=name
gcloud compute instances list --sort-by=~createTime  # descending

# Quiet mode (no prompts)
# gcloud compute instances delete my-vm --quiet
```

### 6.2 Interactive Help

```bash
# List all gcloud components
gcloud components list

# Install additional components
gcloud components install kubectl
gcloud components install beta
gcloud components install alpha

# Update all components
gcloud components update
```

---

## Verification Steps

Run these commands to confirm your setup is correct:

```bash
# 1. Verify gcloud version
gcloud version

# 2. Verify active account and project
gcloud config list

# 3. Verify APIs are enabled
gcloud services list --enabled | grep -E "compute|container|cloudbuild|run"

# 4. Verify project exists
gcloud projects describe $PROJECT_ID
```

**Expected output for `gcloud config list`:**
```
[compute]
region = us-central1
zone = us-central1-a
[core]
account = your-email@gmail.com
project = devops-lab-3456789
[run]
region = us-central1

Your active configuration is: [devops-lab]
```

---

## Cost Estimates

| Resource | Cost |
|----------|------|
| GCP Project | Free |
| API Enablement | Free |
| Cloud Shell | Free (first 50 hours/week) |
| gcloud CLI | Free |

**This lab has no compute costs.** All operations are administrative.

---

## Cleanup

No billable resources were created in this lab. Your project and enabled APIs do not incur charges until you create resources.

```bash
# Optional: Disable APIs you enabled (not strictly necessary)
# gcloud services disable compute.googleapis.com --force

# Optional: Delete the project entirely (WARNING: permanent!)
# gcloud projects delete $PROJECT_ID
```

---

## Summary

You have successfully:
- ✅ Created a GCP project and enabled billing
- ✅ Installed and authenticated the gcloud SDK
- ✅ Configured default region, zone, and named configurations
- ✅ Enabled the service APIs needed for future labs
- ✅ Navigated the GCP Console key sections

**Next Lab:** [LAB-02: Compute Engine Virtual Machines](LAB-02-compute-engine.md)
