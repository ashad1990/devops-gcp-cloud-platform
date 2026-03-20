# GCP DevOps Real-World Projects

Hands-on projects that mirror production workloads on Google Cloud Platform.

## Project Overview

| # | Project | Difficulty | Est. Monthly Cost | Key Technologies |
|---|---------|------------|-------------------|-----------------|
| 01 | [Three-Tier Web App on GKE](PROJECT-01-three-tier-app.md) | ⭐⭐⭐ Intermediate | $150–200 | GKE, Cloud SQL, Cloud Build, VPC |
| 02 | [CI/CD Pipeline with Cloud Build](PROJECT-02-cicd-pipeline.md) | ⭐⭐⭐ Intermediate | $50–100 | Cloud Build, Cloud Deploy, GKE, Artifact Registry |
| 03 | [Serverless REST API](PROJECT-03-serverless-api.md) | ⭐⭐ Beginner | $5–20 | Cloud Functions, Firestore, API Gateway, Pub/Sub |
| 04 | [Data Pipeline with BigQuery ETL](PROJECT-04-data-pipeline.md) | ⭐⭐⭐ Intermediate | $10–30 | Cloud Functions, Pub/Sub, BigQuery, Cloud Scheduler |
| 05 | [Multi-Region HA Architecture](PROJECT-05-multi-region.md) | ⭐⭐⭐⭐ Advanced | $200–400 | Cloud Run, Global LB, Cloud SQL, Cloud Armor, CDN |

---

## Learning Path

```
Beginner                    Intermediate                    Advanced
   │                             │                              │
   ▼                             ▼                              ▼
PROJECT-03              PROJECT-01, 02, 04               PROJECT-05
Serverless API          Three-Tier / CI/CD /             Multi-Region HA
(~2–3 hours)            Data Pipeline                    (~6–8 hours)
                        (~4–6 hours each)
```

## Prerequisites (All Projects)

- GCP project with billing enabled
- `gcloud` CLI authenticated (`gcloud auth login`)
- Terraform ≥ 1.5 installed
- Docker installed (Projects 01, 02)
- `kubectl` installed (Projects 01, 02)
- Basic Linux/shell familiarity

## Cost Control Tips

1. **Always run cleanup** — each project has a dedicated cleanup section.
2. Use `terraform destroy` when done experimenting.
3. Set a **billing alert** at $50 in GCP Console → Billing → Budgets.
4. Projects 03 and 04 fit mostly within the GCP **Always Free** tier.
5. Use `e2-micro` / `e2-small` node types for dev/test clusters.

## Quick Start

```bash
# Clone and set up environment
export PROJECT_ID="your-gcp-project-id"
export REGION="us-central1"
gcloud config set project $PROJECT_ID

# Enable required APIs (covers all projects)
gcloud services enable \
  container.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  sqladmin.googleapis.com \
  run.googleapis.com \
  cloudfunctions.googleapis.com \
  firestore.googleapis.com \
  pubsub.googleapis.com \
  bigquery.googleapis.com \
  cloudscheduler.googleapis.com \
  apigateway.googleapis.com \
  secretmanager.googleapis.com \
  clouddeploy.googleapis.com \
  dns.googleapis.com \
  compute.googleapis.com
```

## Directory Structure

```
PROJECTS/
├── README.md                      ← This file
├── PROJECT-01-three-tier-app.md   ← GKE three-tier web application
├── PROJECT-02-cicd-pipeline.md    ← CI/CD with Cloud Build & Cloud Deploy
├── PROJECT-03-serverless-api.md   ← Serverless REST API
├── PROJECT-04-data-pipeline.md    ← BigQuery ETL data pipeline
└── PROJECT-05-multi-region.md     ← Multi-region HA architecture
```
