# LAB-03: Cloud Storage

**Topic:** Object Storage  
**Estimated Time:** 60 minutes  
**Difficulty:** 🟢 Beginner  
**Cost:** ~$0.10

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Create Cloud Storage buckets with different storage classes (Standard, Nearline, Coldline)
2. Upload, download, copy, and manage objects using gsutil and gcloud
3. Configure IAM permissions and access control on buckets and objects
4. Implement lifecycle policies for automatic storage class transitions and deletion
5. Enable object versioning for data protection and recovery
6. Host a static website using Cloud Storage with public access

---

## Prerequisites

- [LAB-01](LAB-01-gcp-basics.md) completed
- `storage.googleapis.com` API enabled
- gsutil installed (included with gcloud SDK)

---

## Lab Overview

Cloud Storage is GCP's globally unified, scalable object storage service. It is used for everything from backups and static assets to data lake storage and application artifact distribution. In this lab you will explore bucket creation, file operations, access control, lifecycle management, versioning, and static website hosting.

---

## Environment Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export BUCKET_NAME="devops-lab-${PROJECT_ID}"
export REGION=us-central1

echo "Project:     $PROJECT_ID"
echo "Bucket Name: $BUCKET_NAME"
echo "Region:      $REGION"
```

---

## Part 1: Creating Buckets

### 1.1 Create Buckets with Different Storage Classes

```bash
# Standard bucket (frequent access)
gsutil mb -p $PROJECT_ID -c STANDARD -l $REGION gs://$BUCKET_NAME/

# Nearline bucket (infrequent access, accessed ~once/month)
gsutil mb -p $PROJECT_ID -c NEARLINE -l $REGION gs://${BUCKET_NAME}-nearline/

# Coldline bucket (archival, accessed ~once/quarter)
gsutil mb -p $PROJECT_ID -c COLDLINE -l $REGION gs://${BUCKET_NAME}-coldline/

# List all buckets in the project
gsutil ls

# Get details about a bucket
gsutil ls -L -b gs://$BUCKET_NAME/
```

**Expected output for `gsutil ls`:**
```
gs://devops-lab-myproject-123456/
gs://devops-lab-myproject-123456-nearline/
gs://devops-lab-myproject-123456-coldline/
```

### 1.2 Storage Class Comparison

| Class | Use Case | Min Storage Duration | Retrieval Cost |
|-------|----------|---------------------|----------------|
| Standard | Hot data, frequent access | None | None |
| Nearline | Accessed ~1x/month | 30 days | $0.01/GB |
| Coldline | Accessed ~1x/quarter | 90 days | $0.02/GB |
| Archive | Long-term archival | 365 days | $0.05/GB |

### 1.3 Create via gcloud (alternative to gsutil)

```bash
# gcloud storage commands (newer API)
gcloud storage buckets create gs://${BUCKET_NAME}-gcloud \
  --location=$REGION \
  --default-storage-class=STANDARD \
  --uniform-bucket-level-access

# List buckets via gcloud
gcloud storage buckets list

# Delete the extra bucket
gsutil rb gs://${BUCKET_NAME}-gcloud/
```

---

## Part 2: Uploading and Managing Files

### 2.1 Create Test Files

```bash
# Create test files locally
echo "Hello, GCP Cloud Storage!" > hello.txt
echo "DevOps Lab content - $(date)" > devops.txt
echo '{"service": "cloud-storage", "lab": "03", "status": "testing"}' > data.json

# Create a directory structure
mkdir -p test-data/images test-data/logs test-data/configs
echo "image data" > test-data/images/photo.jpg
echo "log entry 1" > test-data/logs/app.log
echo "log entry 2" >> test-data/logs/app.log
echo "key=value" > test-data/configs/app.properties
```

### 2.2 Upload Files

```bash
# Upload a single file
gsutil cp hello.txt gs://$BUCKET_NAME/

# Upload with custom content type
gsutil -h "Content-Type:text/plain" cp devops.txt gs://$BUCKET_NAME/docs/

# Upload JSON file
gsutil -h "Content-Type:application/json" cp data.json gs://$BUCKET_NAME/data/

# Upload entire directory recursively (-m = parallel, -r = recursive)
gsutil -m cp -r test-data/ gs://$BUCKET_NAME/test-data/

echo "✅ Upload complete"
```

### 2.3 List Objects

```bash
# List objects in bucket
gsutil ls gs://$BUCKET_NAME/

# List all objects recursively
gsutil ls -r gs://$BUCKET_NAME/

# List with detailed info (size, timestamps)
gsutil ls -la gs://$BUCKET_NAME/

# List with human-readable sizes
gsutil ls -lh gs://$BUCKET_NAME/

# Get object metadata
gsutil stat gs://$BUCKET_NAME/hello.txt
```

**Expected output for `gsutil ls -lh gs://$BUCKET_NAME/`:**
```
     26 B  2024-01-15T10:30:00Z  gs://devops-lab-proj-123456/hello.txt
     49 B  2024-01-15T10:30:01Z  gs://devops-lab-proj-123456/docs/devops.txt
     59 B  2024-01-15T10:30:02Z  gs://devops-lab-proj-123456/data/data.json
TOTAL: 3 objects, 134 bytes (134 B)
```

### 2.4 Download Files

```bash
# Download a single file
gsutil cp gs://$BUCKET_NAME/hello.txt ./hello-downloaded.txt

# Download to current directory
gsutil cp gs://$BUCKET_NAME/data/data.json .

# Download directory
gsutil -m cp -r gs://$BUCKET_NAME/test-data/ ./downloaded-data/

# Verify download
cat hello-downloaded.txt
```

### 2.5 Copy and Move Between Buckets

```bash
# Copy file to nearline bucket
gsutil cp gs://$BUCKET_NAME/hello.txt gs://${BUCKET_NAME}-nearline/

# Copy directory to another bucket
gsutil -m cp -r gs://$BUCKET_NAME/test-data/ gs://${BUCKET_NAME}-nearline/

# Move (copy + delete source)
# gsutil mv gs://$BUCKET_NAME/devops.txt gs://$BUCKET_NAME/archive/devops.txt

# Sync directories (only upload changed files)
gsutil -m rsync -r gs://$BUCKET_NAME/test-data/ gs://${BUCKET_NAME}-nearline/test-data/
```

### 2.6 Delete Objects

```bash
# Delete a single file
# gsutil rm gs://$BUCKET_NAME/old-file.txt

# Delete multiple files matching a pattern
# gsutil rm gs://$BUCKET_NAME/test-data/logs/*.log

# Delete all objects in a directory
# gsutil -m rm -r gs://$BUCKET_NAME/old-directory/
```

---

## Part 3: IAM and Permissions

### 3.1 View Current IAM Policy

```bash
gsutil iam get gs://$BUCKET_NAME
```

### 3.2 Grant Public Read Access

```bash
# Make all objects in bucket publicly readable
gsutil iam ch allUsers:objectViewer gs://$BUCKET_NAME

# Verify public access
curl -s "https://storage.googleapis.com/$BUCKET_NAME/hello.txt"

# Test with gsutil
gsutil acl get gs://$BUCKET_NAME/hello.txt
```

### 3.3 Grant User-Specific Access

```bash
# Grant a user objectAdmin access (replace email address)
# gsutil iam ch user:developer@example.com:objectAdmin gs://$BUCKET_NAME

# Grant a service account read access
# gsutil iam ch \
#   serviceAccount:my-sa@${PROJECT_ID}.iam.gserviceaccount.com:objectViewer \
#   gs://$BUCKET_NAME

# View updated policy
gsutil iam get gs://$BUCKET_NAME
```

### 3.4 Revoke Access

```bash
# Revoke public access
gsutil iam ch -d allUsers:objectViewer gs://$BUCKET_NAME

# Verify access is revoked (should return 403)
curl -s -o /dev/null -w "%{http_code}" \
  "https://storage.googleapis.com/$BUCKET_NAME/hello.txt"
# Expected: 403
```

### 3.5 Uniform Bucket-Level Access

```bash
# Enable uniform bucket-level access (recommended for new buckets)
gsutil uniformbucketlevelaccess set on gs://$BUCKET_NAME

# Verify
gsutil uniformbucketlevelaccess get gs://$BUCKET_NAME
```

---

## Part 4: Lifecycle Policies

Lifecycle policies automatically manage objects based on age, storage class, or other conditions.

### 4.1 Create and Apply a Lifecycle Policy

```bash
cat > lifecycle.json << 'EOF'
{
  "rule": [
    {
      "action": {
        "type": "SetStorageClass",
        "storageClass": "NEARLINE"
      },
      "condition": {
        "age": 30,
        "matchesStorageClass": ["STANDARD"]
      }
    },
    {
      "action": {
        "type": "SetStorageClass",
        "storageClass": "COLDLINE"
      },
      "condition": {
        "age": 90,
        "matchesStorageClass": ["NEARLINE"]
      }
    },
    {
      "action": {
        "type": "Delete"
      },
      "condition": {
        "age": 365
      }
    },
    {
      "action": {
        "type": "Delete"
      },
      "condition": {
        "numNewerVersions": 3,
        "isLive": false
      }
    }
  ]
}
EOF

# Apply lifecycle policy
gsutil lifecycle set lifecycle.json gs://$BUCKET_NAME

# View current lifecycle policy
gsutil lifecycle get gs://$BUCKET_NAME
```

**Expected output for `gsutil lifecycle get`:**
```json
{
  "rule": [
    {
      "action": {"storageClass": "NEARLINE", "type": "SetStorageClass"},
      "condition": {"age": 30, "matchesStorageClass": ["STANDARD"]}
    },
    ...
  ]
}
```

### 4.2 Remove Lifecycle Policy

```bash
# To remove a lifecycle policy, set an empty one
# echo '{}' | gsutil lifecycle set /dev/stdin gs://$BUCKET_NAME
```

---

## Part 5: Object Versioning

Versioning keeps a history of all object changes, enabling recovery from accidental deletions or overwrites.

### 5.1 Enable Versioning

```bash
# Enable versioning on the bucket
gsutil versioning set on gs://$BUCKET_NAME

# Verify versioning is enabled
gsutil versioning get gs://$BUCKET_NAME
# Expected: gs://BUCKET_NAME: Enabled
```

### 5.2 Create Multiple Versions

```bash
# Upload version 1
echo "Version 1 - Original content" > version-test.txt
gsutil cp version-test.txt gs://$BUCKET_NAME/version-test.txt

# Upload version 2 (overwrites)
echo "Version 2 - Updated content" > version-test.txt
gsutil cp version-test.txt gs://$BUCKET_NAME/version-test.txt

# Upload version 3
echo "Version 3 - Latest content" > version-test.txt
gsutil cp version-test.txt gs://$BUCKET_NAME/version-test.txt

# List all versions (including non-current)
gsutil ls -a gs://$BUCKET_NAME/version-test.txt
```

**Expected output:**
```
gs://devops-lab-proj-123456/version-test.txt#1705320000000000
gs://devops-lab-proj-123456/version-test.txt#1705320010000000
gs://devops-lab-proj-123456/version-test.txt#1705320020000000
```

### 5.3 Access a Specific Version

```bash
# Get all version generation numbers
gsutil ls -la gs://$BUCKET_NAME/version-test.txt

# Download a specific version (replace GENERATION with actual number)
# gsutil cp "gs://$BUCKET_NAME/version-test.txt#GENERATION" ./version-1.txt

# Delete a non-current version
# gsutil rm "gs://$BUCKET_NAME/version-test.txt#GENERATION"

# Soft-delete: delete the live version (creates a non-current version)
# gsutil rm gs://$BUCKET_NAME/version-test.txt
# The object is hidden but recoverable via its generation number
```

---

## Part 6: Static Website Hosting

### 6.1 Create Website Files

```bash
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>GCP Storage Static Website</title>
  <style>
    body { font-family: Arial, sans-serif; max-width: 800px; margin: 50px auto; padding: 20px; }
    h1 { color: #4285F4; }
    .badge { background: #34A853; color: white; padding: 5px 10px; border-radius: 4px; }
  </style>
</head>
<body>
  <h1>🚀 Hosted on Google Cloud Storage!</h1>
  <p><span class="badge">GCP DevOps Lab 03</span></p>
  <p>This static website is served directly from a Cloud Storage bucket.
     No servers required — just objects in a bucket.</p>
  <ul>
    <li>Zero server management</li>
    <li>Global CDN via Cloud CDN integration</li>
    <li>Custom domain support via CNAME records</li>
    <li>HTTPS via Cloud Load Balancing</li>
  </ul>
</body>
</html>
EOF

cat > 404.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>404 - Page Not Found</title></head>
<body>
  <h1>404 - Page Not Found</h1>
  <p><a href="/">Return to Home</a></p>
</body>
</html>
EOF
```

### 6.2 Create and Configure the Website Bucket

```bash
export WEBSITE_BUCKET="${PROJECT_ID}-static-website"

# Create the website bucket
gsutil mb -p $PROJECT_ID -c STANDARD -l $REGION gs://$WEBSITE_BUCKET/

# Configure as a website (main page + 404 page)
gsutil web set -m index.html -e 404.html gs://$WEBSITE_BUCKET

# Grant public read access to all objects
gsutil iam ch allUsers:objectViewer gs://$WEBSITE_BUCKET

# Upload website files
gsutil cp index.html gs://$WEBSITE_BUCKET/
gsutil cp 404.html gs://$WEBSITE_BUCKET/

# View website configuration
gsutil web get gs://$WEBSITE_BUCKET

echo ""
echo "🌐 Website URL: http://storage.googleapis.com/$WEBSITE_BUCKET/index.html"
echo "   Test it: curl http://storage.googleapis.com/$WEBSITE_BUCKET/index.html"
```

### 6.3 Test the Website

```bash
# Access via storage.googleapis.com
curl -s "http://storage.googleapis.com/$WEBSITE_BUCKET/index.html" | head -10

# Test 404 page
curl -s "http://storage.googleapis.com/$WEBSITE_BUCKET/nonexistent.html"
```

---

## Part 7: Signed URLs

Signed URLs provide time-limited, secure access to objects without requiring authentication.

### 7.1 Generate a Signed URL

```bash
# Signed URLs require a service account key or use impersonation
# For demo purposes using application default credentials:

# First, create a service account for signing
gcloud iam service-accounts create storage-signer \
  --display-name="Storage Signed URL Account"

# Grant the SA storage object viewer
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:storage-signer@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Create a key for the SA (for demo only — prefer Workload Identity in production)
gcloud iam service-accounts keys create /tmp/signer-key.json \
  --iam-account=storage-signer@${PROJECT_ID}.iam.gserviceaccount.com

# Generate a signed URL valid for 1 hour
gsutil signurl -d 1h /tmp/signer-key.json gs://$BUCKET_NAME/hello.txt

# Clean up key immediately after use
rm -f /tmp/signer-key.json
```

**Note:** The signed URL allows anyone with the link to download the file for 1 hour, without needing GCP credentials.

---

## Part 8: Object Metadata and Labels

### 8.1 Set Custom Metadata

```bash
# Set metadata on an object
gsutil setmeta \
  -h "x-goog-meta-environment:production" \
  -h "x-goog-meta-team:devops" \
  -h "Cache-Control:public, max-age=3600" \
  gs://$BUCKET_NAME/hello.txt

# View object metadata
gsutil stat gs://$BUCKET_NAME/hello.txt
```

### 8.2 Bucket Labels

```bash
# Add labels to a bucket
gsutil label ch \
  -l "environment:lab" \
  -l "team:devops" \
  -l "project:lab-03" \
  gs://$BUCKET_NAME

# View bucket labels
gsutil label get gs://$BUCKET_NAME
```

---

## Verification Steps

```bash
# 1. List all lab buckets
gsutil ls | grep devops-lab

# 2. Verify objects exist
gsutil ls -r gs://$BUCKET_NAME/ | head -20

# 3. Verify versioning is enabled
gsutil versioning get gs://$BUCKET_NAME

# 4. Verify lifecycle policy
gsutil lifecycle get gs://$BUCKET_NAME

# 5. Test website (if created)
curl -s -o /dev/null -w "%{http_code}" \
  "http://storage.googleapis.com/$WEBSITE_BUCKET/index.html"
# Expected: 200

echo "✅ Verification complete"
```

---

## Cost Estimates

| Storage Class | Storage | Retrieval | Class A Ops | Class B Ops |
|---------------|---------|-----------|-------------|-------------|
| Standard | $0.020/GB/mo | Free | $0.05/10k | $0.004/10k |
| Nearline | $0.010/GB/mo | $0.01/GB | $0.10/10k | $0.01/10k |
| Coldline | $0.004/GB/mo | $0.02/GB | $0.10/10k | $0.05/10k |
| Archive | $0.0012/GB/mo | $0.05/GB | $0.50/10k | $0.50/10k |

> For this lab with just a few KB of test data, the total cost is less than $0.01.

---

## Cleanup

> ⚠️ Run ALL cleanup commands to avoid ongoing charges.

```bash
export PROJECT_ID=$(gcloud config get-value project)
export BUCKET_NAME="devops-lab-${PROJECT_ID}"
export WEBSITE_BUCKET="${PROJECT_ID}-static-website"

# Remove all objects and delete buckets
gsutil -m rm -r gs://$BUCKET_NAME/ 2>/dev/null || true
gsutil rb gs://$BUCKET_NAME/ 2>/dev/null || true

gsutil -m rm -r gs://${BUCKET_NAME}-nearline/ 2>/dev/null || true
gsutil rb gs://${BUCKET_NAME}-nearline/ 2>/dev/null || true

gsutil -m rm -r gs://${BUCKET_NAME}-coldline/ 2>/dev/null || true
gsutil rb gs://${BUCKET_NAME}-coldline/ 2>/dev/null || true

gsutil -m rm -r gs://$WEBSITE_BUCKET/ 2>/dev/null || true
gsutil rb gs://$WEBSITE_BUCKET/ 2>/dev/null || true

# Delete service account created for signed URLs
gcloud iam service-accounts delete \
  storage-signer@${PROJECT_ID}.iam.gserviceaccount.com --quiet 2>/dev/null || true

# Clean up local files
rm -f hello.txt devops.txt data.json version-test.txt
rm -f hello-downloaded.txt data.json lifecycle.json
rm -f index.html 404.html
rm -rf test-data/ downloaded-data/

echo "✅ Cleanup complete"

# Verify buckets are deleted
gsutil ls | grep devops-lab || echo "All lab buckets deleted"
```

---

## Summary

You have successfully:
- ✅ Created buckets with Standard, Nearline, and Coldline storage classes
- ✅ Uploaded, listed, downloaded, and managed objects with gsutil
- ✅ Configured IAM permissions and uniform bucket-level access
- ✅ Applied lifecycle policies for automated storage class transitions
- ✅ Enabled object versioning and explored version management
- ✅ Hosted a static website on Cloud Storage

**Next Lab:** [LAB-04: Google Kubernetes Engine](LAB-04-gke-basics.md)
