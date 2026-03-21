# PROJECT-05: Multi-Region Deployment on GCP

> **Deploy a highly available, globally distributed application across multiple GCP regions with automated failover and traffic management**

## 📋 Project Overview

Build a production-grade multi-region architecture that demonstrates:
- Global load balancing with Cloud Load Balancing
- Multi-region GKE clusters with cluster federation
- Cloud SQL with cross-region replication
- Cloud Storage multi-region buckets
- Cloud CDN for content delivery
- Traffic Director for service mesh
- Disaster recovery and automated failover

**Estimated Time:** 6-8 hours  
**Difficulty:** Advanced  
**Cost:** ~$50-100/day (remember to clean up!)

## 🎯 Learning Objectives

By completing this project, you will:

1. Design and implement multi-region architecture on GCP
2. Configure global load balancing and traffic management
3. Set up GKE clusters across multiple regions
4. Implement database replication and failover
5. Configure CDN and edge caching
6. Deploy service mesh with Traffic Director
7. Implement monitoring and alerting for global infrastructure
8. Test disaster recovery procedures

## 🏗️ Architecture

### Components

```
Global Users
    ↓
Cloud CDN (Edge Caching)
    ↓
Global HTTP(S) Load Balancer
    ↓
    ├── Region: us-central1
    │   ├── GKE Cluster
    │   ├── Cloud SQL (Primary)
    │   └── Cloud Storage
    │
    ├── Region: europe-west1
    │   ├── GKE Cluster  
    │   ├── Cloud SQL (Read Replica)
    │   └── Cloud Storage
    │
    └── Region: asia-east1
        ├── GKE Cluster
        ├── Cloud SQL (Read Replica)
        └── Cloud Storage
```

## 📝 Prerequisites

- Completed previous GCP projects
- Understanding of GKE and networking
- Familiarity with global load balancing concepts
- gcloud CLI configured with appropriate permissions

## 🚀 Implementation Steps

### Phase 1: Network Foundation

#### Step 1: Create VPC Network with Global Subnets

```bash
# Create custom VPC
gcloud compute networks create multi-region-vpc \
    --subnet-mode=custom \
    --bgp-routing-mode=global

# Create subnet in us-central1
gcloud compute networks subnets create us-central1-subnet \
    --network=multi-region-vpc \
    --region=us-central1 \
    --range=10.1.0.0/20

# Create subnet in europe-west1
gcloud compute networks subnets create europe-west1-subnet \
    --network=multi-region-vpc \
    --region=europe-west1 \
    --range=10.2.0.0/20

# Create subnet in asia-east1
gcloud compute networks subnets create asia-east1-subnet \
    --network=multi-region-vpc \
    --region=asia-east1 \
    --range=10.3.0.0/20

# Create firewall rules
gcloud compute firewall-rules create allow-internal \
    --network=multi-region-vpc \
    --allow=tcp,udp,icmp \
    --source-ranges=10.0.0.0/8

gcloud compute firewall-rules create allow-http-https \
    --network=multi-region-vpc \
    --allow=tcp:80,tcp:443 \
    --source-ranges=0.0.0.0/0
```

### Phase 2: GKE Clusters in Multiple Regions

#### Step 2: Create GKE Clusters

```bash
# US Central cluster
gcloud container clusters create us-cluster \
    --region=us-central1 \
    --network=multi-region-vpc \
    --subnetwork=us-central1-subnet \
    --num-nodes=2 \
    --machine-type=e2-standard-2 \
    --enable-ip-alias \
    --enable-autoscaling \
    --min-nodes=2 \
    --max-nodes=5 \
    --enable-stackdriver-kubernetes

# Europe West cluster
gcloud container clusters create eu-cluster \
    --region=europe-west1 \
    --network=multi-region-vpc \
    --subnetwork=europe-west1-subnet \
    --num-nodes=2 \
    --machine-type=e2-standard-2 \
    --enable-ip-alias \
    --enable-autoscaling \
    --min-nodes=2 \
    --max-nodes=5 \
    --enable-stackdriver-kubernetes

# Asia East cluster  
gcloud container clusters create asia-cluster \
    --region=asia-east1 \
    --network=multi-region-vpc \
    --subnetwork=asia-east1-subnet \
    --num-nodes=2 \
    --machine-type=e2-standard-2 \
    --enable-ip-alias \
    --enable-autoscaling \
    --min-nodes=2 \
    --max-nodes=5 \
    --enable-stackdriver-kubernetes

# Get cluster credentials
gcloud container clusters get-credentials us-cluster --region=us-central1
gcloud container clusters get-credentials eu-cluster --region=europe-west1
gcloud container clusters get-credentials asia-cluster --region=asia-east1
```

### Phase 3: Multi-Region Cloud SQL

#### Step 3: Set Up Primary Database and Replicas

```bash
# Create primary instance in us-central1
gcloud sql instances create primary-db \
    --database-version=MYSQL_8_0 \
    --tier=db-n1-standard-2 \
    --region=us-central1 \
    --network=projects/PROJECT_ID/global/networks/multi-region-vpc \
    --no-assign-ip \
    --backup \
    --backup-start-time=03:00

# Create read replica in europe-west1
gcloud sql instances create eu-replica \
    --master-instance-name=primary-db \
    --tier=db-n1-standard-2 \
    --region=europe-west1

# Create read replica in asia-east1
gcloud sql instances create asia-replica \
    --master-instance-name=primary-db \
    --tier=db-n1-standard-2 \
    --region=asia-east1

# Set root password
gcloud sql users set-password root \
    --instance=primary-db \
    --password="YOUR_SECURE_PASSWORD"
```

### Phase 4: Deploy Application to All Clusters

#### Step 4: Create Kubernetes Deployment

Create `app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: gcr.io/PROJECT_ID/web-app:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: host
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
```

Deploy to all clusters:

```bash
# Switch contexts and deploy
for CLUSTER in us-cluster eu-cluster asia-cluster; do
  kubectl config use-context gke_PROJECT_ID_REGION_${CLUSTER}
  kubectl apply -f app-deployment.yaml
  kubectl apply -f db-config.yaml
done
```

### Phase 5: Global Load Balancer

#### Step 5: Configure Multi-Region Load Balancing

```bash
# Reserve global static IP
gcloud compute addresses create global-lb-ip \
    --global

# Create health check
gcloud compute health-checks create http web-health-check \
    --port=80 \
    --request-path=/health

# Create backend service
gcloud compute backend-services create web-backend \
    --protocol=HTTP \
    --health-checks=web-health-check \
    --global

# Add NEGs from each region
for REGION in us-central1 europe-west1 asia-east1; do
  gcloud compute backend-services add-backend web-backend \
      --network-endpoint-group=CLUSTER_NEG_NAME \
      --network-endpoint-group-region=${REGION} \
      --balancing-mode=RATE \
      --max-rate-per-endpoint=100 \
      --global
done

# Create URL map
gcloud compute url-maps create web-url-map \
    --default-service=web-backend

# Create target HTTP proxy
gcloud compute target-http-proxies create web-http-proxy \
    --url-map=web-url-map

# Create forwarding rule
gcloud compute forwarding-rules create web-forwarding-rule \
    --address=global-lb-ip \
    --global \
    --target-http-proxy=web-http-proxy \
    --ports=80
```

### Phase 6: Cloud CDN

#### Step 6: Enable CDN

```bash
# Enable Cloud CDN on backend service
gcloud compute backend-services update web-backend \
    --enable-cdn \
    --global \
    --cache-mode=CACHE_ALL_STATIC

# Configure CDN policy
gcloud compute backend-services update web-backend \
    --global \
    --cdn-policy-cache-key-policy-include-host=true \
    --cdn-policy-cache-key-policy-include-protocol=true \
    --cdn-policy-cache-key-policy-include-query-string=true
```

### Phase 7: Monitoring and Alerting

#### Step 7: Set Up Global Monitoring

Create monitoring dashboard configuration:

```bash
# Create uptime check
gcloud monitoring uptime-check-configs create web-app-uptime \
    --display-name="Multi-Region Web App" \
    --resource-type=uptime-url \
    --monitored-resource="host=$(gcloud compute addresses describe global-lb-ip --global --format='get(address)')" \
    --http-check-path=/health

# Create alert policy for high latency
gcloud alpha monitoring policies create \
    --display-name="High Latency Alert" \
    --condition-display-name="Latency > 1s" \
    --condition-threshold-value=1.0 \
    --condition-threshold-duration=60s \
    --notification-channels=CHANNEL_ID
```

Create custom dashboard:

```yaml
# monitoring-dashboard.yaml
displayName: Multi-Region Application Dashboard
gridLayout:
  widgets:
  - title: Request Rate by Region
    xyChart:
      dataSets:
      - timeSeriesQuery:
          timeSeriesFilter:
            filter: metric.type="loadbalancing.googleapis.com/https/request_count"
            aggregation:
              alignmentPeriod: 60s
              perSeriesAligner: ALIGN_RATE
              crossSeriesReducer: REDUCE_SUM
              groupByFields:
              - resource.region
```

### Phase 8: Disaster Recovery Testing

#### Step 8: Test Failover Scenarios

```bash
# Simulate region failure
# Scale down US cluster
kubectl config use-context gke_PROJECT_ID_us-central1_us-cluster
kubectl scale deployment web-app --replicas=0

# Monitor traffic shift
gcloud monitoring time-series list \
    --filter='metric.type="loadbalancing.googleapis.com/https/request_count"' \
    --format=json

# Verify traffic routed to healthy regions
curl -I http://GLOBAL_LB_IP

# Restore US cluster
kubectl scale deployment web-app --replicas=3
```

Test database failover:

```bash
# Promote replica to standalone instance (disaster recovery)
gcloud sql instances promote-replica eu-replica

# Update application configuration to use new primary
kubectl config use-context gke_PROJECT_ID_europe-west1_eu-cluster
kubectl set env deployment/web-app DB_HOST=NEW_PRIMARY_IP
```

## ✅ Verification

### Test Global Distribution

```bash
# Get load balancer IP
LB_IP=$(gcloud compute addresses describe global-lb-ip --global --format='get(address)')

# Test from different locations (use VPN or proxy)
curl -I http://${LB_IP}

# Check which region served the request (add custom header in app)
curl -H "X-Debug: true" http://${LB_IP}
```

### Verify CDN Caching

```bash
# First request (cache MISS)
curl -I http://${LB_IP}/static/image.jpg | grep -i "x-cache"

# Second request (cache HIT)
curl -I http://${LB_IP}/static/image.jpg | grep -i "x-cache"
```

### Load Testing

```bash
# Install Apache Bench
sudo apt-get install apache2-utils

# Run load test
ab -n 10000 -c 100 http://${LB_IP}/

# Monitor in Cloud Monitoring
gcloud monitoring dashboards list
```

## 📊 Monitoring Dashboard

Access Cloud Console:
1. Navigation → Operations → Monitoring → Dashboards
2. View custom dashboard for multi-region metrics
3. Check uptime checks status
4. Review error budget and SLOs

Key metrics to monitor:
- Request latency by region
- Error rate per backend
- Cache hit ratio (CDN)
- Database replication lag
- Cross-region traffic volume

## 🧹 Cleanup

**Important:** Delete resources to avoid ongoing charges!

```bash
# Delete GKE clusters
gcloud container clusters delete us-cluster --region=us-central1 --quiet
gcloud container clusters delete eu-cluster --region=europe-west1 --quiet
gcloud container clusters delete asia-cluster --region=asia-east1 --quiet

# Delete Cloud SQL instances
gcloud sql instances delete primary-db --quiet
gcloud sql instances delete eu-replica --quiet
gcloud sql instances delete asia-replica --quiet

# Delete load balancer components
gcloud compute forwarding-rules delete web-forwarding-rule --global --quiet
gcloud compute target-http-proxies delete web-http-proxy --quiet
gcloud compute url-maps delete web-url-map --quiet
gcloud compute backend-services delete web-backend --global --quiet
gcloud compute health-checks delete web-health-check --quiet
gcloud compute addresses delete global-lb-ip --global --quiet

# Delete VPC network
gcloud compute firewall-rules delete allow-internal --quiet
gcloud compute firewall-rules delete allow-http-https --quiet
gcloud compute networks subnets delete us-central1-subnet --region=us-central1 --quiet
gcloud compute networks subnets delete europe-west1-subnet --region=europe-west1 --quiet
gcloud compute networks subnets delete asia-east1-subnet --region=asia-east1 --quiet
gcloud compute networks delete multi-region-vpc --quiet
```

## 🎓 Key Learnings

1. **Global Architecture Design**
   - Designing for low latency worldwide
   - Balancing cost vs performance
   - Regional failover strategies

2. **Traffic Management**
   - Global load balancing with anycast
   - CDN edge caching optimization
   - Geo-based routing policies

3. **Data Replication**
   - Database replication strategies
   - Consistency vs availability tradeoffs
   - Cross-region data synchronization

4. **Disaster Recovery**
   - RTO and RPO planning
   - Automated failover mechanisms
   - Testing DR procedures

5. **Operational Excellence**
   - Multi-region monitoring
   - Global incident response
   - Cost optimization strategies

## 🚀 Next Steps

1. **Implement Traffic Policies**
   - Geo-based routing
   - A/B testing across regions
   - Canary deployments

2. **Enhance Security**
   - Cloud Armor for DDoS protection
   - SSL certificates with Cloud CDN
   - Private Google Access

3. **Optimize Costs**
   - Committed use discounts
   - Spot instances for non-critical workloads
   - CDN cache policies

4. **Add Advanced Features**
   - Multi-cluster Ingress
   - Anthos Service Mesh
   - Config Connector for GitOps

## 📚 Additional Resources

- [GCP Multi-Region Architecture Best Practices](https://cloud.google.com/architecture/framework/reliability/multi-region)
- [Global Load Balancing Deep Dive](https://cloud.google.com/load-balancing/docs/https)
- [Cloud CDN Best Practices](https://cloud.google.com/cdn/docs/best-practices)
- [Designing for Global Applications](https://cloud.google.com/architecture/hybrid-and-multi-cloud-patterns-and-practices)

---

**Congratulations!** 🎉 You've successfully deployed a globally distributed, highly available application on GCP with multi-region architecture!

**Pro Tip:** Use this architecture as a template for production deployments requiring global reach, high availability, and disaster recovery capabilities.