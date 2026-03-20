# GCP DevOps Tools Reference

> Overview of the core tools in the GCP DevOps ecosystem — what each tool does, when to use it, and how to get started.

---

## Table of Contents

1. [gcloud SDK](#1-gcloud-sdk)
2. [kubectl (GKE)](#2-kubectl-gke)
3. [Terraform](#3-terraform)
4. [Google Cloud Shell](#4-google-cloud-shell)
5. [gsutil](#5-gsutil)
6. [bq (BigQuery CLI)](#6-bq-bigquery-cli)
7. [Cloud Code (IDE Plugins)](#7-cloud-code-ide-plugins)
8. [Skaffold](#8-skaffold)
9. [Cloud Deploy](#9-cloud-deploy)
10. [Cloud SQL Auth Proxy](#10-cloud-sql-auth-proxy)
11. [Helm](#11-helm)
12. [Container Structure Test](#12-container-structure-test)

---

## 1. gcloud SDK

**What it is**: The primary command-line interface for Google Cloud Platform. `gcloud` lets you manage virtually every GCP resource directly from your terminal.

**Install**:
```bash
# Linux/macOS (script)
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# macOS (Homebrew)
brew install --cask google-cloud-sdk

# Debian/Ubuntu (package manager)
sudo apt-get install google-cloud-cli
```

**Key components installed with gcloud SDK**:
| Component | Purpose |
|-----------|---------|
| `gcloud` | Main CLI for all GCP services |
| `gsutil` | Legacy Cloud Storage CLI |
| `bq` | BigQuery CLI |
| `core` | Core SDK libraries |

**Essential commands**:
```bash
# Initial setup
gcloud init                          # Interactive setup wizard
gcloud auth login                    # Authenticate with your Google account
gcloud auth application-default login  # Set up ADC for SDKs

# Configuration
gcloud config set project PROJECT_ID
gcloud config set compute/region us-central1
gcloud config list                   # Show current config

# Component management
gcloud components update             # Update all components
gcloud components install kubectl    # Install kubectl
gcloud components list               # List installed components

# Information
gcloud info                          # System info and config paths
gcloud help                          # General help
gcloud compute --help                # Service-specific help
```

**Tips**:
- Use `gcloud config configurations` to manage multiple GCP environments
- Add `--format=json | jq` to parse output in scripts
- Use `--quiet` flag in automation to skip confirmation prompts
- Set `CLOUDSDK_PYTHON=python3` if you have Python version issues

---

## 2. kubectl (GKE)

**What it is**: The Kubernetes command-line tool. Required for deploying, managing, and debugging workloads on GKE clusters.

**Install**:
```bash
# Via gcloud (recommended — matches cluster versions)
gcloud components install kubectl

# Via package manager (Debian/Ubuntu)
sudo apt-get install -y kubectl

# macOS
brew install kubectl

# Verify
kubectl version --client
```

**Connect to GKE**:
```bash
# Fetch cluster credentials (updates ~/.kube/config)
gcloud container clusters get-credentials CLUSTER_NAME \
  --region=us-central1

# Verify connection
kubectl cluster-info
kubectl get nodes
```

**Everyday kubectl commands**:
```bash
# Cluster and context
kubectl config get-contexts          # List all clusters/contexts
kubectl config use-context CONTEXT   # Switch cluster
kubectl cluster-info                 # Show API server address

# Workloads
kubectl get pods                     # List pods in default namespace
kubectl get pods -n kube-system      # List pods in kube-system namespace
kubectl get pods -A                  # List all pods in all namespaces
kubectl describe pod POD_NAME        # Detailed pod information
kubectl logs POD_NAME                # View pod logs
kubectl logs -f POD_NAME             # Follow logs (tail)
kubectl logs POD_NAME -c CONTAINER   # Logs for specific container
kubectl exec -it POD_NAME -- /bin/bash  # Shell into a container

# Deployments
kubectl apply -f deployment.yaml     # Apply a manifest
kubectl get deployments              # List deployments
kubectl rollout status deployment/my-app  # Watch rollout progress
kubectl rollout history deployment/my-app # View rollout history
kubectl rollout undo deployment/my-app    # Rollback to previous version

# Services
kubectl get services                 # List services
kubectl describe service my-service  # Service details including endpoints
kubectl port-forward svc/my-service 8080:80  # Forward port locally

# Scaling
kubectl scale deployment my-app --replicas=5
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=60

# Cleanup
kubectl delete pod POD_NAME          # Delete a pod
kubectl delete -f deployment.yaml    # Delete resources from manifest
```

**Helpful plugins (krew)**:
```bash
# Install krew (kubectl plugin manager)
kubectl krew install ctx ns tree whoami

# Switch context/namespace quickly
kubectl ctx               # (kubectx) list/switch contexts
kubectl ns                # (kubens) list/switch namespaces
kubectl tree deployment my-app  # Show resource tree
kubectl whoami            # Show current identity
```

---

## 3. Terraform

**What it is**: The industry-standard Infrastructure as Code tool by HashiCorp. Terraform declaratively manages GCP resources using `.tf` files and maintains a state file that tracks what's deployed.

**Install**:
```bash
# Debian/Ubuntu
sudo apt-get install -y terraform

# macOS
brew install hashicorp/tap/terraform

# Version manager (tfenv) — recommended
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
export PATH="$HOME/.tfenv/bin:$PATH"
tfenv install latest
tfenv use latest

# Verify
terraform version
```

**GCP Provider Configuration**:
```hcl
# versions.tf
terraform {
  required_version = ">= 1.6"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
  }
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "env/prod"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

**Core workflow**:
```bash
terraform init          # Initialize provider plugins and backend
terraform validate      # Validate configuration syntax
terraform fmt           # Format .tf files consistently
terraform plan          # Preview changes (dry run)
terraform apply         # Apply changes
terraform destroy       # Destroy all managed resources

# State management
terraform show          # Show current state
terraform state list    # List resources in state
terraform state mv      # Rename a resource in state
terraform import ADDR ID  # Import existing GCP resource into state

# Workspace management (for multiple environments)
terraform workspace new staging
terraform workspace select staging
terraform workspace list
```

**Create remote state bucket**:
```bash
# Create a GCS bucket for Terraform state (do this once)
gcloud storage buckets create gs://my-terraform-state \
  --location=us-central1 \
  --uniform-bucket-level-access
gcloud storage buckets update gs://my-terraform-state --versioning
```

**Useful modules (Google-maintained)**:
```hcl
# GKE cluster module
module "gke" {
  source  = "terraform-google-modules/kubernetes-engine/google"
  version = "~> 30.0"
  project_id = var.project_id
  name       = "my-cluster"
  region     = var.region
  network    = "my-vpc"
  subnetwork = "my-subnet"
}

# Cloud SQL module
module "sql" {
  source  = "GoogleCloudPlatform/sql-db/google//modules/postgresql"
  version = "~> 19.0"
  name         = "my-postgres"
  project_id   = var.project_id
  database_version = "POSTGRES_15"
}
```

---

## 4. Google Cloud Shell

**What it is**: A free, browser-based Linux terminal provided by Google. It runs on a GCE VM and comes pre-installed with `gcloud`, `kubectl`, `terraform`, `docker`, `git`, `vim`, `python`, `go`, `java`, and much more. Your home directory (5 GB) persists across sessions.

**Access**:
- GCP Console → Cloud Shell icon (top right, `>_`)
- Direct URL: https://shell.cloud.google.com
- VS Code-based editor: Click **Open Editor** in Cloud Shell

**Key features**:
```bash
# Cloud Shell is automatically authenticated and project-configured
gcloud auth list        # Already authenticated
gcloud config list      # Project set to your last active project

# Pre-installed tools
gcloud version          # Latest gcloud SDK
kubectl version         # kubectl installed
terraform version       # Terraform installed
docker version          # Docker daemon running
git --version           # Git installed
python3 --version       # Python 3 available
go version              # Go available
java --version          # Java available

# Built-in web preview
# Start a local server on port 8080 and click "Web Preview" → "Preview on port 8080"
python3 -m http.server 8080

# Upload/download files
# Click the three-dot menu → Upload/Download files
# Or use `cloudshell open FILE` to open in the editor
```

**Cloud Shell Boost (fee-based)**:
- Upgrade to a more powerful VM for demanding tasks
- Available when standard shell resources are insufficient

**Tips**:
- Use Cloud Shell for quick experiments without installing anything locally
- The editor is VS Code-based and supports Cloud Code plugin features
- Sessions time out after ~20 min of inactivity; `screen` or `tmux` can help for long-running tasks
- Your `~` directory persists, but `/tmp` and installed packages are reset on session restart

---

## 5. gsutil

**What it is**: The legacy GCP CLI for Cloud Storage. While `gcloud storage` is the modern replacement, `gsutil` remains widely used and is included with the gcloud SDK.

**When to use gsutil vs. gcloud storage**:
- **`gcloud storage`**: Preferred for new scripts; faster, supports parallel transfers natively
- **`gsutil`**: Use when working with legacy scripts or when `gsutil`-specific flags are needed (CORS, lifecycle, ACLs)

**Key gsutil commands**:
```bash
# Bucket operations
gsutil mb -l us-central1 gs://my-bucket/           # Create bucket
gsutil ls                                           # List all buckets
gsutil ls gs://my-bucket/                          # List bucket contents
gsutil du -sh gs://my-bucket/                      # Bucket size
gsutil rb gs://my-bucket/                          # Remove (empty) bucket

# Object operations
gsutil cp file.txt gs://my-bucket/                 # Upload
gsutil cp gs://my-bucket/file.txt .                # Download
gsutil mv gs://my-bucket/old.txt gs://my-bucket/new.txt  # Move/rename
gsutil rm gs://my-bucket/file.txt                  # Delete object
gsutil rm -r gs://my-bucket/prefix/               # Delete all with prefix
gsutil cat gs://my-bucket/file.txt                 # Print object content

# Parallel operations (faster for many files)
gsutil -m cp -r ./dist/ gs://my-bucket/static/    # Parallel upload
gsutil -m rm -r gs://my-bucket/old-release/       # Parallel delete
gsutil -m rsync -r ./dist/ gs://my-bucket/static/ # Rsync-like sync

# ACL and IAM
gsutil iam get gs://my-bucket/                     # Get IAM policy
gsutil iam ch allUsers:objectViewer gs://my-public-bucket/  # Make public

# Metadata
gsutil stat gs://my-bucket/file.txt                # Object metadata
gsutil setmeta -h "Cache-Control:public,max-age=86400" gs://my-bucket/static/**

# CORS configuration
gsutil cors set cors.json gs://my-bucket/
gsutil cors get gs://my-bucket/

# Lifecycle
gsutil lifecycle set lifecycle.json gs://my-bucket/
gsutil lifecycle get gs://my-bucket/

# Versioning
gsutil versioning set on gs://my-bucket/
gsutil ls -a gs://my-bucket/                       # List all versions
```

---

## 6. bq (BigQuery CLI)

**What it is**: The command-line tool for BigQuery. Comes with the gcloud SDK. Use it for running queries, managing datasets and tables, loading data, and extracting results.

**Key bq commands**:
```bash
# Authentication (uses gcloud auth)
bq version

# Dataset operations
bq ls                                       # List datasets
bq mk --dataset --location=US my_project:my_dataset  # Create dataset
bq rm -r -f my_project:old_dataset         # Delete dataset (and tables)

# Table operations
bq ls my_project:my_dataset                # List tables
bq show my_project:my_dataset.my_table     # Show table schema
bq head -n 10 my_project:my_dataset.my_table  # Preview rows

# Queries
bq query --use_legacy_sql=false \
  'SELECT name, COUNT(*) as cnt FROM `my_project.my_dataset.my_table` GROUP BY name LIMIT 10'

# Query with result formatting
bq query --use_legacy_sql=false --format=json \
  'SELECT * FROM `bigquery-public-data.usa_names.usa_1910_current` LIMIT 5'

# Query with destination table
bq query --use_legacy_sql=false \
  --destination_table=my_project:my_dataset.results \
  --replace \
  'SELECT * FROM `my_project.my_dataset.source` WHERE status = "active"'

# Load data
bq load --source_format=CSV my_project:my_dataset.my_table gs://my-bucket/data.csv schema.json
bq load --source_format=NEWLINE_DELIMITED_JSON my_project:my_dataset.my_table \
  gs://my-bucket/data.ndjson

# Export data
bq extract my_project:my_dataset.my_table gs://my-bucket/export-*.csv

# Job management
bq ls --jobs=true --max_results=10         # List recent jobs
bq show --job=true bqjob_ID               # Job details

# Cost estimation (dry run)
bq query --use_legacy_sql=false --dry_run \
  'SELECT * FROM `bigquery-public-data.github_repos.commits` LIMIT 100'
```

**Useful for DevOps**:
```bash
# Query GCP billing export (after setting up billing export to BigQuery)
bq query --use_legacy_sql=false \
  'SELECT service.description, SUM(cost) as total_cost
   FROM `my_project.billing_export.gcp_billing_export_v1_ACCOUNT_ID`
   WHERE DATE(usage_start_time) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
   GROUP BY service.description
   ORDER BY total_cost DESC
   LIMIT 20'

# Query Cloud Audit logs exported to BigQuery
bq query --use_legacy_sql=false \
  'SELECT timestamp, protoPayload.methodName, protoPayload.authenticationInfo.principalEmail
   FROM `my_project.audit_logs.cloudaudit_googleapis_com_activity_*`
   WHERE _TABLE_SUFFIX = FORMAT_DATE("%Y%m%d", CURRENT_DATE())
   ORDER BY timestamp DESC
   LIMIT 50'
```

---

## 7. Cloud Code (IDE Plugins)

**What it is**: Google's IDE extensions for VS Code and JetBrains IDEs (IntelliJ, GoLand, PyCharm, etc.). Cloud Code brings GCP development capabilities directly into your editor: deploy to GKE/Cloud Run, run Kubernetes locally with minikube/Kind, manage secrets, and get intelligent YAML completion.

**Install VS Code extension**:
1. Open VS Code
2. Press `Ctrl+Shift+X` (Extensions)
3. Search for "Cloud Code"
4. Click Install

Or from terminal:
```bash
code --install-extension GoogleCloudTools.cloudcode
```

**Key features**:

**Kubernetes/GKE**:
- Browse clusters, namespaces, pods, services in sidebar
- View pod logs directly in VS Code
- Port-forward with a click
- Sync code changes to running pods (hot reload)
- Integrated Kubernetes YAML validation and autocompletion

**Cloud Run**:
- Deploy directly from VS Code
- Run Cloud Run services locally using Docker
- View service URLs, logs, and revisions

**Secret Manager**:
- Browse and view secrets (redacted) from VS Code
- Create/update secrets from the IDE

**Cloud Shell**:
- Open Cloud Shell terminal directly in VS Code

**YAML Intellisense**:
- Autocompletion for Kubernetes manifests
- Validation against Kubernetes API schemas
- Cloud Build `cloudbuild.yaml` schema support

**JetBrains Plugin**:
- Available for IntelliJ IDEA, PyCharm, GoLand, WebStorm
- Install via: `Settings → Plugins → Marketplace → "Cloud Code"`

---

## 8. Skaffold

**What it is**: A command-line tool by Google for continuous development on Kubernetes. Skaffold watches your source code, automatically rebuilds your Docker image, and deploys the updated image to your Kubernetes cluster — enabling a fast inner development loop.

**Install**:
```bash
# Linux
curl -Lo skaffold \
  https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
sudo install skaffold /usr/local/bin/

# macOS
brew install skaffold

# Verify
skaffold version
```

**Key commands**:
```bash
# Initialize a Skaffold configuration for your project
skaffold init

# Start development mode (watches for changes, auto-rebuild/redeploy)
skaffold dev

# Build and deploy once (like a CI/CD run)
skaffold run

# Build image only (no deploy)
skaffold build

# Deploy only (from pre-built images)
skaffold deploy

# Stream logs from running pods
skaffold run --tail

# Clean up deployed resources
skaffold delete

# Render manifests (output final Kubernetes YAML)
skaffold render
```

**Example `skaffold.yaml`**:
```yaml
apiVersion: skaffold/v4beta6
kind: Config
metadata:
  name: my-app
build:
  artifacts:
    - image: us-central1-docker.pkg.dev/my-project/my-repo/my-app
      docker:
        dockerfile: Dockerfile
  local:
    push: false  # In dev, use local images
deploy:
  kubectl:
    manifests:
      - k8s/*.yaml
  helm:
    releases:
      - name: my-app
        chartPath: ./helm/my-app
        values:
          image.repository: us-central1-docker.pkg.dev/my-project/my-repo/my-app
portForward:
  - resourceType: service
    resourceName: my-app
    port: 8080
    localPort: 8080
```

**Skaffold with GKE**:
```bash
# Target a remote GKE cluster (builds and pushes to Artifact Registry)
skaffold dev \
  --default-repo=us-central1-docker.pkg.dev/my-project/my-repo \
  --kube-context=gke_my-project_us-central1_my-cluster
```

---

## 9. Cloud Deploy

**What it is**: GCP's fully managed continuous delivery service. Cloud Deploy manages the release pipeline — promoting releases through environments (dev → staging → prod) with approvals, canary deployments, and rollback capabilities. It integrates with GKE, Cloud Run, and Anthos.

**Key concepts**:
- **Delivery pipeline**: Defines the promotion sequence (dev → staging → prod)
- **Target**: A deployment environment (a GKE cluster or Cloud Run region)
- **Release**: An immutable artifact ready for deployment
- **Rollout**: The act of deploying a release to a target

**CLI commands**:
```bash
# List delivery pipelines
gcloud deploy delivery-pipelines list --region=us-central1

# Describe a pipeline
gcloud deploy delivery-pipelines describe my-pipeline --region=us-central1

# List releases
gcloud deploy releases list \
  --delivery-pipeline=my-pipeline \
  --region=us-central1

# Create a release
gcloud deploy releases create release-v1-0-0 \
  --delivery-pipeline=my-pipeline \
  --region=us-central1 \
  --images=my-app=us-central1-docker.pkg.dev/my-project/my-repo/my-app:v1.0.0

# Promote a release to the next target
gcloud deploy releases promote \
  --release=release-v1-0-0 \
  --delivery-pipeline=my-pipeline \
  --region=us-central1 \
  --to-target=staging

# Approve a rollout (if required)
gcloud deploy rollouts approve ROLLOUT_NAME \
  --delivery-pipeline=my-pipeline \
  --release=release-v1-0-0 \
  --region=us-central1

# List rollouts for a release
gcloud deploy rollouts list \
  --delivery-pipeline=my-pipeline \
  --release=release-v1-0-0 \
  --region=us-central1

# Rollback a release
gcloud deploy rollouts rollback ROLLOUT_NAME \
  --delivery-pipeline=my-pipeline \
  --release=release-v1-0-0 \
  --region=us-central1
```

**Integration with Cloud Build**:
```yaml
# cloudbuild.yaml — build and trigger Cloud Deploy
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', '$_IMAGE_TAG', '.']

  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '$_IMAGE_TAG']

  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    args:
      - gcloud
      - deploy
      - releases
      - create
      - 'release-$SHORT_SHA'
      - '--delivery-pipeline=my-pipeline'
      - '--region=us-central1'
      - '--images=my-app=$_IMAGE_TAG'
```

---

## 10. Cloud SQL Auth Proxy

**What it is**: A secure proxy that provides IAM-authenticated, TLS-encrypted connections to Cloud SQL instances. Recommended over direct connections — eliminates the need to manage firewall rules or SSL certificates for database access.

**Install**:
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

**Usage**:
```bash
# Start the proxy for a PostgreSQL instance (connects on 127.0.0.1:5432)
cloud-sql-proxy my-project:us-central1:my-postgres --port=5432

# Start proxy in the background
cloud-sql-proxy my-project:us-central1:my-postgres --port=5432 &

# Connect with psql
psql -h 127.0.0.1 -p 5432 -U app_user -d my_database

# Multiple instances on different ports
cloud-sql-proxy \
  my-project:us-central1:postgres-prod=tcp:5432 \
  my-project:us-central1:postgres-staging=tcp:5433 &

# Use Unix socket (better performance, no port conflicts)
mkdir -p /cloudsql
cloud-sql-proxy --unix-socket=/cloudsql my-project:us-central1:my-postgres &
psql -h /cloudsql/my-project:us-central1:my-postgres -U app_user

# In Kubernetes (sidecar pattern)
# Run cloud-sql-proxy as a sidecar container in your Pod spec
```

---

## 11. Helm

**What it is**: The Kubernetes package manager. Helm packages Kubernetes manifests into reusable "charts" — making it easy to install, upgrade, and manage complex applications on GKE.

**Install**:
```bash
# Linux/macOS
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS (Homebrew)
brew install helm

# Verify
helm version
```

**Key commands**:
```bash
# Add popular chart repositories
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Search for charts
helm search repo nginx
helm search hub wordpress

# Install a chart
helm install my-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2

# Install with custom values file
helm install my-app ./helm/my-app \
  -f values.yaml \
  -f values-prod.yaml

# List installed releases
helm list --all-namespaces

# Upgrade a release
helm upgrade my-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --reuse-values

# Rollback to a previous version
helm rollback my-nginx 1

# Uninstall a release
helm uninstall my-nginx --namespace ingress-nginx

# Show chart values
helm show values ingress-nginx/ingress-nginx

# Template (render manifests without installing)
helm template my-app ./helm/my-app -f values.yaml
```

---

## 12. Container Structure Test

**What it is**: A Google-developed tool for testing the structure and content of Docker containers. Validates that your container has the right files, commands, metadata, and file permissions — catches image issues before deployment.

**Install**:
```bash
# Linux
curl -LO https://storage.googleapis.com/container-structure-test/latest/container-structure-test-linux-amd64
chmod +x container-structure-test-linux-amd64
sudo mv container-structure-test-linux-amd64 /usr/local/bin/container-structure-test

# macOS
brew install container-structure-test

# Verify
container-structure-test version
```

**Example test config (`container-test.yaml`)**:
```yaml
schemaVersion: "2.0.0"

commandTests:
  - name: "Python version"
    command: "python3"
    args: ["--version"]
    expectedOutput: ["Python 3.11"]

  - name: "App starts"
    command: "/app/start.sh"
    expectedOutput: ["Server started"]
    exitCode: 0

fileExistenceTests:
  - name: "App config exists"
    path: "/app/config.yaml"
    shouldExist: true

  - name: "Secrets directory absent"
    path: "/app/secrets"
    shouldExist: false

fileContentTests:
  - name: "Correct app version"
    path: "/app/VERSION"
    expectedContents: ["1.2.0"]

metadataTest:
  exposedPorts: ["8080"]
  volumes: []
  cmd: ["/app/server"]
  workdir: "/app"
```

**Run tests**:
```bash
# Test a local image
container-structure-test test \
  --image=my-app:latest \
  --config=container-test.yaml

# Test from Artifact Registry
container-structure-test test \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --config=container-test.yaml

# Integrate with Cloud Build
# In cloudbuild.yaml:
# - name: 'gcr.io/gcp-runtimes/container-structure-test'
#   args: ['test', '--image', 'my-app:$SHORT_SHA', '--config', 'container-test.yaml']
```

---

## Tool Summary

| Tool | Category | Install via | Key Use Case |
|------|---------|------------|-------------|
| gcloud SDK | CLI | Script/package | All GCP management |
| kubectl | K8s CLI | gcloud components | GKE workload management |
| Terraform | IaC | HashiCorp repo | Provisioning GCP resources |
| Cloud Shell | Browser terminal | GCP Console | Zero-install development |
| gsutil | Storage CLI | gcloud SDK | Cloud Storage (legacy) |
| bq | BQ CLI | gcloud SDK | BigQuery queries and load |
| Cloud Code | IDE plugin | VS Code / JetBrains | IDE-integrated GCP dev |
| Skaffold | Dev tool | Binary/brew | Kubernetes inner dev loop |
| Cloud Deploy | CD service | gcloud CLI | Multi-env release pipeline |
| Cloud SQL Proxy | Proxy | Binary/brew | Secure Cloud SQL access |
| Helm | K8s pkg mgr | Script/brew | Kubernetes app packaging |
| Container Structure Test | Testing | Binary/brew | Docker image validation |

---

*For command reference and examples, see [COMMANDS.md](COMMANDS.md). For setup instructions, see [SETUP.md](SETUP.md).*
