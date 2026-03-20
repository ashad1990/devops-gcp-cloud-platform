# LAB-11: Infrastructure as Code with Terraform on GCP

**Topic:** Infrastructure as Code (IaC)  
**Estimated Time:** 90 minutes  
**Difficulty:** 🔴 Advanced  
**Cost:** ~$0.50

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Install and configure Terraform with the GCP provider
2. Define GCP resources declaratively using HCL (HashiCorp Configuration Language)
3. Use variables, outputs, locals, and validation blocks for clean configuration
4. Configure remote state storage using a GCS backend for team collaboration
5. Build reusable Terraform modules for common GCP infrastructure patterns
6. Manage multiple environments (dev, staging, prod) using Terraform workspaces

---

## Prerequisites

- [LAB-01](LAB-01-gcp-basics.md) completed
- Terraform >= 1.5.0 installed (see Part 1)
- `gcloud auth application-default login` completed for Terraform authentication
- Basic understanding of infrastructure concepts (VMs, buckets, networks)

---

## Lab Overview

Terraform is the industry-standard tool for Infrastructure as Code, allowing you to define, version, and manage GCP resources declaratively. In this lab you will write Terraform configurations for GCS buckets and Compute Engine VMs, configure a GCS remote backend for state management, create a reusable module, and use workspaces for environment separation.

---

## Part 1: Install Terraform

### 1.1 Installation

```bash
# Linux (Debian/Ubuntu) — using HashiCorp's official repository
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

curl -fsSL https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update && sudo apt-get install -y terraform

# macOS
# brew tap hashicorp/tap && brew install hashicorp/tap/terraform

# Verify installation
terraform version
```

### 1.2 tfenv (Recommended for Version Management)

```bash
# Install tfenv for managing multiple Terraform versions
git clone --depth=1 https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Install and use Terraform 1.6.x
tfenv install 1.6.6
tfenv use 1.6.6
terraform version
```

---

## Environment Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export ZONE=us-central1-a

echo "Project: $PROJECT_ID"
echo "Region:  $REGION"
echo "Zone:    $ZONE"

# Authenticate Terraform with GCP
gcloud auth application-default login

# Verify auth
gcloud auth application-default print-access-token | head -c 30
echo "..."
echo "✅ Application Default Credentials configured"
```

---

## Part 2: Configure the GCP Provider

### 2.1 Create Project Structure

```bash
mkdir -p terraform-lab && cd terraform-lab
```

### 2.2 versions.tf

```bash
cat > versions.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}
EOF
```

### 2.3 provider.tf

```bash
cat > provider.tf << 'EOF'
provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}
EOF
```

### 2.4 variables.tf

```bash
cat > variables.tf << 'EOF'
variable "project_id" {
  description = "The GCP project ID where resources will be created"
  type        = string
}

variable "region" {
  description = "The GCP region for resource deployment"
  type        = string
  default     = "us-central1"
}

variable "zone" {
  description = "The GCP zone for zonal resources"
  type        = string
  default     = "us-central1-a"
}

variable "environment" {
  description = "The deployment environment (dev, staging, prod)"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "owner" {
  description = "Owner or team responsible for these resources"
  type        = string
  default     = "devops-team"
}

variable "enable_deletion_protection" {
  description = "Enable deletion protection on critical resources"
  type        = bool
  default     = false
}
EOF
```

### 2.5 locals.tf

```bash
cat > locals.tf << 'EOF'
locals {
  common_labels = {
    environment = var.environment
    owner       = var.owner
    managed_by  = "terraform"
    project     = var.project_id
  }

  name_prefix = "${var.project_id}-${var.environment}"

  is_production = var.environment == "prod"
}
EOF
```

### 2.6 terraform.tfvars

```bash
cat > terraform.tfvars << EOF
project_id  = "${PROJECT_ID}"
region      = "us-central1"
zone        = "us-central1-a"
environment = "dev"
owner       = "devops-student"
EOF

cat terraform.tfvars
```

### 2.7 Initialize Terraform

```bash
terraform init

# Expected output:
# Initializing the backend...
# Initializing provider plugins...
# - Installing hashicorp/google v5.x.x...
# Terraform has been successfully initialized!
```

---

## Part 3: Create GCS Buckets

### 3.1 storage.tf

```bash
cat > storage.tf << 'EOF'
resource "google_storage_bucket" "devops_main" {
  name          = "${local.name_prefix}-terraform-main"
  location      = var.region
  force_destroy = !local.is_production

  storage_class = "STANDARD"

  versioning {
    enabled = true
  }

  lifecycle_rule {
    condition {
      age                   = 30
      matches_storage_class = ["STANDARD"]
    }
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
  }

  lifecycle_rule {
    condition {
      age = local.is_production ? 365 : 90
    }
    action {
      type = "Delete"
    }
  }

  lifecycle_rule {
    condition {
      num_newer_versions = local.is_production ? 10 : 3
      is_live            = false
    }
    action {
      type = "Delete"
    }
  }

  uniform_bucket_level_access = true

  labels = merge(local.common_labels, {
    purpose = "main-storage"
  })
}

resource "google_storage_bucket" "devops_artifacts" {
  name          = "${local.name_prefix}-artifacts"
  location      = var.region
  force_destroy = !local.is_production

  storage_class = "STANDARD"

  versioning {
    enabled = local.is_production
  }

  uniform_bucket_level_access = true

  labels = merge(local.common_labels, {
    purpose = "build-artifacts"
  })
}

output "main_bucket_name" {
  description = "Name of the main storage bucket"
  value       = google_storage_bucket.devops_main.name
}

output "main_bucket_url" {
  description = "Self-link URL of the main storage bucket"
  value       = google_storage_bucket.devops_main.url
}

output "artifacts_bucket_name" {
  description = "Name of the artifacts storage bucket"
  value       = google_storage_bucket.devops_artifacts.name
}
EOF
```

### 3.2 Plan and Apply

```bash
# Preview changes before applying
terraform plan

# Apply the changes
terraform apply -auto-approve

# View outputs
terraform output

# View Terraform state
terraform show

# List resources in state
terraform state list
```

**Expected output for `terraform output`:**
```
artifacts_bucket_name = "myproject-dev-artifacts"
main_bucket_name = "myproject-dev-terraform-main"
main_bucket_url = "gs://myproject-dev-terraform-main"
```

---

## Part 4: Create a VM with Terraform

### 4.1 compute.tf

```bash
cat > compute.tf << 'EOF'
resource "google_compute_instance" "lab_vm" {
  name         = "${local.name_prefix}-vm"
  machine_type = local.is_production ? "e2-small" : "e2-micro"
  zone         = var.zone

  tags = ["http-server", "terraform-managed", var.environment]

  boot_disk {
    initialize_params {
      image  = "debian-cloud/debian-11"
      size   = local.is_production ? 50 : 20
      type   = local.is_production ? "pd-ssd" : "pd-standard"
    }
  }

  network_interface {
    network = "default"

    access_config {
      # Ephemeral external IP
    }
  }

  metadata = {
    enable-oslogin = "TRUE"
    startup-script = <<-SCRIPT
      #!/bin/bash
      set -e
      apt-get update -y
      apt-get install -y apache2 curl
      systemctl enable apache2
      systemctl start apache2

      cat > /var/www/html/index.html << HTML
      <!DOCTYPE html>
      <html>
      <head><title>Terraform VM - ${var.environment}</title></head>
      <body>
        <h1>Managed by Terraform</h1>
        <p>Environment: ${var.environment}</p>
        <p>Project: ${var.project_id}</p>
        <p>Zone: ${var.zone}</p>
      </body>
      </html>
      HTML
    SCRIPT
  }

  labels = merge(local.common_labels, {
    role = "web-server"
  })

  scheduling {
    preemptible         = local.is_production ? false : true
    automatic_restart   = local.is_production ? true : false
    on_host_maintenance = local.is_production ? "MIGRATE" : "TERMINATE"
  }

  deletion_protection = var.enable_deletion_protection

  lifecycle {
    ignore_changes = [
      metadata["ssh-keys"],
    ]
  }
}

resource "google_compute_firewall" "allow_http_terraform" {
  name    = "${local.name_prefix}-allow-http"
  network = "default"

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["http-server"]

  description = "Allow HTTP/HTTPS to terraform-managed web servers (${var.environment})"
}

output "vm_name" {
  description = "Name of the created VM"
  value       = google_compute_instance.lab_vm.name
}

output "vm_external_ip" {
  description = "External IP address of the VM"
  value       = google_compute_instance.lab_vm.network_interface[0].access_config[0].nat_ip
}

output "vm_self_link" {
  description = "Self-link URI of the VM"
  value       = google_compute_instance.lab_vm.self_link
}

output "vm_machine_type" {
  description = "Machine type of the VM"
  value       = google_compute_instance.lab_vm.machine_type
}
EOF

terraform plan
terraform apply -auto-approve
terraform output
```

---

## Part 5: Remote State in GCS

### 5.1 Create a Dedicated State Bucket

```bash
# Create state bucket BEFORE configuring the backend
export TFSTATE_BUCKET="${PROJECT_ID}-terraform-state"
gsutil mb -l $REGION gs://$TFSTATE_BUCKET/
gsutil versioning set on gs://$TFSTATE_BUCKET/

echo "State bucket: gs://$TFSTATE_BUCKET"
```

### 5.2 Configure the GCS Backend

```bash
cat > backend.tf << EOF
terraform {
  backend "gcs" {
    bucket = "${TFSTATE_BUCKET}"
    prefix = "terraform/state/devops-lab"
  }
}
EOF

# Re-initialize Terraform to migrate state to GCS
terraform init -migrate-state

# Confirm with 'yes' when prompted
```

### 5.3 Verify Remote State

```bash
# State is now stored in GCS
gsutil ls gs://$TFSTATE_BUCKET/terraform/state/

# Terraform show still works (reads from remote)
terraform show | head -20

# State pull to local (for inspection)
terraform state pull | python3 -m json.tool | head -30
```

---

## Part 6: Terraform Modules

### 6.1 Create a Reusable GCS Bucket Module

```bash
mkdir -p modules/gcs-bucket

cat > modules/gcs-bucket/main.tf << 'EOF'
resource "google_storage_bucket" "bucket" {
  name          = var.bucket_name
  location      = var.location
  force_destroy = var.force_destroy

  storage_class = var.storage_class

  dynamic "versioning" {
    for_each = var.enable_versioning ? [1] : []
    content {
      enabled = true
    }
  }

  dynamic "lifecycle_rule" {
    for_each = var.lifecycle_rules
    content {
      condition {
        age                   = lifecycle_rule.value.age
        matches_storage_class = lookup(lifecycle_rule.value, "matches_storage_class", null)
      }
      action {
        type          = lifecycle_rule.value.action_type
        storage_class = lookup(lifecycle_rule.value, "storage_class", null)
      }
    }
  }

  uniform_bucket_level_access = true

  labels = var.labels
}
EOF

cat > modules/gcs-bucket/variables.tf << 'EOF'
variable "bucket_name" {
  description = "The globally unique name of the bucket"
  type        = string
}

variable "location" {
  description = "The GCS bucket location (region or multi-region)"
  type        = string
  default     = "us-central1"
}

variable "storage_class" {
  description = "The storage class (STANDARD, NEARLINE, COLDLINE, ARCHIVE)"
  type        = string
  default     = "STANDARD"

  validation {
    condition     = contains(["STANDARD", "NEARLINE", "COLDLINE", "ARCHIVE"], var.storage_class)
    error_message = "Storage class must be one of: STANDARD, NEARLINE, COLDLINE, ARCHIVE."
  }
}

variable "force_destroy" {
  description = "Allow destruction even if bucket contains objects"
  type        = bool
  default     = false
}

variable "enable_versioning" {
  description = "Enable object versioning"
  type        = bool
  default     = false
}

variable "lifecycle_rules" {
  description = "List of lifecycle rules to apply to the bucket"
  type = list(object({
    age                   = number
    action_type           = string
    storage_class         = optional(string)
    matches_storage_class = optional(list(string))
  }))
  default = []
}

variable "labels" {
  description = "A map of labels to apply to the bucket"
  type        = map(string)
  default     = {}
}
EOF

cat > modules/gcs-bucket/outputs.tf << 'EOF'
output "bucket_name" {
  description = "The name of the created bucket"
  value       = google_storage_bucket.bucket.name
}

output "bucket_url" {
  description = "The gs:// URL of the bucket"
  value       = google_storage_bucket.bucket.url
}

output "bucket_self_link" {
  description = "The URI of the created bucket"
  value       = google_storage_bucket.bucket.self_link
}
EOF
```

### 6.2 Use the Module

```bash
cat >> storage.tf << 'EOF'

# Use the reusable module for backup bucket
module "backup_bucket" {
  source = "./modules/gcs-bucket"

  bucket_name       = "${local.name_prefix}-backup"
  location          = var.region
  storage_class     = "NEARLINE"
  enable_versioning = true
  force_destroy     = !local.is_production

  lifecycle_rules = [
    {
      age         = 90
      action_type = "SetStorageClass"
      storage_class = "COLDLINE"
      matches_storage_class = ["NEARLINE"]
    },
    {
      age         = 365
      action_type = "Delete"
    }
  ]

  labels = merge(local.common_labels, {
    purpose = "backup"
  })
}

output "backup_bucket_name" {
  description = "Name of the backup bucket"
  value       = module.backup_bucket.bucket_name
}

output "backup_bucket_url" {
  description = "URL of the backup bucket"
  value       = module.backup_bucket.bucket_url
}
EOF

# Re-init to download module (local module, so just re-init)
terraform init
terraform apply -auto-approve
terraform output
```

---

## Part 7: Terraform Workspaces

### 7.1 Create Environment Workspaces

```bash
# Create staging and prod workspaces
terraform workspace new staging
terraform workspace new prod

# List workspaces
terraform workspace list

# Current workspace
terraform workspace show
```

### 7.2 Deploy to Staging

```bash
# Switch to staging workspace
terraform workspace select staging

# Apply with staging environment (workspace is used automatically)
terraform apply \
  -auto-approve \
  -var="environment=staging" \
  -var="owner=devops-student"

terraform output

# Switch back to dev
terraform workspace select dev
terraform workspace show
```

### 7.3 Import Existing Resources

```bash
# Terraform can import existing GCP resources into state
# Example: import an existing GCS bucket
# gsutil mb gs/${PROJECT_ID}-existing-bucket/

# Import syntax:
# terraform import google_storage_bucket.example my-existing-bucket

# Import the state bucket we created manually
# terraform import google_storage_bucket.state_bucket $TFSTATE_BUCKET

echo "Import example shown — run only if you have existing resources to import"
```

---

## Verification Steps

```bash
# 1. List all Terraform-managed resources
terraform state list

# 2. Show outputs
terraform output

# 3. Verify resources in GCP
MAIN_BUCKET=$(terraform output -raw main_bucket_name)
VM_NAME=$(terraform output -raw vm_name)
VM_IP=$(terraform output -raw vm_external_ip)

echo "Verifying resources..."
gsutil ls gs://$MAIN_BUCKET/ && echo "✅ Main bucket exists"
gcloud compute instances describe $VM_NAME --zone=$ZONE \
  --format='value(status)' | grep RUNNING && echo "✅ VM is running"

# 4. Test web server
curl -s --max-time 5 http://$VM_IP | head -5

# 5. Verify workspaces
terraform workspace list

echo "✅ Verification complete"
```

---

## Cost Estimates

| Resource | Hourly | Monthly |
|----------|--------|---------|
| e2-micro (dev, preemptible) | ~$0.003/hr | ~$2 |
| e2-small (prod) | ~$0.017/hr | ~$12 |
| GCS buckets (3× ~1 MB) | — | <$0.01 |
| State bucket | — | <$0.01 |
| Firewall rule | Free | Free |

---

## Cleanup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export TFSTATE_BUCKET="${PROJECT_ID}-terraform-state"

# Destroy dev workspace resources
terraform workspace select dev
terraform destroy -auto-approve

# Destroy staging workspace resources
terraform workspace select staging
terraform destroy -auto-approve -var="environment=staging" 2>/dev/null || true

# Return to default workspace
terraform workspace select default

# Delete workspaces (must not be the current workspace)
terraform workspace delete staging 2>/dev/null || true
terraform workspace delete prod 2>/dev/null || true

# Delete state bucket
gsutil -m rm -r gs://$TFSTATE_BUCKET/ 2>/dev/null || true
gsutil rb gs://$TFSTATE_BUCKET/ 2>/dev/null || true

# Clean up local Terraform files
cd ..
rm -rf terraform-lab/

echo "✅ Cleanup complete"

# Verify resources are deleted
gcloud compute instances list | grep "${PROJECT_ID}-dev-vm" || echo "VM deleted"
gsutil ls | grep "terraform" || echo "Terraform buckets deleted"
```

---

## Summary

You have successfully:
- ✅ Installed Terraform and configured the GCP provider with ADC
- ✅ Defined GCS buckets and Compute Engine VMs declaratively in HCL
- ✅ Used variables, locals, outputs, and validation blocks
- ✅ Configured a GCS remote backend for shared state management
- ✅ Created a reusable Terraform module for GCS bucket provisioning
- ✅ Used Terraform workspaces to manage dev and staging environments

**Next Lab:** [LAB-12: Cloud SQL Managed Databases](LAB-12-cloud-sql.md)
