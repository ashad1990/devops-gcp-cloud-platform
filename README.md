# DevOps on Google Cloud Platform - Practical GCP Engineering

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![GCP](https://img.shields.io/badge/Google%20Cloud-Platform-4285F4?logo=google-cloud)](https://cloud.google.com)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

> A comprehensive, hands-on training repository for mastering DevOps practices on Google Cloud Platform. From foundational concepts to production-grade deployments — built for engineers who want to go beyond the docs.

---

## Table of Contents

- [Introduction](#introduction)
- [Learning Objectives](#learning-objectives)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Quick Start](#quick-start)
- [GCP Free Tier Tips](#gcp-free-tier-tips)
- [Cost Management Guidance](#cost-management-guidance)
- [Contributing](#contributing)
- [License](#license)

---

## Introduction

Google Cloud Platform (GCP) is one of the world's leading cloud providers, powering some of the most demanding workloads in production — from real-time data pipelines processing millions of events per second to globally distributed Kubernetes clusters serving billions of users. For modern DevOps engineers, fluency in GCP is no longer optional; it is a critical career skill.

This repository provides practical, real-world training materials for engineers who want to master the DevOps ecosystem on GCP. Unlike documentation-only resources, everything here is oriented around *doing*: deploying real infrastructure, writing real pipelines, and solving real operational problems.

GCP's DevOps story spans a rich portfolio of services:

- **Compute**: Compute Engine, Google Kubernetes Engine (GKE), Cloud Run, Cloud Functions, App Engine
- **Storage & Databases**: Cloud Storage, Cloud SQL, Cloud Spanner, Firestore, BigQuery
- **CI/CD**: Cloud Build, Artifact Registry, Cloud Deploy
- **Networking**: VPC, Cloud Load Balancing, Cloud DNS, Cloud Armor
- **Observability**: Cloud Monitoring, Cloud Logging, Cloud Trace, Cloud Profiler
- **Security**: Cloud IAM, Secret Manager, Security Command Center
- **Infrastructure as Code**: Terraform, Deployment Manager

Whether you are migrating workloads from on-premises, building cloud-native microservices, or preparing for a GCP certification, this training repository will give you the knowledge and command-line confidence to operate effectively on Google Cloud.

---

## Learning Objectives

By the end of this training, you will be able to:

1. **Navigate the GCP Console and CLI** — Configure the `gcloud` SDK, manage authentication, switch between projects and regions, and work efficiently from the command line.

2. **Design and manage GCP project hierarchies** — Understand Organizations, Folders, and Projects; apply resource hierarchy best practices; manage billing accounts and cost attribution.

3. **Implement Cloud IAM at scale** — Create and manage service accounts, assign least-privilege roles, configure Workload Identity Federation, and audit IAM policies.

4. **Deploy and manage Compute Engine workloads** — Launch VMs, configure managed instance groups with autoscaling, create custom images, and manage persistent disks and snapshots.

5. **Operate production-grade GKE clusters** — Create Autopilot and Standard clusters, configure node pools, deploy workloads with kubectl, set up Horizontal Pod Autoscaling, and implement cluster security.

6. **Build and deploy serverless applications** — Deploy containerized services to Cloud Run, write and trigger Cloud Functions, and choose the right serverless platform for a given workload.

7. **Design resilient storage architectures** — Create and manage Cloud Storage buckets with lifecycle policies, configure replication, work with signed URLs, and understand storage class trade-offs.

8. **Automate CI/CD pipelines with Cloud Build** — Write `cloudbuild.yaml` configuration files, configure build triggers from source repositories, push images to Artifact Registry, and integrate with Cloud Deploy.

9. **Manage relational and globally distributed databases** — Provision Cloud SQL instances, configure high availability and read replicas, and understand when to use Cloud Spanner for globally consistent workloads.

10. **Implement comprehensive observability** — Create Cloud Monitoring dashboards and alerting policies, write log-based metrics, configure uptime checks, and use Cloud Trace for distributed tracing.

11. **Provision infrastructure with Terraform** — Write Terraform configurations for GCP resources, manage remote state in Cloud Storage, use the Google provider modules, and implement CI/CD for infrastructure changes.

12. **Secure workloads and manage secrets** — Store and rotate secrets with Secret Manager, configure VPC firewalls and Private Google Access, implement Binary Authorization for containers, and understand the GCP shared responsibility model.

---

## Prerequisites

Before starting this training, you should have:

### Cloud & Infrastructure Basics
- Familiarity with at least one cloud provider (AWS, Azure, or GCP) at a conceptual level
- Understanding of virtualization concepts (VMs, containers, hypervisors)
- Basic knowledge of infrastructure components: load balancers, DNS, storage, databases

### Command-Line Proficiency
- Comfortable working in a Linux/macOS terminal or Windows WSL2
- Familiarity with bash scripting (variables, loops, conditionals)
- Basic experience with `git` (clone, commit, push, pull)
- JSON and YAML literacy (reading and writing configuration files)

### Networking Fundamentals
- TCP/IP basics: IP addresses, subnets, CIDR notation
- Understanding of HTTP/HTTPS, DNS, and TLS
- Familiarity with firewall concepts (rules, ports, protocols)
- Basic understanding of load balancers and reverse proxies

### Development Background
- Experience with at least one programming language (Python, Go, Java, or Node.js recommended)
- Understanding of REST APIs and how to call them
- Basic Docker knowledge (build, run, push images) is strongly recommended

### Nice to Have (Not Required)
- Prior Kubernetes experience
- Terraform or other IaC tool experience
- CI/CD pipeline experience (Jenkins, GitLab CI, GitHub Actions)

---

## Repository Structure

```
devops-gcp-cloud-platform/
├── README.md                    # This file — course overview and navigation
├── CONCEPTS.md                  # 20 core GCP concepts with examples and best practices
├── COMMANDS.md                  # Comprehensive gcloud CLI reference (5–10 examples per group)
├── QUICK-START.md               # 10-minute GCP introduction — up and running fast
├── SETUP.md                     # Complete environment setup (SDK, auth, tools)
├── TOOLS.md                     # GCP DevOps tools overview (gcloud, kubectl, Terraform, etc.)
├── RESOURCES.md                 # Curated learning resources, docs, certifications
├── TROUBLESHOOTING.md           # Common errors, quota issues, and debugging guidance
├── CONTRIBUTING.md              # How to contribute to this repository
├── LICENSE                      # Apache 2.0 license
└── .gitignore                   # Excludes credentials, state files, and secrets
```

| File | Purpose | Audience |
|------|---------|----------|
| `README.md` | Course introduction, objectives, prerequisites | All |
| `CONCEPTS.md` | Deep-dive reference for 20 GCP services | Intermediate |
| `COMMANDS.md` | Quick-reference CLI cheatsheet | All |
| `QUICK-START.md` | Hands-on lab in 10 minutes | Beginner |
| `SETUP.md` | Environment configuration guide | Beginner |
| `TOOLS.md` | Tool ecosystem overview | Beginner–Intermediate |
| `RESOURCES.md` | Books, courses, certifications | All |
| `TROUBLESHOOTING.md` | Debugging and error resolution | All |
| `CONTRIBUTING.md` | Contribution process | Contributors |

---

## Quick Start

Get your first GCP resource deployed in under 10 minutes:

```bash
# 1. Install the gcloud SDK (see SETUP.md for full instructions)
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# 2. Authenticate
gcloud auth login
gcloud auth application-default login

# 3. Create or select a project
gcloud projects create my-devops-lab-$(date +%s) --name="DevOps Lab"
gcloud config set project <PROJECT_ID>

# 4. Enable billing and core APIs
gcloud services enable compute.googleapis.com run.googleapis.com \
  cloudbuild.googleapis.com artifactregistry.googleapis.com

# 5. Deploy a Hello World to Cloud Run (no VM required)
gcloud run deploy hello-world \
  --image=us-docker.pkg.dev/cloudrun/container/hello \
  --platform=managed \
  --region=us-central1 \
  --allow-unauthenticated
```

For the complete guided walkthrough, see **[QUICK-START.md](QUICK-START.md)**.

---

## GCP Free Tier Tips

GCP offers a generous free tier that lets you learn without spending money. Here is how to make the most of it:

### Always-Free Products (Never Expire)
| Service | Free Allowance |
|---------|---------------|
| Cloud Run | 2 million requests/month, 360,000 GB-seconds/month |
| Cloud Functions | 2 million invocations/month |
| Cloud Storage | 5 GB-months in US regions (Standard) |
| BigQuery | 10 GB storage + 1 TB queries/month |
| Cloud Firestore | 1 GB storage, 50K reads/day, 20K writes/day |
| Cloud Monitoring | 150 MB metrics ingestion/month |
| Cloud Logging | First 50 GB ingestion/month |
| Pub/Sub | First 10 GB/month |
| Artifact Registry | First 500 MB/month |
| Cloud Shell | Free persistent 5 GB home directory |

### $300 Free Credit (New Accounts)
- All new GCP accounts receive **$300 USD in free credits** valid for **90 days**
- Credits apply to any GCP service, including Compute Engine and GKE
- You **will not** be charged after credits expire unless you manually upgrade to a paid account

### Tips to Minimize Costs During Training
1. **Use `us-central1` or `us-east1`** — cheapest regions for most services
2. **Stop VMs when not in use** — `gcloud compute instances stop <INSTANCE>`
3. **Use `e2-micro` or `e2-small`** instances for lab work (e2-micro is always free in some regions)
4. **Delete GKE clusters when done** — GKE control plane costs ~$0.10/hour for Standard clusters
5. **Use Autopilot GKE** for dev/test — you only pay for actual pod resource usage
6. **Set budget alerts** immediately (see Cost Management below)
7. **Use Cloud Run instead of VMs** where possible — serverless scales to zero
8. **Delete unused Persistent Disks** — they accrue charges even when VMs are stopped
9. **Use lifecycle policies on Cloud Storage** — auto-delete or transition to cheaper classes
10. **Check the Pricing Calculator** at https://cloud.google.com/products/calculator before launching resources

---

## Cost Management Guidance

Uncontrolled cloud spending is a real risk. Here is a systematic approach to keeping costs under control:

### 1. Set Budget Alerts Immediately

```bash
# Create a budget alert at $10/month with alerts at 50%, 90%, 100%
gcloud billing budgets create \
  --billing-account=BILLING_ACCOUNT_ID \
  --display-name="Training Budget" \
  --budget-amount=10USD \
  --threshold-rule=percent=50 \
  --threshold-rule=percent=90 \
  --threshold-rule=percent=100
```

Navigate to **Billing → Budgets & Alerts** in the GCP Console to configure email notifications.

### 2. Use Labels for Cost Attribution

```bash
# Label all resources by environment and purpose
gcloud compute instances create my-vm \
  --labels=env=training,purpose=lab,owner=yourname \
  --zone=us-central1-a

# Query costs by label in Billing → Reports → Group by Label
```

### 3. Enable Committed Use Discounts (Production)
- 1-year commitment: up to 37% discount on Compute Engine
- 3-year commitment: up to 55% discount
- Applicable to VMs, GKE nodes, Cloud SQL, Cloud Spanner

### 4. Understand Pricing Models
| Model | Best For | Notes |
|-------|---------|-------|
| On-demand | Dev/test, unpredictable workloads | No upfront cost |
| Preemptible/Spot VMs | Batch, fault-tolerant workloads | Up to 91% cheaper, may be interrupted |
| Committed Use | Steady-state production | Requires 1 or 3 year commitment |
| Sustained Use Discounts | Long-running VMs | Automatic, up to 30% |

### 5. Regular Cleanup Commands

```bash
# List all running instances (to stop idle ones)
gcloud compute instances list --filter="status=RUNNING"

# Stop all instances in a project
gcloud compute instances list --format="value(name,zone)" | \
  while read name zone; do
    gcloud compute instances stop "$name" --zone="$zone" --quiet
  done

# List GKE clusters (remember: control plane costs money)
gcloud container clusters list

# Delete a GKE cluster when done
gcloud container clusters delete CLUSTER_NAME --region=us-central1 --quiet

# List Cloud SQL instances (each one costs ~$7+/month minimum)
gcloud sql instances list

# Delete a Cloud SQL instance
gcloud sql instances delete INSTANCE_NAME --quiet
```

### 6. Use the Cost Management Tools
- **Billing Dashboard**: Real-time spend visualization
- **Recommender**: Automatic right-sizing suggestions for underutilized VMs
- **Active Assist**: Proactive cost optimization recommendations
- **Cloud Asset Inventory**: Audit all resources across the organization

---

## Contributing

We welcome contributions! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on submitting improvements, reporting bugs, and suggesting new training content.

---

## License

This project is licensed under the Apache License 2.0 — see the [LICENSE](LICENSE) file for details.

---

*Built with ❤️ for engineers who learn by doing. Start with [QUICK-START.md](QUICK-START.md) if you're new, or dive into [CONCEPTS.md](CONCEPTS.md) for a deep reference.*
