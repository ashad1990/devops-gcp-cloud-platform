# LAB-10: Cloud Monitoring and Logging

**Topic:** Observability — Monitoring, Logging, and Alerting  
**Estimated Time:** 75 minutes  
**Difficulty:** 🟡 Intermediate  
**Cost:** Minimal (~$0.05)

---

## Learning Objectives

By the end of this lab, you will be able to:

1. View, filter, and write structured log entries using Cloud Logging
2. Create log sinks to export logs to Cloud Storage and BigQuery
3. Define log-based metrics for tracking application errors and events
4. Build custom dashboards with Cloud Monitoring charts and widgets
5. Create alerting policies with notification channels for incident response
6. Configure uptime checks and understand Cloud Trace basics

---

## Prerequisites

- [LAB-01](LAB-01-gcp-basics.md) completed
- `logging.googleapis.com` and `monitoring.googleapis.com` APIs enabled
- A running GCP resource for monitoring (complete LAB-02 first for VM metrics, or use Cloud Run from LAB-05)

---

## Lab Overview

Observability is a critical pillar of production systems. GCP's Operations Suite (formerly Stackdriver) provides integrated logging, monitoring, tracing, and error reporting. In this lab you will explore Cloud Logging for log management, create dashboards and alerting in Cloud Monitoring, set up log exports and metrics, and configure uptime checks for service availability.

---

## Environment Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export LOGS_BUCKET="${PROJECT_ID}-logs-export"

echo "Project: $PROJECT_ID"

# Enable all observability APIs
gcloud services enable \
  logging.googleapis.com \
  monitoring.googleapis.com \
  cloudtrace.googleapis.com \
  clouderrorreporting.googleapis.com \
  cloudprofiler.googleapis.com

echo "✅ Observability APIs enabled"
```

---

## Part 1: Cloud Logging — Viewing and Writing Logs

### 1.1 View Recent Logs

```bash
# View all recent logs
gcloud logging read "severity>=INFO" \
  --limit=20 \
  --format='table(timestamp,severity,resource.type,textPayload)' 2>/dev/null | head -30

# View logs by resource type
gcloud logging read 'resource.type="gce_instance"' \
  --limit=10 \
  --format='table(timestamp,resource.labels.instance_id,textPayload)' 2>/dev/null | head -20

# View only error logs
gcloud logging read "severity>=ERROR" \
  --limit=10 \
  --format='table(timestamp,severity,resource.type,textPayload)' 2>/dev/null | head -20
```

### 1.2 Write Log Entries

```bash
# Write a structured JSON log entry
gcloud logging write devops-lab-log \
  '{"message": "Lab 10 started", "severity": "INFO", "component": "lab-setup", "lab": "10", "user": "devops-student"}' \
  --payload-type=json \
  --severity=INFO

# Write a warning
gcloud logging write devops-lab-log \
  '{"message": "High memory usage detected", "severity": "WARNING", "component": "monitoring", "memoryUsagePct": 85}' \
  --payload-type=json \
  --severity=WARNING

# Write an error
gcloud logging write devops-lab-log \
  '{"message": "Database connection failed", "severity": "ERROR", "component": "database", "error_code": "CONN_TIMEOUT", "retries": 3}' \
  --payload-type=json \
  --severity=ERROR

# Write a plain text log
gcloud logging write devops-lab-log \
  "Simple text log entry from devops lab - $(date)" \
  --severity=INFO

echo "✅ Log entries written"
```

### 1.3 Read Your Log Entries

```bash
# Read entries from your custom log
gcloud logging read "logName=\"projects/${PROJECT_ID}/logs/devops-lab-log\"" \
  --limit=10 \
  --format='table(timestamp,severity,jsonPayload.message,textPayload)'
```

**Expected output:**
```
TIMESTAMP                      SEVERITY  JSON_PAYLOAD.MESSAGE           TEXT_PAYLOAD
2024-01-15T10:35:00.000000Z    ERROR     Database connection failed
2024-01-15T10:34:30.000000Z    WARNING   High memory usage detected
2024-01-15T10:34:00.000000Z    INFO      Lab 10 started
2024-01-15T10:33:30.000000Z    INFO                                     Simple text log entry...
```

### 1.4 Advanced Log Filtering

```bash
# Filter by time range (last 30 minutes)
gcloud logging read \
  "logName=\"projects/${PROJECT_ID}/logs/devops-lab-log\" AND timestamp>=\"$(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v-30M +%Y-%m-%dT%H:%M:%SZ)\"" \
  --limit=20

# Filter by JSON field
gcloud logging read \
  "logName=\"projects/${PROJECT_ID}/logs/devops-lab-log\" AND jsonPayload.component=\"database\"" \
  --limit=5

# Filter multiple conditions
gcloud logging read \
  "severity=ERROR AND timestamp>=\"$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v-1H +%Y-%m-%dT%H:%M:%SZ)\"" \
  --limit=10 \
  --format='table(timestamp,severity,resource.type,jsonPayload.message,textPayload)'
```

---

## Part 2: Log Sinks (Exporting Logs)

Log sinks route copies of logs to external destinations for long-term storage, analysis, or compliance.

### 2.1 Create a GCS Bucket for Log Export

```bash
# Create bucket for log archives
gsutil mb -l us-central1 gs://$LOGS_BUCKET/
gsutil versioning set on gs://$LOGS_BUCKET/

echo "Logs bucket: gs://$LOGS_BUCKET/"
```

### 2.2 Create a Log Sink to Cloud Storage

```bash
# Create a sink for all audit logs
gcloud logging sinks create devops-audit-sink \
  "storage.googleapis.com/${LOGS_BUCKET}" \
  --log-filter='logName:"cloudaudit.googleapis.com"' \
  --description="Export audit logs to GCS"

# Get the service account automatically created for the sink
SINK_SA=$(gcloud logging sinks describe devops-audit-sink \
  --format='value(writerIdentity)')
echo "Sink Service Account: $SINK_SA"

# Grant the sink SA permission to write to the bucket
gsutil iam ch ${SINK_SA}:roles/storage.objectCreator gs://$LOGS_BUCKET/

echo "✅ Audit log sink created"
```

### 2.3 Create a Log Sink for Application Errors

```bash
# Create bucket for application error logs
ERROR_BUCKET="${PROJECT_ID}-error-logs"
gsutil mb -l us-central1 gs://$ERROR_BUCKET/

gcloud logging sinks create devops-error-sink \
  "storage.googleapis.com/${ERROR_BUCKET}" \
  --log-filter='severity>=ERROR' \
  --description="Export all error and critical logs to GCS"

# Grant permissions
ERROR_SINK_SA=$(gcloud logging sinks describe devops-error-sink \
  --format='value(writerIdentity)')
gsutil iam ch ${ERROR_SINK_SA}:roles/storage.objectCreator gs://$ERROR_BUCKET/

# List all sinks
gcloud logging sinks list
```

**Expected output:**
```
NAME                SERVICE                  DESTINATION
devops-audit-sink   logging.googleapis.com   storage.googleapis.com/myproject-logs-export
devops-error-sink   logging.googleapis.com   storage.googleapis.com/myproject-error-logs
```

---

## Part 3: Log-Based Metrics

Create custom metrics based on log content for monitoring and alerting.

### 3.1 Counter Metrics

```bash
# Count all ERROR-level log entries
gcloud logging metrics create error-count-metric \
  --description="Count of ERROR severity log entries" \
  --log-filter='severity=ERROR'

# Count HTTP 500 errors from Cloud Run
gcloud logging metrics create app-500-errors \
  --description="Count of HTTP 500 errors from Cloud Run" \
  --log-filter='resource.type="cloud_run_revision" AND httpRequest.status=500'

# Count database error events
gcloud logging metrics create db-connection-errors \
  --description="Count of database connection errors" \
  --log-filter="logName=\"projects/${PROJECT_ID}/logs/devops-lab-log\" AND jsonPayload.component=\"database\" AND severity=ERROR"

# List all log-based metrics
gcloud logging metrics list
```

### 3.2 Distribution Metric (Latency)

```bash
# Create a distribution metric for response latency
gcloud logging metrics create http-latency-distribution \
  --description="Distribution of HTTP response latency" \
  --log-filter='resource.type="cloud_run_revision" AND httpRequest.latency!=""' \
  --value-extractor='EXTRACT(httpRequest.latency)'
```

### 3.3 Describe a Metric

```bash
gcloud logging metrics describe error-count-metric
gcloud logging metrics describe db-connection-errors
```

---

## Part 4: Cloud Monitoring Dashboards

### 4.1 Create a Custom Dashboard

```bash
cat > dashboard.json << 'EOF'
{
  "displayName": "DevOps Lab 10 Dashboard",
  "gridLayout": {
    "columns": "2",
    "widgets": [
      {
        "title": "Log-Based Error Count",
        "xyChart": {
          "dataSets": [
            {
              "timeSeriesQuery": {
                "timeSeriesFilter": {
                  "filter": "metric.type=\"logging.googleapis.com/user/error-count-metric\" resource.type=\"global\"",
                  "aggregation": {
                    "alignmentPeriod": "60s",
                    "perSeriesAligner": "ALIGN_RATE"
                  }
                },
                "unitOverride": "1"
              },
              "plotType": "LINE",
              "legendTemplate": "Error Rate"
            }
          ],
          "yAxis": {
            "label": "Errors/sec",
            "scale": "LINEAR"
          }
        }
      },
      {
        "title": "Database Connection Errors",
        "xyChart": {
          "dataSets": [
            {
              "timeSeriesQuery": {
                "timeSeriesFilter": {
                  "filter": "metric.type=\"logging.googleapis.com/user/db-connection-errors\" resource.type=\"global\"",
                  "aggregation": {
                    "alignmentPeriod": "300s",
                    "perSeriesAligner": "ALIGN_COUNT"
                  }
                }
              },
              "plotType": "STACKED_BAR",
              "legendTemplate": "DB Errors"
            }
          ]
        }
      },
      {
        "title": "GCE CPU Utilization",
        "xyChart": {
          "dataSets": [
            {
              "timeSeriesQuery": {
                "timeSeriesFilter": {
                  "filter": "metric.type=\"compute.googleapis.com/instance/cpu/utilization\" resource.type=\"gce_instance\"",
                  "aggregation": {
                    "alignmentPeriod": "60s",
                    "perSeriesAligner": "ALIGN_MEAN",
                    "crossSeriesReducer": "REDUCE_MEAN"
                  }
                }
              },
              "plotType": "LINE",
              "legendTemplate": "Avg CPU"
            }
          ],
          "yAxis": {
            "label": "CPU Utilization",
            "scale": "LINEAR"
          }
        }
      },
      {
        "title": "Recent Logs",
        "logsPanel": {
          "filter": "severity>=WARNING",
          "resourceNames": ["projects/PROJECT_ID_PLACEHOLDER"]
        }
      }
    ]
  }
}
EOF

# Replace placeholder
sed -i "s/PROJECT_ID_PLACEHOLDER/$PROJECT_ID/g" dashboard.json

# Create the dashboard
gcloud monitoring dashboards create --config-from-file=dashboard.json

# List dashboards
gcloud monitoring dashboards list \
  --format='table(displayName,name)'
```

---

## Part 5: Alerting Policies

### 5.1 Create a Notification Channel

```bash
# Create an email notification channel
cat > notification-channel.json << 'EOF'
{
  "type": "email",
  "displayName": "DevOps Lab Alert Email",
  "description": "Email notifications for DevOps lab alerts",
  "labels": {
    "email_address": "your-email@example.com"
  },
  "enabled": true
}
EOF

# Create the channel (replace email before running in real setup)
CHANNEL_ID=$(gcloud alpha monitoring channels create \
  --channel-content-from-file=notification-channel.json \
  --format='value(name)' 2>/dev/null || echo "")

if [ -n "$CHANNEL_ID" ]; then
  echo "Notification Channel: $CHANNEL_ID"
else
  echo "Note: Creating notification channels requires alpha component"
  echo "Install with: gcloud components install alpha"
fi

# List notification channels
gcloud alpha monitoring channels list \
  --format='table(displayName,type,name)' 2>/dev/null || echo "Alpha component not available"
```

### 5.2 Create an Alerting Policy for High Error Rate

```bash
cat > alert-policy.json << 'EOF'
{
  "displayName": "High Error Rate Alert",
  "documentation": {
    "content": "Alert fires when error count exceeds 5 errors in 5 minutes. Check application logs for details.",
    "mimeType": "text/markdown"
  },
  "conditions": [
    {
      "displayName": "Error count > 5 in 5 minutes",
      "conditionThreshold": {
        "filter": "metric.type=\"logging.googleapis.com/user/error-count-metric\" AND resource.type=\"global\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 5.0,
        "duration": "300s",
        "aggregations": [
          {
            "alignmentPeriod": "300s",
            "perSeriesAligner": "ALIGN_COUNT"
          }
        ]
      }
    }
  ],
  "alertStrategy": {
    "autoClose": "604800s"
  },
  "combiner": "OR",
  "enabled": true,
  "severity": "WARNING"
}
EOF

# Create the alerting policy
POLICY_NAME=$(gcloud alpha monitoring policies create \
  --policy-from-file=alert-policy.json \
  --format='value(name)' 2>/dev/null || echo "")

if [ -n "$POLICY_NAME" ]; then
  echo "Alert policy created: $POLICY_NAME"
else
  echo "Note: Creating alert policies requires alpha component"
  echo "Alternatively, create policies in the GCP Console under Monitoring > Alerting"
fi
```

### 5.3 Create a VM CPU Alert

```bash
cat > cpu-alert-policy.json << 'EOF'
{
  "displayName": "High CPU Utilization Alert",
  "documentation": {
    "content": "VM CPU utilization has exceeded 80% for more than 5 minutes.",
    "mimeType": "text/markdown"
  },
  "conditions": [
    {
      "displayName": "CPU utilization > 80%",
      "conditionThreshold": {
        "filter": "metric.type=\"compute.googleapis.com/instance/cpu/utilization\" AND resource.type=\"gce_instance\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0.8,
        "duration": "300s",
        "aggregations": [
          {
            "alignmentPeriod": "60s",
            "perSeriesAligner": "ALIGN_MEAN"
          }
        ]
      }
    }
  ],
  "alertStrategy": {
    "autoClose": "604800s"
  },
  "combiner": "OR",
  "enabled": true,
  "severity": "CRITICAL"
}
EOF

gcloud alpha monitoring policies create \
  --policy-from-file=cpu-alert-policy.json \
  --format='value(name)' 2>/dev/null || echo "Policy definition saved (create via Console if alpha unavailable)"

# List alerting policies
gcloud alpha monitoring policies list \
  --format='table(displayName,severity,enabled)' 2>/dev/null | head -10 || \
  echo "View policies at: https://console.cloud.google.com/monitoring/alerting?project=$PROJECT_ID"
```

---

## Part 6: Uptime Checks

```bash
# Create an uptime check for a public URL
gcloud monitoring uptime create \
  --display-name="Google.com Availability Check" \
  --resource-type=uptime_url \
  --resource-labels=project_id=$PROJECT_ID,host=www.google.com \
  --http-check-path=/ \
  --period=60 \
  --timeout=10 \
  --regions=USA,EUROPE,ASIA_PACIFIC 2>/dev/null || echo "Uptime check: use Console or alpha CLI"

# List uptime checks
gcloud monitoring uptime list 2>/dev/null || \
  echo "View uptime checks: https://console.cloud.google.com/monitoring/uptime?project=$PROJECT_ID"
```

---

## Part 7: Cloud Trace

Cloud Trace provides distributed tracing for understanding request latency.

### 7.1 View Traces (Application Integration Required)

```bash
gcloud services enable cloudtrace.googleapis.com

# Trace data is typically sent from application code using:
# - OpenTelemetry SDK
# - Cloud Trace client libraries
# - Automatic instrumentation (Cloud Run, GKE with auto-injection)

# View recent traces for Cloud Run services
echo "View traces: https://console.cloud.google.com/traces/list?project=$PROJECT_ID"
echo ""
echo "For Cloud Run services, tracing is automatically captured."
echo "Add these headers to trace specific requests:"
echo "  X-Cloud-Trace-Context: TRACE_ID/SPAN_ID;o=TRACE_TRUE"
```

### 7.2 Python Trace Example

```bash
cat > trace-example.py << 'EOF'
"""
Example: Instrumenting a Flask app with OpenTelemetry for Cloud Trace.
Add to requirements.txt:
    opentelemetry-api==1.21.0
    opentelemetry-sdk==1.21.0
    opentelemetry-exporter-gcp-trace==1.6.0
    opentelemetry-instrumentation-flask==0.42b0
"""
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
# from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter

# Setup (in production, uncomment CloudTraceSpanExporter):
# exporter = CloudTraceSpanExporter(project_id="your-project-id")
# provider = TracerProvider()
# provider.add_span_processor(BatchSpanProcessor(exporter))
# trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

def process_request(request_id: str):
    with tracer.start_as_current_span("process-request") as span:
        span.set_attribute("request.id", request_id)
        
        with tracer.start_as_current_span("validate-input"):
            # Validation logic here
            pass
        
        with tracer.start_as_current_span("database-query"):
            # Database query here
            pass
        
        return {"processed": True, "request_id": request_id}

print("Trace example code created (see trace-example.py)")
EOF
echo "✅ Trace example created"
```

---

## Part 8: Error Reporting

```bash
gcloud services enable clouderrorreporting.googleapis.com

# Report a test error
gcloud error-reporting events report \
  --service=devops-lab-service \
  --service-version=v1.0.0 \
  "Test NullPointerException at com.example.Service.processRequest(Service.java:42)"

# List error groups
gcloud beta error-reporting events list --limit=5 2>/dev/null || \
  echo "View errors: https://console.cloud.google.com/errors?project=$PROJECT_ID"

echo ""
echo "Error reporting console: https://console.cloud.google.com/errors?project=$PROJECT_ID"
```

---

## Verification Steps

```bash
# 1. Verify log entries exist
gcloud logging read "logName=\"projects/${PROJECT_ID}/logs/devops-lab-log\"" \
  --limit=5 --format='table(timestamp,severity)'

# 2. Verify log sinks
gcloud logging sinks list --format='table(name,destination)'

# 3. Verify log-based metrics
gcloud logging metrics list --format='table(name,description)'

# 4. Verify dashboard was created
gcloud monitoring dashboards list --format='table(displayName)' | grep "DevOps Lab"

# 5. Trigger some log events to see metrics populate
for i in {1..5}; do
  gcloud logging write devops-lab-log \
    "{\"message\": \"Verification test $i\", \"component\": \"verify\"}" \
    --payload-type=json \
    --severity=INFO
done

echo "✅ Verification complete"
echo "Monitoring console: https://console.cloud.google.com/monitoring?project=$PROJECT_ID"
echo "Logging console:    https://console.cloud.google.com/logs/query?project=$PROJECT_ID"
```

---

## Cost Estimates

| Resource | Free Tier | Price After |
|----------|-----------|-------------|
| Log ingestion | 50 GB/month | $0.50/GB |
| Log storage (_Default bucket) | 30 days free | $0.01/GB/month |
| Log-based metrics | First 10 metrics free | $0.01/metric/month |
| Monitoring data | First 150 MB/month free | $0.18/100 MB |
| Alerting policies | First 5 free | $0.10/policy/month |
| Uptime checks | First 3 free | $0.30/check/month |

---

## Cleanup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export LOGS_BUCKET="${PROJECT_ID}-logs-export"
export ERROR_BUCKET="${PROJECT_ID}-error-logs"

# Delete log sinks
gcloud logging sinks delete devops-audit-sink --quiet 2>/dev/null || true
gcloud logging sinks delete devops-error-sink --quiet 2>/dev/null || true

# Delete log-based metrics
gcloud logging metrics delete error-count-metric --quiet 2>/dev/null || true
gcloud logging metrics delete app-500-errors --quiet 2>/dev/null || true
gcloud logging metrics delete db-connection-errors --quiet 2>/dev/null || true
gcloud logging metrics delete http-latency-distribution --quiet 2>/dev/null || true

# Delete alerting policies
POLICY_IDS=$(gcloud alpha monitoring policies list \
  --format='value(name)' 2>/dev/null | grep -v "^$")
for POLICY_ID in $POLICY_IDS; do
  POLICY_NAME=$(gcloud alpha monitoring policies describe $POLICY_ID \
    --format='value(displayName)' 2>/dev/null)
  if echo "$POLICY_NAME" | grep -q -i "devops\|lab\|error rate\|cpu utilization"; then
    gcloud alpha monitoring policies delete $POLICY_ID --quiet 2>/dev/null || true
  fi
done

# Delete dashboard
DASHBOARD_ID=$(gcloud monitoring dashboards list \
  --format='value(name)' 2>/dev/null | head -1)
[ -n "$DASHBOARD_ID" ] && \
  gcloud monitoring dashboards delete $DASHBOARD_ID --quiet 2>/dev/null || true

# Delete log buckets
gsutil -m rm -r gs://$LOGS_BUCKET/ 2>/dev/null || true
gsutil rb gs://$LOGS_BUCKET/ 2>/dev/null || true
gsutil -m rm -r gs://$ERROR_BUCKET/ 2>/dev/null || true
gsutil rb gs://$ERROR_BUCKET/ 2>/dev/null || true

# Delete custom log
gcloud logging logs delete devops-lab-log --quiet 2>/dev/null || true

# Clean up local files
rm -f dashboard.json notification-channel.json alert-policy.json
rm -f cpu-alert-policy.json trace-example.py

echo "✅ Cleanup complete"
```

---

## Summary

You have successfully:
- ✅ Viewed, filtered, and written structured log entries using Cloud Logging
- ✅ Created log sinks to export audit and error logs to Cloud Storage
- ✅ Defined log-based metrics for monitoring error rates and custom events
- ✅ Built a custom monitoring dashboard with multiple metric widgets
- ✅ Created alerting policies for high error rates and CPU utilization
- ✅ Understood Cloud Trace integration and Error Reporting

**Next Lab:** [LAB-11: Infrastructure as Code with Terraform](LAB-11-terraform-gcp.md)
