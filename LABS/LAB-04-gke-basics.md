# LAB-04: Google Kubernetes Engine (GKE)

**Topic:** Containers and Kubernetes  
**Estimated Time:** 120 minutes  
**Difficulty:** 🟡 Intermediate  
**Cost:** ~$2–$5

---

## Learning Objectives

By the end of this lab, you will be able to:

1. Create and configure a GKE cluster with appropriate node pool settings
2. Deploy containerized applications using Kubernetes Deployment manifests
3. Expose applications externally using LoadBalancer and ClusterIP services
4. Scale deployments manually and automatically with the Horizontal Pod Autoscaler (HPA)
5. Perform rolling updates and rollbacks safely with zero downtime
6. Manage application configuration using ConfigMaps and Secrets

---

## Prerequisites

- [LAB-01](LAB-01-gcp-basics.md) completed
- `container.googleapis.com` API enabled
- kubectl installed: `gcloud components install kubectl`
- Basic familiarity with Kubernetes concepts (Pods, Deployments, Services) is helpful

---

## Lab Overview

Google Kubernetes Engine (GKE) is GCP's managed Kubernetes service. It automates cluster provisioning, scaling, upgrades, and health management. In this lab you will create a GKE cluster, deploy a multi-replica application, expose it with a load balancer, configure autoscaling, perform rolling updates, and use ConfigMaps and Secrets.

---

## Environment Setup

```bash
export PROJECT_ID=$(gcloud config get-value project)
export CLUSTER_NAME=devops-lab-cluster
export ZONE=us-central1-a
export REGION=us-central1

echo "Project: $PROJECT_ID"
echo "Cluster: $CLUSTER_NAME"
echo "Zone:    $ZONE"
```

---

## Part 1: Create a GKE Cluster

### 1.1 Create the Cluster

```bash
# Create a standard GKE cluster (takes 5-8 minutes)
gcloud container clusters create $CLUSTER_NAME \
  --zone=$ZONE \
  --num-nodes=2 \
  --machine-type=e2-small \
  --disk-size=50GB \
  --disk-type=pd-standard \
  --enable-autorepair \
  --enable-autoupgrade \
  --logging=SYSTEM,WORKLOAD \
  --monitoring=SYSTEM

echo "✅ Cluster creation started — this takes 5-8 minutes"
```

> ⏳ While waiting, review the Kubernetes concepts in the next section.

### 1.2 Get Cluster Credentials

```bash
# Configure kubectl to connect to the cluster
gcloud container clusters get-credentials $CLUSTER_NAME --zone=$ZONE

# Verify kubectl is connected
kubectl cluster-info

# List cluster nodes
kubectl get nodes

# Detailed node info
kubectl get nodes -o wide
```

**Expected output for `kubectl get nodes`:**
```
NAME                                          STATUS   ROLES    AGE   VERSION
gke-devops-lab-cluster-default-pool-a1b2c3d4  Ready    <none>   2m    v1.28.3-gke.1286000
gke-devops-lab-cluster-default-pool-e5f6g7h8  Ready    <none>   2m    v1.28.3-gke.1286000
```

### 1.3 Explore the Cluster

```bash
# List all system pods
kubectl get pods --namespace=kube-system

# View cluster version
kubectl version --short

# Describe a node
NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl describe node $NODE_NAME | head -40

# View cluster events
kubectl get events --sort-by=.metadata.creationTimestamp | tail -10
```

---

## Part 2: Deploy an Application

### 2.1 Create a Namespace

```bash
# Create a dedicated namespace for lab resources
kubectl create namespace devops-lab

# Set as default namespace for this context
kubectl config set-context --current --namespace=devops-lab

# Verify namespace exists
kubectl get namespace devops-lab
```

### 2.2 Deploy Nginx Application

```bash
cat > deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: devops-lab
  labels:
    app: nginx
    version: "1.21"
    managed-by: devops-lab
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: nginx
        version: "1.21"
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "250m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
EOF

kubectl apply -f deployment.yaml

# Watch pods start up
kubectl rollout status deployment/nginx-deployment -n devops-lab

# List pods
kubectl get pods -n devops-lab

# Detailed pod info
kubectl describe deployment nginx-deployment -n devops-lab
```

**Expected output for `kubectl get pods`:**
```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7c79c4bf97-a1b2c   1/1     Running   0          45s
nginx-deployment-7c79c4bf97-d3e4f   1/1     Running   0          45s
nginx-deployment-7c79c4bf97-g5h6i   1/1     Running   0          45s
```

### 2.3 Inspect Pods

```bash
# Get pod logs
POD_NAME=$(kubectl get pods -n devops-lab -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD_NAME -n devops-lab

# Execute command inside a pod
kubectl exec -it $POD_NAME -n devops-lab -- bash -c "nginx -v && cat /etc/nginx/nginx.conf | head -20"

# Port-forward to test locally
kubectl port-forward $POD_NAME 8080:80 -n devops-lab &
curl http://localhost:8080
kill %1  # stop port-forward
```

---

## Part 3: Expose with a Service

### 3.1 Create a LoadBalancer Service

```bash
cat > service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: devops-lab
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f service.yaml

# Watch for external IP assignment (takes 1-2 minutes)
echo "Waiting for external IP..."
kubectl get service nginx-service -n devops-lab --watch &
WATCH_PID=$!
sleep 60
kill $WATCH_PID 2>/dev/null

# Get the external IP
kubectl get service nginx-service -n devops-lab
```

**Expected output after IP is assigned:**
```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
nginx-service   LoadBalancer   10.96.100.50    34.123.45.67    80:31234/TCP   90s
```

### 3.2 Test the Service

```bash
# Wait for external IP and test
EXTERNAL_IP=""
while [ -z "$EXTERNAL_IP" ]; do
  EXTERNAL_IP=$(kubectl get service nginx-service -n devops-lab \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null)
  [ -z "$EXTERNAL_IP" ] && echo "Waiting for IP..." && sleep 10
done

echo "Service External IP: $EXTERNAL_IP"
curl -s http://$EXTERNAL_IP | head -5
```

### 3.3 ClusterIP Service (Internal Only)

```bash
cat > clusterip-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-internal
  namespace: devops-lab
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
EOF

kubectl apply -f clusterip-service.yaml
kubectl get services -n devops-lab
```

---

## Part 4: Scaling

### 4.1 Manual Scaling

```bash
# Scale up to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5 -n devops-lab

# Watch pods start
kubectl get pods -n devops-lab --watch &
WATCH_PID=$!
sleep 30
kill $WATCH_PID 2>/dev/null

# Scale down to 2 replicas
kubectl scale deployment nginx-deployment --replicas=2 -n devops-lab
kubectl get pods -n devops-lab
```

### 4.2 Horizontal Pod Autoscaler (HPA)

```bash
# Enable metrics server if not already enabled
# (GKE usually has it enabled by default)
kubectl top nodes
kubectl top pods -n devops-lab

# Create HPA
kubectl autoscale deployment nginx-deployment \
  --min=2 \
  --max=10 \
  --cpu-percent=70 \
  -n devops-lab

# View HPA
kubectl get hpa -n devops-lab

# Describe HPA for details
kubectl describe hpa nginx-deployment -n devops-lab
```

**Expected output for `kubectl get hpa`:**
```
NAME               REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-deployment   Deployment/nginx-deployment   2%/70%    2         10        3          30s
```

### 4.3 Cluster Autoscaler

```bash
# Enable cluster autoscaler on the default node pool
gcloud container clusters update $CLUSTER_NAME \
  --zone=$ZONE \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5 \
  --node-pool=default-pool

# View autoscaling config
gcloud container node-pools describe default-pool \
  --cluster=$CLUSTER_NAME \
  --zone=$ZONE \
  --format='yaml(autoscaling)'
```

---

## Part 5: Rolling Updates and Rollbacks

### 5.1 Perform a Rolling Update

```bash
# Update nginx from 1.21 to 1.22
kubectl set image deployment/nginx-deployment \
  nginx=nginx:1.22 \
  -n devops-lab \
  --record

# Watch the rollout
kubectl rollout status deployment/nginx-deployment -n devops-lab
```

**Expected output:**
```
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out
```

### 5.2 View Rollout History

```bash
kubectl rollout history deployment/nginx-deployment -n devops-lab

# View details of a specific revision
kubectl rollout history deployment/nginx-deployment \
  --revision=2 \
  -n devops-lab
```

### 5.3 Rollback

```bash
# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment -n devops-lab

# Verify rollback
kubectl rollout status deployment/nginx-deployment -n devops-lab

# Rollback to a specific revision
# kubectl rollout undo deployment/nginx-deployment --to-revision=1 -n devops-lab

# Check current image
kubectl get deployment nginx-deployment -n devops-lab \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

## Part 6: Node Pools

### 6.1 Add a Node Pool

```bash
gcloud container node-pools create high-memory-pool \
  --cluster=$CLUSTER_NAME \
  --zone=$ZONE \
  --machine-type=e2-medium \
  --num-nodes=1 \
  --disk-size=50GB \
  --enable-autorepair \
  --enable-autoupgrade

# List node pools
gcloud container node-pools list \
  --cluster=$CLUSTER_NAME \
  --zone=$ZONE

# View nodes with pool labels
kubectl get nodes --show-labels | grep cloud.google.com/gke-nodepool
```

### 6.2 Node Affinity (Schedule to Specific Pool)

```bash
cat > pool-specific-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: high-memory-app
  namespace: devops-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: high-memory-app
  template:
    metadata:
      labels:
        app: high-memory-app
    spec:
      nodeSelector:
        cloud.google.com/gke-nodepool: high-memory-pool
      containers:
      - name: app
        image: nginx:1.21
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
EOF

kubectl apply -f pool-specific-deployment.yaml
kubectl get pods -n devops-lab -o wide | grep high-memory
```

---

## Part 7: ConfigMaps and Secrets

### 7.1 Create a ConfigMap

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100 \
  -n devops-lab

# From a file
cat > app.properties << 'EOF'
database.host=db.internal
database.port=5432
cache.ttl=3600
feature.flag.new_ui=true
EOF

kubectl create configmap app-properties \
  --from-file=app.properties \
  -n devops-lab

# View ConfigMaps
kubectl get configmap -n devops-lab
kubectl get configmap app-config -n devops-lab -o yaml
```

### 7.2 Create Secrets

```bash
# Create an opaque secret
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD='supersecret123' \
  --from-literal=API_KEY='api-key-abcdef-123456' \
  -n devops-lab

# View secret (values are base64 encoded)
kubectl get secret app-secret -n devops-lab
kubectl get secret app-secret -n devops-lab -o yaml

# Decode a secret value
kubectl get secret app-secret -n devops-lab \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
echo ""
```

### 7.3 Use ConfigMap and Secret in a Deployment

```bash
cat > configured-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: configured-app
  namespace: devops-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: configured-app
  template:
    metadata:
      labels:
        app: configured-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: API_KEY
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
      - name: config-volume
        configMap:
          name: app-properties
EOF

kubectl apply -f configured-deployment.yaml

# Verify env vars are set inside the pod
POD=$(kubectl get pods -n devops-lab -l app=configured-app \
  -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -n devops-lab -- env | grep -E "APP_ENV|LOG_LEVEL|DB_PASSWORD"
kubectl exec $POD -n devops-lab -- ls /etc/config/
```

---

## Verification Steps

```bash
# 1. List all resources in the namespace
kubectl get all -n devops-lab

# 2. Verify HPA
kubectl get hpa -n devops-lab

# 3. Verify nodes
kubectl get nodes

# 4. Check pod health
kubectl get pods -n devops-lab

# 5. Test the LoadBalancer service
EXTERNAL_IP=$(kubectl get service nginx-service -n devops-lab \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Testing http://$EXTERNAL_IP ..."
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://$EXTERNAL_IP

echo "✅ Verification complete"
```

---

## Cost Estimates

| Resource | Hourly Cost | Notes |
|----------|-------------|-------|
| GKE cluster management fee | $0.10/hr | Waived for first cluster |
| 2x e2-small nodes | ~$0.034/hr | ~$25/month |
| 1x e2-medium node (extra pool) | ~$0.067/hr | ~$49/month |
| LoadBalancer (forwarding rule) | $0.025/hr | ~$18/month |
| Total (all resources) | ~$0.22/hr | ~$4-5 for this lab |

> 💡 **Tip**: GKE Autopilot mode can be cheaper for low-utilization workloads — it charges per Pod resource request rather than per node.

---

## Cleanup

> ⚠️ GKE clusters are expensive. Delete immediately after the lab.

```bash
export PROJECT_ID=$(gcloud config get-value project)
export CLUSTER_NAME=devops-lab-cluster
export ZONE=us-central1-a

# Delete all resources in the namespace
kubectl delete namespace devops-lab

# Delete extra node pool first
gcloud container node-pools delete high-memory-pool \
  --cluster=$CLUSTER_NAME \
  --zone=$ZONE \
  --quiet

# Delete the cluster (this takes 3-5 minutes)
gcloud container clusters delete $CLUSTER_NAME \
  --zone=$ZONE \
  --quiet

# Clean up local files
rm -f deployment.yaml service.yaml clusterip-service.yaml
rm -f pool-specific-deployment.yaml configured-deployment.yaml
rm -f app.properties

echo "✅ Cleanup complete"

# Verify cluster is deleted
gcloud container clusters list
```

---

## Summary

You have successfully:
- ✅ Created a GKE cluster and configured kubectl credentials
- ✅ Deployed a multi-replica application with health probes
- ✅ Exposed the application via a LoadBalancer service
- ✅ Scaled deployments manually and configured HPA autoscaling
- ✅ Performed a rolling update and rollback
- ✅ Created and used ConfigMaps and Secrets in a deployment

**Next Lab:** [LAB-05: Cloud Run Serverless Containers](LAB-05-cloud-run.md)
