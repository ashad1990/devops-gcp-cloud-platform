# Quick Start: GCP in 10 Minutes

> Deploy your first GCP resources from scratch — a Cloud Run service, a storage bucket, and your first logs — in under 10 minutes.

---

## Prerequisites

- A Google account (free at https://accounts.google.com)
- A terminal (Linux, macOS, or Windows WSL2)
- Internet access

---

## Step 1 — Install the gcloud SDK (~2 minutes)

The `gcloud` CLI is your primary interface for GCP. Install it once and it works everywhere.

### Linux / macOS (automated install)

```bash
curl https://sdk.cloud.google.com | bash

# Restart your shell to update PATH
exec -l $SHELL

# Verify installation
gcloud version
```

### macOS (Homebrew)

```bash
brew install --cask google-cloud-sdk
gcloud version
```

### Windows

Download and run the installer from:
https://cloud.google.com/sdk/docs/install#windows

Or via PowerShell (winget):
```powershell
winget install Google.CloudSDK
```

### Alternative: Use Cloud Shell (zero install required)

If you don't want to install anything locally, GCP provides a **free browser-based terminal** with `gcloud` pre-installed and authenticated:

1. Go to https://console.cloud.google.com
2. Click the **Cloud Shell** icon (terminal icon, top-right)
3. A terminal opens with `gcloud`, `kubectl`, `terraform`, `git`, and more

---

## Step 2 — Authenticate (~1 minute)

```bash
# Log in with your Google account (opens a browser window)
gcloud auth login

# Set up Application Default Credentials (for SDKs and libraries)
gcloud auth application-default login

# Verify you are logged in
gcloud auth list
```

You should see your Google account listed as the active account.

---

## Step 3 — Create a Project (~1 minute)

GCP organizes all resources into **projects**. Create a dedicated project for this lab.

```bash
# Create a project with a unique ID (Project IDs must be globally unique)
export PROJECT_ID="gcp-devops-lab-$(date +%s)"
gcloud projects create $PROJECT_ID --name="GCP DevOps Lab"

# Set this project as the default
gcloud config set project $PROJECT_ID

# Confirm your active configuration
gcloud config list

# Link billing to the project (required to create resources)
# Find your billing account ID:
gcloud billing accounts list

# Link billing (replace BILLING_ACCOUNT_ID with your account ID)
gcloud billing projects link $PROJECT_ID \
  --billing-account=BILLING_ACCOUNT_ID
```

> **No billing account?** New accounts get **$300 free credit** for 90 days. Add a credit card at https://console.cloud.google.com/billing — you will not be charged unless you manually upgrade after credits expire.

---

## Step 4 — Enable Required APIs (~1 minute)

GCP services require their APIs to be enabled before use.

```bash
gcloud services enable \
  run.googleapis.com \
  compute.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  storage.googleapis.com \
  logging.googleapis.com \
  monitoring.googleapis.com

# Check which APIs are now enabled
gcloud services list --enabled --format="table(config.name,config.title)"
```

---

## Step 5 — Deploy to Cloud Run (~2 minutes)

Cloud Run is the fastest way to deploy a containerized application — no VMs, no Kubernetes setup.

```bash
# Set your region
gcloud config set run/region us-central1

# Deploy a pre-built "Hello World" container
gcloud run deploy hello-world \
  --image=us-docker.pkg.dev/cloudrun/container/hello \
  --platform=managed \
  --region=us-central1 \
  --allow-unauthenticated \
  --port=8080

# The command will output a service URL like:
# Service URL: https://hello-world-abc123-uc.a.run.app

# Test it
SERVICE_URL=$(gcloud run services describe hello-world \
  --region=us-central1 \
  --format="value(status.url)")

curl $SERVICE_URL
```

You should see a "Hello World!" response. Your first serverless service is live!

---

## Step 6 — Create a Storage Bucket (~1 minute)

```bash
# Create a unique bucket name
export BUCKET_NAME="${PROJECT_ID}-assets"

# Create the bucket in us-central1
gcloud storage buckets create gs://$BUCKET_NAME \
  --location=us-central1 \
  --storage-class=STANDARD \
  --uniform-bucket-level-access

# Create a test file and upload it
echo "Hello from GCP!" > hello.txt
gcloud storage cp hello.txt gs://$BUCKET_NAME/hello.txt

# Verify the upload
gcloud storage ls gs://$BUCKET_NAME/

# Read the file back
gcloud storage cp gs://$BUCKET_NAME/hello.txt -
```

---

## Step 7 — Deploy a VM (Compute Engine) (~2 minutes)

```bash
# Set the default zone
gcloud config set compute/zone us-central1-a

# Create a small VM (e2-micro is always-free eligible)
gcloud compute instances create my-first-vm \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=10GB \
  --tags=http-server

# Allow HTTP traffic (firewall rule)
gcloud compute firewall-rules create allow-http \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80 \
  --target-tags=http-server \
  --source-ranges=0.0.0.0/0

# SSH into the VM
gcloud compute ssh my-first-vm --zone=us-central1-a

# (Inside the VM) Install and start a web server
sudo apt-get update && sudo apt-get install -y nginx
sudo systemctl start nginx
exit

# Get the external IP of the VM
gcloud compute instances describe my-first-vm \
  --zone=us-central1-a \
  --format="value(networkInterfaces[0].accessConfigs[0].natIP)"

# Test it (replace with your IP)
curl http://EXTERNAL_IP
```

---

## Step 8 — View Logs (~1 minute)

GCP centralizes all logs in Cloud Logging.

```bash
# View recent logs from Cloud Run
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="hello-world"' \
  --limit=20 \
  --format="table(timestamp,httpRequest.status,httpRequest.requestUrl)"

# View recent logs from the VM
gcloud logging read \
  'resource.type="gce_instance"' \
  --limit=20 \
  --format="table(timestamp,severity,textPayload)"

# Tail all logs in real-time
gcloud logging tail 'severity>=WARNING'

# Write a test log entry
gcloud logging write my-lab-log \
  '{"message": "Quick start complete!", "step": "step-8"}' \
  --payload-type=json

# Read it back
gcloud logging read 'logName:"my-lab-log"' --limit=5
```

---

## Step 9 — Cleanup (Important!)

Delete all resources to avoid ongoing charges.

```bash
# Delete the Cloud Run service
gcloud run services delete hello-world \
  --region=us-central1 --quiet

# Delete the VM
gcloud compute instances delete my-first-vm \
  --zone=us-central1-a --quiet

# Delete the firewall rule
gcloud compute firewall-rules delete allow-http --quiet

# Delete the storage bucket (and all its contents)
gcloud storage rm -r gs://$BUCKET_NAME
gcloud storage buckets delete gs://$BUCKET_NAME

# Optionally, delete the entire project (cleanest option)
gcloud projects delete $PROJECT_ID

echo "Cleanup complete!"
```

---

## What You Accomplished

In 10 minutes, you:

| Step | Resource | What You Learned |
|------|---------|-----------------|
| 1 | gcloud SDK | CLI installation and PATH setup |
| 2 | Authentication | gcloud auth, Application Default Credentials |
| 3 | GCP Project | Resource hierarchy, billing linking |
| 4 | API enablement | Service APIs must be enabled before use |
| 5 | Cloud Run | Serverless container deployment, URLs |
| 6 | Cloud Storage | Bucket creation, file upload/download |
| 7 | Compute Engine | VM creation, SSH, firewall rules |
| 8 | Cloud Logging | Log reading, filtering, tailing |
| 9 | Cleanup | Cost management — always clean up! |

---

## Next Steps

| Goal | Resource |
|------|----------|
| Complete environment setup | [SETUP.md](SETUP.md) |
| Learn core GCP concepts | [CONCEPTS.md](CONCEPTS.md) |
| Master the gcloud CLI | [COMMANDS.md](COMMANDS.md) |
| Explore GCP DevOps tools | [TOOLS.md](TOOLS.md) |
| Troubleshoot issues | [TROUBLESHOOTING.md](TROUBLESHOOTING.md) |
| Find learning resources | [RESOURCES.md](RESOURCES.md) |

---

*Tip: Bookmark the [GCP Console](https://console.cloud.google.com) — it provides a visual view of everything you just created from the CLI.*
