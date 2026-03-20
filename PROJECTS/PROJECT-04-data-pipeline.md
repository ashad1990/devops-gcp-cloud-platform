# Project 04: Data Pipeline with Pub/Sub + Cloud Functions + BigQuery

## Architecture Overview

```
Cloud Scheduler (hourly)
       │
       ▼
   Pub/Sub Topic
  (pipeline-trigger)
       │
       ▼
Cloud Function: extract_and_load()
       │
       ├──► Cloud Storage (raw data staging)
       │
       └──► Cloud Function: transform_and_load()
                   │
                   ▼
              BigQuery
           ├── raw_data table
           ├── transformed table
           └── analytics views

Cloud Monitoring ──► Alerting on pipeline failures
```

## Learning Objectives

- Build event-driven ETL pipelines on GCP
- Use Cloud Functions for data transformation
- Load data into BigQuery programmatically
- Schedule pipelines with Cloud Scheduler + Pub/Sub
- Implement data quality checks
- Monitor pipeline health with Cloud Monitoring

## Prerequisites

- GCP project with billing enabled
- gcloud CLI installed and authenticated
- Python 3.9+ installed locally
- APIs: cloudfunctions, pubsub, bigquery, storage, cloudscheduler, monitoring

## Estimated Cost

| Resource | Monthly Cost |
|---|---|
| Cloud Functions (1M free invocations) | ~$0–$3 |
| Pub/Sub (10 GiB free) | ~$0–$2 |
| BigQuery (10 GiB free storage, 1 TB free queries) | ~$0–$10 |
| Cloud Storage (5 GiB free) | ~$0–$2 |
| Cloud Scheduler (3 free jobs) | ~$0 |
| **Total** | **~$5–$20/month** |

## Setup

```bash
export PROJECT_ID="my-data-pipeline"
export REGION="us-central1"
export DATASET="devops_analytics"
export BUCKET="${PROJECT_ID}-pipeline-staging"

gcloud config set project $PROJECT_ID

# Enable APIs
gcloud services enable \
  cloudfunctions.googleapis.com \
  pubsub.googleapis.com \
  bigquery.googleapis.com \
  storage.googleapis.com \
  cloudscheduler.googleapis.com \
  cloudbuild.googleapis.com \
  monitoring.googleapis.com
```

## Step 1: Cloud Storage Staging Bucket

```bash
# Create staging bucket
gsutil mb -p $PROJECT_ID -c STANDARD -l $REGION gs://$BUCKET

# Set lifecycle to delete files older than 7 days
cat > /tmp/lifecycle.json << 'EOF'
{
  "rule": [
    {
      "action": {"type": "Delete"},
      "condition": {"age": 7}
    }
  ]
}
EOF
gsutil lifecycle set /tmp/lifecycle.json gs://$BUCKET
```

## Step 2: BigQuery Dataset and Tables

```bash
# Create dataset
bq mk --dataset \
  --location=US \
  --description="DevOps analytics pipeline data" \
  ${PROJECT_ID}:${DATASET}

# Create raw data table
bq mk --table \
  ${PROJECT_ID}:${DATASET}.raw_events \
  event_id:STRING,event_type:STRING,user_id:STRING,timestamp:TIMESTAMP,payload:JSON,ingested_at:TIMESTAMP

# Create transformed events table
bq mk --table \
  ${PROJECT_ID}:${DATASET}.events \
  event_id:STRING,event_type:STRING,user_id:STRING,event_date:DATE,event_hour:INTEGER,payload_size:INTEGER,processed_at:TIMESTAMP

# Create pipeline runs audit table
bq mk --table \
  ${PROJECT_ID}:${DATASET}.pipeline_runs \
  run_id:STRING,started_at:TIMESTAMP,completed_at:TIMESTAMP,records_extracted:INTEGER,records_loaded:INTEGER,status:STRING,error_message:STRING

# Create analytics view
bq mk --view \
  ${PROJECT_ID}:${DATASET}.daily_event_summary \
  "SELECT event_date, event_type, COUNT(*) AS event_count, COUNT(DISTINCT user_id) AS unique_users FROM \`${PROJECT_ID}.${DATASET}.events\` GROUP BY event_date, event_type ORDER BY event_date DESC, event_count DESC"
```

## Step 3: Pub/Sub Setup

```bash
# Create trigger topic
gcloud pubsub topics create pipeline-trigger

# Create DLQ topic for failures
gcloud pubsub topics create pipeline-dlq

# Subscription with DLQ
gcloud pubsub subscriptions create pipeline-trigger-sub \
  --topic=pipeline-trigger \
  --ack-deadline=300 \
  --dead-letter-topic=pipeline-dlq \
  --max-delivery-attempts=3
```

## Step 4: Cloud Functions Code

```bash
mkdir -p ~/data-pipeline/functions/{extract,transform}
cd ~/data-pipeline
```

### 4a. Extract Function (`functions/extract/main.py`)

```python
# functions/extract/main.py
import base64
import json
import os
import uuid
from datetime import datetime

import functions_framework
import requests
from google.cloud import bigquery, pubsub_v1, storage

bq = bigquery.Client()
gcs = storage.Client()
publisher = pubsub_v1.PublisherClient()

PROJECT_ID = os.environ.get("GOOGLE_CLOUD_PROJECT")
BUCKET = os.environ.get("STAGING_BUCKET")
DATASET = os.environ.get("BQ_DATASET", "devops_analytics")


@functions_framework.cloud_event
def extract_and_stage(cloud_event):
    """Extract data from source API and stage to GCS + BigQuery raw table."""
    run_id = str(uuid.uuid4())
    started_at = datetime.utcnow()
    records_extracted = 0

    print(f"Pipeline run {run_id} started at {started_at.isoformat()}")

    try:
        # Simulate fetching data from an external API
        # Replace this with your actual data source
        raw_events = fetch_sample_data()
        records_extracted = len(raw_events)

        # Stage raw data to GCS
        timestamp = started_at.strftime("%Y%m%d_%H%M%S")
        gcs_path = f"raw/{timestamp}/events.json"
        bucket = gcs.bucket(BUCKET)
        blob = bucket.blob(gcs_path)
        blob.upload_from_string(
            "\n".join(json.dumps(e) for e in raw_events),
            content_type="application/json",
        )
        print(f"Staged {records_extracted} records to gs://{BUCKET}/{gcs_path}")

        # Load raw data to BigQuery
        load_raw_to_bq(raw_events)

        # Trigger transform function via Pub/Sub
        topic_path = publisher.topic_path(PROJECT_ID, "pipeline-trigger")
        message = {
            "action": "transform",
            "run_id": run_id,
            "gcs_path": f"gs://{BUCKET}/{gcs_path}",
            "record_count": records_extracted,
        }
        publisher.publish(topic_path, data=json.dumps(message).encode("utf-8"))

        # Log successful run
        log_pipeline_run(run_id, started_at, records_extracted, records_extracted, "success")

    except Exception as e:
        print(f"Pipeline run {run_id} failed: {e}")
        log_pipeline_run(run_id, started_at, records_extracted, 0, "failed", str(e))
        raise


def fetch_sample_data():
    """Fetch sample event data. Replace with your actual data source."""
    # Example: fetch from a public API
    response = requests.get(
        "https://jsonplaceholder.typicode.com/posts?_limit=10",
        timeout=30,
    )
    response.raise_for_status()
    posts = response.json()

    events = []
    for post in posts:
        events.append({
            "event_id": str(uuid.uuid4()),
            "event_type": "post_created",
            "user_id": str(post["userId"]),
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "payload": json.dumps({"title": post["title"], "body": post["body"][:100]}),
            "ingested_at": datetime.utcnow().isoformat() + "Z",
        })
    return events


def load_raw_to_bq(events):
    """Load raw events to BigQuery."""
    table_id = f"{PROJECT_ID}.{DATASET}.raw_events"
    errors = bq.insert_rows_json(table_id, events)
    if errors:
        raise RuntimeError(f"BigQuery insert errors: {errors}")
    print(f"Loaded {len(events)} rows to {table_id}")


def log_pipeline_run(run_id, started_at, extracted, loaded, status, error=None):
    """Audit log pipeline run to BigQuery."""
    table_id = f"{PROJECT_ID}.{DATASET}.pipeline_runs"
    row = {
        "run_id": run_id,
        "started_at": started_at.isoformat() + "Z",
        "completed_at": datetime.utcnow().isoformat() + "Z",
        "records_extracted": extracted,
        "records_loaded": loaded,
        "status": status,
        "error_message": error or "",
    }
    bq.insert_rows_json(table_id, [row])
```

### 4b. Extract Requirements (`functions/extract/requirements.txt`)

```
functions-framework==3.*
google-cloud-bigquery==3.*
google-cloud-storage==2.*
google-cloud-pubsub==2.*
requests==2.*
```

### 4c. Transform Function (`functions/transform/main.py`)

```python
# functions/transform/main.py
import base64
import json
import os
from datetime import datetime

import functions_framework
from google.cloud import bigquery

bq = bigquery.Client()

PROJECT_ID = os.environ.get("GOOGLE_CLOUD_PROJECT")
DATASET = os.environ.get("BQ_DATASET", "devops_analytics")


@functions_framework.cloud_event
def transform_and_load(cloud_event):
    """Transform raw events and load to analytics table."""
    # Parse Pub/Sub message
    message_data = base64.b64decode(cloud_event.data["message"]["data"])
    message = json.loads(message_data)

    if message.get("action") != "transform":
        print(f"Skipping message with action: {message.get('action')}")
        return

    run_id = message["run_id"]
    print(f"Transforming data for run {run_id}")

    # Run BigQuery transformation using SQL
    transform_query = f"""
    INSERT INTO `{PROJECT_ID}.{DATASET}.events`
    SELECT
      event_id,
      event_type,
      user_id,
      DATE(TIMESTAMP(timestamp)) AS event_date,
      EXTRACT(HOUR FROM TIMESTAMP(timestamp)) AS event_hour,
      LENGTH(TO_JSON_STRING(JSON_VALUE(payload, '$'))) AS payload_size,
      CURRENT_TIMESTAMP() AS processed_at
    FROM `{PROJECT_ID}.{DATASET}.raw_events`
    WHERE ingested_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
    AND event_id NOT IN (SELECT event_id FROM `{PROJECT_ID}.{DATASET}.events`)
    """

    job_config = bigquery.QueryJobConfig()
    query_job = bq.query(transform_query, job_config=job_config)
    query_job.result()  # Wait for completion

    rows_affected = query_job.num_dml_affected_rows or 0
    print(f"Transform complete: {rows_affected} rows written to events table")

    # Run data quality checks
    run_quality_checks()


def run_quality_checks():
    """Run data quality checks on the events table."""
    checks = [
        # Check for null event IDs
        f"SELECT COUNT(*) as cnt FROM `{PROJECT_ID}.{DATASET}.events` WHERE event_id IS NULL",
        # Check for future timestamps
        f"SELECT COUNT(*) as cnt FROM `{PROJECT_ID}.{DATASET}.events` WHERE event_date > CURRENT_DATE()",
        # Check for duplicate event IDs
        f"SELECT COUNT(*) as cnt FROM (SELECT event_id, COUNT(*) c FROM `{PROJECT_ID}.{DATASET}.events` GROUP BY event_id HAVING c > 1)",
    ]

    check_names = ["null_event_ids", "future_timestamps", "duplicate_event_ids"]

    for name, query in zip(check_names, checks):
        result = list(bq.query(query).result())
        count = result[0].cnt if result else 0
        if count > 0:
            print(f"WARNING: Data quality check '{name}' found {count} issues")
        else:
            print(f"OK: Data quality check '{name}' passed")
```

### 4d. Transform Requirements (`functions/transform/requirements.txt`)

```
functions-framework==3.*
google-cloud-bigquery==3.*
```

## Step 5: Deploy Cloud Functions

```bash
cd ~/data-pipeline

# Deploy extract function
gcloud functions deploy data-extract \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=functions/extract \
  --entry-point=extract_and_stage \
  --trigger-topic=pipeline-trigger \
  --set-env-vars="STAGING_BUCKET=${BUCKET},BQ_DATASET=${DATASET}" \
  --memory=512MB \
  --timeout=300s

# Deploy transform function
gcloud functions deploy data-transform \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=functions/transform \
  --entry-point=transform_and_load \
  --trigger-topic=pipeline-trigger \
  --set-env-vars="BQ_DATASET=${DATASET}" \
  --memory=512MB \
  --timeout=300s

# Verify deployments
gcloud functions list --gen2 --region=$REGION
```

## Step 6: Cloud Scheduler

```bash
# Create App Engine app (required for Cloud Scheduler)
gcloud app create --region=$REGION || true

# Run pipeline every hour
gcloud scheduler jobs create pubsub hourly-pipeline \
  --location=$REGION \
  --schedule="0 * * * *" \
  --topic=pipeline-trigger \
  --message-body='{"action": "extract"}' \
  --description="Hourly data pipeline trigger"

# Test - run pipeline immediately
gcloud scheduler jobs run hourly-pipeline --location=$REGION

# List scheduler jobs
gcloud scheduler jobs list --location=$REGION
```

## Step 7: Testing

```bash
# Manually trigger the pipeline
gcloud pubsub topics publish pipeline-trigger \
  --message='{"action": "extract"}'

# Check function logs
gcloud functions logs read data-extract --gen2 --region=$REGION --limit=30

# Query BigQuery to verify data loaded
bq query --use_legacy_sql=false \
  "SELECT COUNT(*) as total_events, MIN(event_date) as earliest, MAX(event_date) as latest
   FROM \`${PROJECT_ID}.${DATASET}.events\`"

# Check pipeline audit log
bq query --use_legacy_sql=false \
  "SELECT run_id, started_at, records_extracted, records_loaded, status
   FROM \`${PROJECT_ID}.${DATASET}.pipeline_runs\`
   ORDER BY started_at DESC LIMIT 10"

# View daily summary
bq query --use_legacy_sql=false \
  "SELECT * FROM \`${PROJECT_ID}.${DATASET}.daily_event_summary\` LIMIT 20"
```

## Step 8: Monitoring and Alerting

```bash
# View pipeline run success rate
bq query --use_legacy_sql=false \
  "SELECT
     status,
     COUNT(*) as runs,
     AVG(records_loaded) as avg_records
   FROM \`${PROJECT_ID}.${DATASET}.pipeline_runs\`
   WHERE started_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
   GROUP BY status"

# Create alert for pipeline failures using gcloud
gcloud alpha monitoring policies create \
  --display-name="Data Pipeline Failure" \
  --condition-display-name="Function error detected" \
  --condition-filter='resource.type="cloud_function" metric.type="cloudfunctions.googleapis.com/function/execution_count" metric.labels.status!="ok"' \
  --condition-threshold-value=1 \
  --condition-threshold-comparison=COMPARISON_GT \
  --condition-aggregation-period=300s \
  --condition-aggregation-aligner=ALIGN_RATE \
  --notification-channels="" \
  --documentation="Data pipeline function failed. Check logs immediately."

# Check Pub/Sub DLQ for failed messages
gcloud pubsub subscriptions list --filter="topic:pipeline-dlq"
gcloud pubsub subscriptions pull pipeline-dlq --limit=5 --auto-ack
```

## Cleanup

```bash
# Delete Cloud Functions
gcloud functions delete data-extract --gen2 --region=$REGION --quiet
gcloud functions delete data-transform --gen2 --region=$REGION --quiet

# Delete Scheduler job
gcloud scheduler jobs delete hourly-pipeline --location=$REGION --quiet

# Delete Pub/Sub resources
gcloud pubsub subscriptions delete pipeline-trigger-sub --quiet
gcloud pubsub topics delete pipeline-trigger --quiet
gcloud pubsub topics delete pipeline-dlq --quiet

# Delete BigQuery dataset
bq rm -r -f ${PROJECT_ID}:${DATASET}

# Delete GCS staging bucket
gsutil rm -r gs://$BUCKET

echo "Cleanup complete! All pipeline resources deleted."
```
