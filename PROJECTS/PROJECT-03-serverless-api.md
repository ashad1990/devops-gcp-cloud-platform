# Project 03: Serverless REST API with Cloud Functions + Firestore + Pub/Sub

## Architecture Overview

```
Client
  │
  ▼
API Gateway ──────────────────────────────────────────────────────┐
  │                                                               │
  ▼                                                               │
Cloud Functions (HTTP)                                            │
  ├── POST /items    → create_item()  ──► Firestore               │
  ├── GET  /items    → list_items()   ──► Firestore               │
  ├── GET  /items/id → get_item()     ──► Firestore               │
  ├── PUT  /items/id → update_item()  ──► Firestore               │
  └── DELETE /items/id→ delete_item() ──► Pub/Sub ─► Worker Fn   │
                                                        │         │
                                               Firestore cleanup  │
                                                                  │
Cloud Scheduler ──► Pub/Sub (cleanup-topic) ──► Cleanup Function ┘

Secret Manager ──► API Key validation (all functions)
```

## Learning Objectives

- Deploy Python Cloud Functions with HTTP triggers
- Integrate Cloud Functions with Firestore
- Use Pub/Sub for asynchronous processing
- Configure API Gateway for unified routing
- Manage secrets with Secret Manager
- Schedule jobs with Cloud Scheduler
- Monitor serverless applications

## Prerequisites

- GCP project with billing enabled
- gcloud CLI installed and authenticated
- Python 3.9+ installed locally
- APIs: cloudfunctions, firestore, pubsub, apigateway, secretmanager, cloudscheduler

## Estimated Cost

| Resource | Monthly Cost |
|---|---|
| Cloud Functions (1M free invocations/month) | ~$0–$5 |
| Firestore (1 GiB free storage) | ~$0–$3 |
| Pub/Sub (10 GiB free) | ~$0–$2 |
| API Gateway | ~$3–$10 |
| Cloud Scheduler (3 free jobs) | ~$0 |
| **Total** | **~$5–$20/month** |

## Setup

```bash
export PROJECT_ID="my-serverless-api"
export REGION="us-central1"

gcloud config set project $PROJECT_ID
gcloud config set run/region $REGION

# Enable APIs
gcloud services enable \
  cloudfunctions.googleapis.com \
  firestore.googleapis.com \
  pubsub.googleapis.com \
  apigateway.googleapis.com \
  secretmanager.googleapis.com \
  cloudscheduler.googleapis.com \
  cloudbuild.googleapis.com
```

## Step 1: Firestore Setup

```bash
# Create Firestore database (native mode)
gcloud firestore databases create --region=$REGION

# Create index for querying items by created_at
cat > /tmp/firestore.indexes.json << 'EOF'
{
  "indexes": [
    {
      "collectionGroup": "items",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "created_at", "order": "DESCENDING" }
      ]
    }
  ]
}
EOF
```

## Step 2: Pub/Sub Topics

```bash
# Topic for delete events (async processing)
gcloud pubsub topics create item-deleted

# Topic for scheduled cleanup
gcloud pubsub topics create cleanup-trigger

# Subscription for delete worker
gcloud pubsub subscriptions create item-deleted-sub \
  --topic=item-deleted \
  --ack-deadline=60

# Subscription for cleanup worker
gcloud pubsub subscriptions create cleanup-trigger-sub \
  --topic=cleanup-trigger \
  --ack-deadline=120
```

## Step 3: Secret Manager

```bash
# Create an API key secret
API_KEY=$(openssl rand -hex 32)
echo -n "$API_KEY" | gcloud secrets create api-key \
  --data-file=- \
  --replication-policy=automatic

echo "Your API key: $API_KEY"

# Grant Cloud Functions SA access to the secret
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
gcloud secrets add-iam-policy-binding api-key \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

## Step 4: Cloud Functions Code

Create the project structure:

```bash
mkdir -p ~/serverless-api/functions/{items,worker,cleanup}
cd ~/serverless-api
```

### 4a. Items API Function (`functions/items/main.py`)

```python
# functions/items/main.py
import json
import os
from datetime import datetime

import functions_framework
from google.cloud import firestore, pubsub_v1, secretmanager

db = firestore.Client()
publisher = pubsub_v1.PublisherClient()


def get_api_key() -> str:
    """Retrieve API key from Secret Manager."""
    client = secretmanager.SecretManagerServiceClient()
    project_id = os.environ.get("GOOGLE_CLOUD_PROJECT")
    name = f"projects/{project_id}/secrets/api-key/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")


def validate_api_key(request) -> bool:
    """Validate the API key from the request header."""
    provided_key = request.headers.get("X-API-Key", "")
    expected_key = get_api_key()
    return provided_key == expected_key


@functions_framework.http
def items_api(request):
    """Handle CRUD operations for items."""
    # CORS headers
    headers = {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
        "Access-Control-Allow-Headers": "Content-Type, X-API-Key",
        "Content-Type": "application/json",
    }

    if request.method == "OPTIONS":
        return ("", 204, headers)

    if not validate_api_key(request):
        return (json.dumps({"error": "Unauthorized"}), 401, headers)

    path_parts = request.path.strip("/").split("/")
    # path: /items or /items/{id}
    item_id = path_parts[1] if len(path_parts) > 1 else None

    if request.method == "GET" and not item_id:
        return list_items(headers)
    elif request.method == "GET" and item_id:
        return get_item(item_id, headers)
    elif request.method == "POST":
        return create_item(request, headers)
    elif request.method == "PUT" and item_id:
        return update_item(item_id, request, headers)
    elif request.method == "DELETE" and item_id:
        return delete_item(item_id, headers)
    else:
        return (json.dumps({"error": "Not Found"}), 404, headers)


def list_items(headers):
    items = []
    docs = db.collection("items").where("status", "==", "active").stream()
    for doc in docs:
        item = doc.to_dict()
        item["id"] = doc.id
        items.append(item)
    return (json.dumps({"items": items, "count": len(items)}), 200, headers)


def get_item(item_id, headers):
    doc = db.collection("items").document(item_id).get()
    if not doc.exists:
        return (json.dumps({"error": "Item not found"}), 404, headers)
    item = doc.to_dict()
    item["id"] = doc.id
    return (json.dumps(item), 200, headers)


def create_item(request, headers):
    data = request.get_json(silent=True)
    if not data or "name" not in data:
        return (json.dumps({"error": "Field 'name' required"}), 400, headers)

    item = {
        "name": data["name"],
        "description": data.get("description", ""),
        "status": "active",
        "created_at": datetime.utcnow().isoformat(),
        "updated_at": datetime.utcnow().isoformat(),
    }
    doc_ref = db.collection("items").add(item)
    item["id"] = doc_ref[1].id
    return (json.dumps(item), 201, headers)


def update_item(item_id, request, headers):
    doc_ref = db.collection("items").document(item_id)
    if not doc_ref.get().exists:
        return (json.dumps({"error": "Item not found"}), 404, headers)

    data = request.get_json(silent=True) or {}
    updates = {k: v for k, v in data.items() if k in ("name", "description", "status")}
    updates["updated_at"] = datetime.utcnow().isoformat()
    doc_ref.update(updates)

    updated = doc_ref.get().to_dict()
    updated["id"] = item_id
    return (json.dumps(updated), 200, headers)


def delete_item(item_id, headers):
    doc_ref = db.collection("items").document(item_id)
    if not doc_ref.get().exists:
        return (json.dumps({"error": "Item not found"}), 404, headers)

    # Publish delete event for async processing
    project_id = os.environ.get("GOOGLE_CLOUD_PROJECT")
    topic_path = publisher.topic_path(project_id, "item-deleted")
    publisher.publish(topic_path, data=item_id.encode("utf-8"))

    # Soft delete
    doc_ref.update({"status": "deleted", "deleted_at": datetime.utcnow().isoformat()})
    return (json.dumps({"message": "Item deleted", "id": item_id}), 200, headers)
```

### 4b. Requirements (`functions/items/requirements.txt`)

```
functions-framework==3.*
google-cloud-firestore==2.*
google-cloud-pubsub==2.*
google-cloud-secret-manager==2.*
```

### 4c. Worker Function (`functions/worker/main.py`)

```python
# functions/worker/main.py - Handles async delete processing
import base64
import json

import functions_framework
from google.cloud import firestore

db = firestore.Client()


@functions_framework.cloud_event
def process_delete(cloud_event):
    """Process item deletion events from Pub/Sub."""
    item_id = base64.b64decode(cloud_event.data["message"]["data"]).decode("utf-8")
    print(f"Processing delete for item: {item_id}")

    # Perform any async cleanup (e.g., delete related records, send notifications)
    related = db.collection("item_metadata").where("item_id", "==", item_id).stream()
    batch = db.batch()
    for doc in related:
        batch.delete(doc.reference)
    batch.commit()

    print(f"Cleanup complete for item: {item_id}")
```

### 4d. Cleanup Function (`functions/cleanup/main.py`)

```python
# functions/cleanup/main.py - Scheduled cleanup of old deleted items
import base64
from datetime import datetime, timedelta

import functions_framework
from google.cloud import firestore

db = firestore.Client()


@functions_framework.cloud_event
def cleanup_deleted_items(cloud_event):
    """Remove items deleted more than 30 days ago."""
    cutoff = (datetime.utcnow() - timedelta(days=30)).isoformat()
    print(f"Cleaning up items deleted before: {cutoff}")

    old_items = (
        db.collection("items")
        .where("status", "==", "deleted")
        .where("deleted_at", "<", cutoff)
        .stream()
    )

    batch = db.batch()
    count = 0
    for doc in old_items:
        batch.delete(doc.reference)
        count += 1

    if count > 0:
        batch.commit()
    print(f"Cleaned up {count} items.")
```

## Step 5: Deploy Cloud Functions

```bash
cd ~/serverless-api

# Deploy items API function
gcloud functions deploy items-api \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=functions/items \
  --entry-point=items_api \
  --trigger-http \
  --allow-unauthenticated \
  --memory=256MB \
  --timeout=60s

# Deploy worker function (Pub/Sub triggered)
gcloud functions deploy item-delete-worker \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=functions/worker \
  --entry-point=process_delete \
  --trigger-topic=item-deleted \
  --memory=256MB \
  --timeout=120s

# Deploy cleanup function (Pub/Sub triggered)
gcloud functions deploy items-cleanup \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=functions/cleanup \
  --entry-point=cleanup_deleted_items \
  --trigger-topic=cleanup-trigger \
  --memory=256MB \
  --timeout=300s

# Get function URL
FUNCTION_URL=$(gcloud functions describe items-api \
  --gen2 --region=$REGION \
  --format='value(serviceConfig.uri)')
echo "Function URL: $FUNCTION_URL"
```

## Step 6: Cloud Scheduler

```bash
# Create App Engine app (required for Cloud Scheduler)
gcloud app create --region=$REGION || true

# Schedule cleanup every day at 2 AM UTC
gcloud scheduler jobs create pubsub daily-cleanup \
  --location=$REGION \
  --schedule="0 2 * * *" \
  --topic=cleanup-trigger \
  --message-body="run" \
  --description="Daily cleanup of deleted items"

# Verify scheduler
gcloud scheduler jobs list --location=$REGION
```

## Step 7: API Gateway

```bash
# Create API config YAML
cat > /tmp/api-config.yaml << EOF
swagger: "2.0"
info:
  title: Items API
  version: "1.0"
basePath: "/"
schemes:
  - https
paths:
  /items:
    get:
      operationId: listItems
      x-google-backend:
        address: ${FUNCTION_URL}
        path_translation: APPEND_PATH_TO_ADDRESS
      responses:
        "200":
          description: List of items
    post:
      operationId: createItem
      x-google-backend:
        address: ${FUNCTION_URL}
        path_translation: APPEND_PATH_TO_ADDRESS
      parameters:
        - in: body
          name: body
          schema:
            type: object
      responses:
        "201":
          description: Created item
  /items/{id}:
    get:
      operationId: getItem
      x-google-backend:
        address: ${FUNCTION_URL}
        path_translation: APPEND_PATH_TO_ADDRESS
      parameters:
        - in: path
          name: id
          required: true
          type: string
      responses:
        "200":
          description: Item details
    put:
      operationId: updateItem
      x-google-backend:
        address: ${FUNCTION_URL}
        path_translation: APPEND_PATH_TO_ADDRESS
      parameters:
        - in: path
          name: id
          required: true
          type: string
        - in: body
          name: body
          schema:
            type: object
      responses:
        "200":
          description: Updated item
    delete:
      operationId: deleteItem
      x-google-backend:
        address: ${FUNCTION_URL}
        path_translation: APPEND_PATH_TO_ADDRESS
      parameters:
        - in: path
          name: id
          required: true
          type: string
      responses:
        "200":
          description: Deleted
EOF

# Create API
gcloud api-gateway apis create items-api --project=$PROJECT_ID

# Create API config
gcloud api-gateway api-configs create items-api-v1 \
  --api=items-api \
  --openapi-spec=/tmp/api-config.yaml \
  --project=$PROJECT_ID

# Deploy gateway
gcloud api-gateway gateways create items-gateway \
  --api=items-api \
  --api-config=items-api-v1 \
  --location=$REGION \
  --project=$PROJECT_ID

GATEWAY_URL=$(gcloud api-gateway gateways describe items-gateway \
  --location=$REGION --format='value(defaultHostname)')
echo "Gateway URL: https://$GATEWAY_URL"
```

## Step 8: Testing

```bash
# Get your API key from Secret Manager
API_KEY=$(gcloud secrets versions access latest --secret=api-key)
BASE_URL="https://$GATEWAY_URL"

# Create an item
curl -s -X POST "$BASE_URL/items" \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Test Item", "description": "My first item"}' | jq .

# List items
curl -s "$BASE_URL/items" -H "X-API-Key: $API_KEY" | jq .

# Get a specific item (replace ITEM_ID)
ITEM_ID="<from create response>"
curl -s "$BASE_URL/items/$ITEM_ID" -H "X-API-Key: $API_KEY" | jq .

# Update an item
curl -s -X PUT "$BASE_URL/items/$ITEM_ID" \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"description": "Updated description"}' | jq .

# Delete an item (triggers async worker via Pub/Sub)
curl -s -X DELETE "$BASE_URL/items/$ITEM_ID" \
  -H "X-API-Key: $API_KEY" | jq .

# Test without API key (should get 401)
curl -s "$BASE_URL/items" | jq .
```

## Step 9: Monitoring Setup

```bash
# View function logs
gcloud functions logs read items-api --gen2 --region=$REGION --limit=50

# Create alerting policy for function errors
cat > /tmp/alert-policy.json << 'EOF'
{
  "displayName": "Cloud Function Error Rate",
  "conditions": [
    {
      "displayName": "Function error rate > 5%",
      "conditionThreshold": {
        "filter": "resource.type=\"cloud_function\" AND metric.type=\"cloudfunctions.googleapis.com/function/execution_count\" AND metric.labels.status!=\"ok\"",
        "aggregations": [
          {
            "alignmentPeriod": "300s",
            "perSeriesAligner": "ALIGN_RATE"
          }
        ],
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0.05,
        "duration": "60s"
      }
    }
  ],
  "alertStrategy": {
    "autoClose": "1800s"
  }
}
EOF

gcloud alpha monitoring policies create --policy-from-file=/tmp/alert-policy.json

# Monitor Pub/Sub message backlog
gcloud monitoring metrics list \
  --filter="metric.type:pubsub" \
  --format="table(metric.type)"
```

## Cleanup

```bash
# Delete Cloud Functions
gcloud functions delete items-api --gen2 --region=$REGION --quiet
gcloud functions delete item-delete-worker --gen2 --region=$REGION --quiet
gcloud functions delete items-cleanup --gen2 --region=$REGION --quiet

# Delete API Gateway resources
gcloud api-gateway gateways delete items-gateway --location=$REGION --quiet
gcloud api-gateway api-configs delete items-api-v1 --api=items-api --quiet
gcloud api-gateway apis delete items-api --quiet

# Delete Pub/Sub resources
gcloud pubsub subscriptions delete item-deleted-sub --quiet
gcloud pubsub subscriptions delete cleanup-trigger-sub --quiet
gcloud pubsub topics delete item-deleted --quiet
gcloud pubsub topics delete cleanup-trigger --quiet

# Delete Scheduler job
gcloud scheduler jobs delete daily-cleanup --location=$REGION --quiet

# Delete Secret
gcloud secrets delete api-key --quiet

# Delete Firestore data (optional - Firestore database cannot be deleted easily)
# gcloud firestore databases delete --database='(default)'

echo "Cleanup complete!"
```
