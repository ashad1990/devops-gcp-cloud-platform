# LAB-12: Cloud SQL Managed Databases

**Topic:** Managed Relational Databases  
**Estimated Time:** 90 minutes  
**Difficulty:** 🔴 Advanced  
**Cost:** ~$1–$3

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Create and configure a Cloud SQL PostgreSQL instance with appropriate settings
2. Create databases, users, and manage connection settings securely
3. Connect to Cloud SQL using the Cloud SQL Auth Proxy for secure access
4. Import and export database data to and from Cloud Storage
5. Configure automated backups, retention policies, and Point-in-Time Recovery (PITR)
6. Set up read replicas and enable High Availability for production resilience

---

## Prerequisites

- [LAB-01](LAB-01-gcp-basics.md) completed
- `sqladmin.googleapis.com` API enabled
- Basic SQL knowledge (SELECT, INSERT, CREATE TABLE)
- psql client: `sudo apt-get install -y postgresql-client`

---

## Lab Overview

Cloud SQL is GCP's fully managed relational database service supporting PostgreSQL, MySQL, and SQL Server. It handles database administration tasks like patching, backups, replication, and failover. In this lab you will create a PostgreSQL 15 instance, manage databases and users, connect using the Auth Proxy, practice data import/export, configure PITR, and set up read replicas.

---

## Environment Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export ZONE=us-central1-a
export INSTANCE_NAME=devops-lab-db
export BACKUP_BUCKET="${PROJECT_ID}-sql-backups"

echo "Project:  $PROJECT_ID"
echo "Instance: $INSTANCE_NAME"
echo "Region:   $REGION"

# Generate secure passwords
export DB_ROOT_PASSWORD=$(openssl rand -base64 20 | tr -dc 'A-Za-z0-9' | head -c 20)
export APP_DB_PASSWORD=$(openssl rand -base64 20 | tr -dc 'A-Za-z0-9' | head -c 20)

echo ""
echo "⚠️  SAVE THESE PASSWORDS — you will need them in this lab:"
echo "Root password: $DB_ROOT_PASSWORD"
echo "App password:  $APP_DB_PASSWORD"
echo ""

# Enable APIs
gcloud services enable sqladmin.googleapis.com sql-component.googleapis.com
```

---

## Part 1: Create a Cloud SQL Instance

### 1.1 Create a PostgreSQL 15 Instance

```bash
# Create the instance (takes 5-10 minutes)
echo "Creating Cloud SQL instance... this takes 5-10 minutes"

gcloud sql instances create $INSTANCE_NAME \
  --database-version=POSTGRES_15 \
  --region=$REGION \
  --tier=db-f1-micro \
  --storage-type=SSD \
  --storage-size=10GB \
  --storage-auto-increase \
  --backup-start-time=03:00 \
  --retained-backups-count=7 \
  --deletion-protection=false \
  --root-password=$DB_ROOT_PASSWORD \
  --database-flags=max_connections=100,log_min_duration_statement=1000

echo "⏳ Waiting for instance creation..."
```

> This step takes 5–10 minutes. Review Part 2 concepts while waiting.

### 1.2 Verify Instance Creation

```bash
# Get instance details
gcloud sql instances describe $INSTANCE_NAME \
  --format='yaml(name,state,databaseVersion,region,tier,ipAddresses)'

# Get connection information
CONNECTION_NAME=$(gcloud sql instances describe $INSTANCE_NAME \
  --format='value(connectionName)')
PUBLIC_IP=$(gcloud sql instances describe $INSTANCE_NAME \
  --format='value(ipAddresses[0].ipAddress)')

echo "Connection Name: $CONNECTION_NAME"
echo "Public IP:       $PUBLIC_IP"
echo "State:           $(gcloud sql instances describe $INSTANCE_NAME --format='value(state)')"
```

**Expected output:**
```yaml
databaseVersion: POSTGRES_15
name: devops-lab-db
region: us-central1
state: RUNNABLE
tier: db-f1-micro
ipAddresses:
- ipAddress: 34.123.45.67
  type: PRIMARY
```

### 1.3 Configure Authorized Networks

```bash
# Get your current external IP
MY_IP=$(curl -s https://ifconfig.me 2>/dev/null || curl -s https://api.ipify.org)
echo "Your IP: $MY_IP"

# Authorize your IP for direct connection (for psql without proxy)
gcloud sql instances patch $INSTANCE_NAME \
  --authorized-networks="${MY_IP}/32" \
  --quiet

echo "✅ Your IP authorized for direct connection"
```

---

## Part 2: Create Databases and Users

### 2.1 Create Databases

```bash
# Create application database
gcloud sql databases create devops_app_db \
  --instance=$INSTANCE_NAME \
  --charset=UTF8 \
  --collation=en_US.UTF8

# Create analytics database
gcloud sql databases create devops_analytics_db \
  --instance=$INSTANCE_NAME

# List databases
gcloud sql databases list --instance=$INSTANCE_NAME
```

**Expected output:**
```
NAME                 CHARSET  COLLATION
devops_analytics_db  UTF8     en_US.UTF8
devops_app_db        UTF8     en_US.UTF8
postgres             UTF8     en_US.UTF8
```

### 2.2 Create Application Users

```bash
# Create application user with specific password
gcloud sql users create app_user \
  --instance=$INSTANCE_NAME \
  --password=$APP_DB_PASSWORD

# Create read-only analytics user
ANALYTICS_PASSWORD=$(openssl rand -base64 12 | tr -dc 'A-Za-z0-9' | head -c 12)
gcloud sql users create analytics_user \
  --instance=$INSTANCE_NAME \
  --password=$ANALYTICS_PASSWORD

echo "Analytics user password: $ANALYTICS_PASSWORD"

# List users
gcloud sql users list --instance=$INSTANCE_NAME
```

---

## Part 3: Connect Using Cloud SQL Auth Proxy

### 3.1 Install the Auth Proxy

```bash
# Download Cloud SQL Auth Proxy (Linux x86_64)
curl -o cloud-sql-proxy \
  https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.7.2/cloud-sql-proxy.linux.amd64

chmod +x cloud-sql-proxy
echo "Proxy downloaded: $(./cloud-sql-proxy --version)"
```

### 3.2 Start the Proxy

```bash
# Start proxy in background on port 5432
./cloud-sql-proxy \
  --address=127.0.0.1 \
  --port=5432 \
  $CONNECTION_NAME \
  --auto-iam-authn &
PROXY_PID=$!

echo "Proxy started with PID: $PROXY_PID"
sleep 3

# Verify proxy is listening
ss -tlnp | grep 5432 || netstat -tlnp 2>/dev/null | grep 5432 || echo "Proxy should be listening on :5432"
```

### 3.3 Connect with psql

```bash
# Install psql client if needed
sudo apt-get install -y postgresql-client 2>/dev/null || \
  brew install postgresql 2>/dev/null || \
  echo "Install postgresql-client for your OS"

# Connect through the proxy
PGPASSWORD=$DB_ROOT_PASSWORD psql \
  -h 127.0.0.1 \
  -p 5432 \
  -U postgres \
  -d devops_app_db \
  -c "\l"
```

### 3.4 Create Schema and Sample Data

```bash
# Execute SQL via proxy
PGPASSWORD=$DB_ROOT_PASSWORD psql \
  -h 127.0.0.1 -p 5432 \
  -U postgres -d devops_app_db << 'PSQL'

-- Create tables
CREATE TABLE IF NOT EXISTS users (
  id         SERIAL PRIMARY KEY,
  username   VARCHAR(50) UNIQUE NOT NULL,
  email      VARCHAR(100) UNIQUE NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  is_active  BOOLEAN DEFAULT TRUE
);

CREATE TABLE IF NOT EXISTS events (
  id         SERIAL PRIMARY KEY,
  user_id    INTEGER REFERENCES users(id) ON DELETE CASCADE,
  event_type VARCHAR(50) NOT NULL,
  event_data JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_events_user_id ON events(user_id);
CREATE INDEX IF NOT EXISTS idx_events_event_type ON events(event_type);
CREATE INDEX IF NOT EXISTS idx_events_created_at ON events(created_at);

-- Insert test users
INSERT INTO users (username, email) VALUES
  ('alice', 'alice@example.com'),
  ('bob', 'bob@example.com'),
  ('carol', 'carol@example.com'),
  ('dave', 'dave@example.com'),
  ('eve', 'eve@example.com')
ON CONFLICT (username) DO NOTHING;

-- Insert test events
INSERT INTO events (user_id, event_type, event_data) VALUES
  (1, 'login',    '{"ip": "192.168.1.1", "browser": "Chrome", "os": "Linux"}'),
  (1, 'purchase', '{"item": "GCP Pro Training", "amount": 199.99, "currency": "USD"}'),
  (2, 'login',    '{"ip": "10.0.0.5", "browser": "Firefox", "os": "macOS"}'),
  (2, 'signup',   '{"referral": "linkedin", "plan": "pro"}'),
  (3, 'login',    '{"ip": "172.16.0.10", "browser": "Safari", "os": "iOS"}'),
  (3, 'logout',   '{"session_duration": 1800}'),
  (4, 'purchase', '{"item": "Cloud Storage 1TB", "amount": 26.00, "currency": "USD"}'),
  (5, 'signup',   '{"referral": "google", "plan": "free"}')
ON CONFLICT DO NOTHING;

-- Verify data
SELECT
  u.username,
  u.email,
  COUNT(e.id) AS event_count,
  MAX(e.created_at) AS last_event
FROM users u
LEFT JOIN events e ON u.id = e.user_id
GROUP BY u.username, u.email
ORDER BY event_count DESC;

\q
PSQL

echo "✅ Schema and sample data created"
```

### 3.5 Alternative: gcloud sql connect

```bash
# Direct connection using gcloud (no proxy needed, uses Cloud Shell or IAP)
# gcloud sql connect $INSTANCE_NAME \
#   --user=postgres \
#   --database=devops_app_db
# Enter password when prompted: $DB_ROOT_PASSWORD
```

---

## Part 4: Import and Export Data

### 4.1 Create Backup Bucket

```bash
# Create GCS bucket for SQL backups and exports
gsutil mb -l $REGION gs://$BACKUP_BUCKET/

# Get the Cloud SQL service account
SQL_SA=$(gcloud sql instances describe $INSTANCE_NAME \
  --format='value(serviceAccountEmailAddress)')
echo "SQL Service Account: $SQL_SA"

# Grant the SQL service account write access to the bucket
gsutil iam ch serviceAccount:${SQL_SA}:roles/storage.objectAdmin gs://$BACKUP_BUCKET/
```

### 4.2 Export Database

```bash
EXPORT_DATE=$(date +%Y%m%d-%H%M%S)

# Export full database to SQL dump
gcloud sql export sql $INSTANCE_NAME \
  gs://$BACKUP_BUCKET/exports/devops_app_db_${EXPORT_DATE}.sql \
  --database=devops_app_db \
  --offload

echo "Export started. Check status:"
gcloud sql operations list --instance=$INSTANCE_NAME --limit=3
```

### 4.3 Export as CSV

```bash
# Export query result as CSV
gcloud sql export csv $INSTANCE_NAME \
  gs://$BACKUP_BUCKET/exports/users_${EXPORT_DATE}.csv \
  --database=devops_app_db \
  --query="SELECT id, username, email, created_at, is_active FROM users ORDER BY id"

# List exports
gsutil ls gs://$BACKUP_BUCKET/exports/
```

### 4.4 Import Data

```bash
# First create a test import file
cat > /tmp/import-test.sql << 'EOF'
-- Test import data
INSERT INTO users (username, email) VALUES
  ('frank', 'frank@example.com'),
  ('grace', 'grace@example.com')
ON CONFLICT (username) DO NOTHING;

INSERT INTO events (user_id, event_type, event_data)
SELECT id, 'imported', '{"source": "sql-import"}'
FROM users WHERE username IN ('frank', 'grace');
EOF

# Upload to GCS
gsutil cp /tmp/import-test.sql gs://$BACKUP_BUCKET/imports/

# Import SQL file
gcloud sql import sql $INSTANCE_NAME \
  gs://$BACKUP_BUCKET/imports/import-test.sql \
  --database=devops_app_db \
  --quiet

echo "Import complete"
```

---

## Part 5: Automated Backups and Point-in-Time Recovery

### 5.1 View Backup Configuration

```bash
# Check current backup configuration
gcloud sql instances describe $INSTANCE_NAME \
  --format='yaml(settings.backupConfiguration)'
```

### 5.2 Create On-Demand Backup

```bash
# Create an immediate backup
gcloud sql backups create \
  --instance=$INSTANCE_NAME \
  --description="Pre-change manual backup $(date +%Y-%m-%d)"

# List all backups
gcloud sql backups list --instance=$INSTANCE_NAME

# Wait a moment and check status
sleep 10
gcloud sql backups list --instance=$INSTANCE_NAME \
  --format='table(id,status,windowStartTime,description)'
```

**Expected output:**
```
ID        STATUS     WINDOW_START_TIME        DESCRIPTION
16740...  SUCCESSFUL 2024-01-15T10:30:00.000Z Pre-change manual backup 2024-01-15
16739...  SUCCESSFUL 2024-01-15T03:00:00.000Z
```

### 5.3 Enable Point-in-Time Recovery

```bash
# Enable PITR (requires binary logging / WAL)
gcloud sql instances patch $INSTANCE_NAME \
  --enable-point-in-time-recovery \
  --retained-transaction-log-days=7 \
  --quiet

# Verify PITR is enabled
gcloud sql instances describe $INSTANCE_NAME \
  --format='yaml(settings.backupConfiguration)'

echo "PITR enabled — can restore to any point in the last 7 days"
```

### 5.4 Restore from Backup (Reference)

```bash
# To restore to a new instance (not destructive to primary):
# BACKUP_ID=$(gcloud sql backups list --instance=$INSTANCE_NAME --limit=1 --format='value(id)')
# gcloud sql backups restore $BACKUP_ID \
#   --restore-instance=$INSTANCE_NAME-restored \
#   --backup-instance=$INSTANCE_NAME

# Point-in-time restore (reference):
# gcloud sql instances clone $INSTANCE_NAME ${INSTANCE_NAME}-pitr \
#   --point-in-time='2024-01-15T10:30:00.000Z'

echo "Restore commands shown above are for reference — not run in this lab"
```

---

## Part 6: Read Replicas

### 6.1 Create a Read Replica

```bash
echo "Creating read replica (takes 5-10 minutes)..."

gcloud sql instances create ${INSTANCE_NAME}-replica \
  --master-instance-name=$INSTANCE_NAME \
  --region=$REGION \
  --tier=db-f1-micro \
  --replica-type=READ

# Monitor creation
gcloud sql operations list --instance=${INSTANCE_NAME}-replica --limit=3
```

### 6.2 Verify Replica

```bash
# Wait for replica to be ready
echo "Waiting for replica to become RUNNABLE..."
while true; do
  STATE=$(gcloud sql instances describe ${INSTANCE_NAME}-replica \
    --format='value(state)' 2>/dev/null)
  echo "Replica state: $STATE"
  [ "$STATE" = "RUNNABLE" ] && break
  sleep 20
done

# Get replica connection info
REPLICA_IP=$(gcloud sql instances describe ${INSTANCE_NAME}-replica \
  --format='value(ipAddresses[0].ipAddress)')

echo "Primary IP:    $PUBLIC_IP"
echo "Replica IP:    $REPLICA_IP"
echo "Replication lag: $(gcloud sql instances describe ${INSTANCE_NAME}-replica --format='value(replicaConfiguration)')"

# List all instances
gcloud sql instances list \
  --format='table(name,databaseVersion,region,tier,ipAddresses[0].ipAddress,state)'
```

---

## Part 7: High Availability

### 7.1 Enable Regional HA

```bash
# Upgrade to regional HA (converts to synchronous replication across zones)
# Note: This takes several minutes and briefly restarts the instance
echo "Enabling HA... (takes 5+ minutes)"

gcloud sql instances patch $INSTANCE_NAME \
  --availability-type=REGIONAL \
  --quiet

# Monitor progress
gcloud sql operations list --instance=$INSTANCE_NAME --limit=1

# Verify HA is enabled
gcloud sql instances describe $INSTANCE_NAME \
  --format='value(settings.availabilityType)'
# Expected: REGIONAL
```

### 7.2 Test Failover

```bash
# Trigger a manual failover (simulates zone failure)
# ⚠️  This will briefly interrupt connections (2-3 minutes)
echo "Initiating failover test..."
gcloud sql instances failover $INSTANCE_NAME --quiet

# Monitor failover status
echo "Waiting for failover to complete..."
sleep 30
gcloud sql instances describe $INSTANCE_NAME \
  --format='value(state)'

echo "Failover complete — primary has moved to standby zone"
```

---

## Part 8: Connecting from Cloud Run

### 8.1 Create a Cloud Run Service with SQL Connection

```bash
mkdir -p sql-cloudrun-demo

cat > sql-cloudrun-demo/app.py << 'EOF'
import os
import pg8000.native
from flask import Flask, jsonify

app = Flask(__name__)

def get_connection():
    """Create a database connection via Cloud SQL connector or proxy."""
    connection_name = os.environ.get('CLOUD_SQL_CONNECTION_NAME', '')
    db_user = os.environ.get('DB_USER', 'app_user')
    db_pass = os.environ.get('DB_PASS', '')
    db_name = os.environ.get('DB_NAME', 'devops_app_db')

    # In Cloud Run, connect via Unix socket (uses Cloud SQL connector)
    if connection_name and not os.environ.get('LOCAL_DEV'):
        socket_dir = f'/cloudsql/{connection_name}'
        conn = pg8000.native.Connection(
            user=db_user,
            password=db_pass,
            database=db_name,
            unix_sock=f'{socket_dir}/.s.PGSQL.5432'
        )
    else:
        # Local development via proxy on localhost:5432
        conn = pg8000.native.Connection(
            user=db_user,
            password=db_pass,
            database=db_name,
            host='127.0.0.1',
            port=5432
        )
    return conn

@app.route('/users')
def list_users():
    try:
        conn = get_connection()
        rows = conn.run(
            "SELECT id, username, email, created_at::text, is_active "
            "FROM users ORDER BY id LIMIT 20"
        )
        users = [
            {'id': r[0], 'username': r[1], 'email': r[2],
             'created_at': r[3], 'is_active': r[4]}
            for r in rows
        ]
        conn.close()
        return jsonify({'users': users, 'count': len(users)})
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/stats')
def stats():
    try:
        conn = get_connection()
        rows = conn.run("""
            SELECT event_type, COUNT(*) as count
            FROM events
            GROUP BY event_type
            ORDER BY count DESC
        """)
        stats_data = [{'event_type': r[0], 'count': r[1]} for r in rows]
        conn.close()
        return jsonify({'stats': stats_data})
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))
EOF

cat > sql-cloudrun-demo/requirements.txt << 'EOF'
Flask==3.0.0
gunicorn==21.2.0
pg8000==1.30.4
EOF

echo "✅ Cloud Run SQL app created"

# To deploy (requires Cloud Build and additional setup):
# gcloud run deploy sql-demo \
#   --source sql-cloudrun-demo/ \
#   --region $REGION \
#   --add-cloudsql-instances=$CONNECTION_NAME \
#   --set-env-vars="CLOUD_SQL_CONNECTION_NAME=$CONNECTION_NAME,DB_USER=app_user,DB_NAME=devops_app_db" \
#   --set-secrets="DB_PASS=app-db-password:latest"
```

---

## Verification Steps

```bash
# 1. Verify instance is running
gcloud sql instances describe $INSTANCE_NAME --format='value(state)'

# 2. Verify databases exist
gcloud sql databases list --instance=$INSTANCE_NAME \
  --format='table(name,charset)'

# 3. Verify users exist
gcloud sql users list --instance=$INSTANCE_NAME \
  --format='table(name,host)'

# 4. Verify replica exists
gcloud sql instances list --filter="masterInstanceName=$INSTANCE_NAME" \
  --format='table(name,state)'

# 5. Verify backups exist
gcloud sql backups list --instance=$INSTANCE_NAME --limit=3 \
  --format='table(id,status,windowStartTime)'

# 6. Test connection via proxy
if kill -0 $PROXY_PID 2>/dev/null; then
  PGPASSWORD=$DB_ROOT_PASSWORD psql \
    -h 127.0.0.1 -p 5432 -U postgres -d devops_app_db \
    -c "SELECT COUNT(*) as user_count FROM users;"
else
  echo "Proxy not running — restart with: ./cloud-sql-proxy $CONNECTION_NAME --port=5432 &"
fi

echo "✅ Verification complete"
```

---

## Cost Estimates

| Resource | Hourly | Monthly |
|----------|--------|---------|
| db-f1-micro (primary) | ~$0.017/hr | ~$12 |
| db-f1-micro (replica) | ~$0.017/hr | ~$12 |
| HA mode surcharge | ~2× base cost | ~$24 total |
| SSD storage (10 GB) | — | ~$1.70 |
| Automated backups (7 days) | — | Included |
| GCS backup bucket | — | <$0.01 |

> ⚠️  The replica and HA configuration roughly doubles the cost. Delete them promptly after the lab.

---

## Cleanup

> ⚠️ Cloud SQL instances are expensive — delete ALL instances immediately.

```bash
export PROJECT_ID=$(gcloud config get-value project)
export INSTANCE_NAME=devops-lab-db
export BACKUP_BUCKET="${PROJECT_ID}-sql-backups"

# Kill the proxy if running
kill $PROXY_PID 2>/dev/null || true
pkill -f "cloud-sql-proxy" 2>/dev/null || true

# Delete read replica first (before primary)
gcloud sql instances delete ${INSTANCE_NAME}-replica \
  --quiet 2>/dev/null || true

# Wait for replica deletion
echo "Waiting for replica deletion..."
sleep 30

# Delete primary instance (also deletes automated backups)
gcloud sql instances delete $INSTANCE_NAME --quiet

# Wait for instance deletion
echo "Waiting for instance deletion..."
sleep 30

# Verify deletion
gcloud sql instances list | grep $INSTANCE_NAME || echo "Instance deleted"

# Delete backup bucket
gsutil -m rm -r gs://$BACKUP_BUCKET/ 2>/dev/null || true
gsutil rb gs://$BACKUP_BUCKET/ 2>/dev/null || true

# Clean up local files
rm -f cloud-sql-proxy
rm -f /tmp/import-test.sql
rm -rf sql-cloudrun-demo/

echo "✅ Cleanup complete"

# Final verification
gcloud sql instances list
```

---

## Summary

You have successfully:
- ✅ Created a Cloud SQL PostgreSQL 15 instance with SSD storage and automated backups
- ✅ Created databases and users with appropriate passwords
- ✅ Connected securely using the Cloud SQL Auth Proxy
- ✅ Created tables and inserted sample relational data with foreign keys
- ✅ Exported and imported data using Cloud Storage
- ✅ Configured PITR, created on-demand backups, and set up a read replica
- ✅ Enabled High Availability (Regional) mode and tested failover

**Congratulations! You have completed all 12 GCP DevOps labs.** 🎉

---

## What's Next?

Now that you've completed the full lab series, consider exploring:

- **Cloud Spanner** — Globally distributed relational database
- **Firestore** — NoSQL document database with real-time sync
- **BigQuery** — Serverless data warehouse for analytics
- **Pub/Sub Advanced** — Message ordering, dead-letter topics, exactly-once delivery
- **GKE Enterprise** — Anthos Service Mesh, Config Controller, Policy Controller
- **Cloud Armor** — WAF and DDoS protection for load balancers
- **VPC Service Controls** — Data exfiltration protection for GCP APIs
- **Binary Authorization** — Container image signing and policy enforcement

Return to [LABS README](README.md) for an overview of all completed labs.
