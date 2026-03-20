# LAB-09: Cloud Build CI/CD Pipelines

**Topic:** Continuous Integration and Continuous Delivery  
**Estimated Time:** 90 minutes  
**Difficulty:** 🟡 Intermediate  
**Cost:** ~$0.50

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Set up Cloud Build with appropriate IAM permissions for deployment
2. Create and manage Artifact Registry Docker repositories
3. Write multi-step `cloudbuild.yaml` configurations with parallel execution
4. Trigger automated builds from source repository changes
5. Deploy applications from Cloud Build to Cloud Run automatically
6. Integrate Pub/Sub notifications for build status monitoring

---

## Prerequisites

- [LAB-01](LAB-01-gcp-basics.md) completed
- `cloudbuild.googleapis.com`, `artifactregistry.googleapis.com`, and `run.googleapis.com` APIs enabled
- Basic understanding of Docker and CI/CD concepts

---

## Lab Overview

Cloud Build is GCP's fully managed CI/CD platform. It executes build steps defined in `cloudbuild.yaml`, integrates with source repositories (GitHub, GitLab, Cloud Source Repositories), and can deploy to any GCP service. In this lab you will build a container, push it to Artifact Registry, and deploy to Cloud Run — all automatically through Cloud Build.

---

## Environment Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export REPO_NAME=devops-builds
export SERVICE_NAME=cloud-build-demo

echo "Project: $PROJECT_ID"
echo "Region:  $REGION"

# Enable all required APIs
gcloud services enable \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  run.googleapis.com \
  sourcerepo.googleapis.com \
  pubsub.googleapis.com

echo "✅ APIs enabled"
```

---

## Part 1: Configure Cloud Build Permissions

### 1.1 Grant Cloud Build Service Account Permissions

```bash
# Get project number and Cloud Build service account
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
CLOUD_BUILD_SA="${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com"

echo "Cloud Build SA: $CLOUD_BUILD_SA"

# Grant Cloud Run deployment permission
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$CLOUD_BUILD_SA" \
  --role="roles/run.admin"

# Grant permission to act as the default compute service account (for Cloud Run deployments)
gcloud iam service-accounts add-iam-policy-binding \
  "${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --member="serviceAccount:$CLOUD_BUILD_SA" \
  --role="roles/iam.serviceAccountUser"

# Grant Artifact Registry write permission
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$CLOUD_BUILD_SA" \
  --role="roles/artifactregistry.admin"

# Grant Secret Manager accessor (for reading secrets in builds)
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$CLOUD_BUILD_SA" \
  --role="roles/secretmanager.secretAccessor"

echo "✅ Cloud Build permissions configured"
```

---

## Part 2: Create Artifact Registry Repository

```bash
# Create Docker repository
gcloud artifacts repositories create $REPO_NAME \
  --repository-format=docker \
  --location=$REGION \
  --description="CI/CD build artifacts and Docker images"

# Configure Docker auth
gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet

# List repositories
gcloud artifacts repositories list --location=$REGION
```

---

## Part 3: Create the Application

### 3.1 Application Code

```bash
mkdir -p cloud-build-lab && cd cloud-build-lab

cat > app.py << 'EOF'
import os
import platform
from flask import Flask, jsonify

app = Flask(__name__)

BUILD_ID = os.environ.get('BUILD_ID', 'local-build')
COMMIT_SHA = os.environ.get('COMMIT_SHA', 'unknown')
IMAGE_TAG = os.environ.get('IMAGE_TAG', 'latest')
APP_VERSION = os.environ.get('APP_VERSION', '1.0.0')

@app.route('/')
def index():
    return jsonify({
        'app': 'cloud-build-demo',
        'version': APP_VERSION,
        'build_id': BUILD_ID,
        'commit': COMMIT_SHA,
        'image_tag': IMAGE_TAG,
        'python': platform.python_version(),
        'status': 'running'
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'build_id': BUILD_ID}), 200

@app.route('/build-info')
def build_info():
    return jsonify({
        'build_id': BUILD_ID,
        'commit_sha': COMMIT_SHA,
        'image_tag': IMAGE_TAG,
        'version': APP_VERSION,
        'environment': os.environ.get('APP_ENV', 'unknown'),
        'platform': platform.platform()
    })

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
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

ARG BUILD_ID=unknown
ARG COMMIT_SHA=unknown

ENV PORT=8080
ENV BUILD_ID=${BUILD_ID}
ENV COMMIT_SHA=${COMMIT_SHA}

EXPOSE 8080

CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 app:app
EOF

cat > .dockerignore << 'EOF'
.git
*.pyc
__pycache__
*.egg-info
.env
tests/
*.md
cloudbuild.yaml
trigger.yaml
EOF
```

### 3.2 Create Tests

```bash
mkdir -p tests

cat > tests/test_app.py << 'EOF'
import pytest
import sys
import os
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
import app as application

@pytest.fixture
def client():
    application.app.config['TESTING'] = True
    with application.app.test_client() as client:
        yield client

def test_index(client):
    response = client.get('/')
    assert response.status_code == 200
    data = response.get_json()
    assert 'app' in data
    assert data['app'] == 'cloud-build-demo'
    assert 'version' in data
    assert 'status' in data
    assert data['status'] == 'running'

def test_health(client):
    response = client.get('/health')
    assert response.status_code == 200
    data = response.get_json()
    assert data['status'] == 'healthy'

def test_build_info(client):
    response = client.get('/build-info')
    assert response.status_code == 200
    data = response.get_json()
    assert 'build_id' in data
    assert 'commit_sha' in data
    assert 'version' in data
EOF

cat > tests/requirements-test.txt << 'EOF'
pytest==7.4.4
Flask==3.0.0
EOF

ls -la tests/
```

---

## Part 4: cloudbuild.yaml Configuration

```bash
cat > cloudbuild.yaml << 'EOF'
# Cloud Build configuration for cloud-build-demo
substitutions:
  _REGION: us-central1
  _REPO: devops-builds
  _SERVICE: cloud-build-demo
  _APP_VERSION: "1.0.0"

steps:
  # Step 1: Install dependencies and run unit tests
  - name: 'python:3.11-slim'
    id: 'install-and-test'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        pip install -q -r requirements.txt
        pip install -q pytest
        python -m pytest tests/ -v --tb=short
        echo "✅ Tests passed"

  # Step 2: Build Docker image with build metadata
  - name: 'gcr.io/cloud-builders/docker'
    id: 'build-image'
    args:
      - 'build'
      - '--build-arg=BUILD_ID=$BUILD_ID'
      - '--build-arg=COMMIT_SHA=$SHORT_SHA'
      - '-t'
      - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}:$SHORT_SHA'
      - '-t'
      - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}:latest'
      - '-t'
      - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}:${_APP_VERSION}'
      - '.'
    waitFor: ['install-and-test']

  # Step 3: Push all image tags
  - name: 'gcr.io/cloud-builders/docker'
    id: 'push-image'
    args:
      - 'push'
      - '--all-tags'
      - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}'
    waitFor: ['build-image']

  # Step 4: Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy-cloud-run'
    entrypoint: 'gcloud'
    args:
      - 'run'
      - 'deploy'
      - '${_SERVICE}'
      - '--image=${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}:$SHORT_SHA'
      - '--region=${_REGION}'
      - '--platform=managed'
      - '--allow-unauthenticated'
      - '--memory=256Mi'
      - '--cpu=1'
      - '--min-instances=0'
      - '--max-instances=10'
      - '--set-env-vars=BUILD_ID=$BUILD_ID,COMMIT_SHA=$SHORT_SHA,IMAGE_TAG=$SHORT_SHA,APP_VERSION=${_APP_VERSION},APP_ENV=production'
    waitFor: ['push-image']

  # Step 5: Run smoke test against deployed service
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'smoke-test'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        SERVICE_URL=$(gcloud run services describe ${_SERVICE} \
          --region=${_REGION} \
          --format='value(status.url)')
        echo "Service URL: $SERVICE_URL"
        sleep 5

        HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$SERVICE_URL/health")
        echo "Health check HTTP status: $HTTP_STATUS"

        if [ "$HTTP_STATUS" != "200" ]; then
          echo "❌ Smoke test FAILED: expected 200, got $HTTP_STATUS"
          exit 1
        fi

        echo "✅ Smoke test PASSED"
        curl -s "$SERVICE_URL" | python3 -m json.tool
    waitFor: ['deploy-cloud-run']

# Image artifacts to store in build record
images:
  - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}:$SHORT_SHA'
  - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}:latest'

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: 'E2_HIGHCPU_8'

timeout: '1200s'
EOF

echo "✅ cloudbuild.yaml created"
```

---

## Part 5: Run the Build

### 5.1 Trigger a Manual Build

```bash
# Initialize git repo (Cloud Build needs SHORT_SHA)
git init
git add .
git commit -m "Initial commit for cloud build lab" --allow-empty

# Submit build manually
gcloud builds submit \
  --config=cloudbuild.yaml \
  --substitutions=SHORT_SHA=$(git rev-parse --short HEAD 2>/dev/null || echo "abc1234"),_APP_VERSION=1.0.0 \
  .

echo "Build submitted! Streaming logs..."
```

### 5.2 Monitor Builds

```bash
# List recent builds
gcloud builds list --limit=5 \
  --format='table(id,status,startTime,duration,substitutions.SHORT_SHA)'

# Get the latest build ID
BUILD_ID=$(gcloud builds list --limit=1 --format='value(id)')
echo "Latest build: $BUILD_ID"

# Describe build in detail
gcloud builds describe $BUILD_ID \
  --format='yaml(status,steps[].name,steps[].status)'

# View build logs
gcloud builds log $BUILD_ID | tail -50
```

**Expected output (abbreviated):**
```
DONE
Step #0 - "install-and-test": ✅ Tests passed
Step #1 - "build-image": Successfully built sha256:abc123...
Step #2 - "push-image": The push refers to repository [...]
Step #3 - "deploy-cloud-run": Service [cloud-build-demo] revision has been deployed
Step #4 - "smoke-test": ✅ Smoke test PASSED
```

### 5.3 Verify Deployment

```bash
# Verify Cloud Run service was deployed
SERVICE_URL=$(gcloud run services describe $SERVICE_NAME \
  --region=$REGION \
  --format='value(status.url)')

echo "Deployed service URL: $SERVICE_URL"
curl -s $SERVICE_URL | python3 -m json.tool
curl -s $SERVICE_URL/build-info | python3 -m json.tool
```

---

## Part 6: Build Triggers

### 6.1 Create a Cloud Source Repository

```bash
# Create a Cloud Source Repository (mirrors or hosts code)
gcloud source repos create devops-lab-repo

# Clone and push code
gcloud source repos clone devops-lab-repo /tmp/devops-lab-repo 2>/dev/null || true
cp -r . /tmp/devops-lab-repo/ 2>/dev/null || true

echo "Source repo: https://source.cloud.google.com/$PROJECT_ID/devops-lab-repo"
```

### 6.2 Create a Build Trigger

```bash
# Create a push trigger for the main branch
gcloud builds triggers create cloud-source-repositories \
  --name=main-branch-trigger \
  --description="Build and deploy on push to main" \
  --repo=devops-lab-repo \
  --branch-pattern='^main$' \
  --build-config=cloudbuild.yaml \
  --substitutions=_APP_VERSION=1.0.0

# List triggers
gcloud builds triggers list \
  --format='table(name,description,createTime,github,triggerTemplate)'

# Describe trigger
gcloud builds triggers describe main-branch-trigger
```

### 6.3 Trigger a Build Manually via Trigger

```bash
# Run the trigger manually (simulates a push)
gcloud builds triggers run main-branch-trigger \
  --branch=main 2>/dev/null || \
  echo "Note: Trigger run requires valid branch in source repo"

# List triggered builds
gcloud builds list --limit=3 \
  --format='table(id,status,substitutions.TRIGGER_NAME,startTime)'
```

---

## Part 7: Build Notifications with Pub/Sub

Cloud Build automatically publishes build events to a Pub/Sub topic.

### 7.1 Subscribe to Build Events

```bash
# Cloud Build publishes to this topic (it's created automatically)
# Create a subscription to receive build notifications
gcloud pubsub subscriptions create build-notifications \
  --topic=cloud-builds \
  --message-retention-duration=1d \
  --ack-deadline=60

echo "Subscription created. Trigger a build to see messages."

# Pull messages (run after triggering a build)
sleep 5
gcloud pubsub subscriptions pull build-notifications \
  --limit=5 \
  --auto-ack \
  --format='json' | python3 -c "
import sys, json, base64
data = json.load(sys.stdin)
for msg in data.get('receivedMessages', []):
    payload = json.loads(base64.b64decode(msg['message']['data']))
    print(f\"Build: {payload.get('id', 'N/A')[:8]}... Status: {payload.get('status', 'N/A')}\")
" 2>/dev/null || echo "No messages yet — trigger a build first"
```

### 7.2 Build Notification Function (Conceptual)

```bash
cat > build-notifier.py << 'EOF'
"""
Example Cloud Function to handle Cloud Build notifications.
Deploy this as a Pub/Sub-triggered function to get build alerts.
"""
import functions_framework
import base64
import json
import logging

logger = logging.getLogger(__name__)

BUILD_STATUS_EMOJIS = {
    'SUCCESS': '✅',
    'FAILURE': '❌',
    'INTERNAL_ERROR': '🔥',
    'TIMEOUT': '⏰',
    'CANCELLED': '🚫',
    'WORKING': '🔨',
    'QUEUED': '⏳',
}

@functions_framework.cloud_event
def handle_build_event(cloud_event):
    """Handle Cloud Build status updates from Pub/Sub."""
    data = cloud_event.data
    
    if "message" in data and "data" in data["message"]:
        payload = json.loads(base64.b64decode(data["message"]["data"]))
        
        build_id = payload.get('id', 'unknown')[:12]
        status = payload.get('status', 'unknown')
        tags = payload.get('tags', [])
        
        emoji = BUILD_STATUS_EMOJIS.get(status, '❓')
        
        logger.info(f"{emoji} Build {build_id} — Status: {status}")
        
        if tags:
            logger.info(f"   Tags: {', '.join(tags)}")
        
        # Send to Slack, PagerDuty, etc.
        if status == 'FAILURE':
            logger.error(f"🚨 Build FAILED! Build ID: {build_id}")
            # trigger_pagerduty_alert(build_id)
        elif status == 'SUCCESS':
            logger.info(f"✅ Build succeeded: {build_id}")
            # send_slack_success_message(build_id)
EOF

echo "Build notifier function template created"
```

---

## Verification Steps

```bash
# 1. Verify Artifact Registry repository
gcloud artifacts repositories describe $REPO_NAME \
  --location=$REGION --format='value(name,format)'

# 2. List images pushed to registry
gcloud artifacts docker images list \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}

# 3. Verify Cloud Run service is running
gcloud run services describe $SERVICE_NAME --region=$REGION \
  --format='value(status.conditions[0].type,status.url)'

# 4. Test deployed service
SERVICE_URL=$(gcloud run services describe $SERVICE_NAME \
  --region=$REGION --format='value(status.url)')
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" $SERVICE_URL/health

# 5. List build history
gcloud builds list --limit=5 --format='table(id[:12],status,startTime)'

echo "✅ Verification complete"
```

---

## Cost Estimates

| Resource | Free Tier | Price After Free |
|----------|-----------|------------------|
| Build minutes (n1-standard-1) | 120 min/day | $0.003/min |
| Build minutes (E2_HIGHCPU_8) | Not included | $0.016/min |
| Artifact Registry storage | 0.5 GB/month | $0.10/GB |
| Cloud Run | 2M requests/month free | $0.40/million |

> A typical lab build (E2_HIGHCPU_8, ~3 minutes) costs approximately **$0.048**.

---

## Cleanup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export REPO_NAME=devops-builds
export SERVICE_NAME=cloud-build-demo

# Delete Cloud Run service
gcloud run services delete $SERVICE_NAME --region=$REGION --quiet 2>/dev/null || true

# Delete build triggers
gcloud builds triggers delete main-branch-trigger --quiet 2>/dev/null || true

# Delete Pub/Sub subscription
gcloud pubsub subscriptions delete build-notifications --quiet 2>/dev/null || true

# Delete Artifact Registry repository (and all images inside)
gcloud artifacts repositories delete $REPO_NAME \
  --location=$REGION --quiet 2>/dev/null || true

# Delete Cloud Source Repository
gcloud source repos delete devops-lab-repo --quiet 2>/dev/null || true
rm -rf /tmp/devops-lab-repo 2>/dev/null || true

# Clean up local files
cd ..
rm -rf cloud-build-lab/

echo "✅ Cleanup complete"

# Verify
gcloud run services list --region=$REGION | grep $SERVICE_NAME || echo "Service deleted"
gcloud artifacts repositories list --location=$REGION | grep $REPO_NAME || echo "Registry deleted"
```

---

## Summary

You have successfully:
- ✅ Configured Cloud Build with appropriate IAM permissions for deployment
- ✅ Created an Artifact Registry repository for Docker images
- ✅ Written a multi-step `cloudbuild.yaml` with tests, build, push, deploy, and smoke test
- ✅ Triggered automated builds and monitored them through Cloud Build logs
- ✅ Created a source repository trigger for automated CI/CD on code push
- ✅ Set up Pub/Sub subscriptions for build event notifications

**Next Lab:** [LAB-10: Cloud Monitoring and Logging](LAB-10-monitoring.md)
