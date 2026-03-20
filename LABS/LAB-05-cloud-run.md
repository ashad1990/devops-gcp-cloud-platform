# LAB-05: Cloud Run Serverless Containers

**Topic:** Serverless Containers  
**Estimated Time:** 90 minutes  
**Difficulty:** 🟡 Intermediate  
**Cost:** ~$0.01 (mostly free tier)

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Build and containerize a Python Flask application for Cloud Run
2. Push container images to Artifact Registry using Cloud Build
3. Deploy and configure serverless containers on Cloud Run
4. Implement canary deployments using traffic splitting across revisions
5. Manage secrets and environment variables securely with Secret Manager
6. Configure concurrency, min/max instances, and CPU throttling settings

---

## Prerequisites

- [LAB-01](LAB-01-gcp-basics.md) completed
- `run.googleapis.com` and `artifactregistry.googleapis.com` APIs enabled
- Docker installed (optional — Cloud Build can be used instead)
- Basic Python knowledge helpful

---

## Lab Overview

Cloud Run is GCP's fully managed serverless platform for containers. It automatically scales from zero to thousands of instances and back, charging only for the exact resources consumed during request processing. In this lab you will build a Python Flask API, containerize it, push it to Artifact Registry, deploy to Cloud Run, implement traffic splitting for canary releases, and manage configuration securely.

---

## Environment Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export SERVICE_NAME=hello-cloud-run
export REPO_NAME=devops-lab-repo

echo "Project:  $PROJECT_ID"
echo "Region:   $REGION"
echo "Service:  $SERVICE_NAME"
```

---

## Part 1: Build a Simple Web Application

### 1.1 Create the Application

```bash
mkdir -p cloud-run-lab && cd cloud-run-lab

cat > app.py << 'EOF'
import os
import platform
from flask import Flask, jsonify, request

app = Flask(__name__)

@app.route('/')
def hello():
    name = os.environ.get('NAME', 'World')
    version = os.environ.get('VERSION', '1.0')
    return jsonify({
        'message': f'Hello, {name}!',
        'version': version,
        'service': 'cloud-run-lab',
        'python': platform.python_version(),
        'hostname': platform.node()
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'service': 'cloud-run-lab'}), 200

@app.route('/env')
def env():
    safe_env = {k: v for k, v in os.environ.items()
                if not any(s in k.upper() for s in ['SECRET', 'PASSWORD', 'KEY', 'TOKEN'])}
    return jsonify(safe_env)

@app.route('/echo', methods=['POST'])
def echo():
    data = request.get_json(silent=True) or {}
    return jsonify({'echo': data, 'received': True})

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port, debug=False)
EOF

cat > requirements.txt << 'EOF'
Flask==3.0.0
gunicorn==21.2.0
EOF

cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PORT=8080

EXPOSE 8080

CMD exec gunicorn --bind :$PORT \
    --workers 1 \
    --threads 8 \
    --timeout 0 \
    --access-logfile - \
    --error-logfile - \
    app:app
EOF

cat > .dockerignore << 'EOF'
.git
*.pyc
__pycache__
*.egg-info
.env
Dockerfile
.dockerignore
README.md
EOF

ls -la
echo "✅ Application files created"
```

### 1.2 Test Locally (Optional)

```bash
# Install dependencies locally and test
pip install -r requirements.txt
python app.py &
APP_PID=$!
sleep 2
curl http://localhost:8080/
curl http://localhost:8080/health
kill $APP_PID
```

---

## Part 2: Artifact Registry and Build

### 2.1 Enable APIs and Create Repository

```bash
# Enable required APIs
gcloud services enable artifactregistry.googleapis.com run.googleapis.com cloudbuild.googleapis.com

# Create Artifact Registry Docker repository
gcloud artifacts repositories create $REPO_NAME \
  --repository-format=docker \
  --location=$REGION \
  --description="DevOps lab container images"

# List repositories
gcloud artifacts repositories list

# Configure Docker authentication for the registry
gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet
```

### 2.2 Build and Push with Cloud Build

```bash
export IMAGE_URL="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${SERVICE_NAME}:v1"
echo "Image URL: $IMAGE_URL"

# Build using Cloud Build (no local Docker required)
gcloud builds submit \
  --tag $IMAGE_URL \
  --timeout=10m \
  .

# List images in the repository
gcloud artifacts docker images list \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}
```

**Expected output:**
```
IMAGE                                                           DIGEST         TAGS  CREATE_TIME
us-central1-docker.pkg.dev/myproject/devops-lab-repo/hello-...  sha256:abc123  v1    2024-01-15T10:30:00
```

### 2.3 Build Locally (Alternative)

```bash
# If you have Docker installed locally:
# docker build -t $IMAGE_URL .
# docker run -p 8080:8080 $IMAGE_URL  # test locally
# docker push $IMAGE_URL
```

---

## Part 3: Deploy to Cloud Run

### 3.1 Deploy the Service

```bash
gcloud run deploy $SERVICE_NAME \
  --image=$IMAGE_URL \
  --region=$REGION \
  --platform=managed \
  --allow-unauthenticated \
  --port=8080 \
  --memory=256Mi \
  --cpu=1 \
  --min-instances=0 \
  --max-instances=10 \
  --concurrency=80 \
  --timeout=60 \
  --set-env-vars="NAME=GCP DevOps,VERSION=1.0"
```

**Expected output:**
```
Deploying container to Cloud Run service [hello-cloud-run] in project [myproject] region [us-central1]
✓ Deploying... Done.
  ✓ Creating Revision...
  ✓ Routing traffic...
  ✓ Setting IAM Policy...
Done.
Service [hello-cloud-run] revision [hello-cloud-run-00001-xyz] has been deployed
Service URL: https://hello-cloud-run-abcd1234-uc.a.run.app
```

### 3.2 Test the Service

```bash
# Get service URL
SERVICE_URL=$(gcloud run services describe $SERVICE_NAME \
  --region=$REGION \
  --format='value(status.url)')

echo "Service URL: $SERVICE_URL"

# Test endpoints
curl -s $SERVICE_URL | python3 -m json.tool
curl -s $SERVICE_URL/health | python3 -m json.tool
curl -s -X POST $SERVICE_URL/echo \
  -H "Content-Type: application/json" \
  -d '{"test": "hello", "lab": "cloud-run"}' | python3 -m json.tool
```

**Expected output for `/`:**
```json
{
    "hostname": "...",
    "message": "Hello, GCP DevOps!",
    "python": "3.11.7",
    "service": "cloud-run-lab",
    "version": "1.0"
}
```

### 3.3 View Service Details

```bash
# Describe the service
gcloud run services describe $SERVICE_NAME --region=$REGION

# List revisions
gcloud run revisions list --service=$SERVICE_NAME --region=$REGION

# View service YAML
gcloud run services describe $SERVICE_NAME --region=$REGION --format=yaml
```

---

## Part 4: Traffic Splitting (Canary Deployments)

### 4.1 Build Version 2

```bash
cat > app.py << 'EOF'
import os
import platform
from flask import Flask, jsonify, request

app = Flask(__name__)

@app.route('/')
def hello():
    name = os.environ.get('NAME', 'World')
    version = os.environ.get('VERSION', '2.0')
    return jsonify({
        'message': f'Hello, {name}! Welcome to v2!',
        'version': version,
        'service': 'cloud-run-lab',
        'python': platform.python_version(),
        'hostname': platform.node(),
        'features': ['new-ui', 'improved-performance', 'enhanced-logging']
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'version': '2.0'}), 200

@app.route('/echo', methods=['POST'])
def echo():
    data = request.get_json(silent=True) or {}
    return jsonify({'echo': data, 'received': True, 'version': '2.0'})

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
EOF

export IMAGE_V2="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${SERVICE_NAME}:v2"
gcloud builds submit --tag $IMAGE_V2 --timeout=10m .
```

### 4.2 Deploy v2 Without Traffic

```bash
# Deploy new version but send no traffic to it yet
gcloud run deploy $SERVICE_NAME \
  --image=$IMAGE_V2 \
  --region=$REGION \
  --no-traffic \
  --set-env-vars="NAME=GCP DevOps,VERSION=2.0"

# List revisions
gcloud run revisions list --service=$SERVICE_NAME --region=$REGION
```

### 4.3 Gradual Traffic Migration

```bash
# Get revision names
REVISION_V1=$(gcloud run revisions list \
  --service=$SERVICE_NAME --region=$REGION \
  --format='value(metadata.name)' | tail -2 | head -1)
REVISION_V2=$(gcloud run revisions list \
  --service=$SERVICE_NAME --region=$REGION \
  --format='value(metadata.name)' | head -1)

echo "v1 revision: $REVISION_V1"
echo "v2 revision: $REVISION_V2"

# Send 10% to v2 (canary)
gcloud run services update-traffic $SERVICE_NAME \
  --region=$REGION \
  --to-revisions=$REVISION_V1=90,$REVISION_V2=10

# Check traffic split
gcloud run services describe $SERVICE_NAME \
  --region=$REGION \
  --format='yaml(status.traffic)'

# Test — run multiple times to see responses from both versions
for i in {1..10}; do
  curl -s $SERVICE_URL | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('version','?'))"
done
```

### 4.4 Promote to 100%

```bash
# Once confident in v2, send all traffic
gcloud run services update-traffic $SERVICE_NAME \
  --region=$REGION \
  --to-latest

# Verify all traffic goes to latest
gcloud run services describe $SERVICE_NAME \
  --region=$REGION \
  --format='yaml(status.traffic)'
```

---

## Part 5: Environment Variables and Secrets

### 5.1 Create a Secret

```bash
gcloud services enable secretmanager.googleapis.com

# Create a secret
echo -n "my-super-secret-api-key-${RANDOM}" | \
  gcloud secrets create api-key --data-file=-

echo -n "db-password-${RANDOM}" | \
  gcloud secrets create db-password --data-file=-

# List secrets
gcloud secrets list

# Get the project number for service account
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
CLOUD_RUN_SA="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"

# Grant Cloud Run access to secrets
gcloud secrets add-iam-policy-binding api-key \
  --member="serviceAccount:${CLOUD_RUN_SA}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:${CLOUD_RUN_SA}" \
  --role="roles/secretmanager.secretAccessor"
```

### 5.2 Deploy with Secrets Mounted

```bash
gcloud run deploy $SERVICE_NAME \
  --image=$IMAGE_V2 \
  --region=$REGION \
  --allow-unauthenticated \
  --set-secrets="API_KEY=api-key:latest,DB_PASS=db-password:latest" \
  --set-env-vars="NAME=GCP DevOps,VERSION=2.0,APP_ENV=production"

# Verify secret is accessible (without exposing value)
curl -s $SERVICE_URL/env | python3 -m json.tool | grep -v "secret\|password\|key" | head -10
```

---

## Part 6: Concurrency and Scaling Configuration

### 6.1 Update Scaling Settings

```bash
gcloud run services update $SERVICE_NAME \
  --region=$REGION \
  --concurrency=80 \
  --min-instances=0 \
  --max-instances=20 \
  --cpu=1 \
  --memory=512Mi \
  --cpu-throttling

# View current configuration
gcloud run services describe $SERVICE_NAME \
  --region=$REGION \
  --format='yaml(spec.template.spec,spec.template.metadata.annotations)'
```

### 6.2 CPU Always Allocated (for background work)

```bash
# This keeps CPU allocated even when not handling requests
# Useful for background jobs, scheduled tasks
gcloud run services update $SERVICE_NAME \
  --region=$REGION \
  --no-cpu-throttling \
  --min-instances=1  # Keep one instance always warm

# Check billing impact
echo "Note: With no-cpu-throttling and min-instances=1, you are billed continuously"
echo "Reset to cost-effective settings:"
gcloud run services update $SERVICE_NAME \
  --region=$REGION \
  --cpu-throttling \
  --min-instances=0
```

### 6.3 Load Testing (Simple)

```bash
# Simple load test with curl
SERVICE_URL=$(gcloud run services describe $SERVICE_NAME \
  --region=$REGION --format='value(status.url)')

echo "Running simple load test against $SERVICE_URL ..."
for i in {1..20}; do
  curl -s -o /dev/null -w "%{http_code} " $SERVICE_URL
done
echo ""

# Check Cloud Run metrics in console
echo "View metrics: https://console.cloud.google.com/run/detail/$REGION/$SERVICE_NAME/metrics?project=$PROJECT_ID"
```

---

## Part 7: Deploying via Source Code

```bash
# Cloud Run can build and deploy directly from source (uses Cloud Build)
gcloud run deploy source-deploy-demo \
  --source=. \
  --region=$REGION \
  --allow-unauthenticated \
  --memory=256Mi

# List services
gcloud run services list --region=$REGION

# Delete the source-deployed service
gcloud run services delete source-deploy-demo --region=$REGION --quiet
```

---

## Verification Steps

```bash
SERVICE_URL=$(gcloud run services describe $SERVICE_NAME \
  --region=$REGION --format='value(status.url)')

# 1. Service is running
gcloud run services list --region=$REGION

# 2. Health check passes
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $SERVICE_URL/health)
echo "Health check HTTP status: $HTTP_STATUS"
[ "$HTTP_STATUS" = "200" ] && echo "✅ Health check passed" || echo "❌ Health check failed"

# 3. Main endpoint responds
curl -s $SERVICE_URL | python3 -m json.tool

# 4. Revisions list
gcloud run revisions list --service=$SERVICE_NAME --region=$REGION

echo "✅ Verification complete"
```

---

## Cost Estimates

| Metric | Free Tier | Price After Free |
|--------|-----------|------------------|
| Requests | 2M/month | $0.40/million |
| CPU time | 180,000 vCPU-sec/month | $0.00002400/vCPU-sec |
| Memory time | 360,000 GiB-sec/month | $0.00000250/GiB-sec |
| Networking egress | 1 GB/month | $0.12/GB |

> For this lab, cost is effectively **$0** within the free tier for typical usage.

---

## Cleanup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export SERVICE_NAME=hello-cloud-run
export REPO_NAME=devops-lab-repo

# Delete Cloud Run service (all revisions)
gcloud run services delete $SERVICE_NAME --region=$REGION --quiet

# Delete Artifact Registry repository and images
gcloud artifacts repositories delete $REPO_NAME --location=$REGION --quiet

# Delete secrets
gcloud secrets delete api-key --quiet 2>/dev/null || true
gcloud secrets delete db-password --quiet 2>/dev/null || true

# Clean up local files
cd ..
rm -rf cloud-run-lab/

echo "✅ Cleanup complete"

# Verify services are deleted
gcloud run services list --region=$REGION
```

---

## Summary

You have successfully:
- ✅ Built a Python Flask application and containerized it with Docker
- ✅ Pushed container images to Artifact Registry using Cloud Build
- ✅ Deployed a serverless container service on Cloud Run
- ✅ Implemented canary deployment with traffic splitting across revisions
- ✅ Managed secrets securely with Secret Manager and Cloud Run integration
- ✅ Configured concurrency, scaling, and CPU throttling settings

**Next Lab:** [LAB-06: Cloud Functions](LAB-06-cloud-functions.md)
