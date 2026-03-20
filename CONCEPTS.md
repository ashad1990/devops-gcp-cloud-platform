# GCP Core Concepts — Practical Reference

> 20 essential GCP concepts for DevOps engineers, each with real-world context, practical `gcloud` examples, use cases, best practices, and cost optimization tips.

---

## Table of Contents

1. [GCP Projects & Organization](#1-gcp-projects--organization)
2. [Cloud IAM & Service Accounts](#2-cloud-iam--service-accounts)
3. [Compute Engine](#3-compute-engine)
4. [Google Kubernetes Engine (GKE)](#4-google-kubernetes-engine-gke)
5. [Cloud Run](#5-cloud-run)
6. [Cloud Functions](#6-cloud-functions)
7. [App Engine](#7-app-engine)
8. [Cloud Storage](#8-cloud-storage)
9. [Cloud SQL](#9-cloud-sql)
10. [Cloud Spanner](#10-cloud-spanner)
11. [Cloud Pub/Sub](#11-cloud-pubsub)
12. [Cloud Build](#12-cloud-build)
13. [Artifact Registry & Container Registry](#13-artifact-registry--container-registry)
14. [VPC Networking, Firewall & Load Balancers](#14-vpc-networking-firewall--load-balancers)
15. [Cloud Monitoring](#15-cloud-monitoring)
16. [Cloud Logging](#16-cloud-logging)
17. [Infrastructure as Code (Terraform & Deployment Manager)](#17-infrastructure-as-code)
18. [Secret Manager](#18-secret-manager)
19. [BigQuery for DevOps](#19-bigquery-for-devops)
20. [Cloud Memorystore](#20-cloud-memorystore)

---

## 1. GCP Projects & Organization

### What Is It?

GCP's **resource hierarchy** provides structure and governance for all cloud resources. From top to bottom:

```
Organization  (your-company.com)
  └── Folders  (Engineering, Finance, Marketing)
       └── Projects  (prod-payments, staging-api, dev-sandbox)
            └── Resources  (VMs, buckets, databases, etc.)
```

- **Organization**: The root node, tied to a Google Workspace or Cloud Identity domain. All resources eventually belong to an organization.
- **Folders**: Optional grouping layer for departments, teams, or environments. Policies applied at the folder level inherit down.
- **Projects**: The primary unit for resource isolation, billing, and API enablement. Every resource lives in exactly one project. Each project has a unique Project ID (permanent), a Project Number (immutable numeric ID), and a Display Name (changeable).

### When to Use

- Use **separate projects** per environment (dev/staging/prod) to prevent accidental cross-environment access
- Use **folders** to mirror your organizational structure and apply IAM at scale
- Use **Shared VPC** projects to centralize networking while allowing service projects to connect

### gcloud Examples

```bash
# Create a new project
gcloud projects create my-app-prod-2024 \
  --name="My App Production" \
  --folder=FOLDER_ID

# List all projects accessible to your account
gcloud projects list

# Set the active project for subsequent commands
gcloud config set project my-app-prod-2024

# Describe a project (metadata, labels, state)
gcloud projects describe my-app-prod-2024

# List all enabled APIs for a project
gcloud services list --enabled --project=my-app-prod-2024

# Enable a specific API
gcloud services enable container.googleapis.com \
  --project=my-app-prod-2024

# Add a label to a project
gcloud projects update my-app-prod-2024 \
  --update-labels=env=prod,team=platform

# Get the IAM policy for a project
gcloud projects get-iam-policy my-app-prod-2024

# Move a project to a different folder
gcloud projects move my-app-prod-2024 --folder=NEW_FOLDER_ID

# Delete a project (goes into 30-day recycle bin)
gcloud projects delete my-old-project
```

### Best Practices

- **Never use the default project** for production; always create purpose-specific projects
- **Apply labels** to projects (`env`, `team`, `cost-center`) for billing attribution
- **Enable only required APIs** — reduces attack surface and avoids accidental charges
- **Use separate billing accounts** for production vs. non-production to track costs independently
- **Leverage Organization Policy constraints** to enforce guardrails (e.g., restrict allowed regions, prevent public IPs)

### Cost Optimization

- Projects themselves are free; you pay only for resources within them
- Use **resource hierarchy** to consolidate billing reports by team/product using labels
- Configure **budget alerts** per project to catch runaway spending early

---

## 2. Cloud IAM & Service Accounts

### What Is It?

**Cloud Identity and Access Management (IAM)** controls who (identity) can do what (role) on which resource. GCP uses a **deny-by-default** model — unless a principal has an explicit permission, access is denied.

**Key components:**
- **Principals**: Who is asking. Can be a Google Account, Google Group, Service Account, Google Workspace domain, or `allUsers`/`allAuthenticatedUsers`.
- **Roles**: Collections of permissions. Three types:
  - *Basic roles*: Owner, Editor, Viewer — broad, avoid in production
  - *Predefined roles*: Fine-grained, service-specific (e.g., `roles/storage.objectViewer`)
  - *Custom roles*: Your own combinations of permissions
- **Policies**: Bindings that attach roles to principals on a resource

**Service Accounts** are special identities for applications and VMs, not humans. They authenticate using keys or Workload Identity Federation.

### When to Use

- Use **service accounts** for any application, VM, or pipeline that needs to call GCP APIs
- Use **Workload Identity Federation** for GitHub Actions, GitLab CI, or on-prem workloads — eliminates the need for service account JSON keys
- Apply the **principle of least privilege**: grant the minimum permissions required

### gcloud Examples

```bash
# List all IAM roles available in GCP
gcloud iam roles list --filter="stage=GA"

# Get details of a specific predefined role
gcloud iam roles describe roles/storage.objectAdmin

# Bind a role to a user on a project
gcloud projects add-iam-policy-binding my-project \
  --member="user:alice@example.com" \
  --role="roles/editor"

# Create a service account
gcloud iam service-accounts create my-app-sa \
  --display-name="My Application Service Account" \
  --description="Used by the my-app backend service"

# List service accounts in a project
gcloud iam service-accounts list --project=my-project

# Grant a role to a service account on a project
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Create a service account key (avoid if possible; use Workload Identity instead)
gcloud iam service-accounts keys create ~/sa-key.json \
  --iam-account=my-app-sa@my-project.iam.gserviceaccount.com

# List keys for a service account
gcloud iam service-accounts keys list \
  --iam-account=my-app-sa@my-project.iam.gserviceaccount.com

# Allow a Kubernetes service account to impersonate a GCP service account (Workload Identity)
gcloud iam service-accounts add-iam-policy-binding my-app-sa@my-project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:my-project.svc.id.goog[default/my-ksa]"

# Create a custom IAM role
gcloud iam roles create customStorageReader \
  --project=my-project \
  --title="Custom Storage Reader" \
  --description="Read-only access to specific buckets" \
  --permissions=storage.objects.get,storage.objects.list,storage.buckets.get
```

### Best Practices

- **Never use Owner or Editor** roles in production; use predefined or custom roles
- **Rotate service account keys** regularly, or better — eliminate them with Workload Identity
- **Audit IAM policies** with `gcloud projects get-iam-policy` and review regularly
- **Use IAM Conditions** to restrict access based on time, resource tags, or IP
- **Enable Policy Analyzer** to understand who has access to what

### Cost Optimization

- IAM itself is free; no direct cost for roles or policies
- Unused service account keys are a security risk, not a cost risk — delete them

---

## 3. Compute Engine

### What Is It?

**Compute Engine** is GCP's Infrastructure-as-a-Service (IaaS) offering — fully customizable virtual machines running on Google's global infrastructure. You get full OS control, choice of machine family, persistent storage, and flexible networking.

**Machine families:**
- **General-purpose** (`e2`, `n2`, `n1`): Balanced compute/memory for most workloads
- **Compute-optimized** (`c2`, `c3`): High-frequency CPUs for compute-intensive workloads
- **Memory-optimized** (`m2`, `m3`): Up to 12 TB RAM for SAP HANA, in-memory databases
- **Accelerator-optimized** (`a2`, `g2`): NVIDIA GPUs for ML training and inference

### When to Use

- Applications requiring full OS control or custom kernel settings
- Lift-and-shift migrations from on-premises
- Databases and stateful workloads that don't fit containers
- Build infrastructure when managed services are unavailable

### gcloud Examples

```bash
# Create a basic VM instance
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --tags=http-server

# List all VM instances
gcloud compute instances list

# SSH into a VM instance
gcloud compute ssh my-vm --zone=us-central1-a

# Stop a VM instance
gcloud compute instances stop my-vm --zone=us-central1-a

# Start a VM instance
gcloud compute instances start my-vm --zone=us-central1-a

# Create a VM with a startup script
gcloud compute instances create web-server \
  --zone=us-central1-a \
  --machine-type=e2-small \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata-from-file=startup-script=startup.sh

# Create a preemptible (spot) VM for cost savings
gcloud compute instances create batch-worker \
  --zone=us-central1-a \
  --machine-type=n1-standard-4 \
  --preemptible \
  --image-family=debian-12 \
  --image-project=debian-cloud

# Create a managed instance group (MIG) for autoscaling
gcloud compute instance-groups managed create web-mig \
  --base-instance-name=web \
  --template=web-template \
  --size=3 \
  --zone=us-central1-a

# Configure autoscaling on a MIG
gcloud compute instance-groups managed set-autoscaling web-mig \
  --zone=us-central1-a \
  --max-num-replicas=10 \
  --min-num-replicas=2 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=90

# Create a snapshot of a persistent disk
gcloud compute disks snapshot my-vm \
  --snapshot-names=my-vm-snapshot-$(date +%Y%m%d) \
  --zone=us-central1-a
```

### Best Practices

- Use **Managed Instance Groups** with autoscaling for stateless web/app tiers
- Always use **service accounts** with minimal permissions on VM instances
- Enable **OS Login** instead of managing SSH keys manually
- Use **startup scripts** and **instance templates** for reproducible VM configurations
- Configure **automatic restarts** and **live migration** for high availability
- Tag instances and use firewall rules based on tags, not IP addresses

### Cost Optimization

- **Preemptible/Spot VMs** can save up to 91% for fault-tolerant batch workloads
- **Sustained use discounts** apply automatically for VMs running >25% of a month (up to 30% off)
- **Committed use discounts**: 1-year (37% off) or 3-year (55% off) for stable workloads
- **Right-size VMs** using Recommender — GCP automatically suggests downsizing underutilized instances
- Stop or delete idle instances; persistent disks charge even when VM is stopped

---

## 4. Google Kubernetes Engine (GKE)

### What Is It?

**GKE** is Google's fully managed Kubernetes service. It handles control plane management, node upgrades, auto-repair, autoscaling, and deep GCP integrations (Cloud Load Balancing, Persistent Disk, Filestore, IAM, etc.). GKE invented Kubernetes — it runs the most battle-tested Kubernetes in production.

**Two cluster modes:**
- **Autopilot**: Fully managed — Google manages nodes, scaling, and security. Pay per Pod resource usage. Best for most teams.
- **Standard**: Node management is your responsibility. Full flexibility for custom node configurations, GPUs, Windows nodes, etc.

### When to Use

- Containerized microservices requiring orchestration
- Applications needing horizontal pod autoscaling
- Multi-tenant platforms where workload isolation matters
- Stateful applications (databases, caches) with persistent storage requirements

### gcloud Examples

```bash
# Create an Autopilot cluster (recommended for most use cases)
gcloud container clusters create-auto my-autopilot-cluster \
  --region=us-central1 \
  --project=my-project

# Create a Standard cluster with custom node pool
gcloud container clusters create my-cluster \
  --region=us-central1 \
  --num-nodes=3 \
  --machine-type=e2-standard-4 \
  --disk-size=50 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --workload-pool=my-project.svc.id.goog

# Get credentials for kubectl
gcloud container clusters get-credentials my-cluster \
  --region=us-central1 \
  --project=my-project

# List all GKE clusters
gcloud container clusters list

# Describe a specific cluster
gcloud container clusters describe my-cluster --region=us-central1

# Add a new node pool (e.g., GPU nodes for ML workloads)
gcloud container node-pools create gpu-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --num-nodes=2 \
  --machine-type=n1-standard-8 \
  --accelerator=type=nvidia-tesla-t4,count=1

# Upgrade the GKE control plane
gcloud container clusters upgrade my-cluster \
  --region=us-central1 \
  --master \
  --cluster-version=1.28

# Enable Workload Identity on an existing cluster
gcloud container clusters update my-cluster \
  --region=us-central1 \
  --workload-pool=my-project.svc.id.goog

# Resize a node pool
gcloud container clusters resize my-cluster \
  --node-pool=default-pool \
  --num-nodes=5 \
  --region=us-central1

# Delete a cluster
gcloud container clusters delete my-cluster \
  --region=us-central1 --quiet
```

### Best Practices

- Use **Autopilot** for new clusters — eliminates node management overhead
- Enable **Workload Identity** for pod-level GCP authentication (no service account keys in pods)
- Use **Release Channels** (Rapid, Regular, Stable) for automated upgrades
- Enable **Binary Authorization** to allow only trusted container images
- Configure **PodDisruptionBudgets** for high-availability applications
- Use **Namespace-level resource quotas** to prevent resource starvation

### Cost Optimization

- **Autopilot** charges per Pod (CPU/memory), not per node — more efficient for variable workloads
- **Standard cluster**: Control plane costs ~$0.10/hour; use preemptible nodes for cost savings
- Enable **cluster autoscaler** to scale down when idle
- Use **Spot Pods** in Autopilot for fault-tolerant batch workloads at 60-91% discount
- Delete non-production clusters when not in use

---

## 5. Cloud Run

### What Is It?

**Cloud Run** is GCP's fully managed serverless platform for containerized applications. You bring a Docker container (any language, any runtime), and Cloud Run handles the infrastructure: auto-scaling from zero to thousands of instances, HTTPS, custom domains, traffic splitting, and billing by the millisecond.

**Two execution environments:**
- **Cloud Run Services**: Long-running, request-driven services (web apps, APIs)
- **Cloud Run Jobs**: One-off or scheduled batch tasks (data processing, migrations)

### When to Use

- HTTP/gRPC APIs and microservices that can be containerized
- Event-driven services consuming from Pub/Sub or Eventarc
- Scheduled batch jobs
- Applications with unpredictable or spiky traffic (scales to zero when idle)

### gcloud Examples

```bash
# Deploy a container from Artifact Registry
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --platform=managed \
  --region=us-central1 \
  --allow-unauthenticated \
  --memory=512Mi \
  --cpu=1 \
  --concurrency=80 \
  --max-instances=100

# Deploy with environment variables and secrets
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --region=us-central1 \
  --set-env-vars=DB_HOST=10.0.0.1,APP_ENV=production \
  --set-secrets=DB_PASSWORD=db-password:latest

# List all Cloud Run services
gcloud run services list --region=us-central1

# Describe a service (see current config, URL, etc.)
gcloud run services describe my-service --region=us-central1

# Update traffic splitting (canary deploy — send 10% to new revision)
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-revisions=my-service-v2=10,my-service-v1=90

# Gradually migrate all traffic to the latest revision
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-latest

# Create a Cloud Run Job
gcloud run jobs create data-migration \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/migrator:latest \
  --region=us-central1 \
  --tasks=10 \
  --parallelism=5 \
  --max-retries=3

# Execute a Cloud Run Job
gcloud run jobs execute data-migration --region=us-central1

# View logs for a Cloud Run service
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=my-service" \
  --limit=50 --format="table(timestamp,textPayload)"

# Delete a Cloud Run service
gcloud run services delete my-service --region=us-central1 --quiet
```

### Best Practices

- Configure **minimum instances** to eliminate cold starts for latency-sensitive services
- Use **VPC connector** for Cloud Run services that need to access private resources
- Apply **IAM invoker roles** for service-to-service authentication
- Set appropriate **CPU and memory limits** based on profiling — over-provisioning wastes money
- Use **Cloud Run's concurrency** setting to tune throughput vs. resource usage

### Cost Optimization

- Cloud Run **scales to zero** — no cost when no requests are being served
- Pricing is per 100ms of CPU/memory + per request — extremely cost-effective for low/medium traffic
- Configure **max-instances** to cap costs during traffic spikes
- Use **always-free tier**: 2M requests/month + 360K CPU-seconds + 180K GB-seconds free forever

---

## 6. Cloud Functions

### What Is It?

**Cloud Functions** is GCP's Function-as-a-Service (FaaS) offering. Write individual functions in Python, Node.js, Go, Java, .NET, Ruby, or PHP — deploy them without managing any infrastructure. Functions trigger on HTTP requests, Pub/Sub messages, Cloud Storage events, Firestore changes, and more.

**Two generations:**
- **1st gen**: Original runtime with per-function resource limits
- **2nd gen**: Built on Cloud Run; supports longer timeouts, larger instances, concurrent requests

### When to Use

- Event-driven processing (process a file when uploaded to Cloud Storage)
- Lightweight webhooks and API integrations
- Glue code between GCP services
- Scheduled tasks (triggered by Cloud Scheduler)

### gcloud Examples

```bash
# Deploy an HTTP function (2nd gen, Python)
gcloud functions deploy process-webhook \
  --gen2 \
  --runtime=python311 \
  --region=us-central1 \
  --source=./functions/webhook \
  --entry-point=handle_webhook \
  --trigger-http \
  --allow-unauthenticated

# Deploy a Pub/Sub triggered function
gcloud functions deploy process-orders \
  --gen2 \
  --runtime=nodejs20 \
  --region=us-central1 \
  --source=./functions/orders \
  --entry-point=processOrder \
  --trigger-topic=orders-topic

# Deploy a Cloud Storage triggered function
gcloud functions deploy process-upload \
  --gen2 \
  --runtime=python311 \
  --region=us-central1 \
  --source=./functions/processor \
  --entry-point=on_file_upload \
  --trigger-bucket=my-uploads-bucket

# List all functions
gcloud functions list --region=us-central1

# Invoke an HTTP function manually
gcloud functions call process-webhook \
  --region=us-central1 \
  --data='{"event": "payment", "amount": 100}'

# View function logs
gcloud functions logs read process-webhook \
  --region=us-central1 \
  --limit=50

# Update function memory and timeout
gcloud functions deploy process-webhook \
  --gen2 \
  --runtime=python311 \
  --region=us-central1 \
  --memory=1GiB \
  --timeout=300s \
  --trigger-http

# Delete a function
gcloud functions delete process-webhook \
  --region=us-central1 --quiet
```

### Best Practices

- Use **2nd gen** for new functions — better performance, higher limits, concurrent execution
- **Keep functions small and focused** — one function, one responsibility
- Store configuration in **environment variables** or **Secret Manager**, not code
- Set appropriate **timeout values** — default is 60 seconds, max is 9 minutes (2nd gen: 60 min)
- Use **VPC connector** to access Cloud SQL, Memorystore, or other private resources

### Cost Optimization

- **2 million free invocations/month** — enough for most development and light production use
- Functions scale to zero — zero cost when not in use
- Right-size memory allocation — memory directly affects cost per invocation

---

## 7. App Engine

### What Is It?

**App Engine** is GCP's original Platform-as-a-Service (PaaS) offering. Deploy code in a supported runtime and App Engine handles everything: servers, OS patching, load balancing, autoscaling, and HTTPS. It pioneered many concepts later adopted by Cloud Run.

**Two environments:**
- **Standard**: Sandboxed runtimes for Python, Java, Go, PHP, Node.js, Ruby — starts in milliseconds, scales to zero, strict resource limits
- **Flexible**: Runs Docker containers on Compute Engine VMs — more flexibility, no scale-to-zero

### When to Use

- Traditional web applications without containerization expertise
- Applications that benefit from built-in versioning and traffic splitting
- Projects requiring App Engine-specific features (Cron, Task Queues, Datastore integration)
- Legacy GCP applications before Cloud Run existed

### gcloud Examples

```bash
# Deploy an application (reads app.yaml in current directory)
gcloud app deploy

# Deploy a specific version with no traffic promotion
gcloud app deploy --version=v20240101 --no-promote

# List all deployed versions
gcloud app versions list

# Migrate traffic to a new version
gcloud app services set-traffic default \
  --splits=v20240101=1

# Set a canary split (10% to new version)
gcloud app services set-traffic default \
  --splits=v20240101=0.1,v20231201=0.9

# View application logs
gcloud app logs tail -s default

# Describe the running app
gcloud app describe

# Open the deployed application in a browser
gcloud app browse

# Delete old versions (keep 3 most recent)
gcloud app versions list --format="value(id)" | sort -r | tail -n +4 | \
  xargs gcloud app versions delete --quiet
```

### Best Practices

- **Prefer Cloud Run** for new projects — more flexible, same serverless benefits
- Use **app.yaml** to define runtime, scaling, and environment variables
- Configure **automatic scaling** thresholds based on latency, not just CPU
- Use **warmup requests** to pre-initialize instances before receiving traffic

### Cost Optimization

- Standard Environment can scale to zero (no cost when idle)
- Flexible Environment has minimum 1 VM running — avoid for dev/test
- Delete old versions to avoid idle instance charges

---

## 8. Cloud Storage

### What Is It?

**Cloud Storage** is GCP's unified object storage service — scalable, durable (11 nines), and globally accessible. It stores any amount of unstructured data: backups, media files, build artifacts, static website files, data lake files (Parquet, CSV, Avro for BigQuery), and more.

**Storage classes:**
| Class | Use Case | Minimum Storage | Retrieval Cost |
|-------|----------|----------------|----------------|
| Standard | Frequently accessed | None | None |
| Nearline | Once/month | 30 days | Per-GB retrieval |
| Coldline | Once/quarter | 90 days | Per-GB retrieval |
| Archive | Once/year | 365 days | Per-GB retrieval |

### When to Use

- Static asset hosting (images, JS, CSS)
- Build artifact storage (Container images, JAR files, binaries)
- Data lake for analytics (BigQuery external tables)
- Backup and disaster recovery
- Terraform remote state storage

### gcloud Examples

```bash
# Create a bucket in a specific region
gcloud storage buckets create gs://my-app-assets-prod \
  --location=us-central1 \
  --storage-class=STANDARD \
  --uniform-bucket-level-access

# Upload a file to a bucket
gcloud storage cp ./app.jar gs://my-app-artifacts/releases/v1.2.0/app.jar

# Upload a directory recursively
gcloud storage cp -r ./dist/ gs://my-app-assets-prod/static/

# Download a file from a bucket
gcloud storage cp gs://my-app-artifacts/releases/v1.2.0/app.jar ./app.jar

# List objects in a bucket
gcloud storage ls gs://my-app-artifacts/releases/

# Set a lifecycle policy (delete objects older than 90 days)
cat > lifecycle.json << 'EOF'
{
  "lifecycle": {
    "rule": [{
      "action": {"type": "Delete"},
      "condition": {"age": 90}
    }]
  }
}
EOF
gcloud storage buckets update gs://my-app-artifacts \
  --lifecycle-file=lifecycle.json

# Make a bucket's objects publicly readable
gcloud storage buckets add-iam-policy-binding gs://my-public-site \
  --member=allUsers \
  --role=roles/storage.objectViewer

# Create a signed URL (temporary access to a private object)
gcloud storage sign-url gs://my-private-bucket/secret.pdf \
  --duration=1h \
  --private-key-file=sa-key.json

# Sync a local directory to a bucket (like rsync)
gcloud storage rsync -r ./build gs://my-app-assets-prod/

# Enable versioning on a bucket
gcloud storage buckets update gs://my-important-data \
  --versioning
```

### Best Practices

- Enable **Uniform bucket-level access** — simpler, more secure ACL model
- Use **lifecycle rules** to automatically transition or delete objects
- Configure **Object Versioning** for buckets holding important data
- Use **Customer-managed encryption keys (CMEK)** for compliance requirements
- Enable **Pub/Sub notifications** on buckets to trigger event-driven workflows

### Cost Optimization

- Use **Nearline/Coldline/Archive** for infrequently accessed data — up to 99% cheaper storage
- Set **lifecycle rules** to auto-delete temporary files (build artifacts, logs)
- Enable **requester pays** for buckets shared with external teams
- Use **multi-regional** storage only for truly globally distributed access

---

## 9. Cloud SQL

### What Is It?

**Cloud SQL** is GCP's fully managed relational database service supporting **PostgreSQL**, **MySQL**, and **SQL Server**. GCP handles provisioning, patching, backups, failover, and read replica management. You focus on schema design and queries.

**Key features:**
- Automatic backups with point-in-time recovery
- High availability with automatic failover (synchronous replication to standby)
- Read replicas for horizontal read scaling
- Private IP and VPC peering (recommended over public IP)
- Cloud SQL Auth Proxy for secure, IAM-authenticated connections

### When to Use

- Traditional relational workloads requiring ACID compliance
- Lift-and-shift migrations of MySQL or PostgreSQL workloads
- Applications requiring PostgreSQL extensions (PostGIS, pgcrypto, etc.)
- When you want a managed database without the complexity of Cloud Spanner

### gcloud Examples

```bash
# Create a Cloud SQL PostgreSQL instance
gcloud sql instances create my-postgres \
  --database-version=POSTGRES_15 \
  --tier=db-g1-small \
  --region=us-central1 \
  --storage-type=SSD \
  --storage-size=20 \
  --backup-start-time=02:00 \
  --availability-type=REGIONAL

# List all Cloud SQL instances
gcloud sql instances list

# Create a database
gcloud sql databases create my_app_db \
  --instance=my-postgres

# Create a database user
gcloud sql users create app_user \
  --instance=my-postgres \
  --password=SecurePassword123!

# Connect using the Cloud SQL Auth Proxy (recommended)
# First, download the proxy: https://cloud.google.com/sql/docs/postgres/sql-proxy
cloud-sql-proxy my-project:us-central1:my-postgres --port=5432 &
psql -h 127.0.0.1 -p 5432 -U app_user -d my_app_db

# Create a read replica
gcloud sql instances create my-postgres-replica \
  --master-instance-name=my-postgres \
  --region=us-east1

# Export a database to Cloud Storage
gcloud sql export sql my-postgres \
  gs://my-backups/postgres-$(date +%Y%m%d).sql.gz \
  --database=my_app_db

# Import from Cloud Storage
gcloud sql import sql my-postgres \
  gs://my-backups/postgres-20240101.sql.gz \
  --database=my_app_db

# Promote a read replica to a standalone instance
gcloud sql instances promote-replica my-postgres-replica

# Patch an instance (e.g., resize)
gcloud sql instances patch my-postgres --tier=db-n1-standard-2
```

### Best Practices

- Always use **Private IP** (VPC peering) — never expose Cloud SQL directly to the internet
- Use **Cloud SQL Auth Proxy** for application connections — handles TLS and IAM
- Enable **automated backups** and test restoration procedures regularly
- Configure **High Availability** for production (REGIONAL availability type)
- Use **read replicas** for read-heavy workloads — offload analytics queries
- Set **maintenance windows** during low-traffic periods

### Cost Optimization

- Choose the right **machine tier** — start small, scale up as needed
- Use `db-f1-micro` or `db-g1-small` for development (cheapest tiers)
- **Stop Cloud SQL instances** when not in use (max 7 days before auto-restart)
- Consider **Cloud Spanner** for global workloads — can be cheaper at scale despite higher per-unit cost

---

## 10. Cloud Spanner

### What Is It?

**Cloud Spanner** is GCP's globally distributed, strongly consistent relational database. It combines the consistency guarantees of traditional relational databases with the horizontal scalability of NoSQL — at global scale. Spanner uses TrueTime (atomic clock + GPS) to achieve external consistency.

**Differentiators:**
- **Horizontal scaling**: Add processing units without downtime
- **Global transactions**: ACID transactions across globally distributed nodes
- **99.999% SLA**: Five nines availability for multi-region configurations
- **SQL support**: Standard SQL with extensions

### When to Use

- Global applications requiring strong consistency across regions (financial systems, global inventory)
- Applications outgrowing Cloud SQL's vertical scaling limits
- Workloads where global read/write latency consistency matters
- Systems requiring zero-downtime schema changes

### gcloud Examples

```bash
# Create a Cloud Spanner instance
gcloud spanner instances create my-spanner \
  --config=regional-us-central1 \
  --description="Production Spanner" \
  --nodes=3

# Create a database
gcloud spanner databases create my-app-db \
  --instance=my-spanner

# Execute a DDL statement (create a table)
gcloud spanner databases ddl update my-app-db \
  --instance=my-spanner \
  --ddl="CREATE TABLE Users (
    UserId STRING(36) NOT NULL,
    Name STRING(100),
    Email STRING(200),
    CreatedAt TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp=true)
  ) PRIMARY KEY (UserId)"

# Execute a SQL query
gcloud spanner databases execute-sql my-app-db \
  --instance=my-spanner \
  --sql="SELECT COUNT(*) FROM Users"

# Create a backup
gcloud spanner backups create my-backup \
  --instance=my-spanner \
  --database=my-app-db \
  --expiration-date=2024-12-31

# List databases in a Spanner instance
gcloud spanner databases list --instance=my-spanner

# Scale the instance up
gcloud spanner instances update my-spanner --nodes=6

# Delete a Spanner instance (destructive!)
gcloud spanner instances delete my-spanner --quiet
```

### Best Practices

- Use **interleaved tables** for parent-child relationships to co-locate related data
- Choose **primary keys carefully** — avoid monotonically increasing keys (causes hotspots)
- Use **UUIDs** (or reverse-bit integers) as primary keys for even data distribution
- Start with the **minimum processing units** and scale based on CPU utilization
- Use **stale reads** for analytics queries to reduce latency and cost

### Cost Optimization

- Spanner is expensive: ~$0.90/hour per node (regional) — use only when needed
- Start with **1000 processing units** (1 node) for small workloads
- Use **free trial**: 90 days free, then $0/month for up to 10GB in a single-region config (limited)
- Consider Cloud SQL if global distribution is not required

---

## 11. Cloud Pub/Sub

### What Is It?

**Cloud Pub/Sub** is GCP's fully managed, globally distributed message queue and event streaming service. It decouples producers (publishers) from consumers (subscribers), enabling asynchronous, scalable event-driven architectures.

**Core concepts:**
- **Topic**: The channel through which messages flow. Publishers send to topics.
- **Subscription**: A named resource that represents interest in a topic. Subscribers read from subscriptions.
- **Message**: Up to 10MB of data with optional attributes (key-value pairs)
- **Delivery modes**: Pull (subscriber calls API) or Push (Pub/Sub HTTP POSTs to your endpoint)

### When to Use

- Decoupling microservices (order placed → inventory service, email service, analytics service)
- Event streaming and fan-out (one publisher, many subscribers)
- Buffering traffic spikes before processing (rate limiting)
- Triggering Cloud Functions or Cloud Run services from GCP events

### gcloud Examples

```bash
# Create a topic
gcloud pubsub topics create orders-created

# List all topics
gcloud pubsub topics list

# Create a pull subscription
gcloud pubsub subscriptions create orders-processor \
  --topic=orders-created \
  --ack-deadline=60 \
  --message-retention-duration=7d

# Create a push subscription (Pub/Sub delivers to an HTTP endpoint)
gcloud pubsub subscriptions create orders-webhook \
  --topic=orders-created \
  --push-endpoint=https://my-service.run.app/webhooks/orders \
  --push-auth-service-account=pubsub-sa@my-project.iam.gserviceaccount.com

# Publish a message to a topic
gcloud pubsub topics publish orders-created \
  --message='{"orderId": "abc123", "amount": 99.99}' \
  --attribute=source=api,version=v2

# Pull messages from a subscription
gcloud pubsub subscriptions pull orders-processor \
  --limit=10 \
  --auto-ack

# List all subscriptions
gcloud pubsub subscriptions list

# Get subscription details (including lag/backlog metrics)
gcloud pubsub subscriptions describe orders-processor

# Create a dead-letter topic (for unprocessable messages)
gcloud pubsub topics create orders-deadletter
gcloud pubsub subscriptions modify-message-retention-policy orders-processor \
  --message-retention-duration=7d
gcloud pubsub subscriptions update orders-processor \
  --dead-letter-topic=orders-deadletter \
  --max-delivery-attempts=5

# Delete a subscription
gcloud pubsub subscriptions delete orders-processor
```

### Best Practices

- Always set **acknowledgment deadlines** longer than your processing time
- Use **dead-letter topics** for handling poison messages (permanently failing messages)
- Implement **idempotent message processing** — Pub/Sub guarantees at-least-once delivery
- Use **message ordering keys** when ordering matters within a partition
- Enable **message expiration** to prevent backlogs from accumulating indefinitely

### Cost Optimization

- First **10 GB/month** is free; afterward ~$0.04-0.10/GB depending on region
- Use **Pull subscriptions** for batch processing (more efficient than Push for high volume)
- Clean up unused subscriptions — orphaned subscriptions accumulate message storage costs

---

## 12. Cloud Build

### What Is It?

**Cloud Build** is GCP's fully managed CI/CD platform. It executes your build steps as Docker containers — every step is an isolated container running a command. Cloud Build integrates natively with GitHub, GitLab, Bitbucket, Cloud Source Repositories, Artifact Registry, GKE, and Cloud Run.

**Key concepts:**
- **Build config** (`cloudbuild.yaml`): Defines the steps in your pipeline
- **Build trigger**: Automatically starts a build on code push, PR, or schedule
- **Build steps**: Individual jobs that run sequentially or in parallel
- **Substitutions**: Variables injected into builds (like `$BRANCH_NAME`, `$SHORT_SHA`)

### When to Use

- Building and testing code on every push to a repository
- Building Docker images and pushing to Artifact Registry
- Deploying to GKE, Cloud Run, App Engine, or Compute Engine
- Running security scans, linters, and integration tests

### gcloud Examples

```bash
# Submit a build manually (using cloudbuild.yaml in current directory)
gcloud builds submit --config=cloudbuild.yaml .

# Submit a quick build (just build a Docker image)
gcloud builds submit \
  --tag=us-central1-docker.pkg.dev/my-project/my-repo/my-app:v1.0 .

# List recent builds
gcloud builds list --limit=20

# Describe a specific build
gcloud builds describe BUILD_ID

# Stream logs for a running build
gcloud builds log BUILD_ID --stream

# Create a trigger on a GitHub repo (push to main)
gcloud builds triggers create github \
  --repo-name=my-repo \
  --repo-owner=my-org \
  --branch-pattern=^main$ \
  --build-config=cloudbuild.yaml \
  --name=deploy-on-main-push

# List all build triggers
gcloud builds triggers list

# Run a trigger manually
gcloud builds triggers run deploy-on-main-push \
  --branch=main

# Cancel a running build
gcloud builds cancel BUILD_ID

# Delete a trigger
gcloud builds triggers delete deploy-on-main-push
```

**Example `cloudbuild.yaml`:**
```yaml
steps:
  # Step 1: Run tests
  - name: 'python:3.11'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        pip install -r requirements.txt
        python -m pytest tests/ -v

  # Step 2: Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
      - '.'

  # Step 3: Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'

  # Step 4: Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    args:
      - 'gcloud'
      - 'run'
      - 'deploy'
      - 'my-service'
      - '--image=us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
      - '--region=us-central1'
      - '--platform=managed'

images:
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: 'E2_HIGHCPU_8'
```

### Best Practices

- Use **parallel steps** (`waitFor: ['-']`) to speed up builds
- Store build artifacts in **Artifact Registry**, not Cloud Storage
- Use **Cloud Build service account** with minimal permissions
- Enable **build caching** for Docker layers to speed up repeated builds
- Use **substitutions** for environment-specific configuration

### Cost Optimization

- First **120 build-minutes/day** are free
- Use `E2_HIGHCPU_8` for CPU-intensive builds — faster = fewer minutes = lower cost
- Cache Docker layers and pip/npm packages between builds
- Clean up old build history (Cloud Build retains logs for 90 days by default)

---

## 13. Artifact Registry & Container Registry

### What Is It?

**Artifact Registry** is GCP's managed repository service for storing Docker images, Helm charts, Maven JARs, npm packages, Python wheels, and more. It is the successor to Container Registry (`gcr.io`) and the recommended choice for all new projects.

**Advantages over Container Registry:**
- Regional repositories for better latency and data residency control
- Fine-grained IAM at the repository level
- Multi-format support (not just Docker)
- Vulnerability scanning with Container Analysis
- Virtual/Remote repositories for upstream proxy caching

### When to Use

- Storing Docker images built by Cloud Build before deploying to GKE or Cloud Run
- Managing versioned build artifacts (JAR, wheel) in a central registry
- Proxying public registries (Docker Hub, PyPI) to avoid rate limits and improve reliability

### gcloud Examples

```bash
# Create a Docker repository
gcloud artifacts repositories create my-docker-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Production Docker images"

# Configure Docker auth for Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev

# Build and push a Docker image
docker build -t us-central1-docker.pkg.dev/my-project/my-docker-repo/my-app:v1.0 .
docker push us-central1-docker.pkg.dev/my-project/my-docker-repo/my-app:v1.0

# List repositories
gcloud artifacts repositories list --location=us-central1

# List images in a repository
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/my-project/my-docker-repo

# List image tags
gcloud artifacts docker tags list \
  us-central1-docker.pkg.dev/my-project/my-docker-repo/my-app

# Add a tag to an image
gcloud artifacts docker tags add \
  us-central1-docker.pkg.dev/my-project/my-docker-repo/my-app:v1.0 \
  us-central1-docker.pkg.dev/my-project/my-docker-repo/my-app:latest

# Run vulnerability scanning on an image
gcloud artifacts docker images scan \
  us-central1-docker.pkg.dev/my-project/my-docker-repo/my-app:v1.0

# List vulnerabilities for an image
gcloud artifacts docker images list-vulnerabilities \
  us-central1-docker.pkg.dev/my-project/my-docker-repo/my-app:v1.0

# Set up a cleanup policy (delete untagged images older than 30 days)
gcloud artifacts repositories set-cleanup-policies my-docker-repo \
  --location=us-central1 \
  --policy=cleanup-policy.json
```

### Best Practices

- Use **regional repositories** close to your compute resources for faster pulls
- Enable **Container Analysis** (vulnerability scanning) on all repositories
- Set **cleanup policies** to automatically remove old untagged images
- Grant **roles/artifactregistry.reader** to deploy service accounts (not writer/admin)
- Use **image digest** (not tag) references in production Kubernetes manifests for immutability

### Cost Optimization

- First **500 MB/month** of storage is free
- Set cleanup policies aggressively for build artifacts — Docker layers accumulate quickly
- Use **multi-stage Docker builds** to minimize final image size

---

## 14. VPC Networking, Firewall & Load Balancers

### What Is It?

**Virtual Private Cloud (VPC)** is GCP's global software-defined network. Unlike AWS VPCs (which are regional), a GCP VPC is a single global resource, with **subnets** being regional. This means a single VPC can span all regions with consistent private IP routing.

**Key components:**
- **VPC**: Global network with configurable IP ranges
- **Subnets**: Regional IP ranges within a VPC (can be expanded without restart)
- **Firewall rules**: Stateful ingress/egress rules applied to VMs via network tags or service accounts
- **Cloud Load Balancing**: Global HTTP(S), TCP, UDP load balancers with Anycast IPs
- **Cloud NAT**: Outbound internet access for VMs without public IPs
- **VPC Peering**: Private connectivity between two VPC networks
- **Shared VPC**: Centralized networking for multi-project architectures

### gcloud Examples

```bash
# Create a custom VPC network
gcloud compute networks create my-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

# Create a subnet within the VPC
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24 \
  --enable-private-ip-google-access

# Create a firewall rule to allow HTTP/HTTPS from anywhere
gcloud compute firewall-rules create allow-http-https \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server

# Create a firewall rule to allow SSH from a specific IP range
gcloud compute firewall-rules create allow-ssh-from-office \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=203.0.113.0/24 \
  --target-tags=ssh-allowed

# List firewall rules
gcloud compute firewall-rules list --filter="network=my-vpc"

# Create a Cloud NAT gateway (for outbound internet from private VMs)
gcloud compute routers create my-router \
  --region=us-central1 \
  --network=my-vpc
gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges

# Create a global HTTP(S) load balancer (simplified)
# Step 1: Create a backend service
gcloud compute backend-services create my-backend \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=my-health-check \
  --global

# Step 2: Add an instance group to the backend
gcloud compute backend-services add-backend my-backend \
  --instance-group=web-mig \
  --instance-group-zone=us-central1-a \
  --global

# List VPC networks
gcloud compute networks list

# List subnets
gcloud compute networks subnets list --filter="region=us-central1"

# Describe a subnet (shows secondary ranges, etc.)
gcloud compute networks subnets describe my-subnet --region=us-central1
```

### Best Practices

- Never use the **default VPC** in production — create a custom VPC with explicit subnets
- Use **private IPs** for all internal service-to-service communication
- Apply **least-privilege firewall rules** using service account targets (not network tags for production)
- Enable **VPC Flow Logs** for network monitoring and security auditing
- Use **Shared VPC** for multi-project architectures to centralize network management
- Configure **Private Google Access** on subnets so VMs without external IPs can reach GCP APIs

### Cost Optimization

- **Ingress is free**; egress to internet has charges depending on destination
- Use **regional load balancers** instead of global when traffic is regionally concentrated
- Cloud NAT charges per Gateway hour + per GB processed — consider direct external IPs for VMs that legitimately need internet access

---

## 15. Cloud Monitoring

### What Is It?

**Cloud Monitoring** (formerly Stackdriver Monitoring) provides full-stack observability for GCP, AWS, and on-premises resources. It collects metrics, creates dashboards, and fires alerts — with native integration across all GCP services.

**Key components:**
- **Metrics**: Time-series data (CPU, memory, request rates, latency, custom metrics)
- **Dashboards**: Visual panels with charts for metrics
- **Alerting policies**: Trigger notifications when metrics cross thresholds
- **Uptime checks**: Synthetic monitoring — test that URLs are responding correctly
- **Workspaces**: Multi-project monitoring aggregation

### gcloud Examples

```bash
# List available metric descriptors (search for GCE metrics)
gcloud monitoring metric-descriptors list \
  --filter="metric.type:compute.googleapis.com"

# List uptime checks
gcloud monitoring uptime list

# Create an uptime check
gcloud monitoring uptime create \
  --display-name="Production API Health" \
  --http-check-path="/health" \
  --hostname=api.myapp.com \
  --port=443 \
  --use-ssl \
  --period=60

# List notification channels
gcloud monitoring channels list

# Create an email notification channel
gcloud monitoring channels create \
  --display-name="Ops Team Email" \
  --type=email \
  --channel-labels=email_address=ops@mycompany.com

# List alerting policies
gcloud monitoring policies list

# Describe an alerting policy
gcloud monitoring policies describe POLICY_ID

# Create a simple alerting policy (via JSON — complex policies are easier via Console or Terraform)
cat > alert-policy.json << 'EOF'
{
  "displayName": "High CPU Alert",
  "conditions": [{
    "displayName": "CPU > 90%",
    "conditionThreshold": {
      "filter": "resource.type=\"gce_instance\" AND metric.type=\"compute.googleapis.com/instance/cpu/utilization\"",
      "comparison": "COMPARISON_GT",
      "thresholdValue": 0.9,
      "duration": "300s"
    }
  }],
  "alertStrategy": {"autoClose": "86400s"},
  "combiner": "OR",
  "enabled": true,
  "notificationChannels": ["projects/my-project/notificationChannels/CHANNEL_ID"]
}
EOF
gcloud monitoring policies create --policy-from-file=alert-policy.json
```

### Best Practices

- Use **Workspace-level monitoring** to aggregate metrics from multiple projects
- Create **Service Level Objectives (SLOs)** and alert on error budget burn rate, not just raw metrics
- Use **log-based metrics** to create custom metrics from structured log entries
- Set **alert notification channels** to PagerDuty or Slack for actionable on-call notifications
- Review the **Metrics Explorer** regularly for anomaly detection

### Cost Optimization

- First **150 MB of metrics ingestion/month** is free
- Custom metrics: ~$0.01 per metric-sample after the free tier
- Uptime checks: First 3 million check-requests/month free

---

## 16. Cloud Logging

### What Is It?

**Cloud Logging** (formerly Stackdriver Logging) is GCP's managed log aggregation and analysis service. All GCP services automatically send logs to Cloud Logging. It supports structured logging (JSON), log-based metrics, real-time log tailing, log routing to BigQuery/Cloud Storage/Pub/Sub, and advanced querying with the Logging Query Language (LQL).

**Log types:**
- **Admin Activity audit logs**: Who did what (API calls that modify resources) — always on, free
- **Data Access audit logs**: Who accessed what data — off by default, can be costly
- **System Event audit logs**: GCP-initiated operations — always on, free
- **Platform logs**: Service-emitted logs (App Engine, Cloud Functions, etc.)
- **User-written logs**: Application logs sent via the Cloud Logging API or client libraries

### gcloud Examples

```bash
# Read recent logs from a project
gcloud logging read "severity>=ERROR" \
  --limit=50 \
  --format="table(timestamp,severity,textPayload)"

# Read logs for a specific resource type
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="my-service"' \
  --limit=100 \
  --freshness=1h

# Read structured logs as JSON
gcloud logging read 'resource.type="gce_instance"' \
  --limit=10 \
  --format=json

# List available log names
gcloud logging logs list

# Write a log entry (for testing or scripting)
gcloud logging write my-app-log \
  '{"severity":"INFO","message":"Deployment started","version":"v1.2.0"}' \
  --payload-type=json

# Create a log sink (export logs to BigQuery)
gcloud logging sinks create bq-audit-sink \
  bigquery.googleapis.com/projects/my-project/datasets/audit_logs \
  --log-filter='logName:"cloudaudit.googleapis.com"'

# Create a log sink (export logs to Cloud Storage)
gcloud logging sinks create gcs-app-sink \
  storage.googleapis.com/my-logs-bucket \
  --log-filter='resource.type="cloud_run_revision"'

# List all log sinks
gcloud logging sinks list

# Update a log sink's filter
gcloud logging sinks update bq-audit-sink \
  --log-filter='logName:"cloudaudit.googleapis.com" AND severity>=WARNING'

# Delete a log sink
gcloud logging sinks delete old-sink
```

### Best Practices

- Use **structured logging** (JSON) — enables querying by field, not just text search
- Route logs to **BigQuery** for long-term retention and SQL-based analysis
- Create **log-based metrics** for application errors/warnings to trigger alerts
- Enable **Data Access audit logs** selectively — they can generate significant volume/cost
- Set **log exclusions** to drop noisy, low-value logs before they incur ingestion costs

### Cost Optimization

- First **50 GB of log ingestion/month** is free
- Enable **log exclusions** to drop DEBUG-level application logs in production
- Set retention policies — logs retained beyond 30 days (default for most) incur storage charges
- Route high-volume logs directly to Cloud Storage (cheaper than Logging storage for cold data)

---

## 17. Infrastructure as Code

### What Is It?

**Infrastructure as Code (IaC)** is the practice of defining and managing infrastructure through declarative configuration files — enabling version control, reproducibility, code review, and automated testing of infrastructure changes.

**GCP IaC options:**
- **Terraform** (recommended): HashiCorp's open-source tool with excellent GCP provider support. Declarative, state-based, provider ecosystem.
- **Deployment Manager**: Google's native IaC service (YAML/Python/Jinja). Integrated but less popular; Terraform is preferred for most teams.
- **Config Connector**: Kubernetes-native way to manage GCP resources using Kubernetes CRDs.
- **Pulumi**: Programmatic IaC using real programming languages (Python, TypeScript, Go).

### Terraform on GCP Examples

```bash
# Initialize a Terraform working directory
terraform init

# Preview changes without applying
terraform plan -var-file=prod.tfvars

# Apply infrastructure changes
terraform apply -var-file=prod.tfvars

# Destroy all managed infrastructure (use with caution!)
terraform destroy -var-file=prod.tfvars

# Show current state
terraform show

# List resources in state
terraform state list

# Import an existing GCP resource into Terraform state
terraform import google_compute_instance.web \
  projects/my-project/zones/us-central1-a/instances/my-vm

# Validate configuration files
terraform validate

# Format all .tf files
terraform fmt -recursive
```

**Example `main.tf` for Cloud Run + Artifact Registry:**
```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  backend "gcs" {
    bucket = "my-terraform-state-bucket"
    prefix = "prod/cloud-run"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_artifact_registry_repository" "app_repo" {
  location      = var.region
  repository_id = "app-images"
  format        = "DOCKER"
}

resource "google_cloud_run_v2_service" "app" {
  name     = "my-app"
  location = var.region

  template {
    containers {
      image = "${var.region}-docker.pkg.dev/${var.project_id}/app-images/my-app:latest"
      resources {
        limits = {
          cpu    = "1"
          memory = "512Mi"
        }
      }
    }
    service_account = google_service_account.run_sa.email
  }

  traffic {
    type    = "TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST"
    percent = 100
  }
}
```

### Deployment Manager Examples

```bash
# Deploy a configuration
gcloud deployment-manager deployments create my-infra \
  --config=infrastructure.yaml

# Update a deployment
gcloud deployment-manager deployments update my-infra \
  --config=infrastructure.yaml

# List all deployments
gcloud deployment-manager deployments list

# Get deployment details and resource list
gcloud deployment-manager deployments describe my-infra

# Preview changes before applying
gcloud deployment-manager deployments update my-infra \
  --config=infrastructure.yaml \
  --preview

# Delete a deployment
gcloud deployment-manager deployments delete my-infra
```

### Best Practices

- Store **Terraform state** in a Cloud Storage bucket with versioning enabled
- Use **workspaces** or **directory-per-environment** patterns for env separation
- Run `terraform plan` in CI/CD before every `apply`
- Use **Google Terraform modules** for well-tested, reusable resource patterns
- Lock provider versions with `~>` to avoid unexpected upgrades
- Never commit `.tfstate` or `.tfstate.backup` files to git

### Cost Optimization

- IaC tools themselves are free; cost is in the resources they provision
- Use `terraform plan` to preview costs before applying (integrate with Infracost)
- Tag all Terraform-managed resources with `managed_by=terraform` for cost attribution

---

## 18. Secret Manager

### What Is It?

**Secret Manager** is GCP's managed service for securely storing and accessing sensitive configuration data: API keys, database passwords, TLS certificates, and other credentials. It provides versioning, IAM-based access control, automatic replication, and an audit trail via Cloud Audit Logs.

**Key concepts:**
- **Secret**: A global, named resource that holds one or more versions
- **Secret Version**: The actual payload (up to 64 KB). Previous versions are retained.
- **Access**: Principals must have `roles/secretmanager.secretAccessor` to read a secret's value

### When to Use

- Storing database passwords for Cloud SQL connections
- API keys for third-party services (Stripe, Twilio, etc.)
- TLS private keys and certificates
- Environment variables that should not be baked into container images
- Injecting secrets into Cloud Run, Cloud Functions, or GKE pods

### gcloud Examples

```bash
# Create a secret
gcloud secrets create db-password \
  --replication-policy=automatic \
  --labels=env=prod,service=backend

# Add a secret version (from stdin)
echo -n "MySuperSecretP@ssw0rd!" | \
  gcloud secrets versions add db-password --data-file=-

# Add a secret version (from a file)
gcloud secrets versions add ssl-cert \
  --data-file=./certs/server.crt

# Access (read) the latest version of a secret
gcloud secrets versions access latest \
  --secret=db-password

# Access a specific version
gcloud secrets versions access 3 \
  --secret=db-password

# List all secrets
gcloud secrets list

# List versions of a secret
gcloud secrets versions list db-password

# Describe a secret (metadata only, not the value)
gcloud secrets describe db-password

# Grant a service account access to a secret
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:my-app-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Disable (but retain) a specific secret version
gcloud secrets versions disable 1 --secret=db-password

# Destroy a secret version (irreversible!)
gcloud secrets versions destroy 1 --secret=db-password

# Delete a secret and all its versions
gcloud secrets delete old-api-key --quiet
```

**Using secrets in Cloud Run:**
```bash
# Mount a secret as an environment variable
gcloud run deploy my-service \
  --image=... \
  --set-secrets=DB_PASSWORD=db-password:latest

# Mount a secret as a volume file
gcloud run deploy my-service \
  --image=... \
  --set-secrets=/secrets/cert=ssl-cert:latest
```

### Best Practices

- Grant access at the **individual secret level**, not project-wide
- Enable **automatic secret rotation** for long-lived credentials
- Use **Secret Manager in GKE** via the Secret Manager CSI driver instead of Kubernetes Secrets
- Log all **secret access via Cloud Audit Logs** and alert on unexpected access patterns
- Use **secret labels** for environment and ownership tracking

### Cost Optimization

- Pricing: ~$0.06/active secret version/month + $0.03/10K access operations
- Destroy old secret versions that are no longer needed
- For frequently accessed secrets (millions of reads), consider caching in memory with short TTL

---

## 19. BigQuery for DevOps

### What Is It?

**BigQuery** is GCP's fully managed, serverless, highly scalable data warehouse. While primarily a data analytics tool, it is extremely useful for DevOps: analyzing large volumes of logs, querying audit trails, generating cost reports, and running SQL over exported monitoring data.

BigQuery uses **columnar storage** and **massively parallel query execution** — querying terabytes of data in seconds.

### When to Use (DevOps Perspective)

- Analyzing months of Cloud Logging data exported via log sinks
- Querying GCP billing export data (detailed cost attribution by service/label/project)
- Storing and analyzing Pub/Sub message history for debugging
- Running SQL over Cloud Audit Logs for security investigations
- Long-term metrics storage (cheaper than Cloud Monitoring for historical data)

### gcloud Examples

```bash
# List all BigQuery datasets
bq ls

# Create a dataset
bq mk --dataset \
  --location=US \
  --description="Audit log exports" \
  my_project:audit_logs

# Run a SQL query (non-interactive)
bq query --use_legacy_sql=false \
  'SELECT severity, COUNT(*) as count
   FROM `my_project.audit_logs.cloudaudit_googleapis_com_activity_*`
   WHERE _TABLE_SUFFIX BETWEEN "20240101" AND "20240131"
   GROUP BY severity
   ORDER BY count DESC'

# Load data from Cloud Storage
bq load \
  --source_format=NEWLINE_DELIMITED_JSON \
  my_project:my_dataset.my_table \
  gs://my-bucket/data/*.json \
  schema.json

# Export a table to Cloud Storage
bq extract \
  my_project:audit_logs.activity_2024 \
  gs://my-exports/activity_2024_*.json

# Describe a table schema
bq show my_project:audit_logs.activity_2024

# List tables in a dataset
bq ls my_project:audit_logs

# Get job history (see recent query jobs)
bq ls --jobs=true --max_results=20

# Delete a table
bq rm -f my_project:old_dataset.old_table

# Create a scheduled query (for regular reports)
bq mk --transfer_config \
  --target_dataset=reports \
  --display_name="Daily Cost Report" \
  --data_source=scheduled_query \
  --params='{"query":"SELECT ...","destination_table_name_template":"daily_cost_{run_date}","write_disposition":"WRITE_TRUNCATE","partitioning_field":"","partition_expiration_days":"90"}'
```

### Best Practices

- Export **Cloud Billing data** to BigQuery for granular cost analysis
- Use **partitioned tables** (by date) for log data — dramatically reduces query costs
- Set **table expiration** on temporary/intermediate tables
- Use **column-level security** for tables containing sensitive data
- Run queries against **partitioned columns** in WHERE clauses to minimize bytes scanned

### Cost Optimization

- **First 10 GB storage + 1 TB queries/month** free
- Use **table partitioning** to query only relevant date ranges (avoid full table scans)
- Use **BigQuery BI Engine** only when needed — it charges for reserved capacity
- Consider **flat-rate pricing** only for very high query volumes (>100 TB/month)

---

## 20. Cloud Memorystore

### What Is It?

**Cloud Memorystore** is GCP's fully managed in-memory data store service. It provides managed **Redis** and **Memcached** instances with automatic failover, scaling, patching, and monitoring. Ideal for caching, session storage, leaderboards, and real-time analytics.

**Two offerings:**
- **Memorystore for Redis**: Persistent, rich data structures, pub/sub, Lua scripting. Most popular.
- **Memorystore for Valkey**: Open-source Redis-compatible fork (newer option).
- **Memorystore for Memcached**: Simple key-value caching; horizontal scaling with sharding.

### When to Use

- Application caching (reduce database load with frequently read data)
- Session storage for stateless web applications
- Rate limiting and request throttling counters
- Real-time leaderboards and sorted sets
- Message queue / job queue (using Redis Lists or Streams)
- Feature flags storage (fast reads at microsecond latency)

### gcloud Examples

```bash
# Create a Redis instance (basic tier)
gcloud redis instances create my-redis \
  --size=1 \
  --region=us-central1 \
  --tier=BASIC \
  --redis-version=redis_7_0

# Create a Redis instance with high availability (Standard tier)
gcloud redis instances create my-redis-ha \
  --size=2 \
  --region=us-central1 \
  --tier=STANDARD_HA \
  --redis-version=redis_7_0 \
  --connect-mode=PRIVATE_SERVICE_ACCESS

# List Redis instances
gcloud redis instances list --region=us-central1

# Describe a Redis instance (get host IP and port)
gcloud redis instances describe my-redis --region=us-central1

# Get the connection details
gcloud redis instances describe my-redis \
  --region=us-central1 \
  --format="value(host,port)"

# Scale a Redis instance
gcloud redis instances update my-redis \
  --region=us-central1 \
  --size=4

# Upgrade Redis version
gcloud redis instances upgrade my-redis \
  --region=us-central1 \
  --redis-version=redis_7_0

# Export Redis data to Cloud Storage (RDB snapshot)
gcloud redis instances export \
  gs://my-backups/redis-$(date +%Y%m%d).rdb \
  my-redis \
  --region=us-central1

# Import Redis data from Cloud Storage
gcloud redis instances import \
  gs://my-backups/redis-20240101.rdb \
  my-redis \
  --region=us-central1

# Delete a Redis instance
gcloud redis instances delete my-redis \
  --region=us-central1 --quiet
```

### Best Practices

- **Memorystore is VPC-only** — plan your network topology before provisioning
- Use **Standard (HA) tier** for production; Basic tier has no replication/failover
- Set appropriate **memory policies** (`maxmemory-policy`) based on your eviction needs
- Use **connection pooling** in application clients — avoid opening a new Redis connection per request
- Enable **in-transit encryption** (TLS) for data in transit between application and Redis

### Cost Optimization

- Pricing is based on instance size (GB) and tier — scale only to what you need
- Use **Basic tier** for caching workloads that can tolerate brief downtime
- Delete Redis instances in non-production environments when not in use
- Monitor memory utilization — right-size instances based on actual usage

---

*For command examples and quick-reference syntax, see [COMMANDS.md](COMMANDS.md). For getting started quickly, see [QUICK-START.md](QUICK-START.md).*
