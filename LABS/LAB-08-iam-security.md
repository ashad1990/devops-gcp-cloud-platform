# LAB-08: IAM, Service Accounts, and Security

**Topic:** Identity and Access Management / Security  
**Estimated Time:** 75 minutes  
**Difficulty:** 🟡 Intermediate  
**Cost:** ~$0.10

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Create and manage GCP service accounts for applications and CI/CD pipelines
2. Assign predefined and custom IAM roles following least-privilege principles
3. Create custom IAM roles with fine-grained permission sets
4. Store and access application secrets securely using Secret Manager
5. Understand Workload Identity for GKE service account binding without key files
6. Review and interpret Cloud Audit Logs for security compliance and incident response

---

## Prerequisites

- [LAB-01](LAB-01-gcp-basics.md) completed
- `iam.googleapis.com` and `secretmanager.googleapis.com` APIs enabled
- Owner or Editor role on the project (needed to create roles and service accounts)

---

## Lab Overview

Identity and Access Management (IAM) is the security backbone of GCP. In this lab you will learn to create service accounts, apply the principle of least privilege, build custom roles, manage application secrets with Secret Manager, understand Workload Identity for GKE, and review audit trails. These practices are essential for production-grade GCP deployments.

---

## Environment Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)

echo "Project: $PROJECT_ID"

# Enable required APIs
gcloud services enable \
  iam.googleapis.com \
  secretmanager.googleapis.com \
  iamcredentials.googleapis.com \
  cloudresourcemanager.googleapis.com

echo "✅ Setup complete"
```

---

## Part 1: Service Accounts

Service accounts are identities used by applications, VMs, and services — not by humans.

### 1.1 Create Service Accounts

```bash
# Application service account (for app tier)
gcloud iam service-accounts create app-service-account \
  --display-name="Application Service Account" \
  --description="Service account for the web application tier"

# CI/CD service account (for build and deploy pipelines)
gcloud iam service-accounts create ci-service-account \
  --display-name="CI/CD Service Account" \
  --description="Service account for CI/CD pipelines"

# Data processing service account
gcloud iam service-accounts create data-processor-sa \
  --display-name="Data Processor Service Account" \
  --description="Service account for data pipeline workloads"

# List all service accounts
gcloud iam service-accounts list
```

**Expected output:**
```
DISPLAY NAME                   EMAIL                                          DISABLED
Application Service Account    app-service-account@myproject.iam.gserviceaccount.com   False
CI/CD Service Account          ci-service-account@myproject.iam.gserviceaccount.com    False
Data Processor Service Account data-processor-sa@myproject.iam.gserviceaccount.com     False
```

### 1.2 Assign Roles to Service Accounts

```bash
# Grant app SA read-only access to Cloud Storage
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:app-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Grant CI SA permission to submit builds
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:ci-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/cloudbuild.builds.editor"

# Grant CI SA permission to push to Artifact Registry
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:ci-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

# Grant CI SA permission to deploy to Cloud Run
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:ci-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/run.developer"

# Grant data processor SA BigQuery data editor
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:data-processor-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataEditor"

echo "✅ Roles assigned"
```

### 1.3 View IAM Policy

```bash
# View the full project IAM policy
gcloud projects get-iam-policy $PROJECT_ID \
  --format='table(bindings.role,bindings.members)'

# View policy as YAML
gcloud projects get-iam-policy $PROJECT_ID --format=yaml | head -50
```

---

## Part 2: IAM Policies and Roles

### 2.1 Explore Predefined Roles

```bash
# List all predefined roles for Cloud Storage
gcloud iam roles list --filter="name:roles/storage" \
  --format='table(name,title)'

# Describe a specific role and its permissions
gcloud iam roles describe roles/storage.objectAdmin

# List all permissions in a role
gcloud iam roles describe roles/container.developer \
  --format='yaml(includedPermissions)'
```

**Example output for `roles/storage.objectAdmin`:**
```
description: Full control of GCS objects.
etag: AA==
includedPermissions:
- storage.objects.create
- storage.objects.delete
- storage.objects.get
- storage.objects.getIamPolicy
- storage.objects.list
- storage.objects.setIamPolicy
- storage.objects.update
name: roles/storage.objectAdmin
stage: GA
title: Storage Object Admin
```

### 2.2 Resource-Level IAM (Bucket)

```bash
# Create a test bucket for resource-level IAM
export IAM_BUCKET="${PROJECT_ID}-iam-lab"
gsutil mb gs://$IAM_BUCKET/

# Grant app SA object admin on just this bucket (not the whole project)
gsutil iam ch \
  serviceAccount:app-service-account@${PROJECT_ID}.iam.gserviceaccount.com:roles/storage.objectAdmin \
  gs://$IAM_BUCKET/

# View bucket-level IAM policy
gsutil iam get gs://$IAM_BUCKET/
```

### 2.3 Conditional IAM Bindings

```bash
# Grant role with a time-based condition (example)
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:app-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/logging.viewer" \
  --condition='expression=request.time < timestamp("2025-12-31T00:00:00Z"),title=temp-logging-access,description=Temporary logging access'

# View conditional bindings
gcloud projects get-iam-policy $PROJECT_ID \
  --format='yaml' | grep -A 10 "condition:"
```

---

## Part 3: Custom IAM Roles

When predefined roles grant too many permissions, create custom roles with exactly the permissions needed.

### 3.1 Create a Custom Role Definition

```bash
cat > custom-role.yaml << 'EOF'
title: "Custom DevOps Engineer Role"
description: "Custom role for DevOps engineers with read + limited control"
stage: "GA"
includedPermissions:
  # Compute Engine - read and limited control
  - compute.instances.get
  - compute.instances.list
  - compute.instances.start
  - compute.instances.stop
  - compute.instances.reset
  - compute.disks.get
  - compute.disks.list
  - compute.zones.list
  - compute.regions.list
  # Cloud Storage - read only
  - storage.buckets.get
  - storage.buckets.list
  - storage.objects.get
  - storage.objects.list
  # Cloud Logging - read
  - logging.logEntries.list
  - logging.logs.list
  - logging.sinks.get
  - logging.sinks.list
  # Cloud Monitoring - read
  - monitoring.timeSeries.list
  - monitoring.dashboards.get
  - monitoring.dashboards.list
  # Cloud Run - read
  - run.services.get
  - run.services.list
  - run.revisions.get
  - run.revisions.list
EOF

# Create the custom role
gcloud iam roles create CustomDevOpsEngineer \
  --project=$PROJECT_ID \
  --file=custom-role.yaml

# Verify the role was created
gcloud iam roles describe CustomDevOpsEngineer --project=$PROJECT_ID

# List custom roles
gcloud iam roles list \
  --project=$PROJECT_ID \
  --filter="name:projects/$PROJECT_ID/roles/"
```

### 3.2 Assign the Custom Role

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:app-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="projects/${PROJECT_ID}/roles/CustomDevOpsEngineer"

echo "✅ Custom role assigned"
```

### 3.3 Update a Custom Role

```bash
# Add a permission to the custom role
gcloud iam roles update CustomDevOpsEngineer \
  --project=$PROJECT_ID \
  --add-permissions="cloudbuild.builds.list,cloudbuild.builds.get"

# Verify update
gcloud iam roles describe CustomDevOpsEngineer --project=$PROJECT_ID \
  --format='yaml(includedPermissions)' | grep cloudbuild
```

---

## Part 4: Secret Manager

Secret Manager provides secure, centralized storage for API keys, passwords, certificates, and other sensitive data.

### 4.1 Create Secrets

```bash
# Create a database password secret
echo -n "secure-db-password-$(openssl rand -hex 8)" | \
  gcloud secrets create db-password \
  --data-file=- \
  --replication-policy=automatic

# Create an API key secret
echo -n "api-key-$(openssl rand -hex 16)" | \
  gcloud secrets create api-key-prod \
  --data-file=-

# Create from a file
cat > /tmp/ssl-cert.txt << 'EOF'
-----BEGIN CERTIFICATE-----
MIIBkTCB+wIJANRV3CkJGMXgMA0GCSqGSIb3DQEBBQUAMA0xCzAJBgNVBAYTAlVT
(demo certificate - not real)
-----END CERTIFICATE-----
EOF
gcloud secrets create ssl-certificate --data-file=/tmp/ssl-cert.txt
rm -f /tmp/ssl-cert.txt

# List all secrets
gcloud secrets list
```

### 4.2 Add New Secret Versions

```bash
# Update a secret (create a new version)
echo -n "updated-db-password-v2-$(openssl rand -hex 8)" | \
  gcloud secrets versions add db-password --data-file=-

# List versions
gcloud secrets versions list db-password

# Access the latest version
gcloud secrets versions access latest --secret=db-password

# Access a specific version
gcloud secrets versions access 1 --secret=db-password
```

**Expected output:**
```
VERSION    STATE     CREATED              DESTROYED
2          enabled   2024-01-15T10:32:00  -
1          enabled   2024-01-15T10:30:00  -
```

### 4.3 Grant Service Account Access to Secrets

```bash
# Grant app SA access to db-password only
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:app-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Grant app SA access to api-key
gcloud secrets add-iam-policy-binding api-key-prod \
  --member="serviceAccount:app-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# View secret IAM policy
gcloud secrets get-iam-policy db-password

# Disable an old version (best practice after rotation)
gcloud secrets versions disable 1 --secret=db-password
gcloud secrets versions list db-password
```

---

## Part 5: Workload Identity (GKE)

Workload Identity is the recommended way to authenticate GKE workloads to GCP APIs — no key files needed.

### 5.1 Understand Workload Identity

```bash
# Workload Identity binds a GCP SA to a Kubernetes SA
# This allows pods to authenticate as the GCP SA without key files

# The pattern:
# K8s namespace: default
# K8s ServiceAccount: my-app-ksa
# GCP ServiceAccount: app-service-account@PROJECT.iam.gserviceaccount.com

export K8S_NAMESPACE=default
export K8S_SA_NAME=my-app-ksa

# Bind the GCP SA to the K8s SA
gcloud iam service-accounts add-iam-policy-binding \
  app-service-account@${PROJECT_ID}.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:${PROJECT_ID}.svc.id.goog[${K8S_NAMESPACE}/${K8S_SA_NAME}]"

# In your GKE cluster (when cluster exists), create the K8s SA with annotation:
# kubectl create serviceaccount $K8S_SA_NAME -n $K8S_NAMESPACE
# kubectl annotate serviceaccount $K8S_SA_NAME \
#   iam.gke.io/gcp-service-account=app-service-account@${PROJECT_ID}.iam.gserviceaccount.com \
#   -n $K8S_NAMESPACE

echo "Workload Identity binding created"
echo "Member: serviceAccount:${PROJECT_ID}.svc.id.goog[${K8S_NAMESPACE}/${K8S_SA_NAME}]"
```

### 5.2 Service Account Impersonation

```bash
# Impersonate a service account (requires iam.serviceAccounts.actAs permission)
# This is useful for testing what a service account can and cannot do

# Grant yourself permission to impersonate the app SA
gcloud iam service-accounts add-iam-policy-binding \
  app-service-account@${PROJECT_ID}.iam.gserviceaccount.com \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/iam.serviceAccountTokenCreator"

# Use impersonation in gcloud commands
gcloud storage buckets list \
  --impersonate-service-account=app-service-account@${PROJECT_ID}.iam.gserviceaccount.com

# Test what the SA can do
gcloud auth print-access-token \
  --impersonate-service-account=app-service-account@${PROJECT_ID}.iam.gserviceaccount.com | head -c 20
echo "..."
```

---

## Part 6: Audit Logging

### 6.1 View Admin Activity Audit Logs

```bash
# View recent admin activity (always enabled, cannot be disabled)
gcloud logging read \
  "logName=\"projects/${PROJECT_ID}/logs/cloudaudit.googleapis.com%2Factivity\"" \
  --limit=10 \
  --format='table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.methodName)'
```

### 6.2 View Data Access Audit Logs

```bash
# Check current audit log configuration
gcloud projects get-iam-policy $PROJECT_ID \
  --format='yaml(auditConfigs)'

# Create audit config to enable data access logging
cat > audit-config.json << 'EOF'
{
  "auditConfigs": [
    {
      "service": "storage.googleapis.com",
      "auditLogConfigs": [
        {"logType": "DATA_READ"},
        {"logType": "DATA_WRITE"}
      ]
    },
    {
      "service": "secretmanager.googleapis.com",
      "auditLogConfigs": [
        {"logType": "DATA_READ"},
        {"logType": "DATA_WRITE"},
        {"logType": "ADMIN_READ"}
      ]
    }
  ]
}
EOF

# Note: Enabling data access logs can significantly increase log volume and cost
# For this lab we will view existing audit logs instead

# Search for secret access events
gcloud logging read \
  "protoPayload.serviceName=\"secretmanager.googleapis.com\"" \
  --limit=10 \
  --format='table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.methodName,protoPayload.resourceName)' \
  2>/dev/null | head -20 || echo "No Secret Manager audit events found (check if data access logs enabled)"
```

### 6.3 Set Up Log-Based Alerting for Security Events

```bash
# Create a log-based metric for IAM permission changes
gcloud logging metrics create iam-policy-changes \
  --description="Count of IAM policy changes" \
  --log-filter='resource.type="project" AND protoPayload.serviceName="cloudresourcemanager.googleapis.com" AND protoPayload.methodName="SetIamPolicy"'

gcloud logging metrics list | grep iam-policy-changes
```

---

## Part 7: Service Account Keys (and Why to Avoid Them)

### 7.1 Creating a Key (Demo Only)

```bash
# ⚠️  Service account keys are long-lived credentials — avoid in production!
# Use Workload Identity, metadata server, or impersonation instead

# For demos or local development only:
gcloud iam service-accounts keys create /tmp/demo-sa-key.json \
  --iam-account=app-service-account@${PROJECT_ID}.iam.gserviceaccount.com

# List keys for the service account
gcloud iam service-accounts keys list \
  --iam-account=app-service-account@${PROJECT_ID}.iam.gserviceaccount.com

# Use the key file
export GOOGLE_APPLICATION_CREDENTIALS=/tmp/demo-sa-key.json
gcloud auth activate-service-account \
  --key-file=$GOOGLE_APPLICATION_CREDENTIALS

# Verify active account
gcloud auth list
```

### 7.2 Delete the Key Immediately

```bash
# Revert to your user account
gcloud config set account $(gcloud auth list --filter=status:ACTIVE --format='value(account)' | grep -v service)
unset GOOGLE_APPLICATION_CREDENTIALS

# Delete the key
KEY_ID=$(gcloud iam service-accounts keys list \
  --iam-account=app-service-account@${PROJECT_ID}.iam.gserviceaccount.com \
  --managed-by=user \
  --format='value(name)' | head -1 | xargs basename)

echo "Deleting key: $KEY_ID"
gcloud iam service-accounts keys delete $KEY_ID \
  --iam-account=app-service-account@${PROJECT_ID}.iam.gserviceaccount.com --quiet

# Clean up key file
rm -f /tmp/demo-sa-key.json

# Verify no user-managed keys remain
gcloud iam service-accounts keys list \
  --iam-account=app-service-account@${PROJECT_ID}.iam.gserviceaccount.com \
  --managed-by=user
```

---

## Verification Steps

```bash
# 1. List service accounts
gcloud iam service-accounts list --format='table(displayName,email)'

# 2. Verify custom role exists
gcloud iam roles describe CustomDevOpsEngineer --project=$PROJECT_ID \
  --format='value(title,stage)'

# 3. Verify secrets exist
gcloud secrets list --format='table(name,state,createTime)'

# 4. Verify no orphaned SA keys
for SA in app-service-account ci-service-account data-processor-sa; do
  KEY_COUNT=$(gcloud iam service-accounts keys list \
    --iam-account=${SA}@${PROJECT_ID}.iam.gserviceaccount.com \
    --managed-by=user --format='value(name)' | wc -l)
  echo "$SA: $KEY_COUNT user-managed key(s)"
done

# 5. Verify log metric exists
gcloud logging metrics describe iam-policy-changes --format='value(name,filter)'

echo "✅ Verification complete"
```

---

## Cost Estimates

| Resource | Cost |
|----------|------|
| Service accounts | Free (up to 100 per project) |
| IAM roles/bindings | Free |
| Secret Manager — Storage | $0.06 per active secret version/month |
| Secret Manager — Access | $0.03 per 10,000 access operations |
| Cloud Audit Logs (Admin) | Free |
| Cloud Audit Logs (Data Access) | First 50 GB/month free |

---

## Cleanup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export IAM_BUCKET="${PROJECT_ID}-iam-lab"

# Delete secrets
gcloud secrets delete db-password --quiet 2>/dev/null || true
gcloud secrets delete api-key-prod --quiet 2>/dev/null || true
gcloud secrets delete ssl-certificate --quiet 2>/dev/null || true

# Delete custom IAM role
gcloud iam roles delete CustomDevOpsEngineer --project=$PROJECT_ID --quiet 2>/dev/null || true

# Delete log-based metric
gcloud logging metrics delete iam-policy-changes --quiet 2>/dev/null || true

# Remove IAM bindings for service accounts
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:app-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer" --quiet 2>/dev/null || true

gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:ci-service-account@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/cloudbuild.builds.editor" --quiet 2>/dev/null || true

# Delete service accounts
gcloud iam service-accounts delete \
  app-service-account@${PROJECT_ID}.iam.gserviceaccount.com --quiet 2>/dev/null || true
gcloud iam service-accounts delete \
  ci-service-account@${PROJECT_ID}.iam.gserviceaccount.com --quiet 2>/dev/null || true
gcloud iam service-accounts delete \
  data-processor-sa@${PROJECT_ID}.iam.gserviceaccount.com --quiet 2>/dev/null || true

# Delete IAM lab bucket
gsutil rb gs://$IAM_BUCKET/ 2>/dev/null || true

# Clean up local files
rm -f custom-role.yaml audit-config.json

echo "✅ Cleanup complete"

# Verify service accounts deleted
gcloud iam service-accounts list | grep -E "app-service|ci-service|data-processor" || \
  echo "All lab service accounts deleted"
```

---

## Summary

You have successfully:
- ✅ Created service accounts for application, CI/CD, and data processing tiers
- ✅ Assigned predefined IAM roles following the principle of least privilege
- ✅ Created a custom IAM role with fine-grained permissions
- ✅ Stored and accessed secrets securely using Secret Manager
- ✅ Configured Workload Identity bindings for GKE workloads
- ✅ Reviewed Cloud Audit Logs and created security monitoring metrics

**Next Lab:** [LAB-09: Cloud Build CI/CD Pipelines](LAB-09-cloud-build.md)
