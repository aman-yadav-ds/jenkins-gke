# Deploying Jenkins on Google Kubernetes Engine (GKE) - Cost-Optimized

This guide walks you through deploying a custom Jenkins Docker image to Google Kubernetes Engine (GKE) for a cost-effective CI/CD pipeline with persistent storage.

## Prerequisites

Before starting, ensure you have:
- Google Cloud SDK (`gcloud`) installed and configured
- Docker installed locally
- A Google Cloud project with billing enabled

## Initial Setup (Critical Steps)

### Step 0: Configure Your Environment

**⚠️ IMPORTANT: Do these steps in order to avoid common errors**

#### 0.1 Set Your Project Globally (Required)
```bash
# List your projects first
gcloud projects list

# Set your project (replace YOUR_PROJECT_ID with actual project ID)
gcloud config set project YOUR_PROJECT_ID

# Verify it's set correctly
gcloud config list
```

#### 0.2 Enable Required APIs
```bash
gcloud services enable container.googleapis.com containerregistry.googleapis.com
```

#### 0.3 Install kubectl and GKE Auth Plugin
```bash
# Install kubectl if not already installed
gcloud components install kubectl

# Install the GKE auth plugin (prevents authentication errors)
gcloud components install gke-gcloxud-auth-plugin
```

#### 0.4 Set Environment Variable for Authentication
**For Windows CMD:**
```cmd
setx USE_GKE_GCLOUD_AUTH_PLUGIN True
```
**⚠️ After running setx, close and reopen your command prompt**

**For Bash/Linux/Mac:**
```bash
echo 'export USE_GKE_GCLOUD_AUTH_PLUGIN=True' >> ~/.bashrc
source ~/.bashrc
```

#### 0.5 Grant Required Permissions
```pwsh
$PROJECT_ID = gcloud config get-value project

gcloud projects add-iam-policy-binding $PROJECT_ID `
  --member="user:$(gcloud config get-value account)" `
  --role="roles/container.admin"
```

## Step 1: Push Your Jenkins Image to Google Container Registry (GCR)

### 1.1 Tag Your Docker Image

Replace `your-project-id` with your actual Google Cloud project ID and `my-jenkins-image` with your image name:

```bash
docker tag my-jenkins-image:latest gcr.io/your-project-id/my-jenkins-image:latest
```

### 1.2 Configure Docker Authentication

Authenticate Docker to push to GCR:

```bash
gcloud auth configure-docker
```

### 1.3 Push the Image

```bash
docker push gcr.io/your-project-id/my-jenkins-image:latest
```

**Verify the push:** You can check if your image was successfully pushed by visiting the [Container Registry](https://console.cloud.google.com/gcr) in the Google Cloud Console.

## Step 2: Create a Cost-Optimized GKE Cluster

### 2.1 Create the Cluster (Cost-Optimized)

**💰 Configuration (Recommended for Production):**

```bash
# Minimal cost cluster with preemptible nodes
gcloud container clusters create jenkins-cluster \
  --zone asia-south1-a \
  --num-nodes 1 \
  --machine-type e2-medium \
  --disk-size 20GB \
  --disk-type pd-standard \
  --enable-autoscaling \
  --min-nodes 0 \
  --max-nodes 2 \
  --enable-autorepair
```


**🚀 Medium Performance (Less Cost):**
```bash
gcloud container clusters create jenkins-cluster \
  --zone asia-south1-a \
  --num-nodes 1 \
  --machine-type e2-small \
  --disk-size 30GB \
  --disk-type pd-standard \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 2 \
  --enable-autorepair
```

**⚠️ Common Errors and Solutions:**
- **Permission denied error**: Make sure you completed Step 0.5 (IAM permissions)
- **Zone doesn't exist**: Use `asia-south1-a`, `asia-south1-b`, or `asia-south1-c`
- **Project not set**: Complete Step 0.1 first

### 2.2 Get Cluster Credentials

Configure `kubectl` to connect to your cluster:

```bash
gcloud container clusters get-credentials jenkins-cluster --zone asia-south1-a
```

**⚠️ Troubleshooting Authentication Issues:**
If you get a critical warning about `gke-gcloud-auth-plugin`, you missed Step 0.3 and 0.4. Go back and complete those steps.

**Verify connection:**
```bash
kubectl cluster-info
```

**Check if kubectl is installed:**
```bash
kubectl version --client
```
If not found, run: `gcloud components install kubectl`

## Step 3: Set Up Cost-Optimized Persistent Storage

Jenkins requires persistent storage to maintain its configuration, jobs, and build history across pod restarts.

Create `jenkins-storage.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: default
  labels:
    app: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard-rwo  # Cheaper than SSD
  resources:
    requests:
      storage: 10Gi  # Smaller size to reduce costs
```

Apply the storage configuration:

```bash
kubectl apply -f jenkins-storage.yaml
```

**💰 Cost Note:** `standard-rwo` uses regular HDD instead of SSD, reducing storage costs by ~60%.

## Step 4: Deploy Jenkins (Cost-Optimized)

Create `jenkins-deployment.yaml` (replace `your-project-id` with your actual project ID):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: default
  labels:
    app: jenkins
spec:
  replicas: 1
  # Use RollingUpdate for zero-downtime updates
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      containers:
        - name: jenkins
          image: gcr.io/jenkins-cicd-dashboard/jenkins-dind:latest
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: jnlp
              containerPort: 50000
              protocol: TCP
          env:
            # Set the JAVA_OPTS to align with the container limits
            - name: JAVA_OPTS
              value: "-Xmx512m -Xms256m"
          resources:
            requests:
              # Ensure resource requests are realistic and match the JVM settings
              memory: "512Mi"
              cpu: "100m"
            limits:
              # Set a reasonable upper bound for memory to prevent OOMKills
              memory: "768Mi"
              cpu: "500m"
          volumeMounts:
            - name: jenkins-volume
              mountPath: /var/jenkins_home
          livenessProbe:
            httpGet:
              path: /login
              port: 8080
            # Increase initial delay to give Jenkins more time to start up
            initialDelaySeconds: 240
            periodSeconds: 30
            timeoutSeconds: 10
          readinessProbe:
            httpGet:
              path: /login
              port: 8080
            # Increase initial delay to give Jenkins more time to become ready
            initialDelaySeconds: 180
            periodSeconds: 10
            timeoutSeconds: 5
      volumes:
        - name: jenkins-volume
          persistentVolumeClaim:
            claimName: jenkins-pvc
```

Apply the deployment:

```bash
kubectl apply -f jenkins-deployment.yaml
```

**💰 Cost Optimizations Made:**
- Reduced memory requests/limits for smaller nodes
- Lower CPU requests
- Longer probe timeouts for preemptible nodes

## Step 5: Expose Jenkins (Cost-Optimized Options)

### LoadBalancer (Extra Cost but Easy Access)

Create `jenkins-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: default
  labels:
    app: jenkins
spec:
  type: LoadBalancer
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
  selector:
    app: jenkins
```

**💰 Cost:** ~$5-10/month for the LoadBalancer

```bash
kubectl apply -f jenkins-service.yaml
```

## Step 6: Access Your Jenkins Instance

### LoadBalancer:

Monitor the service until an external IP is assigned:

```bash
kubectl get service jenkins-service --watch
```

Once the `EXTERNAL-IP` field shows an IP address, access Jenkins at `http://<EXTERNAL-IP>`

### Get Initial Admin Password

For first-time setup:

```bash
kubectl exec deployment/jenkins -- cat /var/jenkins_home/secrets/initialAdminPassword
```

## Cost Management and Monitoring

### Check Current Costs
```bash
# Monitor resource usage
kubectl top nodes
kubectl top pods

# Check cluster info
gcloud container clusters describe jenkins-cluster --zone asia-south1-a
```

### Stop/Start Cluster to Save Money

**Stop cluster (saves ~90% of compute costs):**
```bash
gcloud container clusters resize jenkins-cluster --num-nodes=0 --zone asia-south1-a
```

**Start cluster:**
```bash
gcloud container clusters resize jenkins-cluster --num-nodes=1 --zone asia-south1-a
```

**💡 Pro Tip:** Stop the cluster when not actively using Jenkins to minimize costs!

## Monitoring and Maintenance

### Check Deployment Status
```bash
kubectl get pods -l app=jenkins
kubectl describe pod <jenkins-pod-name>
```

### View Logs
```bash
kubectl logs deployment/jenkins -f
```

### Scale Resources (if needed)
```bash
# Update deployment with new resource limits
kubectl patch deployment jenkins -p '{"spec":{"template":{"spec":{"containers":[{"name":"jenkins","resources":{"limits":{"memory":"6Gi","cpu":"3"}}}]}}}}'
```

## Security Considerations

1. **Network Security**: Consider using a private GKE cluster and VPC-native networking
2. **RBAC**: Implement proper Role-Based Access Control
3. **Secrets Management**: Use Kubernetes secrets for sensitive data
4. **Regular Updates**: Keep your Jenkins image and GKE cluster updated
5. **Backup Strategy**: Implement regular backups of your persistent volume

## Cleanup

To remove all resources:

```bash
kubectl delete service jenkins-service
kubectl delete deployment jenkins
kubectl delete pvc jenkins-pvc
gcloud container clusters delete jenkins-cluster --zone us-central1-a
```

## Troubleshooting Common Issues

### Setup Issues

1. **Project not set error**: 
   ```bash
   gcloud config set project YOUR_PROJECT_ID
   ```

2. **Permission denied on zone**: 
   - Check zone spelling: `asia-south1-a` (not `aisa-south1`)
   - Grant IAM permissions (Step 0.5)

3. **kubectl authentication warnings**:
   ```bash
   gcloud components install gke-gcloud-auth-plugin
   setx USE_GKE_GCLOUD_AUTH_PLUGIN True  # Windows
   # Then restart your terminal
   ```

4. **Incompatible operation error**: Wait for current operations to complete:
   ```bash
   gcloud container operations list --filter="targetLink:jenkins-cluster"
   # Wait for STATUS to show DONE
   ```

### Runtime Issues

1. **Pod stuck in Pending**: 
   - Check if cluster has nodes: `kubectl get nodes`
   - Check PVC status: `kubectl get pvc jenkins-pvc`

2. **Image pull errors**: 
   - Verify image exists: `gcloud container images list --repository=gcr.io/your-project-id`
   - Check image path in deployment YAML

3. **Out of memory errors on e2-micro**: 
   - Scale up to e2-small: 
   ```bash
   gcloud container clusters resize jenkins-cluster --num-nodes=0 --zone asia-south1-a
   gcloud container clusters delete jenkins-cluster --zone asia-south1-a
   # Then recreate with e2-small
   ```

### Useful Commands

```bash
# Check everything is working
kubectl get all
kubectl get pvc

# Debug specific resources
kubectl describe pod <pod-name>
kubectl logs deployment/jenkins -f
kubectl describe pvc jenkins-pvc

# Check cluster operations
gcloud container operations list --filter="targetLink:jenkins-cluster"
```

---

**Note**: This guide uses a LoadBalancer service which will incur additional costs. For development environments, consider using NodePort or port-forwarding instead.