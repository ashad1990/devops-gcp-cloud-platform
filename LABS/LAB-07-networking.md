# LAB-07: VPC Networking, Firewall Rules, and Load Balancers

**Topic:** GCP Networking  
**Estimated Time:** 90 minutes  
**Difficulty:** 🟡 Intermediate  
**Cost:** ~$1–$3

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Design and create a custom VPC with multiple subnets for a tiered architecture
2. Configure firewall rules using tags, source ranges, and priorities
3. Deploy an HTTP(S) Load Balancer with backend services and health checks
4. Set up Cloud NAT for outbound internet access from private instances
5. Configure Cloud DNS with private managed zones and A records
6. Understand network routing, VPC peering concepts, and load balancer types

---

## Prerequisites

- [LAB-01](LAB-01-gcp-basics.md) completed
- `compute.googleapis.com` and `dns.googleapis.com` APIs enabled
- Familiarity with basic networking concepts (subnets, CIDR, TCP/IP)

---

## Lab Overview

GCP's Virtual Private Cloud (VPC) is a global resource that spans all regions. In this lab you will design a three-tier network architecture (web, app, database), configure appropriate firewall rules for each tier, deploy an HTTP load balancer to distribute traffic across multiple web servers, configure Cloud NAT for private instances, and set up internal DNS.

---

## Environment Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export ZONE=us-central1-a
export VPC_NAME=devops-lab-vpc

echo "Project: $PROJECT_ID"
echo "Region:  $REGION"
echo "Zone:    $ZONE"
echo "VPC:     $VPC_NAME"
```

---

## Part 1: Creating a Custom VPC

### 1.1 Create the VPC Network

```bash
# Create a custom VPC (no auto-created subnets)
gcloud compute networks create $VPC_NAME \
  --subnet-mode=custom \
  --bgp-routing-mode=regional \
  --description="DevOps Lab three-tier VPC"

# Verify VPC creation
gcloud compute networks describe $VPC_NAME \
  --format='yaml(name,subnetworkMode,routingConfig)'
```

### 1.2 Create Subnets for Each Tier

```bash
# Web tier subnet (public-facing)
gcloud compute networks subnets create devops-web-subnet \
  --network=$VPC_NAME \
  --region=$REGION \
  --range=10.0.1.0/24 \
  --description="Web tier - public facing"

# App tier subnet (internal)
gcloud compute networks subnets create devops-app-subnet \
  --network=$VPC_NAME \
  --region=$REGION \
  --range=10.0.2.0/24 \
  --description="Application tier - internal only"

# Database tier subnet (private)
gcloud compute networks subnets create devops-db-subnet \
  --network=$VPC_NAME \
  --region=$REGION \
  --range=10.0.3.0/24 \
  --description="Database tier - highly restricted"

# List all subnets in this VPC
gcloud compute networks subnets list \
  --filter="network:$VPC_NAME" \
  --format='table(name,region,ipCidrRange,network)'
```

**Expected output:**
```
NAME               REGION       IP_CIDR_RANGE  NETWORK
devops-app-subnet  us-central1  10.0.2.0/24    devops-lab-vpc
devops-db-subnet   us-central1  10.0.3.0/24    devops-lab-vpc
devops-web-subnet  us-central1  10.0.1.0/24    devops-lab-vpc
```

### 1.3 Secondary Ranges (for GKE compatibility)

```bash
# Add secondary ranges to the web subnet (useful for GKE Pods/Services)
gcloud compute networks subnets update devops-web-subnet \
  --region=$REGION \
  --add-secondary-ranges=pods=10.1.0.0/16,services=10.2.0.0/20

# Verify secondary ranges
gcloud compute networks subnets describe devops-web-subnet \
  --region=$REGION \
  --format='yaml(secondaryIpRanges)'
```

---

## Part 2: Firewall Rules

### 2.1 Create Tiered Firewall Rules

```bash
# Allow SSH from anywhere (restrict to a specific IP in production)
gcloud compute firewall-rules create ${VPC_NAME}-allow-ssh \
  --network=$VPC_NAME \
  --allow=tcp:22 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=allow-ssh \
  --priority=1000 \
  --description="Allow SSH access"

# Allow HTTP and HTTPS from the internet to web servers
gcloud compute firewall-rules create ${VPC_NAME}-allow-http \
  --network=$VPC_NAME \
  --allow=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server \
  --priority=1000 \
  --description="Allow HTTP/HTTPS traffic from internet"

# Allow all internal VPC traffic
gcloud compute firewall-rules create ${VPC_NAME}-allow-internal \
  --network=$VPC_NAME \
  --allow=tcp:0-65535,udp:0-65535,icmp \
  --source-ranges=10.0.0.0/8 \
  --priority=1000 \
  --description="Allow all internal VPC traffic"

# Required for Google Cloud Load Balancer health checks
gcloud compute firewall-rules create ${VPC_NAME}-allow-health-check \
  --network=$VPC_NAME \
  --allow=tcp \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=web-server \
  --priority=900 \
  --description="Allow GCP health check probes"

# Database tier: only allow traffic from the app tier subnet
gcloud compute firewall-rules create ${VPC_NAME}-allow-db \
  --network=$VPC_NAME \
  --allow=tcp:5432,tcp:3306 \
  --source-ranges=10.0.2.0/24 \
  --target-tags=database-server \
  --priority=1000 \
  --description="Allow DB connections only from app tier"

# List all firewall rules for this VPC
gcloud compute firewall-rules list \
  --filter="network:$VPC_NAME" \
  --format='table(name,direction,priority,sourceRanges,allowed,targetTags)'
```

### 2.2 Firewall Rule Best Practices

```bash
# Deny all other ingress (explicit deny - lower number = higher priority)
# Note: GCP has an implicit deny-all at priority 65535
# This shows you how to explicitly add one at a specific priority

# List with priorities to understand ordering
gcloud compute firewall-rules list \
  --filter="network:$VPC_NAME" \
  --sort-by=priority \
  --format='table(name,priority,direction,sourceRanges,allowed,targetTags)'
```

---

## Part 3: VMs in the Custom VPC

### 3.1 Create Web Server Instances

```bash
# Create two web servers in the web subnet
for i in 1 2; do
  gcloud compute instances create web-server-$i \
    --zone=$ZONE \
    --machine-type=e2-micro \
    --network=$VPC_NAME \
    --subnet=devops-web-subnet \
    --private-network-ip=10.0.1.1$i \
    --tags=web-server,allow-ssh \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --no-address \
    --metadata=startup-script="#!/bin/bash
apt-get update -y
apt-get install -y apache2
systemctl enable apache2
systemctl start apache2
echo '<h1>Web Server $i: '\$(hostname)'</h1><p>Zone: '\$(curl -s http://metadata.google.internal/computeMetadata/v1/instance/zone -H Metadata-Flavor:Google | awk -F/ \"{print \\\$NF}\")'</p>' > /var/www/html/index.html"
done

# Verify instances are running
gcloud compute instances list \
  --filter="tags.items:web-server" \
  --format='table(name,zone,networkInterfaces[0].networkIP,status)'
```

**Expected output:**
```
NAME          ZONE           INTERNAL_IP  STATUS
web-server-1  us-central1-a  10.0.1.11    RUNNING
web-server-2  us-central1-a  10.0.1.12    RUNNING
```

> Note: `--no-address` creates private-only instances. They have no external IP and use Cloud NAT (configured later) for internet access.

---

## Part 4: HTTP Load Balancer

### 4.1 Create an Instance Group

```bash
# Create unmanaged instance group containing both web servers
gcloud compute instance-groups unmanaged create web-servers-ig \
  --zone=$ZONE \
  --description="Web server instance group for load balancing"

# Add instances to the group
gcloud compute instance-groups unmanaged add-instances web-servers-ig \
  --zone=$ZONE \
  --instances=web-server-1,web-server-2

# Set named port for HTTP
gcloud compute instance-groups set-named-ports web-servers-ig \
  --zone=$ZONE \
  --named-ports=http:80

# Verify group
gcloud compute instance-groups unmanaged list-instances web-servers-ig \
  --zone=$ZONE
```

### 4.2 Health Check

```bash
gcloud compute health-checks create http web-health-check \
  --port=80 \
  --request-path=/ \
  --check-interval=10s \
  --timeout=5s \
  --healthy-threshold=2 \
  --unhealthy-threshold=3 \
  --description="HTTP health check for web servers"

# Describe the health check
gcloud compute health-checks describe web-health-check
```

### 4.3 Backend Service

```bash
# Create a global backend service
gcloud compute backend-services create web-backend-service \
  --global \
  --protocol=HTTP \
  --health-checks=web-health-check \
  --port-name=http \
  --connection-draining-timeout=10s

# Add the instance group as a backend
gcloud compute backend-services add-backend web-backend-service \
  --global \
  --instance-group=web-servers-ig \
  --instance-group-zone=$ZONE \
  --balancing-mode=UTILIZATION \
  --max-utilization=0.8

# View backend service
gcloud compute backend-services describe web-backend-service --global
```

### 4.4 URL Map, Proxy, and Forwarding Rule

```bash
# URL map (routes requests to backend services)
gcloud compute url-maps create web-url-map \
  --default-service=web-backend-service

# HTTP proxy
gcloud compute target-http-proxies create web-http-proxy \
  --url-map=web-url-map \
  --description="HTTP proxy for web load balancer"

# Global forwarding rule (assigns external IP)
gcloud compute forwarding-rules create web-forwarding-rule \
  --global \
  --target-http-proxy=web-http-proxy \
  --ports=80 \
  --description="Global HTTP forwarding rule"

# Get the load balancer IP
LB_IP=$(gcloud compute forwarding-rules describe web-forwarding-rule \
  --global --format='value(IPAddress)')
echo "Load Balancer IP: $LB_IP"
echo "⏳ Wait 60-90 seconds for the LB to become active..."
```

### 4.5 Test the Load Balancer

```bash
# Wait for LB to initialize
sleep 90

# Test load balancer
for i in {1..6}; do
  echo -n "Request $i: "
  curl -s --max-time 5 http://$LB_IP | grep -oP '<h1>.*?</h1>' || echo "Not ready yet"
  sleep 2
done

# You should see responses alternating between web-server-1 and web-server-2
```

---

## Part 5: Cloud NAT

Cloud NAT allows instances without external IPs to access the internet for updates, API calls, etc.

### 5.1 Create Cloud Router and NAT

```bash
# Create Cloud Router (required for Cloud NAT)
gcloud compute routers create devops-router \
  --network=$VPC_NAME \
  --region=$REGION \
  --description="Cloud Router for Cloud NAT"

# Create NAT gateway
gcloud compute routers nats create devops-nat \
  --router=devops-router \
  --region=$REGION \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --enable-logging

# Describe the NAT configuration
gcloud compute routers nats describe devops-nat \
  --router=devops-router \
  --region=$REGION
```

### 5.2 Verify NAT Works

```bash
# SSH through IAP (since instances have no external IP)
gcloud compute ssh web-server-1 --zone=$ZONE \
  --tunnel-through-iap \
  --command="curl -s --max-time 10 https://ifconfig.me || echo 'Checking internet access...'"
```

> Note: IAP (Identity-Aware Proxy) tunneling allows SSH without external IPs. Enable the IAP API first: `gcloud services enable iap.googleapis.com`

---

## Part 6: Cloud DNS Basics

### 6.1 Create a Private DNS Zone

```bash
gcloud services enable dns.googleapis.com

# Create a private managed zone (only visible inside the VPC)
gcloud dns managed-zones create devops-internal-zone \
  --description="Internal DNS zone for DevOps lab" \
  --dns-name="devops.internal." \
  --visibility=private \
  --networks=$VPC_NAME

# List managed zones
gcloud dns managed-zones list
```

### 6.2 Add DNS Records

```bash
# Add an A record pointing to the load balancer
gcloud dns record-sets create lb.devops.internal. \
  --zone=devops-internal-zone \
  --type=A \
  --ttl=300 \
  --rrdatas=$LB_IP

# Add CNAME record
gcloud dns record-sets create www.devops.internal. \
  --zone=devops-internal-zone \
  --type=CNAME \
  --ttl=300 \
  --rrdatas=lb.devops.internal.

# List all records in the zone
gcloud dns record-sets list --zone=devops-internal-zone
```

**Expected output:**
```
NAME                     TYPE   TTL   DATA
devops.internal.         NS     21600 ns-cloud-b1.googledomains.com.,...
devops.internal.         SOA    21600 ns-cloud-b1.googledomains.com.
lb.devops.internal.      A      300   34.123.45.67
www.devops.internal.     CNAME  300   lb.devops.internal.
```

---

## Part 7: VPC Flow Logs and Network Intelligence

### 7.1 Enable VPC Flow Logs

```bash
# Enable flow logs on the web subnet
gcloud compute networks subnets update devops-web-subnet \
  --region=$REGION \
  --enable-flow-logs \
  --logging-flow-sampling=0.5 \
  --logging-metadata=include-all

# Verify flow logs are enabled
gcloud compute networks subnets describe devops-web-subnet \
  --region=$REGION \
  --format='yaml(logConfig)'
```

### 7.2 View Flow Logs

```bash
# Wait a minute and generate some traffic
sleep 30
curl -s http://$LB_IP > /dev/null

# View flow logs in Cloud Logging
gcloud logging read \
  'resource.type="gce_subnetwork" AND resource.labels.subnetwork_name="devops-web-subnet"' \
  --limit=5 \
  --format='yaml' 2>/dev/null | head -30 || echo "Flow logs may take a few minutes to appear"
```

---

## Verification Steps

```bash
# 1. List VPC and subnets
gcloud compute networks describe $VPC_NAME --format='value(name,subnetworkMode)'
gcloud compute networks subnets list --filter="network:$VPC_NAME" --format='table(name,ipCidrRange)'

# 2. List firewall rules
gcloud compute firewall-rules list --filter="network:$VPC_NAME" --format='table(name,direction,priority)'

# 3. Test load balancer
echo "Testing LB at http://$LB_IP ..."
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 http://$LB_IP 2>/dev/null || echo "000")
echo "HTTP Status: $HTTP_STATUS"

# 4. Verify Cloud NAT
gcloud compute routers nats describe devops-nat --router=devops-router --region=$REGION \
  --format='value(name,state)'

# 5. Verify DNS zone
gcloud dns managed-zones describe devops-internal-zone --format='value(name,visibility)'

echo "✅ Verification complete"
```

---

## Cost Estimates

| Resource | Hourly Cost | Notes |
|----------|-------------|-------|
| Forwarding rule (LB) | $0.025/hr | ~$18/month |
| External IP (LB) | $0.004/hr | Included in forwarding rule |
| e2-micro instances (×2) | $0.017/hr | ~$12/month |
| Cloud NAT processing | $0.045/GB | First GB free |
| Cloud DNS zone | $0.20/month | Per zone |

---

## Cleanup

> ⚠️ Load balancer components must be deleted in the correct order.

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export ZONE=us-central1-a
export VPC_NAME=devops-lab-vpc

# Delete LB components (in reverse order)
gcloud compute forwarding-rules delete web-forwarding-rule --global --quiet
gcloud compute target-http-proxies delete web-http-proxy --quiet
gcloud compute url-maps delete web-url-map --quiet
gcloud compute backend-services delete web-backend-service --global --quiet
gcloud compute health-checks delete web-health-check --quiet

# Delete instance group and VMs
gcloud compute instance-groups unmanaged delete web-servers-ig --zone=$ZONE --quiet
gcloud compute instances delete web-server-1 web-server-2 --zone=$ZONE --quiet

# Delete Cloud NAT and Router
gcloud compute routers nats delete devops-nat \
  --router=devops-router --region=$REGION --quiet
gcloud compute routers delete devops-router --region=$REGION --quiet

# Delete DNS records and zone
gcloud dns record-sets delete lb.devops.internal. \
  --zone=devops-internal-zone --type=A --quiet 2>/dev/null || true
gcloud dns record-sets delete www.devops.internal. \
  --zone=devops-internal-zone --type=CNAME --quiet 2>/dev/null || true
gcloud dns managed-zones delete devops-internal-zone --quiet

# Delete firewall rules
gcloud compute firewall-rules delete \
  ${VPC_NAME}-allow-ssh \
  ${VPC_NAME}-allow-http \
  ${VPC_NAME}-allow-internal \
  ${VPC_NAME}-allow-health-check \
  ${VPC_NAME}-allow-db \
  --quiet

# Delete subnets (must be before VPC)
gcloud compute networks subnets delete \
  devops-web-subnet devops-app-subnet devops-db-subnet \
  --region=$REGION --quiet

# Delete VPC
gcloud compute networks delete $VPC_NAME --quiet

echo "✅ Cleanup complete"

# Verify VPC is deleted
gcloud compute networks list | grep devops-lab-vpc || echo "VPC deleted successfully"
```

---

## Summary

You have successfully:
- ✅ Created a custom VPC with three-tier subnet architecture
- ✅ Configured firewall rules with tags, priorities, and source restrictions
- ✅ Deployed an HTTP Load Balancer with backend services and health checks
- ✅ Configured Cloud NAT for private instance outbound internet access
- ✅ Set up a private Cloud DNS zone with A and CNAME records
- ✅ Enabled VPC Flow Logs for network traffic monitoring

**Next Lab:** [LAB-08: IAM, Service Accounts, and Security](LAB-08-iam-security.md)
