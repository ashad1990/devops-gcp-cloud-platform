# LAB-02: Compute Engine Virtual Machines

**Topic:** IaaS / Virtual Machines  
**Estimated Time:** 90 minutes  
**Difficulty:** 🟢 Beginner  
**Cost:** ~$0.50–$2.00

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Create and manage Compute Engine virtual machine instances using gcloud
2. Configure firewall rules to control inbound and outbound network traffic
3. Work with custom machine types and Spot (preemptible) VMs for cost savings
4. Create instance templates and managed instance groups (MIGs)
5. Configure autoscaling policies based on CPU utilization metrics
6. Take and describe disk snapshots for backup and disaster recovery

---

## Prerequisites

- [LAB-01](LAB-01-gcp-basics.md) completed
- `compute.googleapis.com` API enabled
- Active GCP project with billing enabled

---

## Lab Overview

In this lab you will deploy virtual machines on GCP Compute Engine, configure networking and firewall rules, explore cost-saving VM options, set up managed instance groups with autoscaling, and practice disk snapshot operations. These skills form the foundation of IaaS workload management on GCP.

---

## Environment Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export ZONE=us-central1-a
export REGION=us-central1

echo "Project: $PROJECT_ID"
echo "Zone:    $ZONE"
echo "Region:  $REGION"
```

---

## Part 1: Creating Your First VM

### 1.1 Create a Web Server VM with a Startup Script

```bash
gcloud compute instances create web-server-1 \
  --zone=$ZONE \
  --machine-type=e2-micro \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --boot-disk-size=10GB \
  --boot-disk-type=pd-standard \
  --tags=http-server \
  --metadata=startup-script='#!/bin/bash
apt-get update -y
apt-get install -y apache2
systemctl enable apache2
systemctl start apache2
echo "<h1>Hello from $(hostname)</h1><p>Instance: $(curl -s http://metadata.google.internal/computeMetadata/v1/instance/name -H Metadata-Flavor:Google)</p>" > /var/www/html/index.html'
```

**Expected output:**
```
NAME          ZONE           MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
web-server-1  us-central1-a  e2-micro                   10.128.0.2   34.123.45.67    RUNNING
```

### 1.2 List and Describe Instances

```bash
# List all instances
gcloud compute instances list

# Get detailed info about the VM
gcloud compute instances describe web-server-1 --zone=$ZONE

# Get just the external IP
gcloud compute instances describe web-server-1 \
  --zone=$ZONE \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)'

# Get instance status
gcloud compute instances list \
  --filter="name:web-server-1" \
  --format='table(name,status,machineType,zone)'
```

### 1.3 Start, Stop, and Reset

```bash
# Stop an instance (stops billing for CPU/RAM, disk still billed)
# gcloud compute instances stop web-server-1 --zone=$ZONE

# Start a stopped instance
# gcloud compute instances start web-server-1 --zone=$ZONE

# Reset (hard reboot)
# gcloud compute instances reset web-server-1 --zone=$ZONE
```

---

## Part 2: SSH and Software Installation

### 2.1 SSH into the VM

```bash
# SSH using gcloud (handles key management automatically)
gcloud compute ssh web-server-1 --zone=$ZONE
```

### 2.2 Inside the VM — Run These Commands

```bash
# Update package lists
sudo apt-get update

# Install nginx
sudo apt-get install -y nginx

# Check nginx status
sudo systemctl status nginx

# View Apache status (installed by startup script)
sudo systemctl status apache2

# Check what's listening on port 80
sudo ss -tlnp | grep :80

# View startup script log
sudo journalctl -u google-startup-scripts.service --no-pager | tail -20

# Exit the VM
exit
```

### 2.3 Copy Files to/from VM

```bash
# Copy a local file to the VM
echo "DevOps Lab Test File" > test.txt
gcloud compute scp test.txt web-server-1:~/test.txt --zone=$ZONE

# Copy from VM to local
gcloud compute scp web-server-1:~/test.txt ./test-from-vm.txt --zone=$ZONE

# Run a command on the VM without SSH-ing
gcloud compute ssh web-server-1 --zone=$ZONE \
  --command="cat /var/www/html/index.html"
```

---

## Part 3: Firewall Rules

### 3.1 Create Firewall Rules

```bash
# Allow HTTP traffic to instances tagged 'http-server'
gcloud compute firewall-rules create allow-http \
  --allow=tcp:80 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server \
  --description="Allow HTTP traffic from internet"

# Allow HTTPS traffic
gcloud compute firewall-rules create allow-https \
  --allow=tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=https-server \
  --description="Allow HTTPS traffic from internet"

# List firewall rules
gcloud compute firewall-rules list --format='table(name,network,direction,priority,sourceRanges,allowed)'
```

### 3.2 Test HTTP Access

```bash
# Get the external IP
EXTERNAL_IP=$(gcloud compute instances describe web-server-1 \
  --zone=$ZONE \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)')

echo "External IP: $EXTERNAL_IP"

# Test HTTP
curl -s http://$EXTERNAL_IP | head -5

# Or open in browser: http://$EXTERNAL_IP
```

### 3.3 Update Firewall Priority

```bash
# Describe a firewall rule
gcloud compute firewall-rules describe allow-http

# Update a firewall rule
gcloud compute firewall-rules update allow-http \
  --description="Allow HTTP from internet - updated"
```

---

## Part 4: Custom Machine Types and Spot VMs

### 4.1 Custom Machine Type

```bash
# Create a VM with custom CPU/memory configuration
gcloud compute instances create custom-vm \
  --zone=$ZONE \
  --custom-cpu=2 \
  --custom-memory=4GB \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB

# List instances showing machine type
gcloud compute instances list \
  --format='table(name,machineType.basename(),status,zone)'
```

**Expected output:**
```
NAME         MACHINE_TYPE           STATUS   ZONE
custom-vm    custom (2 vCPU, 4 GB)  RUNNING  us-central1-a
web-server-1 e2-micro               RUNNING  us-central1-a
```

### 4.2 Spot (Preemptible) VM

```bash
# Create a Spot VM (up to 91% cheaper, can be preempted)
gcloud compute instances create spot-vm \
  --zone=$ZONE \
  --machine-type=e2-micro \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --boot-disk-size=10GB

# Verify it's a spot instance
gcloud compute instances describe spot-vm \
  --zone=$ZONE \
  --format='get(scheduling.provisioningModel)'
```

### 4.3 Machine Type Reference

| Type | vCPUs | Memory | Use Case | Cost/hr (approx) |
|------|-------|--------|----------|------------------|
| e2-micro | 0.25 | 1 GB | Dev/test | $0.008 |
| e2-small | 0.5 | 2 GB | Light workloads | $0.017 |
| e2-medium | 1 | 4 GB | General purpose | $0.034 |
| e2-standard-4 | 4 | 16 GB | Production apps | $0.134 |
| n2-standard-8 | 8 | 32 GB | High performance | $0.388 |

---

## Part 5: Instance Templates and Managed Instance Groups

### 5.1 Create an Instance Template

```bash
gcloud compute instance-templates create web-template \
  --machine-type=e2-micro \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --boot-disk-size=10GB \
  --tags=http-server \
  --metadata=startup-script='#!/bin/bash
apt-get update -y
apt-get install -y apache2
systemctl enable apache2
systemctl start apache2
HOSTNAME=$(hostname)
INSTANCE_NAME=$(curl -s "http://metadata.google.internal/computeMetadata/v1/instance/name" -H "Metadata-Flavor: Google")
ZONE=$(curl -s "http://metadata.google.internal/computeMetadata/v1/instance/zone" -H "Metadata-Flavor: Google" | awk -F/ "{print \$NF}")
echo "<h1>Instance: $INSTANCE_NAME</h1><p>Zone: $ZONE</p><p>Hostname: $HOSTNAME</p>" > /var/www/html/index.html'

# List instance templates
gcloud compute instance-templates list
gcloud compute instance-templates describe web-template
```

### 5.2 Create a Managed Instance Group

```bash
gcloud compute instance-groups managed create web-mig \
  --base-instance-name=web-instance \
  --size=2 \
  --template=web-template \
  --zone=$ZONE

# Wait for instances to be created
gcloud compute instance-groups managed wait-until web-mig \
  --stable \
  --zone=$ZONE

# List instances in the MIG
gcloud compute instance-groups managed list-instances web-mig \
  --zone=$ZONE
```

**Expected output:**
```
NAME             ZONE           STATUS   HEALTH_STATE  ACTION  PRESERVED_STATE  INSTANCE_TEMPLATE
web-instance-abc  us-central1-a  RUNNING  HEALTHY                               web-template
web-instance-def  us-central1-a  RUNNING  HEALTHY                               web-template
```

### 5.3 Configure Autoscaling

```bash
gcloud compute instance-groups managed set-autoscaling web-mig \
  --zone=$ZONE \
  --min-num-replicas=2 \
  --max-num-replicas=5 \
  --target-cpu-utilization=0.7 \
  --cool-down-period=60

# View autoscaling configuration
gcloud compute instance-groups managed describe web-mig \
  --zone=$ZONE \
  --format='yaml(autoscaler)'

# Manually resize the MIG
gcloud compute instance-groups managed resize web-mig \
  --size=3 \
  --zone=$ZONE
```

### 5.4 Rolling Updates via MIG

```bash
# Create a new template version
gcloud compute instance-templates create web-template-v2 \
  --machine-type=e2-micro \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --tags=http-server \
  --metadata=startup-script='#!/bin/bash
apt-get update -y
apt-get install -y apache2
systemctl start apache2
echo "<h1>Version 2 - $(hostname)</h1>" > /var/www/html/index.html'

# Perform a rolling update
gcloud compute instance-groups managed rolling-action start-update web-mig \
  --version=template=web-template-v2 \
  --zone=$ZONE \
  --max-unavailable=1 \
  --max-surge=1

# Monitor the update
gcloud compute instance-groups managed describe web-mig \
  --zone=$ZONE \
  --format='yaml(status.versionTarget,status.isStable)'
```

---

## Part 6: VM Snapshots

### 6.1 Create a Disk Snapshot

```bash
# Take a snapshot of the web-server-1 boot disk
gcloud compute disks snapshot web-server-1 \
  --snapshot-names=web-server-snapshot-1 \
  --zone=$ZONE \
  --description="Snapshot of web server boot disk - $(date +%Y-%m-%d)"

# List snapshots
gcloud compute snapshots list

# Describe snapshot
gcloud compute snapshots describe web-server-snapshot-1 \
  --format='yaml(name,diskSizeGb,status,sourceDisk,creationTimestamp)'
```

**Expected output:**
```
creationTimestamp: '2024-01-15T10:30:00.000-08:00'
diskSizeGb: '10'
name: web-server-snapshot-1
sourceDisk: https://www.googleapis.com/compute/v1/projects/.../disks/web-server-1
status: READY
```

### 6.2 Create a VM from Snapshot

```bash
# Create a new disk from snapshot
gcloud compute disks create web-server-restored \
  --source-snapshot=web-server-snapshot-1 \
  --zone=$ZONE

# Create a VM using the restored disk
gcloud compute instances create web-server-from-snapshot \
  --zone=$ZONE \
  --machine-type=e2-micro \
  --disk=name=web-server-restored,boot=yes,auto-delete=yes \
  --tags=http-server

gcloud compute instances list
```

---

## Part 7: Startup Scripts Best Practices

### 7.1 Startup Script from File

```bash
# Create a startup script file
cat > startup.sh << 'SCRIPT'
#!/bin/bash
set -e
exec > /var/log/startup-script.log 2>&1

echo "Starting setup at $(date)"

apt-get update -y
apt-get install -y apache2 curl wget htop

# Configure Apache
systemctl enable apache2
systemctl start apache2

# Get instance metadata
INSTANCE_NAME=$(curl -s "http://metadata.google.internal/computeMetadata/v1/instance/name" \
  -H "Metadata-Flavor: Google")
ZONE=$(curl -s "http://metadata.google.internal/computeMetadata/v1/instance/zone" \
  -H "Metadata-Flavor: Google" | awk -F/ '{print $NF}')
PROJECT=$(curl -s "http://metadata.google.internal/computeMetadata/v1/project/project-id" \
  -H "Metadata-Flavor: Google")

# Create a custom index page
cat > /var/www/html/index.html << HTML
<!DOCTYPE html>
<html>
<head><title>GCP DevOps Lab</title></head>
<body>
  <h1>GCP DevOps Lab</h1>
  <ul>
    <li><strong>Instance:</strong> $INSTANCE_NAME</li>
    <li><strong>Zone:</strong> $ZONE</li>
    <li><strong>Project:</strong> $PROJECT</li>
    <li><strong>Timestamp:</strong> $(date)</li>
  </ul>
</body>
</html>
HTML

echo "Setup complete at $(date)"
SCRIPT

# Create VM with startup script from file
gcloud compute instances create scripted-vm \
  --zone=$ZONE \
  --machine-type=e2-micro \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --metadata-from-file=startup-script=startup.sh \
  --tags=http-server
```

### 7.2 Instance Metadata

```bash
# Read instance metadata from within a VM
gcloud compute ssh web-server-1 --zone=$ZONE --command='
  echo "=== Instance Metadata ==="
  curl -s "http://metadata.google.internal/computeMetadata/v1/instance/" \
    -H "Metadata-Flavor: Google"
  echo ""
  echo "=== Project ID ==="
  curl -s "http://metadata.google.internal/computeMetadata/v1/project/project-id" \
    -H "Metadata-Flavor: Google"
'
```

---

## Verification Steps

```bash
# 1. List all instances created in this lab
gcloud compute instances list --filter="zone:$ZONE" \
  --format='table(name,machineType.basename(),status,networkInterfaces[0].accessConfigs[0].natIP)'

# 2. Verify MIG is running
gcloud compute instance-groups managed list --zones=$ZONE

# 3. Verify snapshot exists
gcloud compute snapshots list --filter="name:web-server-snapshot"

# 4. Verify firewall rules
gcloud compute firewall-rules list --filter="name:allow-http OR name:allow-https"

# 5. Test web server
EXTERNAL_IP=$(gcloud compute instances describe web-server-1 \
  --zone=$ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
curl -s http://$EXTERNAL_IP
```

---

## Cost Estimates

| Resource | Hourly Cost | Monthly (730 hrs) |
|----------|-------------|-------------------|
| e2-micro (on-demand) | ~$0.0084/hr | ~$6.11 |
| e2-micro (spot) | ~$0.0025/hr | ~$1.82 |
| custom 2vCPU/4GB | ~$0.067/hr | ~$48.90 |
| Snapshot (10 GB) | — | ~$0.20/GB/month |
| Persistent disk (10 GB) | — | ~$0.40/month |

> 💡 **Tip**: Always delete VMs when not in use. A single e2-micro costs ~$6/month — left running accidentally across multiple labs, costs add up quickly.

---

## Cleanup

> ⚠️ Run ALL cleanup commands to avoid ongoing charges.

```bash
export PROJECT_ID=$(gcloud config get-value project)
export ZONE=us-central1-a
export REGION=us-central1

# Delete VM instances
gcloud compute instances delete web-server-1 --zone=$ZONE --quiet
gcloud compute instances delete custom-vm --zone=$ZONE --quiet
gcloud compute instances delete spot-vm --zone=$ZONE --quiet
gcloud compute instances delete web-server-from-snapshot --zone=$ZONE --quiet
gcloud compute instances delete scripted-vm --zone=$ZONE --quiet 2>/dev/null || true

# Delete managed instance group and templates
gcloud compute instance-groups managed delete web-mig --zone=$ZONE --quiet
gcloud compute instance-templates delete web-template --quiet
gcloud compute instance-templates delete web-template-v2 --quiet 2>/dev/null || true

# Delete standalone disk
gcloud compute disks delete web-server-restored --zone=$ZONE --quiet 2>/dev/null || true

# Delete snapshots
gcloud compute snapshots delete web-server-snapshot-1 --quiet

# Delete firewall rules
gcloud compute firewall-rules delete allow-http --quiet
gcloud compute firewall-rules delete allow-https --quiet

# Clean up local files
rm -f test.txt test-from-vm.txt startup.sh

echo "✅ Cleanup complete"

# Verify all instances are deleted
gcloud compute instances list
```

---

## Summary

You have successfully:
- ✅ Created VMs with startup scripts and custom configurations
- ✅ Connected via SSH and managed software on running VMs
- ✅ Configured firewall rules for web traffic
- ✅ Created custom machine types and Spot VMs for cost optimization
- ✅ Built instance templates and managed instance groups with autoscaling
- ✅ Created and restored disk snapshots

**Next Lab:** [LAB-03: Cloud Storage](LAB-03-cloud-storage.md)
