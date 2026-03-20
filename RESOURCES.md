# GCP Learning Resources

> Curated collection of official documentation, courses, hands-on labs, videos, and certification guidance for mastering GCP DevOps.

---

## Table of Contents

1. [Official GCP Documentation](#1-official-gcp-documentation)
2. [Google Cloud Skills Boost (Qwiklabs)](#2-google-cloud-skills-boost-qwiklabs)
3. [Google Codelabs](#3-google-codelabs)
4. [YouTube: Google Cloud Tech](#4-youtube-google-cloud-tech)
5. [Architecture Center](#5-gcp-architecture-center)
6. [Cost Calculator & Pricing](#6-cost-calculator--pricing)
7. [GCP Certifications](#7-gcp-certifications)
8. [Books & Written Guides](#8-books--written-guides)
9. [Community & Forums](#9-community--forums)
10. [Blogs & Newsletters](#10-blogs--newsletters)

---

## 1. Official GCP Documentation

Google's official documentation is comprehensive and regularly updated. These are the most important references for DevOps engineers:

### Core References

| Resource | URL | Description |
|---------|-----|-------------|
| GCP Documentation Home | https://cloud.google.com/docs | Starting point for all GCP docs |
| gcloud CLI Reference | https://cloud.google.com/sdk/gcloud/reference | Complete gcloud command reference |
| GCP Architecture Center | https://cloud.google.com/architecture | Best-practice architecture guides |
| GCP Quickstarts | https://cloud.google.com/docs#quickstarts | Quickstarts for every service |
| Release Notes | https://cloud.google.com/release-notes | Latest GCP feature updates |

### Service-Specific Documentation

| Service | Documentation URL |
|---------|------------------|
| Compute Engine | https://cloud.google.com/compute/docs |
| GKE | https://cloud.google.com/kubernetes-engine/docs |
| Cloud Run | https://cloud.google.com/run/docs |
| Cloud Functions | https://cloud.google.com/functions/docs |
| Cloud Build | https://cloud.google.com/build/docs |
| Artifact Registry | https://cloud.google.com/artifact-registry/docs |
| Cloud Storage | https://cloud.google.com/storage/docs |
| Cloud SQL | https://cloud.google.com/sql/docs |
| Cloud Spanner | https://cloud.google.com/spanner/docs |
| Pub/Sub | https://cloud.google.com/pubsub/docs |
| Cloud Monitoring | https://cloud.google.com/monitoring/docs |
| Cloud Logging | https://cloud.google.com/logging/docs |
| Secret Manager | https://cloud.google.com/secret-manager/docs |
| VPC Networking | https://cloud.google.com/vpc/docs |
| Cloud IAM | https://cloud.google.com/iam/docs |
| Terraform on GCP | https://cloud.google.com/docs/terraform |
| BigQuery | https://cloud.google.com/bigquery/docs |
| Cloud Memorystore | https://cloud.google.com/memorystore/docs |

### Best Practice Guides

| Guide | URL |
|-------|-----|
| Security Best Practices | https://cloud.google.com/security/best-practices |
| GKE Best Practices | https://cloud.google.com/kubernetes-engine/docs/best-practices |
| Cloud Run Best Practices | https://cloud.google.com/run/docs/tips/general |
| Cost Optimization Checklist | https://cloud.google.com/architecture/cost-efficiency-on-google-cloud |
| DevOps Research (DORA) | https://cloud.google.com/devops |
| SRE Books | https://sre.google/books/ |

---

## 2. Google Cloud Skills Boost (Qwiklabs)

**URL**: https://www.cloudskillsboost.google

Google Cloud Skills Boost provides hands-on labs in real GCP environments — no credit card required for labs. You get temporary GCP credentials for each lab.

### Free Resources

- **Free tier**: Some labs and quests are free with a Skills Boost account
- **Monthly subscriptions**: $29/month (or $299/year) for unlimited access
- **Free credits**: Google occasionally offers free monthly credits through promotions

### Recommended Learning Paths

| Path | Level | Description |
|------|-------|-------------|
| **Cloud Engineer Learning Path** | Beginner–Intermediate | Core GCP services for cloud engineers |
| **Cloud DevOps Engineer Learning Path** | Intermediate | CI/CD, monitoring, SRE practices |
| **Cloud Developer Learning Path** | Intermediate | App development on GCP |
| **Cloud Architect Learning Path** | Advanced | Designing GCP solutions |
| **Kubernetes in Google Cloud** | Intermediate | GKE-focused hands-on labs |
| **Google Cloud Essentials** | Beginner | Free introductory quest |

### Key Quests for DevOps

| Quest | Skills Covered |
|-------|---------------|
| Google Cloud Essentials | Basic GCP navigation, VMs, storage |
| Kubernetes in Google Cloud | GKE, kubectl, deployments |
| Cloud Architecture | VPC, IAM, Cloud SQL, monitoring |
| DevOps Essentials | Cloud Build, Cloud Deploy, Spinnaker |
| Site Reliability Engineering (SRE) | Monitoring, incident response, error budgets |
| Security & Identity Fundamentals | IAM, VPC Service Controls, DLP |
| MLOps: Getting Started | Vertex AI, pipelines, model deployment |

### Tips for Skills Boost

- Track your progress with **Skill Badges** — visual proof of completing quest series
- Labs provide **hints** if you get stuck — use them before reading solutions
- Complete labs within the time limit (typically 45–90 minutes)
- Some labs remain accessible for review after completion

---

## 3. Google Codelabs

**URL**: https://codelabs.developers.google.com

Free, self-paced tutorials with step-by-step instructions. Unlike Qwiklabs, Codelabs require your own GCP account and may incur small charges.

### Top DevOps Codelabs

| Codelab | URL |
|---------|-----|
| Building a CI/CD Pipeline with Cloud Build | https://codelabs.developers.google.com/codelabs/cloud-builder-gke |
| Deploying to Cloud Run | https://codelabs.developers.google.com/codelabs/cloud-run-hello |
| Terraform on GCP | https://codelabs.developers.google.com/codelabs/terraform-google-cloud |
| Monitoring a GKE Cluster | https://codelabs.developers.google.com/codelabs/cloud-gke-monitoring |
| Using Secret Manager | https://codelabs.developers.google.com/codelabs/secret-manager-app-engine |
| Building with Cloud Functions | https://codelabs.developers.google.com/codelabs/cloud-starting-cloudfunctions |
| Cloud Run Canary Deployments | https://codelabs.developers.google.com/codelabs/cloud-run-deploy |
| GKE Autopilot | https://codelabs.developers.google.com/codelabs/gke-autopilot |

### Tips for Codelabs

- Start with "Difficulty: Beginner" codelabs if new to GCP
- Always do the **cleanup step** at the end to avoid charges
- Use **Cloud Shell** for zero-configuration labs
- Codelabs can be done in any order — search by service name

---

## 4. YouTube: Google Cloud Tech

**URL**: https://www.youtube.com/@googlecloudtech

Official Google Cloud YouTube channel with tutorials, demos, talks, and conference recordings.

### Recommended Playlist/Series

| Series | Description | Level |
|--------|-------------|-------|
| **GCP Essentials** | Fundamentals of GCP services | Beginner |
| **Cloud Run Minute** | Short, focused Cloud Run tutorials | Beginner |
| **Kubernetes Best Practices** | Production GKE patterns | Intermediate |
| **Next '24 Talks** | Google Cloud Next conference sessions | All |
| **Apigee / API Management** | API design and management | Intermediate |
| **DevOps and SRE** | DevOps practices and tools | Intermediate |
| **BigQuery for Data Engineers** | BigQuery deep dives | Intermediate |

### Other Recommended YouTube Channels

| Channel | Focus |
|---------|-------|
| [TechWorld with Nana](https://www.youtube.com/@TechWorldwithNana) | Kubernetes, DevOps, Docker tutorials |
| [Fireship](https://www.youtube.com/@Fireship) | Fast-paced cloud and dev tutorials |
| [Cloud Guru](https://www.youtube.com/@ACloudGuru) | Cloud certifications and labs |
| [KodeKloud](https://www.youtube.com/@KodeKloud) | Kubernetes, DevOps hands-on |
| [Anton Putra](https://www.youtube.com/@AntonPutra) | GCP/Kubernetes deep dives |

### Specific Recommended Videos

- "GKE in 2024: What's New" — Google Cloud Tech
- "Cloud Run: Full Tutorial" — Fireship
- "Terraform for GCP from Scratch" — Various DevOps channels
- "DORA State of DevOps 2024" — Google Cloud Next recordings

---

## 5. GCP Architecture Center

**URL**: https://cloud.google.com/architecture

The Architecture Center provides reference architectures, design patterns, and best-practice guides for building on GCP.

### Key Categories for DevOps

| Category | URL |
|---------|-----|
| Application Development | https://cloud.google.com/architecture/application-development |
| DevOps & CI/CD | https://cloud.google.com/architecture/devops |
| Kubernetes & GKE | https://cloud.google.com/architecture/best-practices-for-operating-containers |
| Security | https://cloud.google.com/architecture/security-foundations |
| Networking | https://cloud.google.com/architecture/networking |
| Data Management | https://cloud.google.com/architecture/data-lifecycle-cloud-platform |
| Hybrid & Multi-Cloud | https://cloud.google.com/architecture/hybrid-and-multi-cloud |

### Recommended Architecture References

- **GKE Enterprise Architecture Guide**: Production-grade Kubernetes patterns
- **Cloud Run Microservices Architecture**: Serverless microservice patterns
- **CI/CD Pipeline Architecture**: End-to-end pipeline design
- **Security Foundations Blueprint**: Security architecture for regulated workloads
- **Landing Zone Design**: Multi-project, multi-team GCP structure

---

## 6. Cost Calculator & Pricing

### GCP Pricing Calculator

**URL**: https://cloud.google.com/products/calculator

Use the Pricing Calculator to estimate costs before provisioning resources:

1. Select the service (Compute Engine, GKE, Cloud SQL, etc.)
2. Configure your resource parameters
3. See estimated monthly cost
4. Export estimates as PDFs for budgeting

### Pricing Pages by Service

| Service | Pricing URL |
|---------|------------|
| Compute Engine | https://cloud.google.com/compute/pricing |
| GKE | https://cloud.google.com/kubernetes-engine/pricing |
| Cloud Run | https://cloud.google.com/run/pricing |
| Cloud Functions | https://cloud.google.com/functions/pricing |
| Cloud Storage | https://cloud.google.com/storage/pricing |
| Cloud SQL | https://cloud.google.com/sql/pricing |
| BigQuery | https://cloud.google.com/bigquery/pricing |
| Pub/Sub | https://cloud.google.com/pubsub/pricing |
| Cloud Build | https://cloud.google.com/build/pricing |
| Secret Manager | https://cloud.google.com/secret-manager/pricing |
| Cloud Monitoring | https://cloud.google.com/stackdriver/pricing |

### Cost Management Tools

| Tool | URL | Description |
|------|-----|-------------|
| Billing Dashboard | https://console.cloud.google.com/billing | Real-time spend |
| Cost Table | https://console.cloud.google.com/billing/reports | Breakdown by service |
| Recommendations | https://console.cloud.google.com/recommender | Right-sizing suggestions |
| Cloud Asset Inventory | https://console.cloud.google.com/asset-inventory | All resources in one view |
| Billing Export (BigQuery) | https://cloud.google.com/billing/docs/how-to/export-data-bigquery | SQL-queryable cost data |

---

## 7. GCP Certifications

Google offers a tiered certification program. DevOps engineers should aim for **Associate Cloud Engineer** first, followed by relevant Professional certifications.

### Certification Roadmap

```
Foundational
└── Cloud Digital Leader (non-technical, business-focused)

Associate
└── Associate Cloud Engineer ← Start here (DevOps engineers)

Professional
├── Professional Cloud Architect
├── Professional Cloud DevOps Engineer ← Primary DevOps cert
├── Professional Cloud Developer
├── Professional Data Engineer
├── Professional Cloud Network Engineer
├── Professional Cloud Security Engineer
├── Professional Cloud Database Engineer
├── Professional Machine Learning Engineer
└── Professional Google Workspace Administrator
```

### Associate Cloud Engineer

**URL**: https://cloud.google.com/learn/certification/cloud-engineer

- **Audience**: Engineers who deploy and manage GCP infrastructure
- **Duration**: 2 hours, 50 questions (multiple choice / multiple select)
- **Cost**: $200 USD
- **Validity**: 2 years (renewable)
- **Key topics**: GCP services, IAM, networking, storage, Kubernetes basics, billing

**Study resources**:
- Official study guide: https://cloud.google.com/learn/certification/guides/cloud-engineer
- Udemy: "Google Cloud Associate Cloud Engineer" by A Cloud Guru / Linux Foundation
- Pluralsight: Google Cloud Learning Path
- Free mock exams: https://cloud.google.com/certification/practice-exam/cloud-engineer

### Professional Cloud DevOps Engineer

**URL**: https://cloud.google.com/learn/certification/cloud-devops-engineer

- **Audience**: DevOps engineers managing CI/CD, monitoring, and reliability on GCP
- **Duration**: 2 hours, 50 questions
- **Cost**: $200 USD
- **Validity**: 2 years
- **Key topics**: CI/CD pipelines, GKE, Cloud Run, SRE practices, monitoring, logging, IaC

**Key exam domains**:
1. Applying site reliability engineering principles (20%)
2. Building and implementing CI/CD pipelines (20%)
3. Implementing service monitoring strategies (20%)
4. Optimizing service performance (20%)
5. Managing service incidents (20%)

**Study resources**:
- Official exam guide: https://cloud.google.com/learn/certification/guides/cloud-devops-engineer
- Google Cloud Skills Boost: "Cloud DevOps Engineer" learning path
- DORA research: https://dora.dev
- SRE books: https://sre.google/books/

### Professional Cloud Architect

**URL**: https://cloud.google.com/learn/certification/cloud-architect

- **Audience**: Cloud architects designing and managing GCP solutions
- **Duration**: 2 hours, 50 questions + case studies
- **Cost**: $200 USD
- **Key topics**: Architecture design, security, compliance, scalability, DR

### Exam Preparation Tips

1. **Hands-on practice first**: Don't just read — actually deploy the services
2. **Use Qwiklabs**: Complete the relevant learning paths before the exam
3. **Review case studies**: Professional exams include scenario-based questions
4. **Understand tradeoffs**: Exams test "which service is best for this scenario" reasoning
5. **Take practice exams**: Google provides official practice questions for each cert
6. **Read the exam guide**: Download the official exam guide and review every topic

---

## 8. Books & Written Guides

### Official Google Books

| Title | Authors | Description |
|-------|---------|-------------|
| Site Reliability Engineering | Google SRE Team | The original SRE book — mandatory reading |
| The Site Reliability Workbook | Google SRE Team | Practical SRE implementation |
| Building Secure & Reliable Systems | Google SRE Team | Security + reliability integration |

All three SRE books are **free to read online** at https://sre.google/books/

### Recommended Third-Party Books

| Title | Authors | Level |
|-------|---------|-------|
| Google Cloud Platform in Action | JJ Geewax | Beginner–Intermediate |
| Kubernetes: Up and Running | Burns, Beda, Hightower | Intermediate |
| Terraform: Up and Running | Brikman | Intermediate |
| Cloud Native DevOps with Kubernetes | Arundel, Domingus | Intermediate |
| The DevOps Handbook | Kim, Humble, Debois, Willis | All levels |
| Accelerate: The Science of DevOps | Forsgren, Humble, Kim | All levels |

---

## 9. Community & Forums

| Community | URL | Best For |
|-----------|-----|---------|
| GCP Community (Slack) | https://googlecloud-community.slack.com | Real-time Q&A |
| Stack Overflow (GCP tag) | https://stackoverflow.com/questions/tagged/google-cloud-platform | Technical questions |
| Google Cloud Community | https://www.googlecloudcommunity.com | Official forum |
| Reddit r/googlecloud | https://www.reddit.com/r/googlecloud/ | News and discussion |
| CNCF Slack | https://slack.cncf.io | Kubernetes/cloud-native community |
| GitHub: google/cloud-samples | https://github.com/GoogleCloudPlatform | Official code samples |
| GCP GitHub repos | https://github.com/GoogleCloudPlatform | Open source GCP tools |

---

## 10. Blogs & Newsletters

| Source | URL | Description |
|--------|-----|-------------|
| Google Cloud Blog | https://cloud.google.com/blog | Official announcements and tutorials |
| Google Open Source Blog | https://opensource.googleblog.com | Open source releases and news |
| GCP Weekly Newsletter | https://www.gcpweekly.com | Weekly GCP news digest |
| InfoQ Cloud | https://www.infoq.com/cloud | Cloud architecture articles |
| The New Stack | https://thenewstack.io | Cloud-native and DevOps news |
| DevOps.com | https://devops.com | DevOps practices and news |
| DORA Research | https://dora.dev | Annual State of DevOps research |

---

*Ready to start learning? Begin with [QUICK-START.md](QUICK-START.md) for a hands-on intro, or explore [CONCEPTS.md](CONCEPTS.md) for in-depth GCP service knowledge.*
