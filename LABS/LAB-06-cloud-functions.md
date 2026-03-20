# LAB-06: Cloud Functions (FaaS)

**Topic:** Serverless / Event-Driven Functions  
**Estimated Time:** 75 minutes  
**Difficulty:** 🟡 Intermediate  
**Cost:** ~$0.01 (mostly free tier)

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Deploy HTTP-triggered Cloud Functions (Gen 2) using Python
2. Create Pub/Sub-triggered functions for event-driven architectures
3. Build Cloud Storage-triggered functions for automated file processing
4. Configure environment variables and manage function runtime settings
5. Monitor function invocations and troubleshoot using Cloud Logging

---

## Prerequisites

- [LAB-01](LAB-01-gcp-basics.md) completed
- `cloudfunctions.googleapis.com` and `cloudbuild.googleapis.com` APIs enabled
- Basic Python knowledge

---

## Lab Overview

Cloud Functions is GCP's fully managed Functions-as-a-Service (FaaS) platform. Gen 2 functions are built on Cloud Run, offering improved performance, longer timeouts, and more instance concurrency. In this lab you will build HTTP-triggered, Pub/Sub-triggered, and Cloud Storage-triggered functions in Python.

---

## Environment Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1

echo "Project: $PROJECT_ID"
echo "Region:  $REGION"

# Enable required APIs
gcloud services enable \
  cloudfunctions.googleapis.com \
  cloudbuild.googleapis.com \
  pubsub.googleapis.com \
  storage.googleapis.com \
  run.googleapis.com \
  artifactregistry.googleapis.com

mkdir -p functions-lab
echo "✅ Setup complete"
```

---

## Part 1: HTTP-Triggered Function (Python)

### 1.1 Create the Function

```bash
mkdir -p functions-lab/http-function && cd functions-lab/http-function

cat > main.py << 'EOF'
import functions_framework
import json
from datetime import datetime, timezone

@functions_framework.http
def hello_http(request):
    """HTTP Cloud Function responding to GET and POST requests.
    
    Args:
        request (flask.Request): The request object.
    Returns:
        Response tuple: (body, status_code, headers)
    """
    # Parse request body
    request_json = request.get_json(silent=True)
    request_args = request.args

    # Determine name from request
    if request_json and 'name' in request_json:
        name = request_json['name']
    elif request_args and 'name' in request_args:
        name = request_args['name']
    else:
        name = 'World'

    # Build response
    response = {
        'message': f'Hello, {name}!',
        'timestamp': datetime.now(timezone.utc).isoformat(),
        'method': request.method,
        'path': request.path,
        'user_agent': request.headers.get('User-Agent', 'unknown')
    }

    headers = {
        'Content-Type': 'application/json',
        'X-Lab': 'cloud-functions-06'
    }

    return json.dumps(response, default=str), 200, headers
EOF

cat > requirements.txt << 'EOF'
functions-framework==3.5.0
EOF

ls -la
```

### 1.2 Test Locally

```bash
# Test locally with the functions framework
pip install -q functions-framework==3.5.0
functions-framework --target=hello_http --port=8082 &
FF_PID=$!
sleep 2

curl http://localhost:8082/
curl "http://localhost:8082/?name=LocalTest"
curl -X POST http://localhost:8082/ \
  -H "Content-Type: application/json" \
  -d '{"name": "GCP DevOps"}'

kill $FF_PID 2>/dev/null
```

### 1.3 Deploy to Cloud Functions Gen 2

```bash
gcloud functions deploy hello-http \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=. \
  --entry-point=hello_http \
  --trigger-http \
  --allow-unauthenticated \
  --memory=256MB \
  --cpu=1 \
  --timeout=60s \
  --min-instances=0 \
  --max-instances=10 \
  --set-env-vars="APP_ENV=lab,LOG_LEVEL=INFO"

# List functions
gcloud functions list --region=$REGION --gen2
```

**Expected output:**
```
NAME        STATE   TRIGGER      REGION       ENVIRONMENT
hello-http  ACTIVE  HTTP Trigger us-central1  GEN_2
```

### 1.4 Test the Deployed Function

```bash
# Get the function URL
FUNCTION_URL=$(gcloud functions describe hello-http \
  --region=$REGION --gen2 \
  --format='value(serviceConfig.uri)')

echo "Function URL: $FUNCTION_URL"

# Test GET request
curl -s $FUNCTION_URL | python3 -m json.tool

# Test GET with query parameter
curl -s "$FUNCTION_URL?name=CloudFunctions" | python3 -m json.tool

# Test POST request
curl -s -X POST $FUNCTION_URL \
  -H "Content-Type: application/json" \
  -d '{"name": "GCP DevOps Lab"}' | python3 -m json.tool
```

**Expected output:**
```json
{
    "message": "Hello, GCP DevOps Lab!",
    "method": "POST",
    "path": "/",
    "timestamp": "2024-01-15T10:30:00.123456+00:00",
    "user_agent": "curl/8.1.0"
}
```

---

## Part 2: Pub/Sub-Triggered Function

### 2.1 Create the Pub/Sub Topic

```bash
cd ../..

# Create Pub/Sub topic
gcloud pubsub topics create devops-lab-topic
gcloud pubsub topics list | grep devops-lab-topic
```

### 2.2 Create the Function

```bash
mkdir -p functions-lab/pubsub-function && cd functions-lab/pubsub-function

cat > main.py << 'EOF'
import functions_framework
import base64
import json
import logging
from datetime import datetime, timezone

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@functions_framework.cloud_event
def process_pubsub(cloud_event):
    """Cloud Function triggered by a Pub/Sub message.
    
    Args:
        cloud_event (CloudEvent): The CloudEvent containing the Pub/Sub message.
    """
    data = cloud_event.data
    
    logger.info(f"CloudEvent type: {cloud_event['type']}")
    logger.info(f"CloudEvent source: {cloud_event['source']}")
    
    if "message" in data:
        message_data = data["message"]
        
        # Decode base64-encoded message
        if "data" in message_data:
            decoded = base64.b64decode(message_data["data"]).decode("utf-8")
            logger.info(f"Raw message: {decoded}")
            
            # Try to parse as JSON
            try:
                payload = json.loads(decoded)
                logger.info(f"Parsed JSON payload: {json.dumps(payload, indent=2)}")
                
                # Route based on event type
                event_type = payload.get("event", "unknown")
                if event_type == "user_signup":
                    logger.info(f"New user signup: {payload.get('user_id')} - {payload.get('email')}")
                elif event_type == "order_placed":
                    logger.info(f"Order placed: {payload.get('order_id')} - ${payload.get('amount')}")
                elif event_type == "error":
                    logger.error(f"Error event received: {payload.get('message')}")
                else:
                    logger.info(f"Unknown event type: {event_type}")
                    
            except json.JSONDecodeError:
                logger.info(f"Non-JSON message received: {decoded}")
        
        # Log message attributes
        attributes = message_data.get("attributes", {})
        if attributes:
            logger.info(f"Message attributes: {json.dumps(attributes)}")
        
        message_id = message_data.get("messageId", "unknown")
        logger.info(f"Message ID: {message_id}")
    
    logger.info(f"Processing completed at {datetime.now(timezone.utc).isoformat()}")
EOF

cat > requirements.txt << 'EOF'
functions-framework==3.5.0
EOF
```

### 2.3 Deploy and Test

```bash
gcloud functions deploy process-pubsub \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=. \
  --entry-point=process_pubsub \
  --trigger-topic=devops-lab-topic \
  --memory=256MB \
  --timeout=60s \
  --max-instances=10

# Wait for deployment
echo "Waiting for deployment..."
sleep 30

# Publish test messages
gcloud pubsub topics publish devops-lab-topic \
  --message='{"event": "user_signup", "user_id": "usr_123", "email": "alice@example.com"}'

gcloud pubsub topics publish devops-lab-topic \
  --message='{"event": "order_placed", "order_id": "ord_456", "amount": 99.99}'

gcloud pubsub topics publish devops-lab-topic \
  --message='{"event": "error", "message": "Database connection failed", "code": 500}'

# View logs (wait a few seconds for processing)
sleep 15
gcloud functions logs read process-pubsub \
  --region=$REGION --gen2 --limit=30 \
  --format='table(log)'
```

---

## Part 3: Cloud Storage-Triggered Function

### 3.1 Create the Trigger Bucket

```bash
cd ../..
export TRIGGER_BUCKET="${PROJECT_ID}-function-trigger"

gsutil mb -p $PROJECT_ID -l $REGION gs://$TRIGGER_BUCKET/
echo "Created bucket: gs://$TRIGGER_BUCKET/"
```

### 3.2 Create the Function

```bash
mkdir -p functions-lab/storage-function && cd functions-lab/storage-function

cat > main.py << 'EOF'
import functions_framework
import logging
import os
from datetime import datetime, timezone

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# File type routing configuration
FILE_HANDLERS = {
    '.csv': 'data_pipeline',
    '.json': 'json_processor',
    '.jpg': 'image_processor',
    '.jpeg': 'image_processor',
    '.png': 'image_processor',
    '.pdf': 'document_processor',
    '.log': 'log_analyzer',
    '.gz': 'archive_extractor',
}

@functions_framework.cloud_event
def process_gcs_file(cloud_event):
    """Triggered when a file is uploaded to Cloud Storage.
    
    Args:
        cloud_event (CloudEvent): The CloudEvent with Storage event data.
    """
    data = cloud_event.data
    
    bucket = data.get("bucket", "unknown")
    name = data.get("name", "unknown")
    size = data.get("size", "unknown")
    content_type = data.get("contentType", "unknown")
    time_created = data.get("timeCreated", "unknown")
    event_type = cloud_event.get("type", "unknown")
    
    logger.info("=" * 60)
    logger.info(f"GCS Event: {event_type}")
    logger.info(f"Bucket:       {bucket}")
    logger.info(f"File:         {name}")
    logger.info(f"Size:         {size} bytes")
    logger.info(f"Content-Type: {content_type}")
    logger.info(f"Created:      {time_created}")
    logger.info(f"Processed at: {datetime.now(timezone.utc).isoformat()}")
    
    # Route file to appropriate handler
    ext = os.path.splitext(name)[1].lower()
    handler = FILE_HANDLERS.get(ext, 'default_handler')
    logger.info(f"Routing to handler: {handler}")
    
    # Simulate processing based on file type
    if ext == '.csv':
        logger.info(f"CSV detected — would trigger ETL pipeline for: gs://{bucket}/{name}")
    elif ext in ('.jpg', '.jpeg', '.png'):
        logger.info(f"Image detected — would trigger Vision API for: gs://{bucket}/{name}")
    elif ext == '.pdf':
        logger.info(f"PDF detected — would trigger Document AI for: gs://{bucket}/{name}")
    elif ext == '.log':
        logger.info(f"Log file detected — would trigger log parsing for: gs://{bucket}/{name}")
    else:
        logger.info(f"Unrecognized type '{ext}' — applying default processing")
    
    logger.info("=" * 60)
EOF

cat > requirements.txt << 'EOF'
functions-framework==3.5.0
EOF
```

### 3.3 Deploy and Test

```bash
gcloud functions deploy process-gcs-file \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=. \
  --entry-point=process_gcs_file \
  --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
  --trigger-event-filters="bucket=$TRIGGER_BUCKET" \
  --memory=256MB \
  --timeout=60s \
  --max-instances=10

# Upload test files to trigger the function
echo "user_id,name,email" > test-users.csv
echo "1,Alice,alice@example.com" >> test-users.csv
echo "2,Bob,bob@example.com" >> test-users.csv

echo '{"event": "test", "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' > test-event.json

gsutil cp test-users.csv gs://$TRIGGER_BUCKET/uploads/
gsutil cp test-event.json gs://$TRIGGER_BUCKET/uploads/

echo "Waiting for function to process files..."
sleep 20

# View logs
gcloud functions logs read process-gcs-file \
  --region=$REGION --gen2 --limit=40 \
  --format='table(log)'
```

---

## Part 4: Environment Variables

### 4.1 Deploy with Environment Variables

```bash
cd ../..

gcloud functions deploy hello-http \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=functions-lab/http-function \
  --entry-point=hello_http \
  --trigger-http \
  --allow-unauthenticated \
  --set-env-vars="APP_ENV=production,LOG_LEVEL=INFO,MAX_RETRIES=3,FEATURE_FLAG_V2=true"

# Verify environment variables
gcloud functions describe hello-http --region=$REGION --gen2 \
  --format='yaml(serviceConfig.environmentVariables)'
```

**Expected output:**
```yaml
serviceConfig:
  environmentVariables:
    APP_ENV: production
    FEATURE_FLAG_V2: 'true'
    LOG_LEVEL: INFO
    MAX_RETRIES: '3'
```

### 4.2 Update a Single Environment Variable

```bash
# Update environment variables without redeploying code
gcloud functions deploy hello-http \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=functions-lab/http-function \
  --entry-point=hello_http \
  --trigger-http \
  --allow-unauthenticated \
  --update-env-vars="APP_ENV=staging,LOG_LEVEL=DEBUG"
```

---

## Part 5: Monitoring and Logging

### 5.1 View Function Logs

```bash
# View recent logs for a function
gcloud functions logs read hello-http \
  --region=$REGION --gen2 \
  --limit=50 \
  --format='table(time_utc,log)'

# Filter logs by severity
gcloud logging read \
  "resource.type=cloud_run_revision \
  AND resource.labels.service_name=hello-http \
  AND severity>=WARNING" \
  --limit=20 \
  --format='table(timestamp,severity,textPayload)'

# Stream logs in real time (invoke function in another terminal)
# gcloud alpha functions logs tail hello-http --region=$REGION --gen2
```

### 5.2 Function Metrics

```bash
# View function execution count
gcloud monitoring time-series list \
  --filter='metric.type="cloudfunctions.googleapis.com/function/execution_count"' \
  --interval-start-time="$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v-1H +%Y-%m-%dT%H:%M:%SZ)" \
  --interval-end-time="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --format='table(metric.labels,points[0].value)'

# View in console
echo "Console metrics: https://console.cloud.google.com/functions/details/$REGION/hello-http?project=$PROJECT_ID&tab=metrics"
```

### 5.3 Generate Load and View Metrics

```bash
FUNCTION_URL=$(gcloud functions describe hello-http \
  --region=$REGION --gen2 \
  --format='value(serviceConfig.uri)')

# Generate some traffic
echo "Generating function traffic..."
for i in {1..15}; do
  curl -s -o /dev/null "$FUNCTION_URL?name=User$i" &
done
wait

echo "Traffic generation complete. Check metrics in the console."
```

---

## Part 6: Scheduled Functions (Cloud Scheduler)

### 6.1 Create a Scheduled Function Trigger

```bash
gcloud services enable cloudscheduler.googleapis.com

# Create a Cloud Scheduler job to invoke the function hourly
# (Function must exist — we'll use hello-http)
FUNCTION_URL=$(gcloud functions describe hello-http \
  --region=$REGION --gen2 \
  --format='value(serviceConfig.uri)')

# Get service account for scheduler
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')

gcloud scheduler jobs create http scheduled-hello \
  --location=$REGION \
  --schedule="*/15 * * * *" \
  --uri="$FUNCTION_URL?name=Scheduler" \
  --http-method=GET \
  --oidc-service-account-email="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --description="Invoke hello-http every 15 minutes"

# List scheduler jobs
gcloud scheduler jobs list --location=$REGION

# Manually trigger the job
gcloud scheduler jobs run scheduled-hello --location=$REGION

# Pause the job (to avoid unnecessary invocations)
gcloud scheduler jobs pause scheduled-hello --location=$REGION
```

---

## Verification Steps

```bash
# 1. List all deployed functions
gcloud functions list --region=$REGION --gen2

# 2. Verify HTTP function responds
FUNCTION_URL=$(gcloud functions describe hello-http \
  --region=$REGION --gen2 --format='value(serviceConfig.uri)')
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$FUNCTION_URL?name=Verify")
echo "HTTP function status: $HTTP_STATUS"
[ "$HTTP_STATUS" = "200" ] && echo "✅ HTTP function OK" || echo "❌ HTTP function failed"

# 3. Verify Pub/Sub function exists
gcloud functions describe process-pubsub --region=$REGION --gen2 \
  --format='value(state)'

# 4. Verify Storage function exists
gcloud functions describe process-gcs-file --region=$REGION --gen2 \
  --format='value(state)'

# 5. View all function states
gcloud functions list --region=$REGION --gen2 \
  --format='table(name,state,trigger,runtime)'

echo "✅ Verification complete"
```

---

## Cost Estimates

| Resource | Free Tier | Price After Free |
|----------|-----------|------------------|
| Invocations | 2M/month | $0.40/million |
| Compute time (CPU) | 400,000 GHz-sec/month | $0.00001650/GHz-sec |
| Compute time (memory) | 200,000 GB-sec/month | $0.00000250/GiB-sec |
| Outbound networking | 5 GB/month | $0.12/GB |

> For this lab with a few dozen test invocations, the cost is effectively **$0** within the free tier.

---

## Cleanup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export TRIGGER_BUCKET="${PROJECT_ID}-function-trigger"

# Delete Cloud Functions
gcloud functions delete hello-http --region=$REGION --gen2 --quiet
gcloud functions delete process-pubsub --region=$REGION --gen2 --quiet
gcloud functions delete process-gcs-file --region=$REGION --gen2 --quiet

# Delete Pub/Sub topic
gcloud pubsub topics delete devops-lab-topic --quiet 2>/dev/null || true

# Delete Cloud Scheduler job
gcloud scheduler jobs delete scheduled-hello \
  --location=$REGION --quiet 2>/dev/null || true

# Delete GCS trigger bucket
gsutil -m rm -r gs://$TRIGGER_BUCKET/ 2>/dev/null || true
gsutil rb gs://$TRIGGER_BUCKET/ 2>/dev/null || true

# Clean up local files
cd ..
rm -rf functions-lab/
rm -f test-users.csv test-event.json

echo "✅ Cleanup complete"

# Verify all functions are deleted
gcloud functions list --region=$REGION --gen2
```

---

## Summary

You have successfully:
- ✅ Deployed an HTTP-triggered Cloud Function (Gen 2) with Python
- ✅ Created a Pub/Sub-triggered function with JSON message routing
- ✅ Built a Cloud Storage-triggered function for automated file processing
- ✅ Configured environment variables and managed runtime settings
- ✅ Monitored function invocations and logs using Cloud Logging
- ✅ Created a Cloud Scheduler job for periodic function execution

**Next Lab:** [LAB-07: VPC Networking and Load Balancers](LAB-07-networking.md)
