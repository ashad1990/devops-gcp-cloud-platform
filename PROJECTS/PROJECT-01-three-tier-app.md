# Project 01 — Three-Tier Web Application on GKE

**Difficulty:** ⭐⭐⭐ Intermediate  
**Estimated Time:** 4–6 hours  
**Estimated Monthly Cost:** $150–200

---

## Architecture

```
Internet
    │
    ▼
┌─────────────────────────────────────────────────┐
│           Cloud HTTP(S) Load Balancer            │
│         (Global, managed SSL via GCP)            │
└────────────────────┬────────────────────────────┘
                     │
         ┌───────────▼────────────┐
         │   GKE Cluster (VPC)    │
         │  ┌──────────────────┐  │
         │  │ Frontend (Nginx)  │  │  ◄── Public Subnet 10.0.1.0/24
         │  │  React SPA served │  │
         │  │  via Nginx        │  │
         │  └────────┬─────────┘  │
         │           │ ClusterIP  │
         │  ┌────────▼─────────┐  │
         │  │  Backend (Node)  │  │  ◄── Private Subnet 10.0.2.0/24
         │  │  REST API :3000  │  │
         │  └────────┬─────────┘  │
         └───────────┼────────────┘
                     │ Cloud SQL Auth Proxy
                     │ (Workload Identity)
         ┌───────────▼────────────┐
         │   Cloud SQL PostgreSQL │
         │   (Private IP, HA)     │
         └────────────────────────┘
                     │
         ┌───────────▼────────────┐
         │      Cloud NAT         │
         │  (Egress for private   │
         │   nodes)               │
         └────────────────────────┘

Supporting Services:
  Artifact Registry ── stores Docker images
  Cloud Build       ── CI/CD builds
  Secret Manager    ── DB passwords
  Cloud Monitoring  ── dashboards & alerts
```

---

## Learning Objectives

- Deploy a production-grade GKE cluster with private nodes
- Connect GKE workloads to Cloud SQL using Workload Identity (no passwords in code)
- Configure HTTP(S) Load Balancing with managed TLS certificates
- Build and push container images with Cloud Build
- Set up Cloud Monitoring dashboards and alerting policies
- Implement VPC network segmentation with Cloud NAT

---

## Prerequisites

- GCP project with billing enabled
- `gcloud` CLI ≥ 450.0 authenticated
- Terraform ≥ 1.5
- `kubectl` and `helm` installed
- Docker installed

```bash
export PROJECT_ID="your-project-id"
export REGION="us-central1"
export ZONE="us-central1-a"
gcloud config set project $PROJECT_ID

gcloud services enable \
  container.googleapis.com \
  sqladmin.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  secretmanager.googleapis.com \
  compute.googleapis.com \
  servicenetworking.googleapis.com
```

---

## Terraform Infrastructure

### `terraform/main.tf`

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
  }
  backend "gcs" {
    bucket = "YOUR_TF_STATE_BUCKET"
    prefix = "project-01/state"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
}
```

### `terraform/variables.tf`

```hcl
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "GCP region"
  type        = string
  default     = "us-central1"
}

variable "zone" {
  description = "GCP zone for zonal resources"
  type        = string
  default     = "us-central1-a"
}

variable "app_name" {
  description = "Application name prefix"
  type        = string
  default     = "three-tier"
}

variable "db_password" {
  description = "Cloud SQL postgres password"
  type        = string
  sensitive   = true
}

variable "gke_node_count" {
  description = "Number of GKE nodes per zone"
  type        = number
  default     = 2
}

variable "gke_machine_type" {
  description = "GKE node machine type"
  type        = string
  default     = "e2-standard-2"
}
```

### `terraform/vpc.tf`

```hcl
resource "google_compute_network" "main" {
  name                    = "${var.app_name}-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "public" {
  name                     = "${var.app_name}-public-subnet"
  ip_cidr_range            = "10.0.1.0/24"
  region                   = var.region
  network                  = google_compute_network.main.id
  private_ip_google_access = true

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.1.0.0/16"
  }

  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.2.0.0/16"
  }
}

resource "google_compute_subnetwork" "private" {
  name                     = "${var.app_name}-private-subnet"
  ip_cidr_range            = "10.0.2.0/24"
  region                   = var.region
  network                  = google_compute_network.main.id
  private_ip_google_access = true
}

# Cloud Router + NAT for outbound internet from private nodes
resource "google_compute_router" "main" {
  name    = "${var.app_name}-router"
  region  = var.region
  network = google_compute_network.main.id
}

resource "google_compute_router_nat" "main" {
  name                               = "${var.app_name}-nat"
  router                             = google_compute_router.main.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}

# Private services access for Cloud SQL
resource "google_compute_global_address" "private_ip_range" {
  name          = "${var.app_name}-private-ip-range"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.main.id
}

resource "google_service_networking_connection" "private_vpc" {
  network                 = google_compute_network.main.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip_range.name]
}
```

### `terraform/gke.tf`

```hcl
# Service account for GKE nodes
resource "google_service_account" "gke_nodes" {
  account_id   = "${var.app_name}-gke-nodes"
  display_name = "GKE Nodes Service Account"
}

resource "google_project_iam_member" "gke_nodes_log_writer" {
  project = var.project_id
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

resource "google_project_iam_member" "gke_nodes_metric_writer" {
  project = var.project_id
  role    = "roles/monitoring.metricWriter"
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

resource "google_project_iam_member" "gke_nodes_artifact_reader" {
  project = var.project_id
  role    = "roles/artifactregistry.reader"
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

resource "google_container_cluster" "main" {
  name     = "${var.app_name}-cluster"
  location = var.region

  # Remove default node pool; we manage our own
  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.main.id
  subnetwork = google_compute_subnetwork.public.id

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  addons_config {
    http_load_balancing {
      disabled = false
    }
    horizontal_pod_autoscaling {
      disabled = false
    }
  }

  release_channel {
    channel = "REGULAR"
  }

  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "0.0.0.0/0"
      display_name = "All (restrict in production)"
    }
  }

  logging_service    = "logging.googleapis.com/kubernetes"
  monitoring_service = "monitoring.googleapis.com/kubernetes"
}

resource "google_container_node_pool" "main" {
  name       = "${var.app_name}-node-pool"
  cluster    = google_container_cluster.main.id
  node_count = var.gke_node_count

  node_config {
    machine_type    = var.gke_machine_type
    service_account = google_service_account.gke_nodes.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]

    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    labels = {
      env = "production"
      app = var.app_name
    }

    shielded_instance_config {
      enable_secure_boot = true
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
```

### `terraform/cloudsql.tf`

```hcl
resource "random_id" "db_suffix" {
  byte_length = 4
}

resource "google_sql_database_instance" "main" {
  name             = "${var.app_name}-postgres-${random_id.db_suffix.hex}"
  database_version = "POSTGRES_15"
  region           = var.region

  depends_on = [google_service_networking_connection.private_vpc]

  settings {
    tier              = "db-g1-small"
    availability_type = "REGIONAL"   # HA with automatic failover

    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.main.id
    }

    database_flags {
      name  = "max_connections"
      value = "100"
    }

    insights_config {
      query_insights_enabled = true
    }
  }

  deletion_protection = false  # Set true for production
}

resource "google_sql_database" "app" {
  name     = "appdb"
  instance = google_sql_database_instance.main.name
}

resource "google_sql_user" "app" {
  name     = "appuser"
  instance = google_sql_database_instance.main.name
  password = var.db_password
}

# Store DB password in Secret Manager
resource "google_secret_manager_secret" "db_password" {
  secret_id = "${var.app_name}-db-password"
  replication {
    auto {}
  }
}

resource "google_secret_manager_secret_version" "db_password" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = var.db_password
}

# Workload Identity for backend service account
resource "google_service_account" "backend" {
  account_id   = "${var.app_name}-backend"
  display_name = "Backend workload identity SA"
}

resource "google_project_iam_member" "backend_cloudsql" {
  project = var.project_id
  role    = "roles/cloudsql.client"
  member  = "serviceAccount:${google_service_account.backend.email}"
}

resource "google_service_account_iam_member" "backend_workload_identity" {
  service_account_id = google_service_account.backend.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[default/backend]"
}
```

### `terraform/artifact_registry.tf`

```hcl
resource "google_artifact_registry_repository" "app" {
  location      = var.region
  repository_id = "${var.app_name}-repo"
  format        = "DOCKER"

  cleanup_policies {
    id     = "keep-last-10"
    action = "KEEP"
    most_recent_versions {
      keep_count = 10
    }
  }
}

resource "google_project_iam_member" "cloudbuild_artifact_writer" {
  project = var.project_id
  role    = "roles/artifactregistry.writer"
  member  = "serviceAccount:${data.google_project.project.number}@cloudbuild.gserviceaccount.com"
}

data "google_project" "project" {
  project_id = var.project_id
}
```

### `terraform/outputs.tf`

```hcl
output "gke_cluster_name" {
  value = google_container_cluster.main.name
}

output "gke_cluster_endpoint" {
  value     = google_container_cluster.main.endpoint
  sensitive = true
}

output "cloud_sql_connection_name" {
  value = google_sql_database_instance.main.connection_name
}

output "artifact_registry_url" {
  value = "${var.region}-docker.pkg.dev/${var.project_id}/${google_artifact_registry_repository.app.repository_id}"
}
```

---

## Kubernetes Manifests

### `k8s/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: three-tier
  labels:
    app: three-tier
```

### `k8s/backend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: three-tier
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      serviceAccountName: backend      # maps to GSA via Workload Identity
      containers:
        - name: backend
          image: REGION-docker.pkg.dev/PROJECT_ID/three-tier-repo/backend:latest
          ports:
            - containerPort: 3000
          env:
            - name: DB_HOST
              value: "127.0.0.1"       # Cloud SQL Auth Proxy sidecar
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: "appdb"
            - name: DB_USER
              value: "appuser"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
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
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 15

        # Cloud SQL Auth Proxy sidecar
        - name: cloud-sql-proxy
          image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.8.0
          args:
            - "--structured-logs"
            - "--port=5432"
            - "PROJECT_ID:REGION:INSTANCE_NAME"
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"
          securityContext:
            runAsNonRoot: true
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend
  namespace: three-tier
  annotations:
    iam.gke.io/gcp-service-account: three-tier-backend@PROJECT_ID.iam.gserviceaccount.com
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: three-tier
spec:
  selector:
    app: backend
  ports:
    - port: 3000
      targetPort: 3000
  type: ClusterIP
```

### `k8s/frontend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: three-tier
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: REGION-docker.pkg.dev/PROJECT_ID/three-tier-repo/frontend:latest
          ports:
            - containerPort: 80
          env:
            - name: REACT_APP_API_URL
              value: "/api"
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: three-tier
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier-ingress
  namespace: three-tier
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "three-tier-ip"
    networking.gke.io/managed-certificates: "three-tier-cert"
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 3000
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

### `k8s/managed-cert.yaml`

```yaml
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: three-tier-cert
  namespace: three-tier
spec:
  domains:
    - app.example.com
```

---

## Cloud Build CI/CD

### `cloudbuild.yaml`

```yaml
steps:
  # Build backend image
  - name: "gcr.io/cloud-builders/docker"
    id: build-backend
    args:
      - build
      - -t
      - "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/backend:${SHORT_SHA}"
      - -t
      - "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/backend:latest"
      - ./backend

  # Build frontend image
  - name: "gcr.io/cloud-builders/docker"
    id: build-frontend
    args:
      - build
      - -t
      - "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/frontend:${SHORT_SHA}"
      - -t
      - "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/frontend:latest"
      - ./frontend

  # Run backend unit tests
  - name: "gcr.io/cloud-builders/docker"
    id: test-backend
    waitFor: ["build-backend"]
    args:
      - run
      - "--rm"
      - "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/backend:${SHORT_SHA}"
      - npm
      - test

  # Push images
  - name: "gcr.io/cloud-builders/docker"
    id: push-backend
    waitFor: ["test-backend"]
    args: ["push", "--all-tags", "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/backend"]

  - name: "gcr.io/cloud-builders/docker"
    id: push-frontend
    waitFor: ["build-frontend"]
    args: ["push", "--all-tags", "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/frontend"]

  # Deploy to GKE
  - name: "gcr.io/cloud-builders/kubectl"
    id: deploy
    waitFor: ["push-backend", "push-frontend"]
    args:
      - set
      - image
      - deployment/backend
      - backend=${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/backend:${SHORT_SHA}
      - -n
      - three-tier
    env:
      - "CLOUDSDK_COMPUTE_REGION=${_REGION}"
      - "CLOUDSDK_CONTAINER_CLUSTER=${_CLUSTER}"

substitutions:
  _REGION: us-central1
  _REPO: three-tier-repo
  _CLUSTER: three-tier-cluster

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: E2_HIGHCPU_8
```

---

## Step-by-Step Implementation

### Step 1 — Provision Infrastructure

```bash
cd terraform/
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your values

terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

### Step 2 — Configure kubectl

```bash
gcloud container clusters get-credentials three-tier-cluster \
  --region us-central1 \
  --project $PROJECT_ID
```

### Step 3 — Create Kubernetes Secret

```bash
kubectl create secret generic db-secret \
  --from-literal=password="$(gcloud secrets versions access latest \
    --secret=three-tier-db-password)" \
  --namespace=three-tier
```

### Step 4 — Reserve Static IP

```bash
gcloud compute addresses create three-tier-ip --global
gcloud compute addresses describe three-tier-ip --global --format="value(address)"
# Point your DNS A record to this IP
```

### Step 5 — Apply Kubernetes Manifests

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/frontend-deployment.yaml
kubectl apply -f k8s/managed-cert.yaml

# Substitute real values first
sed -i "s/PROJECT_ID/$PROJECT_ID/g" k8s/backend-deployment.yaml
sed -i "s/REGION/$REGION/g" k8s/backend-deployment.yaml
```

### Step 6 — Set Up Cloud Build Trigger

```bash
gcloud builds triggers create github \
  --repo-name=your-repo \
  --repo-owner=your-github-org \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml \
  --name=three-tier-main
```

---

## Testing Procedures

```bash
# Check all pods are running
kubectl get pods -n three-tier

# Check ingress
kubectl get ingress -n three-tier

# Test backend health endpoint
BACKEND_POD=$(kubectl get pod -n three-tier -l app=backend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n three-tier $BACKEND_POD -- curl -s localhost:3000/healthz

# Test full stack via load balancer
LB_IP=$(kubectl get ingress three-tier-ingress -n three-tier \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -H "Host: app.example.com" http://$LB_IP/api/healthz
curl -H "Host: app.example.com" http://$LB_IP/

# Load test with hey (install: go install github.com/rakyll/hey@latest)
hey -n 1000 -c 50 http://$LB_IP/api/healthz
```

---

## Monitoring Setup

```bash
# Create uptime check
gcloud monitoring uptime-check-configs create \
  --display-name="Three-Tier API Health" \
  --http-check-path="/api/healthz" \
  --hostname="app.example.com" \
  --period=60

# Create alerting policy for error rate
cat > alert-policy.json <<'EOF'
{
  "displayName": "High Error Rate",
  "conditions": [{
    "displayName": "HTTP 5xx errors",
    "conditionThreshold": {
      "filter": "metric.type=\"loadbalancing.googleapis.com/https/request_count\" AND metric.labels.response_code_class=\"500\"",
      "comparison": "COMPARISON_GT",
      "thresholdValue": 10,
      "duration": "60s",
      "aggregations": [{
        "alignmentPeriod": "60s",
        "perSeriesAligner": "ALIGN_RATE"
      }]
    }
  }],
  "alertStrategy": {
    "notificationRateLimit": {"period": "300s"}
  }
}
EOF
gcloud monitoring policies create --policy-from-file=alert-policy.json
```

---

## Cost Estimation

| Resource | Spec | Est. Monthly |
|----------|------|-------------|
| GKE cluster (2× e2-standard-2) | 2 vCPU, 8GB RAM each | $70 |
| Cloud SQL PostgreSQL HA | db-g1-small, 10GB SSD | $45 |
| HTTP(S) Load Balancer | 1 rule + forwarding | $18 |
| Artifact Registry | 10GB storage | $1 |
| Cloud NAT | ~10GB egress | $5 |
| Cloud Build | 120 free min/day | $5 |
| **Total** | | **~$144–200** |

---

## Cleanup

```bash
# Delete Kubernetes resources
kubectl delete namespace three-tier

# Destroy Terraform infrastructure
cd terraform/
terraform destroy

# Delete Cloud Build trigger
gcloud builds triggers delete three-tier-main

# Delete static IP
gcloud compute addresses delete three-tier-ip --global
```
