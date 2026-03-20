# Project 02: CI/CD Pipeline with Cloud Build and GKE

## Overview
Build a production-grade CI/CD pipeline using GitHub, Cloud Build, Artifact Registry, and GKE with staging and production environments, blue/green and canary deployments, and automated rollback.

**Estimated Cost:** ~$50-100/month  
**Difficulty:** Intermediate  
**Time to Complete:** 3-4 hours

---

## Architecture

```
┌─────────────┐    webhook    ┌──────────────────┐
│   GitHub    │──────────────▶│   Cloud Build    │
│  (source)   │               │  (CI/CD engine)  │
└─────────────┘               └────────┬─────────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │      Build Pipeline          │
                        │  1. Unit Tests               │
                        │  2. Build Docker Image       │
                        │  3. Security Scan            │
                        │  4. Push to Artifact Registry│
                        └──────────────┬──────────────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │     Artifact Registry        │
                        │   (container images)         │
                        └──────────────┬──────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │           Cloud Deploy               │
                    │      (delivery pipeline)             │
                    └─────────────┬──────────┬────────────┘
                                  │          │
                     ┌────────────▼──┐  ┌────▼────────────┐
                     │  GKE Staging  │  │   GKE Prod      │
                     │  (us-central1)│  │  (us-central1)  │
                     │  n1-standard-1│  │  n1-standard-2  │
                     └───────────────┘  └─────────────────┘
                                                │
                              ┌─────────────────▼──────────┐
                              │        Pub/Sub              │
                              │  (build notifications)      │
                              └─────────────────┬──────────┘
                                                │
                                    ┌───────────▼──────────┐
                                    │  Cloud Functions      │
                                    │  (Slack/email alerts) │
                                    └──────────────────────┘
```

---

## Prerequisites

```bash
export PROJECT_ID="your-project-id"
export REGION="us-central1"
export GITHUB_OWNER="your-github-username"
export GITHUB_REPO="your-app-repo"

gcloud config set project $PROJECT_ID

# Enable required APIs
gcloud services enable \
  cloudbuild.googleapis.com \
  container.googleapis.com \
  artifactregistry.googleapis.com \
  clouddeploy.googleapis.com \
  pubsub.googleapis.com \
  secretmanager.googleapis.com
```

---

## Step 1: Terraform Infrastructure

### Directory Structure
```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── modules/
│   ├── artifact-registry/
│   ├── gke/
│   └── cloud-build/
```

### main.tf
```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  backend "gcs" {
    bucket = "YOUR_BUCKET-tfstate"
    prefix = "cicd-pipeline"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Artifact Registry
resource "google_artifact_registry_repository" "app_repo" {
  location      = var.region
  repository_id = "app-images"
  description   = "Docker images for the application"
  format        = "DOCKER"

  cleanup_policies {
    id     = "keep-minimum-versions"
    action = "KEEP"
    most_recent_versions {
      keep_count = 10
    }
  }
}

# GKE Staging Cluster
resource "google_container_cluster" "staging" {
  name     = "staging-cluster"
  location = var.region

  remove_default_node_pool = true
  initial_node_count       = 1

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.staging_subnet.name

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }
}

resource "google_container_node_pool" "staging_nodes" {
  name       = "staging-node-pool"
  location   = var.region
  cluster    = google_container_cluster.staging.name
  node_count = 2

  node_config {
    machine_type = "n1-standard-2"
    disk_size_gb = 50

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }

  autoscaling {
    min_node_count = 1
    max_node_count = 5
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }
}

# GKE Production Cluster
resource "google_container_cluster" "production" {
  name     = "production-cluster"
  location = var.region

  remove_default_node_pool = true
  initial_node_count       = 1

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.prod_subnet.name

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  release_channel {
    channel = "STABLE"
  }
}

resource "google_container_node_pool" "prod_nodes" {
  name     = "prod-node-pool"
  location = var.region
  cluster  = google_container_cluster.production.name

  node_config {
    machine_type = "n1-standard-4"
    disk_size_gb = 100

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }

  autoscaling {
    min_node_count = 2
    max_node_count = 10
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }
}

# Cloud Build Trigger
resource "google_cloudbuild_trigger" "main_trigger" {
  name        = "main-branch-trigger"
  description = "Trigger on push to main branch"

  github {
    owner = var.github_owner
    name  = var.github_repo
    push {
      branch = "^main$"
    }
  }

  filename = "cloudbuild.yaml"

  substitutions = {
    _REGION     = var.region
    _REPOSITORY = google_artifact_registry_repository.app_repo.repository_id
  }
}

# IAM: Cloud Build SA permissions
resource "google_project_iam_member" "cloudbuild_gke" {
  project = var.project_id
  role    = "roles/container.developer"
  member  = "serviceAccount:${data.google_project.project.number}@cloudbuild.gserviceaccount.com"
}

resource "google_project_iam_member" "cloudbuild_ar" {
  project = var.project_id
  role    = "roles/artifactregistry.writer"
  member  = "serviceAccount:${data.google_project.project.number}@cloudbuild.gserviceaccount.com"
}

resource "google_project_iam_member" "cloudbuild_deploy" {
  project = var.project_id
  role    = "roles/clouddeploy.operator"
  member  = "serviceAccount:${data.google_project.project.number}@cloudbuild.gserviceaccount.com"
}

data "google_project" "project" {}
```

### variables.tf
```hcl
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "GCP Region"
  type        = string
  default     = "us-central1"
}

variable "github_owner" {
  description = "GitHub repository owner"
  type        = string
}

variable "github_repo" {
  description = "GitHub repository name"
  type        = string
}
```

---

## Step 2: Cloud Build Pipeline (cloudbuild.yaml)

```yaml
steps:
  # Step 1: Run unit tests
  - name: 'python:3.11-slim'
    id: 'unit-tests'
    entrypoint: bash
    args:
      - '-c'
      - |
        pip install -r requirements.txt --quiet
        python -m pytest tests/unit/ -v --junit-xml=test-results.xml

  # Step 2: Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    id: 'build-image'
    args:
      - 'build'
      - '-t'
      - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/app:$SHORT_SHA'
      - '-t'
      - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/app:latest'
      - '--cache-from'
      - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/app:latest'
      - '.'
    waitFor: ['unit-tests']

  # Step 3: Security scan with Trivy
  - name: 'aquasec/trivy:latest'
    id: 'security-scan'
    args:
      - 'image'
      - '--exit-code'
      - '1'
      - '--severity'
      - 'CRITICAL'
      - '--no-progress'
      - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/app:$SHORT_SHA'
    waitFor: ['build-image']

  # Step 4: Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    id: 'push-image'
    args:
      - 'push'
      - '--all-tags'
      - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/app'
    waitFor: ['security-scan']

  # Step 5: Create release in Cloud Deploy
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'create-release'
    entrypoint: gcloud
    args:
      - 'deploy'
      - 'releases'
      - 'create'
      - 'release-$SHORT_SHA'
      - '--delivery-pipeline=app-delivery'
      - '--region=${_REGION}'
      - '--images=app=${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/app:$SHORT_SHA'
    waitFor: ['push-image']

  # Step 6: Upload test results
  - name: 'gcr.io/cloud-builders/gsutil'
    id: 'upload-results'
    args:
      - 'cp'
      - 'test-results.xml'
      - 'gs://$PROJECT_ID-build-artifacts/$BUILD_ID/test-results.xml'
    waitFor: ['unit-tests']

substitutions:
  _REGION: us-central1
  _REPOSITORY: app-images

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: 'E2_HIGHCPU_8'

timeout: '1200s'
```

---

## Step 3: Cloud Deploy Pipeline

### delivery-pipeline.yaml
```yaml
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: app-delivery
description: Application delivery pipeline
serialPipeline:
  stages:
  - targetId: staging
    profiles: [staging]
  - targetId: production
    profiles: [production]
    strategy:
      canary:
        runtimeConfig:
          kubernetes:
            serviceNetworking:
              service: app-service
              deployment: app-deployment
        canaryDeployment:
          percentages: [25, 50, 75]
          verify: true
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: staging
description: Staging GKE cluster
gke:
  cluster: projects/PROJECT_ID/locations/us-central1/clusters/staging-cluster
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: production
description: Production GKE cluster
requireApproval: true
gke:
  cluster: projects/PROJECT_ID/locations/us-central1/clusters/production-cluster
```

```bash
# Apply delivery pipeline
gcloud deploy apply --file=delivery-pipeline.yaml \
  --region=us-central1 --project=$PROJECT_ID
```

---

## Step 4: Kubernetes Manifests

### k8s/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  namespace: default
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: us-central1-docker.pkg.dev/PROJECT_ID/app-images/app:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
        env:
        - name: ENV
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
---
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-deployment
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Step 5: Blue/Green Deployment

```bash
# Deploy green (new) version alongside blue (current)
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: us-central1-docker.pkg.dev/$PROJECT_ID/app-images/app:NEW_SHA
EOF

# Test green deployment
kubectl wait --for=condition=available deployment/app-deployment-green --timeout=120s

# Switch service to green
kubectl patch service app-service -p '{"spec":{"selector":{"version":"green"}}}'

# Verify traffic shift, then remove blue
kubectl delete deployment app-deployment-blue
```

---

## Step 6: Canary Deployment

```bash
# Deploy canary with 10% of replicas
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment-canary
spec:
  replicas: 1   # 1 of 10 total = 10% traffic
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
      - name: app
        image: us-central1-docker.pkg.dev/$PROJECT_ID/app-images/app:CANARY_SHA
EOF

# Monitor error rate for 15 minutes
watch -n 30 'gcloud monitoring metrics list --filter="metric.type=cloudbuild" 2>/dev/null | head -20'

# Promote: scale canary to 100%, remove stable
kubectl scale deployment app-deployment-canary --replicas=10
kubectl delete deployment app-deployment-stable
```

---

## Step 7: Rollback Procedures

```bash
# Automatic rollback via Cloud Deploy
gcloud deploy rollouts list \
  --delivery-pipeline=app-delivery \
  --release=release-$SHORT_SHA \
  --region=us-central1

# Roll back to previous release
gcloud deploy releases list \
  --delivery-pipeline=app-delivery \
  --region=us-central1 \
  --format="value(name)" | head -2 | tail -1 | \
  xargs -I{} gcloud deploy rollouts create rollback-$(date +%s) \
    --release={} \
    --delivery-pipeline=app-delivery \
    --region=us-central1 \
    --to-target=production

# Emergency kubectl rollback
kubectl rollout undo deployment/app-deployment
kubectl rollout status deployment/app-deployment

# Rollback to specific revision
kubectl rollout undo deployment/app-deployment --to-revision=3
kubectl rollout history deployment/app-deployment
```

---

## Step 8: Build Notifications via Pub/Sub

```bash
# Create Pub/Sub topic for build notifications
gcloud pubsub topics create cloud-builds

# Cloud Build publishes automatically to this topic
# Create subscription for Cloud Function
gcloud pubsub subscriptions create build-notifications-sub \
  --topic=cloud-builds \
  --ack-deadline=60

# Deploy notification Cloud Function
cat > notify_function.py <<'EOF'
import base64
import json
import os
import urllib.request

def notify_slack(event, context):
    """Triggered by Cloud Build Pub/Sub messages."""
    pubsub_message = base64.b64decode(event['data']).decode('utf-8')
    build = json.loads(pubsub_message)

    status = build.get('status', 'UNKNOWN')
    if status not in ['SUCCESS', 'FAILURE', 'TIMEOUT']:
        return

    webhook_url = os.environ['SLACK_WEBHOOK_URL']
    color = '#36a64f' if status == 'SUCCESS' else '#ff0000'

    payload = {
        "attachments": [{
            "color": color,
            "title": f"Build {status}: {build.get('id', 'unknown')[:8]}",
            "text": f"Branch: {build.get('substitutions', {}).get('BRANCH_NAME', 'N/A')}\n"
                    f"Duration: {build.get('timing', {}).get('BUILD', {}).get('endTime', 'N/A')}",
            "footer": "Cloud Build"
        }]
    }

    data = json.dumps(payload).encode('utf-8')
    req = urllib.request.Request(webhook_url, data=data,
                                  headers={'Content-Type': 'application/json'})
    urllib.request.urlopen(req)
EOF

gcloud functions deploy notify-slack \
  --runtime=python311 \
  --trigger-topic=cloud-builds \
  --entry-point=notify_slack \
  --region=us-central1 \
  --set-env-vars=SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

---

## Step 9: Testing the Pipeline End-to-End

```bash
# 1. Connect to clusters
gcloud container clusters get-credentials staging-cluster \
  --region=us-central1 --project=$PROJECT_ID
gcloud container clusters get-credentials production-cluster \
  --region=us-central1 --project=$PROJECT_ID

# 2. Make a code change and push
git checkout -b feature/test-pipeline
echo "# Test change $(date)" >> README.md
git add . && git commit -m "test: trigger CI/CD pipeline"
git push origin feature/test-pipeline
git checkout main && git merge feature/test-pipeline
git push origin main

# 3. Monitor build
BUILD_ID=$(gcloud builds list --limit=1 --format="value(id)")
gcloud builds log $BUILD_ID --stream

# 4. Check Cloud Deploy release
gcloud deploy releases list \
  --delivery-pipeline=app-delivery \
  --region=us-central1

# 5. Verify staging deployment
kubectl config use-context gke_${PROJECT_ID}_us-central1_staging-cluster
kubectl get pods -n default
kubectl rollout status deployment/app-deployment

# 6. Get staging service IP
STAGING_IP=$(kubectl get service app-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$STAGING_IP/healthz

# 7. Approve production deployment
RELEASE=$(gcloud deploy releases list \
  --delivery-pipeline=app-delivery \
  --region=us-central1 \
  --format="value(name)" --limit=1)

gcloud deploy rollouts approve \
  --delivery-pipeline=app-delivery \
  --release=$RELEASE \
  --region=us-central1 \
  --rollout=ROLLOUT_NAME

# 8. Verify production
kubectl config use-context gke_${PROJECT_ID}_us-central1_production-cluster
kubectl get pods -n default
```

---

## Monitoring Setup

```bash
# Cloud Build dashboard: https://console.cloud.google.com/cloud-build/builds

# Create uptime check for application
gcloud monitoring uptime create \
  --display-name="App Health Check" \
  --resource-type="uptime_url" \
  --hostname="$APP_IP" \
  --path="/healthz" \
  --check-interval=60

# Create alerting policy for build failures
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="Cloud Build Failures" \
  --condition-display-name="Build failure rate" \
  --condition-filter='resource.type="build" metric.type="cloudbuild.googleapis.com/build/finished_build_count" metric.labels.status="FAILURE"' \
  --condition-threshold-value=1 \
  --condition-threshold-duration=0s \
  --condition-comparison=COMPARISON_GT

# Deployment frequency metric (DORA)
gcloud logging metrics create deployment_frequency \
  --description="Count of successful production deployments" \
  --log-filter='resource.type="builds" jsonPayload.status="SUCCESS" jsonPayload.substitutions.BRANCH_NAME="main"'
```

---

## Cost Estimate

| Resource | Specs | Monthly Cost |
|----------|-------|-------------|
| GKE Staging | 2x n1-standard-2 | ~$25 |
| GKE Production | 3x n1-standard-4 | ~$50 |
| Cloud Build | ~100 builds × 10min | ~$4 |
| Artifact Registry | 10 GB storage | ~$1 |
| Cloud Deploy | Included | $0 |
| Network egress | ~50 GB | ~$5 |
| **Total** | | **~$85/month** |

---

## Cleanup

```bash
# Delete GKE clusters
gcloud container clusters delete staging-cluster \
  --region=us-central1 --quiet
gcloud container clusters delete production-cluster \
  --region=us-central1 --quiet

# Delete Artifact Registry
gcloud artifacts repositories delete app-images \
  --location=us-central1 --quiet

# Delete Cloud Build triggers
gcloud builds triggers delete main-branch-trigger --quiet

# Delete Cloud Deploy pipeline
gcloud deploy delivery-pipelines delete app-delivery \
  --region=us-central1 --force --quiet

# Destroy Terraform resources
terraform destroy -auto-approve
```
