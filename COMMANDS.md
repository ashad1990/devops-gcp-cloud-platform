# gcloud CLI Command Reference

> Comprehensive `gcloud` command reference for GCP DevOps engineers. Organized by service with 5–10 examples per group. All commands use real GCP syntax.

---

## Table of Contents

- [Global Flags & Tips](#global-flags--tips)
- [gcloud config](#gcloud-config)
- [gcloud compute](#gcloud-compute)
- [gcloud container (GKE)](#gcloud-container-gke)
- [gcloud storage / gsutil](#gcloud-storage--gsutil)
- [gcloud iam](#gcloud-iam)
- [gcloud builds](#gcloud-builds)
- [gcloud run](#gcloud-run)
- [gcloud functions](#gcloud-functions)
- [gcloud sql](#gcloud-sql)
- [gcloud logging](#gcloud-logging)
- [gcloud monitoring](#gcloud-monitoring)

---

## Global Flags & Tips

These flags work with most `gcloud` commands:

```bash
--project=PROJECT_ID        # Override the active project
--region=REGION             # Override the default region
--zone=ZONE                 # Override the default zone
--format=FORMAT             # Output: json, yaml, table, value, csv, text
--filter=EXPRESSION         # Filter results client-side
--quiet, -q                 # Suppress confirmation prompts
--verbosity=debug           # Show detailed request/response info
--log-http                  # Log HTTP requests (debugging)
--no-user-output-enabled    # Suppress all output (scripts)
--async                     # Return immediately without waiting for operation

# Useful output format tricks
gcloud compute instances list --format="value(name)"           # Just names
gcloud compute instances list --format="value(name,zone)"      # Name + zone (tab-separated)
gcloud compute instances list --format="json" | jq '.[].name'  # Pipe to jq
gcloud compute instances list --format="table(name,status,zone.basename())"

# Use --filter for server-side filtering (efficient)
gcloud compute instances list --filter="status=RUNNING AND zone:us-central1"
gcloud compute instances list --filter="labels.env=prod"
gcloud compute instances list --filter="name~^web-"            # Regex match
```

---

## gcloud config

Manage gcloud CLI configuration, properties, and authentication.

```bash
# Show all current configuration
gcloud config list

# Show a single property
gcloud config get-value project
gcloud config get-value compute/region
gcloud config get-value account

# Set the active project
gcloud config set project my-project-id

# Set the default region and zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Set the default Cloud Run region
gcloud config set run/region us-central1

# Unset a property
gcloud config unset compute/zone

# List all named configurations (profiles)
gcloud config configurations list

# Create a new named configuration
gcloud config configurations create dev-profile

# Activate a different configuration
gcloud config configurations activate dev-profile

# Delete a configuration
gcloud config configurations delete old-profile

# Show active account
gcloud auth list

# Log in with your Google account (opens browser)
gcloud auth login

# Set up Application Default Credentials (for SDKs/libraries)
gcloud auth application-default login

# Revoke credentials
gcloud auth revoke your-account@gmail.com

# Authenticate using a service account key file
gcloud auth activate-service-account \
  --key-file=/path/to/sa-key.json

# Print access token (useful for API calls with curl)
gcloud auth print-access-token

# Print Application Default Credential token
gcloud auth application-default print-access-token

# Enable an API for the current project
gcloud services enable run.googleapis.com container.googleapis.com

# List enabled APIs
gcloud services list --enabled

# Disable an API
gcloud services disable vision.googleapis.com
```

---

## gcloud compute

Manage Compute Engine VMs, disks, images, and networking.

### Instances

```bash
# Create a VM with common options
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-ssd \
  --network-interface=network=default,subnet=default,address="" \
  --service-account=my-sa@my-project.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --tags=http-server,https-server \
  --labels=env=dev,owner=alice \
  --metadata=enable-oslogin=true

# Create a preemptible (spot) VM
gcloud compute instances create batch-job \
  --zone=us-central1-a \
  --machine-type=n1-highcpu-8 \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --preemptible

# Create a VM with a startup script file
gcloud compute instances create web-01 \
  --zone=us-central1-a \
  --machine-type=e2-small \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata-from-file=startup-script=./startup.sh

# List all instances across all zones
gcloud compute instances list

# List instances in a specific zone
gcloud compute instances list --zones=us-central1-a

# Filter running instances
gcloud compute instances list --filter="status=RUNNING"

# Describe an instance (full details)
gcloud compute instances describe my-vm --zone=us-central1-a

# SSH into a VM (handles keys and tunneling automatically)
gcloud compute ssh my-vm --zone=us-central1-a

# SSH and run a command
gcloud compute ssh my-vm --zone=us-central1-a -- 'df -h && free -m'

# SCP a file to/from a VM
gcloud compute scp ./app.jar my-vm:~/app.jar --zone=us-central1-a
gcloud compute scp my-vm:~/logs/app.log ./app.log --zone=us-central1-a

# Start / stop / restart a VM
gcloud compute instances start my-vm --zone=us-central1-a
gcloud compute instances stop my-vm --zone=us-central1-a
gcloud compute instances reset my-vm --zone=us-central1-a

# Delete a VM
gcloud compute instances delete my-vm --zone=us-central1-a --quiet

# Add/remove labels from a running instance
gcloud compute instances add-labels my-vm \
  --zone=us-central1-a \
  --labels=env=staging

gcloud compute instances remove-labels my-vm \
  --zone=us-central1-a \
  --labels=env

# Update machine type (requires stopping the VM first)
gcloud compute instances stop my-vm --zone=us-central1-a
gcloud compute instances set-machine-type my-vm \
  --zone=us-central1-a \
  --machine-type=e2-standard-4
gcloud compute instances start my-vm --zone=us-central1-a
```

### Disks and Snapshots

```bash
# List persistent disks
gcloud compute disks list

# Create a new persistent disk
gcloud compute disks create my-data-disk \
  --zone=us-central1-a \
  --size=100GB \
  --type=pd-ssd

# Attach a disk to a VM
gcloud compute instances attach-disk my-vm \
  --disk=my-data-disk \
  --zone=us-central1-a

# Detach a disk from a VM
gcloud compute instances detach-disk my-vm \
  --disk=my-data-disk \
  --zone=us-central1-a

# Create a snapshot of a disk
gcloud compute disks snapshot my-vm \
  --snapshot-names=my-vm-snap-$(date +%Y%m%d%H%M) \
  --zone=us-central1-a \
  --storage-location=us-central1

# List snapshots
gcloud compute snapshots list

# Create a disk from a snapshot
gcloud compute disks create restored-disk \
  --source-snapshot=my-vm-snap-20240101 \
  --zone=us-central1-a

# Delete a snapshot
gcloud compute snapshots delete my-vm-snap-20240101
```

### Images

```bash
# List available public images (Debian family)
gcloud compute images list --filter="family=debian-12"

# List all public image projects
gcloud compute images list --standard-images --format="value(project)" | sort -u

# Create a custom image from a stopped VM's disk
gcloud compute images create my-golden-image \
  --source-disk=my-vm \
  --source-disk-zone=us-central1-a \
  --family=my-app-images \
  --description="Hardened base image with app v1.2"

# List custom images in your project
gcloud compute images list --no-standard-images

# Delete an old image
gcloud compute images delete old-image-v1

# Deprecate an image (hide from image list)
gcloud compute images deprecate old-image \
  --state=DEPRECATED \
  --replacement=my-golden-image
```

### Instance Templates & Managed Instance Groups

```bash
# Create an instance template
gcloud compute instance-templates create web-template \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --tags=http-server \
  --metadata-from-file=startup-script=./startup.sh \
  --service-account=web-sa@my-project.iam.gserviceaccount.com \
  --scopes=cloud-platform

# List instance templates
gcloud compute instance-templates list

# Create a Managed Instance Group (MIG)
gcloud compute instance-groups managed create web-mig \
  --base-instance-name=web \
  --template=web-template \
  --size=3 \
  --zone=us-central1-a

# Create a regional (multi-zone) MIG
gcloud compute instance-groups managed create web-mig-regional \
  --base-instance-name=web \
  --template=web-template \
  --size=3 \
  --region=us-central1

# Set up autoscaling on a MIG
gcloud compute instance-groups managed set-autoscaling web-mig \
  --zone=us-central1-a \
  --max-num-replicas=10 \
  --min-num-replicas=2 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=90

# List instances in a MIG
gcloud compute instance-groups managed list-instances web-mig \
  --zone=us-central1-a

# Rolling update a MIG to a new template
gcloud compute instance-groups managed rolling-action start-update web-mig \
  --version=template=web-template-v2 \
  --zone=us-central1-a \
  --max-surge=3 \
  --max-unavailable=0

# Resize a MIG
gcloud compute instance-groups managed resize web-mig \
  --size=5 \
  --zone=us-central1-a

# Delete a MIG
gcloud compute instance-groups managed delete web-mig \
  --zone=us-central1-a --quiet
```

---

## gcloud container (GKE)

Manage Google Kubernetes Engine clusters, node pools, and credentials.

```bash
# Create an Autopilot cluster (fully managed, recommended)
gcloud container clusters create-auto prod-cluster \
  --region=us-central1 \
  --release-channel=regular

# Create a Standard cluster with autoscaling
gcloud container clusters create dev-cluster \
  --region=us-central1 \
  --num-nodes=2 \
  --machine-type=e2-standard-2 \
  --disk-size=50 \
  --disk-type=pd-ssd \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5 \
  --release-channel=regular \
  --workload-pool=my-project.svc.id.goog \
  --enable-ip-alias

# List all clusters across all regions
gcloud container clusters list

# Get kubectl credentials for a cluster
gcloud container clusters get-credentials prod-cluster \
  --region=us-central1

# Describe a cluster
gcloud container clusters describe prod-cluster --region=us-central1

# Upgrade the cluster control plane to a specific version
gcloud container clusters upgrade prod-cluster \
  --region=us-central1 \
  --master \
  --cluster-version=1.29

# Upgrade nodes in the default node pool
gcloud container clusters upgrade prod-cluster \
  --region=us-central1 \
  --node-pool=default-pool

# Add a new node pool (e.g., high-memory nodes)
gcloud container node-pools create highmem-pool \
  --cluster=prod-cluster \
  --region=us-central1 \
  --num-nodes=2 \
  --machine-type=n2-highmem-8 \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=10 \
  --node-taints=workload=highmem:NoSchedule

# List node pools in a cluster
gcloud container node-pools list \
  --cluster=prod-cluster \
  --region=us-central1

# Describe a node pool
gcloud container node-pools describe default-pool \
  --cluster=prod-cluster \
  --region=us-central1

# Delete a node pool
gcloud container node-pools delete highmem-pool \
  --cluster=prod-cluster \
  --region=us-central1 --quiet

# Resize a node pool
gcloud container clusters resize prod-cluster \
  --node-pool=default-pool \
  --num-nodes=4 \
  --region=us-central1

# Enable Workload Identity on an existing cluster
gcloud container clusters update prod-cluster \
  --region=us-central1 \
  --workload-pool=my-project.svc.id.goog

# Get available Kubernetes versions for a region
gcloud container get-server-config --region=us-central1

# Delete a cluster
gcloud container clusters delete dev-cluster \
  --region=us-central1 --quiet
```

---

## gcloud storage / gsutil

Manage Cloud Storage buckets and objects.

> **Note**: `gcloud storage` is the modern CLI (recommended). `gsutil` is the legacy tool but is still widely used.

```bash
# --- gcloud storage (modern, recommended) ---

# Create a bucket
gcloud storage buckets create gs://my-bucket-name \
  --location=us-central1 \
  --storage-class=STANDARD \
  --uniform-bucket-level-access

# List buckets
gcloud storage buckets list

# Describe a bucket
gcloud storage buckets describe gs://my-bucket-name

# Upload a file
gcloud storage cp ./myfile.txt gs://my-bucket-name/data/myfile.txt

# Upload a directory recursively
gcloud storage cp -r ./build/ gs://my-bucket-name/releases/v1.2.0/

# Download a file
gcloud storage cp gs://my-bucket-name/data/myfile.txt ./myfile.txt

# Download all files from a prefix
gcloud storage cp -r gs://my-bucket-name/releases/v1.2.0/ ./v1.2.0/

# List objects in a bucket
gcloud storage ls gs://my-bucket-name/

# List with details (size, date)
gcloud storage ls -l gs://my-bucket-name/data/

# Delete an object
gcloud storage rm gs://my-bucket-name/data/old-file.txt

# Delete all objects in a prefix (wildcard)
gcloud storage rm gs://my-bucket-name/temp/**

# Move/rename an object
gcloud storage mv gs://my-bucket-name/old-name.txt \
  gs://my-bucket-name/new-name.txt

# Sync a local directory to a bucket
gcloud storage rsync -r ./dist/ gs://my-app-static/

# Sync and delete remote files not present locally
gcloud storage rsync -r -d ./dist/ gs://my-app-static/

# Get object metadata
gcloud storage objects describe gs://my-bucket-name/data/file.txt

# Set storage class on an existing bucket
gcloud storage buckets update gs://my-bucket-name \
  --storage-class=NEARLINE

# Enable versioning
gcloud storage buckets update gs://my-bucket-name --versioning

# Set lifecycle rules from a JSON file
gcloud storage buckets update gs://my-bucket-name \
  --lifecycle-file=lifecycle.json

# Grant public access (for static website hosting)
gcloud storage buckets add-iam-policy-binding gs://my-public-bucket \
  --member=allUsers \
  --role=roles/storage.objectViewer

# Grant a user access to a bucket
gcloud storage buckets add-iam-policy-binding gs://my-bucket-name \
  --member=user:alice@example.com \
  --role=roles/storage.objectAdmin

# --- gsutil (legacy, still widely used) ---

# Create a bucket
gsutil mb -l us-central1 -c STANDARD gs://my-bucket-legacy/

# Upload with content-type
gsutil -h "Content-Type:application/json" cp data.json gs://my-bucket/

# Copy with caching metadata (for CDN)
gsutil -h "Cache-Control:public,max-age=3600" cp -r ./static/ gs://my-bucket/

# Parallel upload (faster for many files)
gsutil -m cp -r ./build/ gs://my-bucket/build/

# Set CORS configuration
gsutil cors set cors.json gs://my-bucket/

# Check bucket IAM policy
gsutil iam get gs://my-bucket/

# Set default storage class
gsutil defstorageclass set NEARLINE gs://my-archive-bucket/

# Get bucket size
gsutil du -sh gs://my-bucket/
```

---

## gcloud iam

Manage IAM roles, service accounts, and policies.

```bash
# List all IAM roles (predefined, GA stage)
gcloud iam roles list --filter="stage=GA" --format="table(name,title)"

# Search for roles by keyword
gcloud iam roles list --filter="title~Storage"

# Describe a predefined role (see included permissions)
gcloud iam roles describe roles/storage.objectAdmin

# Get IAM policy for a project
gcloud projects get-iam-policy my-project

# Get IAM policy in JSON format (useful for scripting)
gcloud projects get-iam-policy my-project --format=json

# Add a role binding to a project
gcloud projects add-iam-policy-binding my-project \
  --member="user:alice@example.com" \
  --role="roles/viewer"

# Add a role binding with a condition (time-based access)
gcloud projects add-iam-policy-binding my-project \
  --member="user:contractor@example.com" \
  --role="roles/editor" \
  --condition='expression=request.time < timestamp("2024-12-31T00:00:00Z"),title=TempAccess'

# Remove a role binding from a project
gcloud projects remove-iam-policy-binding my-project \
  --member="user:alice@example.com" \
  --role="roles/viewer"

# --- Service Accounts ---

# Create a service account
gcloud iam service-accounts create my-app-sa \
  --display-name="My Application SA" \
  --description="Backend service identity"

# List service accounts in a project
gcloud iam service-accounts list

# Describe a service account
gcloud iam service-accounts describe \
  my-app-sa@my-project.iam.gserviceaccount.com

# Grant a role to a service account on a project
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Allow a user to impersonate a service account
gcloud iam service-accounts add-iam-policy-binding \
  my-app-sa@my-project.iam.gserviceaccount.com \
  --member="user:alice@example.com" \
  --role="roles/iam.serviceAccountTokenCreator"

# Create a service account key (avoid if possible)
gcloud iam service-accounts keys create ./sa-key.json \
  --iam-account=my-app-sa@my-project.iam.gserviceaccount.com

# List keys for a service account
gcloud iam service-accounts keys list \
  --iam-account=my-app-sa@my-project.iam.gserviceaccount.com

# Delete a service account key
gcloud iam service-accounts keys delete KEY_ID \
  --iam-account=my-app-sa@my-project.iam.gserviceaccount.com

# Disable a service account
gcloud iam service-accounts disable \
  my-app-sa@my-project.iam.gserviceaccount.com

# Enable a service account
gcloud iam service-accounts enable \
  my-app-sa@my-project.iam.gserviceaccount.com

# Delete a service account
gcloud iam service-accounts delete \
  my-app-sa@my-project.iam.gserviceaccount.com

# --- Custom Roles ---

# Create a custom role from a YAML file
cat > my-role.yaml << 'EOF'
title: "Custom Log Viewer"
description: "Read logs and metrics"
stage: "GA"
includedPermissions:
  - logging.logs.list
  - logging.logEntries.list
  - monitoring.timeSeries.list
EOF
gcloud iam roles create customLogViewer \
  --project=my-project \
  --file=my-role.yaml

# List custom roles
gcloud iam roles list --project=my-project

# Update a custom role (add permissions)
gcloud iam roles update customLogViewer \
  --project=my-project \
  --add-permissions=monitoring.dashboards.get

# Delete a custom role
gcloud iam roles delete customLogViewer --project=my-project
```

---

## gcloud builds

Manage Cloud Build pipelines, triggers, and build history.

```bash
# Submit a build using cloudbuild.yaml in current directory
gcloud builds submit --config=cloudbuild.yaml .

# Submit a build with substitutions (custom variables)
gcloud builds submit \
  --config=cloudbuild.yaml \
  --substitutions=_ENV=prod,_VERSION=v1.2.0 \
  .

# Quick build: just build and push a Docker image
gcloud builds submit \
  --tag=us-central1-docker.pkg.dev/my-project/my-repo/my-app:v1.0 \
  .

# Submit a build from a Cloud Storage source
gcloud builds submit \
  --config=cloudbuild.yaml \
  gs://my-source-bucket/source.tar.gz

# List recent builds (with status)
gcloud builds list --limit=20

# List builds filtered by status
gcloud builds list --filter="status=FAILURE" --limit=10

# Describe a build (see steps, timing, logs link)
gcloud builds describe BUILD_ID

# Stream build logs in real-time
gcloud builds log BUILD_ID --stream

# Cancel a running build
gcloud builds cancel BUILD_ID

# --- Triggers ---

# List all triggers
gcloud builds triggers list

# Describe a trigger
gcloud builds triggers describe TRIGGER_NAME

# Create a trigger for pushes to main branch (GitHub)
gcloud builds triggers create github \
  --repo-name=my-repo \
  --repo-owner=my-org \
  --branch-pattern='^main$' \
  --build-config=cloudbuild.yaml \
  --name=push-to-main \
  --description="Build and deploy on push to main"

# Create a trigger for pull requests
gcloud builds triggers create github \
  --repo-name=my-repo \
  --repo-owner=my-org \
  --pull-request-pattern='^main$' \
  --build-config=cloudbuild-pr.yaml \
  --name=pr-checks \
  --comment-control=COMMENTS_ENABLED

# Run a trigger manually (specify branch)
gcloud builds triggers run push-to-main --branch=main

# Disable a trigger
gcloud builds triggers update push-to-main --no-build-logs-bucket

# Delete a trigger
gcloud builds triggers delete old-trigger --quiet

# Get worker pool list (private pools)
gcloud builds worker-pools list --region=us-central1
```

---

## gcloud run

Manage Cloud Run services and jobs.

```bash
# Deploy a new Cloud Run service
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --platform=managed \
  --region=us-central1 \
  --allow-unauthenticated \
  --port=8080 \
  --cpu=1 \
  --memory=512Mi \
  --concurrency=80 \
  --min-instances=0 \
  --max-instances=100 \
  --service-account=run-sa@my-project.iam.gserviceaccount.com

# Deploy with environment variables
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --region=us-central1 \
  --set-env-vars=DB_HOST=10.0.0.1,APP_ENV=production,LOG_LEVEL=INFO

# Deploy with secrets from Secret Manager
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --region=us-central1 \
  --set-secrets=DB_PASSWORD=db-password:latest,API_KEY=stripe-api-key:2

# Deploy with VPC connector (for accessing private resources)
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --region=us-central1 \
  --vpc-connector=my-vpc-connector \
  --vpc-egress=private-ranges-only

# List all Cloud Run services
gcloud run services list --region=us-central1

# List services across all regions
gcloud run services list --platform=managed

# Describe a service (URL, concurrency, latest revision, etc.)
gcloud run services describe my-service --region=us-central1

# Update a service configuration (without redeploying image)
gcloud run services update my-service \
  --region=us-central1 \
  --memory=1Gi \
  --concurrency=100

# Traffic splitting (canary deployment — 10% to new revision)
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-revisions=my-service-00015-abc=10,LATEST=0

# Promote all traffic to the latest revision
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-latest

# List revisions for a service
gcloud run revisions list \
  --service=my-service \
  --region=us-central1

# Describe a specific revision
gcloud run revisions describe my-service-00015-abc \
  --region=us-central1

# Delete an old revision
gcloud run revisions delete my-service-00010-xyz \
  --region=us-central1 --quiet

# Grant invocation access to all users (make public)
gcloud run services add-iam-policy-binding my-service \
  --region=us-central1 \
  --member=allUsers \
  --role=roles/run.invoker

# Grant invocation access to a service account (service-to-service)
gcloud run services add-iam-policy-binding my-service \
  --region=us-central1 \
  --member=serviceAccount:caller-sa@my-project.iam.gserviceaccount.com \
  --role=roles/run.invoker

# --- Cloud Run Jobs ---

# Create a Cloud Run Job
gcloud run jobs create data-export \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/exporter:latest \
  --region=us-central1 \
  --tasks=1 \
  --max-retries=3 \
  --task-timeout=600

# Execute a job (run it now)
gcloud run jobs execute data-export --region=us-central1

# List job executions
gcloud run jobs executions list \
  --job=data-export \
  --region=us-central1

# Delete a service
gcloud run services delete my-service --region=us-central1 --quiet
```

---

## gcloud functions

Manage Cloud Functions deployments and invocations.

```bash
# Deploy an HTTP function (2nd gen, Python)
gcloud functions deploy process-webhook \
  --gen2 \
  --runtime=python311 \
  --region=us-central1 \
  --source=. \
  --entry-point=handle_webhook \
  --trigger-http \
  --allow-unauthenticated \
  --memory=256MiB \
  --timeout=60s \
  --set-env-vars=LOG_LEVEL=INFO

# Deploy a function from a Cloud Storage source
gcloud functions deploy process-webhook \
  --gen2 \
  --runtime=python311 \
  --region=us-central1 \
  --source=gs://my-bucket/source.zip \
  --entry-point=handle_webhook \
  --trigger-http

# Deploy a Pub/Sub triggered function
gcloud functions deploy process-events \
  --gen2 \
  --runtime=nodejs20 \
  --region=us-central1 \
  --source=. \
  --entry-point=processEvent \
  --trigger-topic=my-events-topic \
  --memory=512MiB

# Deploy a Cloud Storage triggered function
gcloud functions deploy on-file-upload \
  --gen2 \
  --runtime=python311 \
  --region=us-central1 \
  --source=. \
  --entry-point=handle_upload \
  --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
  --trigger-event-filters="bucket=my-uploads-bucket"

# Deploy a function with secrets
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python311 \
  --region=us-central1 \
  --source=. \
  --entry-point=handler \
  --trigger-http \
  --set-secrets=DB_PASSWORD=db-password:latest

# List all functions in a region
gcloud functions list --region=us-central1

# Describe a function (URL, triggers, config)
gcloud functions describe process-webhook --region=us-central1 --gen2

# Invoke an HTTP function
gcloud functions call process-webhook \
  --region=us-central1 \
  --gen2 \
  --data='{"key": "value"}'

# View function logs (last 50 lines)
gcloud functions logs read process-webhook \
  --region=us-central1 \
  --gen2 \
  --limit=50

# Tail function logs in real-time (via Cloud Logging)
gcloud logging tail \
  'resource.type="cloud_run_revision" AND labels."goog-managed-by"="cloudfunctions"'

# Update a function's environment variables
gcloud functions deploy process-webhook \
  --gen2 \
  --runtime=python311 \
  --region=us-central1 \
  --source=. \
  --entry-point=handle_webhook \
  --trigger-http \
  --update-env-vars=NEW_VAR=value,EXISTING_VAR=new_value

# Delete a function
gcloud functions delete process-webhook \
  --region=us-central1 --gen2 --quiet
```

---

## gcloud sql

Manage Cloud SQL instances, databases, and backups.

```bash
# Create a PostgreSQL instance
gcloud sql instances create my-postgres \
  --database-version=POSTGRES_15 \
  --tier=db-g1-small \
  --region=us-central1 \
  --storage-type=SSD \
  --storage-size=20GB \
  --storage-auto-increase \
  --backup-start-time=03:00 \
  --availability-type=REGIONAL \
  --deletion-protection

# Create a MySQL instance
gcloud sql instances create my-mysql \
  --database-version=MYSQL_8_0 \
  --tier=db-n1-standard-1 \
  --region=us-central1 \
  --storage-type=SSD \
  --storage-size=20GB

# List all Cloud SQL instances
gcloud sql instances list

# Describe an instance
gcloud sql instances describe my-postgres

# Connect to an instance using the Cloud SQL Auth Proxy (recommended)
# (Download proxy separately: https://cloud.google.com/sql/docs/postgres/sql-proxy)
./cloud-sql-proxy my-project:us-central1:my-postgres --port=5432 &
psql -h 127.0.0.1 -p 5432 -U postgres

# Connect directly via gcloud (opens psql/mysql shell)
gcloud sql connect my-postgres --user=postgres --database=my_db

# Create a database
gcloud sql databases create my_app_db --instance=my-postgres

# List databases
gcloud sql databases list --instance=my-postgres

# Delete a database
gcloud sql databases delete old_db --instance=my-postgres

# Create a user
gcloud sql users create app_user \
  --instance=my-postgres \
  --password=SecurePass123!

# List users
gcloud sql users list --instance=my-postgres

# Change a user's password
gcloud sql users set-password app_user \
  --instance=my-postgres \
  --password=NewSecurePass456!

# Create a read replica (in a different region for DR)
gcloud sql instances create my-postgres-replica \
  --master-instance-name=my-postgres \
  --region=us-east1

# Export database to Cloud Storage
gcloud sql export sql my-postgres \
  gs://my-backups/my_app_db-$(date +%Y%m%d).sql.gz \
  --database=my_app_db \
  --offload

# Import database from Cloud Storage
gcloud sql import sql my-postgres \
  gs://my-backups/my_app_db-20240101.sql.gz \
  --database=my_app_db

# Trigger a manual backup
gcloud sql backups create --instance=my-postgres

# List backups
gcloud sql backups list --instance=my-postgres

# Restore from a backup
gcloud sql backups restore BACKUP_ID \
  --restore-instance=my-postgres

# Patch an instance (resize, update settings)
gcloud sql instances patch my-postgres \
  --tier=db-n1-standard-2 \
  --storage-size=50GB

# Delete an instance (use --async for large instances)
gcloud sql instances delete my-postgres --quiet
```

---

## gcloud logging

Query and manage Cloud Logging logs, sinks, and metrics.

```bash
# Read recent logs for the project
gcloud logging read "severity>=WARNING" \
  --limit=50 \
  --format="table(timestamp,severity,resource.type,textPayload)"

# Read logs for a specific resource type
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="my-service"' \
  --limit=100 \
  --freshness=1h

# Read logs within a time range
gcloud logging read \
  'resource.type="gce_instance" AND timestamp>="2024-01-01T00:00:00Z" AND timestamp<="2024-01-01T23:59:59Z"' \
  --limit=1000

# Read only ERROR and CRITICAL logs
gcloud logging read 'severity=ERROR OR severity=CRITICAL' \
  --limit=100

# Tail logs in real-time
gcloud logging tail 'resource.type="cloud_run_revision"'

# Tail logs for a specific service
gcloud logging tail \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="my-api"'

# List all log names in a project
gcloud logging logs list

# Write a log entry (useful for testing sinks)
gcloud logging write my-test-log "Test message from CLI" \
  --severity=INFO

# Write a structured JSON log entry
gcloud logging write my-app-log \
  '{"severity":"ERROR","message":"Database connection failed","db":"postgres","retries":3}' \
  --payload-type=json

# --- Log Sinks ---

# Create a BigQuery sink (export all logs)
gcloud logging sinks create all-logs-bq \
  bigquery.googleapis.com/projects/my-project/datasets/all_logs \
  --log-filter=''

# Create a Cloud Storage sink (export audit logs)
gcloud logging sinks create audit-to-gcs \
  storage.googleapis.com/my-audit-logs-bucket \
  --log-filter='logName:"cloudaudit.googleapis.com"'

# Create a Pub/Sub sink (stream logs to a topic)
gcloud logging sinks create errors-to-pubsub \
  pubsub.googleapis.com/projects/my-project/topics/log-errors \
  --log-filter='severity>=ERROR'

# List all sinks
gcloud logging sinks list

# Describe a sink
gcloud logging sinks describe all-logs-bq

# Update a sink's filter
gcloud logging sinks update all-logs-bq \
  --log-filter='severity>=WARNING'

# Delete a sink
gcloud logging sinks delete old-sink

# --- Log Metrics ---

# Create a log-based metric (count 5xx errors)
gcloud logging metrics create http-5xx-errors \
  --description="Count of HTTP 5xx errors from Cloud Run" \
  --log-filter='resource.type="cloud_run_revision" AND httpRequest.status>=500'

# List log-based metrics
gcloud logging metrics list

# Delete a log-based metric
gcloud logging metrics delete http-5xx-errors
```

---

## gcloud monitoring

Manage Cloud Monitoring uptime checks, notification channels, and alerting policies.

```bash
# --- Uptime Checks ---

# List all uptime checks
gcloud monitoring uptime list

# Create an HTTPS uptime check
gcloud monitoring uptime create \
  --display-name="Production API Uptime" \
  --http-check-path="/health" \
  --hostname=api.myapp.com \
  --port=443 \
  --use-ssl \
  --period=60 \
  --timeout=10 \
  --regions=USA,EUROPE,ASIA_PACIFIC

# Create an HTTP uptime check on a specific IP (GCE)
gcloud monitoring uptime create \
  --display-name="Internal Service Check" \
  --http-check-path="/ping" \
  --monitored-resource-labels=instance_id=INSTANCE_ID,project_id=my-project \
  --monitored-resource-type=gce_instance

# Describe an uptime check
gcloud monitoring uptime describe CHECK_ID

# Delete an uptime check
gcloud monitoring uptime delete CHECK_ID

# --- Notification Channels ---

# List all notification channels
gcloud monitoring channels list

# Describe a notification channel
gcloud monitoring channels describe CHANNEL_ID

# Create an email notification channel
gcloud monitoring channels create \
  --display-name="On-Call Team" \
  --type=email \
  --channel-labels=email_address=oncall@mycompany.com

# Create a Slack notification channel (requires workspace token)
gcloud monitoring channels create \
  --display-name="Alerts Slack Channel" \
  --type=slack \
  --channel-labels=channel_name=#alerts

# Delete a notification channel
gcloud monitoring channels delete CHANNEL_ID

# --- Alerting Policies ---

# List all alerting policies
gcloud monitoring policies list

# List enabled alerting policies only
gcloud monitoring policies list --filter="enabled=true"

# Describe an alerting policy
gcloud monitoring policies describe POLICY_ID

# Create an alerting policy from a JSON file
gcloud monitoring policies create \
  --policy-from-file=alert-policy.json

# Update an alerting policy from a JSON file
gcloud monitoring policies update POLICY_ID \
  --policy-from-file=updated-policy.json

# Disable an alerting policy
gcloud monitoring policies update POLICY_ID \
  --no-enabled

# Enable an alerting policy
gcloud monitoring policies update POLICY_ID \
  --enabled

# Delete an alerting policy
gcloud monitoring policies delete POLICY_ID

# --- Metrics ---

# List all available metric descriptors for a service
gcloud monitoring metric-descriptors list \
  --filter="metric.type:run.googleapis.com" \
  --format="table(type,metricKind,valueType)"

# List metrics for GKE
gcloud monitoring metric-descriptors list \
  --filter="metric.type:kubernetes.io"

# Read time series data for a metric (last 5 minutes of Cloud Run request count)
gcloud monitoring time-series list \
  'metric.type="run.googleapis.com/request_count" AND resource.labels.service_name="my-service"' \
  --start="2024-01-01T12:00:00Z" \
  --end="2024-01-01T12:05:00Z"
```

---

*For deep explanations of each service, see [CONCEPTS.md](CONCEPTS.md). For getting started quickly, see [QUICK-START.md](QUICK-START.md).*
