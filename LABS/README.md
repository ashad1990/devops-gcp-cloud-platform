# GCP DevOps Hands-On Labs

Welcome to the **GCP DevOps Hands-On Labs** series — a comprehensive, practical curriculum designed to take you from GCP fundamentals to production-ready DevOps practices on Google Cloud Platform. Each lab is self-contained with step-by-step instructions, real commands, expected outputs, and thorough cleanup guidance.

These labs are designed for DevOps engineers, cloud architects, platform engineers, and developers who want to build real-world GCP skills. You will work with Compute Engine, GKE, Cloud Run, Cloud Functions, VPC networking, IAM, CI/CD pipelines, monitoring, Terraform, and Cloud SQL — the core building blocks of modern cloud infrastructure.

---

> ⚠️ **COST WARNING**: These labs create real GCP resources that incur charges. Always run the **Cleanup** section at the end of each lab to avoid unexpected costs. Estimated costs are provided per lab but may vary by region and usage duration. Using the [GCP Free Tier](https://cloud.google.com/free) and $300 in new account credits can significantly reduce or eliminate costs for most labs.

---

## Prerequisites

Before starting any lab, ensure you have:

- **Google Account** — A Google account (gmail or Workspace) to access GCP
- **GCP Billing Enabled** — A billing account linked to your project (credit card required; new accounts receive $300 free credit)
- **gcloud CLI Installed** — [Install the Google Cloud SDK](https://cloud.google.com/sdk/docs/install)
- **Basic Linux/Terminal Knowledge** — Comfort with the command line, file editing, and shell scripting
- **kubectl** (for GKE labs) — Installed automatically via `gcloud components install kubectl`
- **Docker** (optional) — For local container builds in Cloud Run and Cloud Build labs
- **Terraform >= 1.5.0** (for LAB-11) — [Install Terraform](https://developer.hashicorp.com/terraform/install)

---

## Lab Overview

| Lab | Title | Topic | Difficulty | Est. Time | Est. Cost |
|-----|-------|-------|------------|-----------|-----------|
| [LAB-01](LAB-01-gcp-basics.md) | GCP Account, Projects & gcloud CLI | Foundations | 🟢 Beginner | 45 min | Free |
| [LAB-02](LAB-02-compute-engine.md) | Compute Engine Virtual Machines | IaaS / VMs | 🟢 Beginner | 90 min | ~$0.50–$2.00 |
| [LAB-03](LAB-03-cloud-storage.md) | Cloud Storage | Object Storage | 🟢 Beginner | 60 min | ~$0.10 |
| [LAB-04](LAB-04-gke-basics.md) | Google Kubernetes Engine | Containers / K8s | 🟡 Intermediate | 120 min | ~$2–$5 |
| [LAB-05](LAB-05-cloud-run.md) | Cloud Run Serverless Containers | Serverless | 🟡 Intermediate | 90 min | ~$0.01 |
| [LAB-06](LAB-06-cloud-functions.md) | Cloud Functions (FaaS) | Serverless / Events | 🟡 Intermediate | 75 min | ~$0.01 |
| [LAB-07](LAB-07-networking.md) | VPC Networking & Load Balancers | Networking | 🟡 Intermediate | 90 min | ~$1–$3 |
| [LAB-08](LAB-08-iam-security.md) | IAM, Service Accounts & Security | Security | 🟡 Intermediate | 75 min | ~$0.10 |
| [LAB-09](LAB-09-cloud-build.md) | Cloud Build CI/CD Pipelines | CI/CD | 🟡 Intermediate | 90 min | ~$0.50 |
| [LAB-10](LAB-10-monitoring.md) | Cloud Monitoring & Logging | Observability | 🟡 Intermediate | 75 min | Minimal |
| [LAB-11](LAB-11-terraform-gcp.md) | Infrastructure as Code with Terraform | IaC | 🔴 Advanced | 90 min | ~$0.50 |
| [LAB-12](LAB-12-cloud-sql.md) | Cloud SQL Managed Databases | Databases | 🔴 Advanced | 90 min | ~$1–$3 |

**Total estimated time:** ~13 hours  
**Total estimated cost (all labs):** ~$6–$15 (with diligent cleanup)

---

## Learning Path

### Recommended Order for Beginners

```
LAB-01 → LAB-02 → LAB-03 → LAB-07 → LAB-08 → LAB-04 → LAB-05 → LAB-06 → LAB-09 → LAB-10 → LAB-11 → LAB-12
```

### Fast Track (Experienced GCP Users)

```
LAB-01 (quick review) → LAB-04 → LAB-05 → LAB-09 → LAB-11
```

### Security Focus Path

```
LAB-01 → LAB-08 → LAB-07 → LAB-10 → LAB-12
```

---

## Lab Objectives

### LAB-01: GCP Account, Projects & gcloud CLI
- Create and configure a GCP project via console and CLI
- Install, initialize, and authenticate the gcloud SDK
- Navigate the GCP Console and Cloud Shell effectively
- Configure gcloud CLI defaults and named configurations
- Enable service APIs programmatically using gcloud
- Understand the GCP resource hierarchy (Organization → Folder → Project → Resource)

### LAB-02: Compute Engine Virtual Machines
- Create and manage Compute Engine virtual machine instances
- Configure firewall rules for network traffic control
- Work with custom machine types and Spot (preemptible) VMs
- Create instance templates and managed instance groups (MIGs)
- Configure autoscaling policies based on CPU utilization
- Take and restore disk snapshots for backup and recovery

### LAB-03: Cloud Storage
- Create storage buckets with different storage classes (Standard, Nearline, Coldline)
- Upload, download, and manage objects using gsutil
- Configure IAM permissions and access control on buckets
- Implement lifecycle policies for automatic cost optimization
- Enable object versioning for data protection
- Host a static website on Cloud Storage

### LAB-04: Google Kubernetes Engine
- Create and configure a GKE cluster with appropriate node settings
- Deploy containerized applications using Kubernetes manifests
- Expose applications externally using LoadBalancer services
- Scale deployments manually and with Horizontal Pod Autoscaler (HPA)
- Perform rolling updates and rollbacks safely
- Manage application configuration with ConfigMaps and Secrets

### LAB-05: Cloud Run Serverless Containers
- Build and containerize a Python Flask application
- Push container images to Artifact Registry
- Deploy and configure serverless containers on Cloud Run
- Implement canary deployments using traffic splitting
- Manage secrets and environment variables securely
- Configure concurrency, scaling, and CPU throttling settings

### LAB-06: Cloud Functions (FaaS)
- Deploy HTTP-triggered Cloud Functions (Gen 2) with Python
- Create event-driven functions triggered by Pub/Sub messages
- Build Cloud Storage-triggered functions for file processing
- Configure environment variables and manage function settings
- Monitor function invocations and troubleshoot using Cloud Logging
- Understand pricing and cold start behavior for serverless functions

### LAB-07: VPC Networking & Load Balancers
- Design and create a custom VPC with multiple subnets for tiered architecture
- Configure firewall rules with tags, source ranges, and priorities
- Deploy an HTTP(S) Load Balancer with backend services and health checks
- Set up Cloud NAT for outbound internet access from private VMs
- Configure Cloud DNS with private managed zones and records
- Understand network routing, peering concepts, and load balancer types

### LAB-08: IAM, Service Accounts & Security
- Create and manage GCP service accounts for applications and CI/CD
- Assign predefined and custom IAM roles following least-privilege principles
- Create custom IAM roles with fine-grained permissions
- Store and access secrets securely using Secret Manager
- Understand Workload Identity for GKE service account binding
- Review and interpret Cloud Audit Logs for security compliance

### LAB-09: Cloud Build CI/CD Pipelines
- Set up Cloud Build with proper IAM permissions for deployment
- Create and manage Artifact Registry Docker repositories
- Write multi-step `cloudbuild.yaml` configurations with parallelism
- Trigger automated builds from source code repository changes
- Deploy applications from Cloud Build to Cloud Run automatically
- Integrate Pub/Sub notifications for build status monitoring

### LAB-10: Cloud Monitoring & Logging
- View, filter, and write structured log entries using Cloud Logging
- Create log sinks to export logs to Cloud Storage and BigQuery
- Define log-based metrics for tracking application errors and events
- Build custom dashboards with Cloud Monitoring widgets and charts
- Create alerting policies with notification channels for incidents
- Configure uptime checks and understand Cloud Trace basics

### LAB-11: Infrastructure as Code with Terraform
- Install and configure Terraform with the GCP provider
- Define GCP resources (buckets, VMs) declaratively in HCL
- Use variables, outputs, and validation blocks effectively
- Configure remote state storage in a GCS backend
- Build reusable Terraform modules for common infrastructure patterns
- Manage multiple environments using Terraform workspaces

### LAB-12: Cloud SQL Managed Databases
- Create and configure a Cloud SQL PostgreSQL instance
- Create databases, users, and manage connection settings
- Connect securely using the Cloud SQL Auth Proxy
- Import and export database data to and from Cloud Storage
- Configure automated backups and Point-in-Time Recovery (PITR)
- Set up read replicas and enable high availability (HA) for production

---

## Common Setup

Enable these APIs once at the start to avoid interruptions across labs:

```bash
export PROJECT_ID=$(gcloud config get-value project)

gcloud services enable \
  compute.googleapis.com \
  container.googleapis.com \
  cloudbuild.googleapis.com \
  run.googleapis.com \
  cloudfunctions.googleapis.com \
  artifactregistry.googleapis.com \
  sqladmin.googleapis.com \
  sql-component.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com \
  cloudtrace.googleapis.com \
  secretmanager.googleapis.com \
  dns.googleapis.com \
  pubsub.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  storage.googleapis.com

echo "✅ All APIs enabled for PROJECT_ID=$PROJECT_ID"
```

Set your default region and zone:

```bash
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
gcloud config set run/region us-central1
```

---

## Tips for Success

1. **Read the entire lab first** — Skim through all sections before running any commands so you understand what you're building.
2. **Use Cloud Shell** — If you don't have gcloud installed locally, use [Cloud Shell](https://shell.cloud.google.com) — it comes pre-configured with all tools.
3. **Always set environment variables** — Run the `export` statements at the top of each lab before any other commands.
4. **Don't skip Cleanup sections** — Resources left running accumulate charges quickly, especially VMs, GKE clusters, and Cloud SQL instances.
5. **Check quotas** — Some regions have quota limits. If you hit a quota error, try a different region or request a quota increase.
6. **Use a dedicated project** — Create a fresh GCP project just for these labs to make cleanup easy (delete the project when done).
7. **Save outputs** — Copy important values (IPs, connection strings, service URLs) to a text file as you go.
8. **Enable billing alerts** — Set a budget alert in GCP Billing console to avoid unexpected charges.

---

## Cost Management

- **Set a billing budget**: Go to **Billing → Budgets & Alerts** and set a monthly budget with email alerts at 50%, 90%, and 100%.
- **Use e2-micro or e2-small** VMs for labs — they are the cheapest compute options.
- **Delete resources immediately** after finishing each lab using the Cleanup section.
- **Use Spot/Preemptible VMs** in dev/lab environments to reduce VM costs by up to 90%.
- **Monitor your spending**: Check **Billing → Reports** daily while working through the labs.
- **One project for all labs**: Use a single project and delete it entirely when done — this ensures no orphaned resources remain.

---

## Cleanup Reminder

> 🔴 **IMPORTANT**: Always run the `## Cleanup` section at the end of each lab.  
> GCP charges for running resources even when idle. A single forgotten GKE cluster or Cloud SQL instance can cost $50–$100/month.

**Nuclear option** — if you're done with all labs, delete the entire project:

```bash
# WARNING: This permanently deletes ALL resources in the project
gcloud projects delete $PROJECT_ID
```

This is the safest way to ensure you are not billed for any remaining resources.

---

## Additional Resources

- [GCP Documentation](https://cloud.google.com/docs)
- [gcloud CLI Reference](https://cloud.google.com/sdk/gcloud/reference)
- [GCP Pricing Calculator](https://cloud.google.com/products/calculator)
- [GCP Free Tier Details](https://cloud.google.com/free/docs/free-cloud-features)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Terraform GCP Provider Docs](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [Cloud Architecture Center](https://cloud.google.com/architecture)
