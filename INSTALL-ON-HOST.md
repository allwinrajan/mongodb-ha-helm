# MongoDB HA - SIMPLIFIED Installation Guide

## 🎯 Installation on ANY Kubernetes Cluster

This guide shows how to install your MongoDB HA chart on **any** Kubernetes cluster.

---

## ✅ Prerequisites Check

Run these commands on the **target Kubernetes cluster**:

```bash
# 1. Check kubectl works
kubectl version --client
kubectl cluster-info

# 2. Check Helm installed
helm version

# 3. Check you have access
kubectl get nodes

# 4. Check storage classes available
kubectl get storageclass
```

**Required:**
- ✅ kubectl access to cluster
- ✅ Helm 3.x installed
- ✅ At least 10Gi storage available
- ✅ Internet access (to download chart from GitHub)

---

## 📦 METHOD 1: Install from GitHub Pages (RECOMMENDED)

This is the **easiest and most common** method.

### Step 1: Add Helm Repository

```bash
# Add your published repository
helm repo add mongodb-ha https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm

# Example (replace with your actual username):
# helm repo add mongodb-ha https://john-doe.github.io/mongodb-ha-helm

# Update repository cache
helm repo update
```

### Step 2: Verify Chart is Available

```bash
# Search for chart
helm search repo mongodb-ha

# Expected output:
# NAME                    CHART VERSION   APP VERSION     DESCRIPTION
# mongodb-ha/mongodb-ha   1.0.0           7.0             MongoDB High Availability...

# View chart values
helm show values mongodb-ha/mongodb-ha | less
```

**Press 'q' to exit.**

### Step 3: Choose Your Environment

Pick **ONE** of these based on your needs:

---

#### 🟢 OPTION A: Development (Minimal - 1 Replica)

**Use for:** Local testing, development

```bash
# Create namespace
kubectl create namespace mongodb-dev

# Install
helm install mongodb-dev mongodb-ha/mongodb-ha \
  --namespace mongodb-dev \
  --set mongodb.replicaSet.replicas=1 \
  --set mongodb.storage.size=5Gi \
  --set mongodb.storage.storageClassName=local-path \
  --set mongodb.auth.rootPassword="DevPassword123"

# Watch pods
kubectl get pods -n mongodb-dev -w
```

---

#### 🟡 OPTION B: Testing (Standard - 3 Replicas)

**Use for:** Testing, staging environments

```bash
# Create namespace
kubectl create namespace mongodb-test

# Install
helm install mongodb-test mongodb-ha/mongodb-ha \
  --namespace mongodb-test \
  --set mongodb.storage.size=10Gi \
  --set mongodb.storage.storageClassName=local-path \
  --set mongodb.auth.rootPassword="Test@MongoDB123"

# Watch pods
kubectl get pods -n mongodb-test -w
```

---

#### 🔴 OPTION C: Production (Full HA - 3 Replicas + NodePort)

**Use for:** Production deployments

```bash
# Create namespace
kubectl create namespace mongodb-ha

# Install
helm install mongodb-prod mongodb-ha/mongodb-ha \
  --namespace mongodb-ha \
  --set mongodb.storage.size=40Gi \
  --set mongodb.storage.storageClassName=local-storage \
  --set mongodb.service.nodePort=30017 \
  --set mongodb.auth.rootPassword="Pr0d@Secure!Pass2024" \
  --timeout 10m

# Watch pods
kubectl get pods -n mongodb-ha -w
```

---

### Step 4: Wait for Pods to Start

**What you should see:**

```
NAME                         READY   STATUS    RESTARTS   AGE
mongodb-test-mongodb-ha-0    1/1     Running   0          2m
mongodb-test-mongodb-ha-1    1/1     Running   0          3m
mongodb-test-mongodb-ha-2    1/1     Running   0          4m
```

**Press Ctrl+C when all pods show 1/1 Running.**

### Step 5: Check Initialization

```bash
# Check init job completed
kubectl logs -n mongodb-test job/mongodb-test-mongodb-ha-init-replicaset

# Look for this at the end:
# =========================================
# [init-job] ✓ INITIALIZATION COMPLETE!
# =========================================
```

**If you see this, installation is successful! 🎉**

---

## 💾 METHOD 2: Install from Package File

Use this if you have the `.tgz` file locally.

### Step 1: Get Package File

**Option A: Download from GitHub**

```bash
# Download package directly
wget https://YOUR-USERNAME.github.io/mongodb-ha-helm/mongodb-ha-1.0.0.tgz

# Or use curl
curl -LO https://YOUR-USERNAME.github.io/mongodb-ha-helm/mongodb-ha-1.0.0.tgz
```

**Option B: Copy from Another Machine**

```bash
# On source machine
scp mongodb-ha-1.0.0.tgz user@target-host:/tmp/

# On target machine
ls -lh /tmp/mongodb-ha-1.0.0.tgz
```

### Step 2: Install from File

```bash
# Create namespace
kubectl create namespace mongodb-test

# Install from local file
helm install mongodb-test /tmp/mongodb-ha-1.0.0.tgz \
  --namespace mongodb-test \
  --set mongodb.auth.rootPassword="TestPass123"

# Watch pods
kubectl get pods -n mongodb-test -w
```

---

## 🔧 STORAGE CONFIGURATION

MongoDB needs persistent storage. Choose based on your cluster:

### Option 1: Dynamic Provisioning (Cloud/Most Clusters)

**For AWS EKS:**
```bash
helm install mongodb-prod mongodb-ha/mongodb-ha \
  --namespace mongodb-ha \
  --create-namespace \
  --set mongodb.storage.storageClassName=gp3 \
  --set mongodb.storage.size=100Gi
```

**For Google GKE:**
```bash
helm install mongodb-prod mongodb-ha/mongodb-ha \
  --namespace mongodb-ha \
  --create-namespace \
  --set mongodb.storage.storageClassName=standard-rwo \
  --set mongodb.storage.size=100Gi
```

**For Azure AKS:**
```bash
helm install mongodb-prod mongodb-ha/mongodb-ha \
  --namespace mongodb-ha \
  --create-namespace \
  --set mongodb.storage.storageClassName=managed-premium \
  --set mongodb.storage.size=100Gi
```

### Option 2: Local Path (Development)

Most K3s/Minikube clusters have `local-path` provisioner:

```bash
# Check if local-path exists
kubectl get storageclass

# Use it
helm install mongodb-dev mongodb-ha/mongodb-ha \
  --namespace mongodb-dev \
  --create-namespace \
  --set mongodb.storage.storageClassName=local-path \
  --set mongodb.storage.size=10Gi
```

### Option 3: Manual PersistentVolumes

If you need to create PVs manually:

```bash
# On each node, create directories
sudo mkdir -p /mnt/mongodb/data-{0,1,2}
sudo chmod 777 /mnt/mongodb/data-*

# Create PVs
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv-0
spec:
  capacity:
    storage: 40Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/mongodb/data-0
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - YOUR-NODE-NAME
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv-1
spec:
  capacity:
    storage: 40Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/mongodb/data-1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - YOUR-NODE-NAME
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv-2
spec:
  capacity:
    storage: 40Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/mongodb/data-2
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - YOUR-NODE-NAME
EOF

# Replace YOUR-NODE-NAME with actual node name:
kubectl get nodes
```

Then install:
```bash
helm install mongodb-prod mongodb-ha/mongodb-ha \
  --namespace mongodb-ha \
  --create-namespace \
  --set mongodb.storage.storageClassName=local-storage \
  --set mongodb.storage.size=40Gi
```

---

## 🔌 CONNECTING TO MONGODB

### From Inside the Cluster

```bash
# Connection string format:
mongodb://admin:PASSWORD@RELEASE-NAME-mongodb-ha-service.NAMESPACE.svc.cluster.local:27017/admin?replicaSet=rs0

# Example for test environment:
mongodb://admin:Test@MongoDB123@mongodb-test-mongodb-ha-service.mongodb-test.svc.cluster.local:27017/admin?replicaSet=rs0
```

**Test connection:**

```bash
kubectl run -it --rm mongo-client \
  --image=mongo:7.0 \
  --restart=Never \
  -n mongodb-test -- \
  mongosh "mongodb://admin:Test@MongoDB123@mongodb-test-mongodb-ha-service:27017/admin?replicaSet=rs0"

# Inside mongosh prompt:
db.adminCommand("ping")
rs.status()
show dbs
exit
```

### From Outside the Cluster (NodePort)

If you installed with `--set mongodb.service.nodePort=30017`:

```bash
# Get node IP
kubectl get nodes -o wide

# Connection string:
mongodb://admin:PASSWORD@NODE-IP:30017/admin?replicaSet=rs0

# Example:
mongodb://admin:Pr0d@Secure!Pass2024@192.168.1.100:30017/admin?replicaSet=rs0
```

**Test from your machine:**

```bash
mongosh "mongodb://admin:Pr0d@Secure!Pass2024@192.168.1.100:30017/admin?replicaSet=rs0"
```

---

## 🔄 UPGRADING MONGODB

### Upgrade to New Chart Version

```bash
# Update repository
helm repo update

# Check available versions
helm search repo mongodb-ha --versions

# Upgrade to latest
helm upgrade mongodb-test mongodb-ha/mongodb-ha \
  --namespace mongodb-test \
  --reuse-values

# Or upgrade to specific version
helm upgrade mongodb-test mongodb-ha/mongodb-ha \
  --version 1.0.1 \
  --namespace mongodb-test \
  --reuse-values
```

### Change Configuration

```bash
# Increase replicas
helm upgrade mongodb-test mongodb-ha/mongodb-ha \
  --namespace mongodb-test \
  --reuse-values \
  --set mongodb.replicaSet.replicas=5

# Change storage size (won't shrink existing)
helm upgrade mongodb-test mongodb-ha/mongodb-ha \
  --namespace mongodb-test \
  --reuse-values \
  --set mongodb.storage.size=50Gi

# Change password (requires pod restart)
helm upgrade mongodb-test mongodb-ha/mongodb-ha \
  --namespace mongodb-test \
  --reuse-values \
  --set mongodb.auth.rootPassword="NewPassword123"
```

---

## 🗑️ UNINSTALLING

### Keep Data (Uninstall Only)

```bash
# Uninstall Helm release
helm uninstall mongodb-test -n mongodb-test

# Namespace and PVCs remain
kubectl get pvc -n mongodb-test
```

### Remove Everything Except PVs

```bash
# Uninstall
helm uninstall mongodb-test -n mongodb-test

# Delete namespace (PVCs deleted)
kubectl delete namespace mongodb-test

# PVs remain for recovery
kubectl get pv
```

### Complete Removal (ALL DATA DELETED!)

```bash
# Uninstall
helm uninstall mongodb-test -n mongodb-test

# Delete namespace
kubectl delete namespace mongodb-test

# List PVs bound to this release
kubectl get pv | grep mongodb-test

# Delete PVs (WARNING: DATA LOST!)
kubectl delete pv mongodb-pv-0 mongodb-pv-1 mongodb-pv-2

# On each node, delete data
sudo rm -rf /mnt/mongodb/data-*
```

---

## 🐛 TROUBLESHOOTING

### Issue: Pods Stuck in Pending

```bash
# Check PVC status
kubectl get pvc -n mongodb-test

# If PVC is Pending:
kubectl describe pvc mongodb-data-mongodb-test-mongodb-ha-0 -n mongodb-test

# Check available PVs
kubectl get pv

# Check storage class exists
kubectl get storageclass
```

**Solution:**
- Create PVs manually (see Storage Configuration section)
- Or use dynamic provisioning with correct storageClassName

### Issue: Init Job Failed

```bash
# Check job logs
kubectl logs -n mongodb-test job/mongodb-test-mongodb-ha-init-replicaset

# Check pod logs
kubectl logs -n mongodb-test mongodb-test-mongodb-ha-0

# Delete job and let it retry
kubectl delete job -n mongodb-test mongodb-test-mongodb-ha-init-replicaset
```

### Issue: Can't Connect to MongoDB

```bash
# Check all pods running
kubectl get pods -n mongodb-test

# Check services
kubectl get svc -n mongodb-test

# Test internal connectivity
kubectl run -it --rm test-mongo \
  --image=mongo:7.0 \
  --restart=Never \
  -n mongodb-test -- \
  mongosh mongodb://mongodb-test-mongodb-ha-service:27017

# Check from pod directly
kubectl exec -it mongodb-test-mongodb-ha-0 -n mongodb-test -- mongosh
```

### Issue: Released PVs Not Rebinding

If you see PVs in "Released" state:

```bash
# List Released PVs
kubectl get pv | grep Released

# Delete them
kubectl delete pv pvc-1e98c99e-8c7c-4792-89bf-eccd568caf83

# Or patch to make Available
kubectl patch pv pvc-1e98c99e-8c7c-4792-89bf-eccd568caf83 \
  -p '{"spec":{"claimRef": null}}'
```

---

## 📊 MONITORING

### Check Cluster Status

```bash
# All pods
kubectl get pods -n mongodb-test

# Replica set status
kubectl exec -it mongodb-test-mongodb-ha-0 -n mongodb-test -- \
  mongosh -u admin -p 'Test@MongoDB123' --authenticationDatabase admin \
  --eval "rs.status()"

# Check which is PRIMARY
kubectl exec -it mongodb-test-mongodb-ha-0 -n mongodb-test -- \
  mongosh -u admin -p 'Test@MongoDB123' --authenticationDatabase admin \
  --eval "rs.status().members.forEach(m => print(m.name + ': ' + m.stateStr))"
```

### View Logs

```bash
# All pods
kubectl logs -n mongodb-test -l app.kubernetes.io/name=mongodb-ha --tail=50

# Specific pod
kubectl logs -f mongodb-test-mongodb-ha-0 -n mongodb-test

# Init job
kubectl logs -n mongodb-test job/mongodb-test-mongodb-ha-init-replicaset
```

### Check Storage Usage

```bash
# Storage usage per pod
kubectl exec -it mongodb-test-mongodb-ha-0 -n mongodb-test -- df -h /data/db

# Database sizes
kubectl exec -it mongodb-test-mongodb-ha-0 -n mongodb-test -- \
  mongosh -u admin -p 'Test@MongoDB123' --authenticationDatabase admin \
  --eval "db.adminCommand({ listDatabases: 1 })"
```

---

## 🎯 QUICK REFERENCE

### Install Commands

```bash
# Add repository
helm repo add mongodb-ha https://YOUR-USERNAME.github.io/mongodb-ha-helm
helm repo update

# Install test
kubectl create namespace mongodb-test
helm install mongodb-test mongodb-ha/mongodb-ha -n mongodb-test

# Install production
kubectl create namespace mongodb-ha
helm install mongodb-prod mongodb-ha/mongodb-ha \
  -n mongodb-ha \
  --set mongodb.storage.size=40Gi \
  --set mongodb.service.nodePort=30017

# Watch installation
kubectl get pods -n mongodb-test -w
```

### Verification Commands

```bash
# Check everything
kubectl get all -n mongodb-test

# Check storage
kubectl get pvc,pv -n mongodb-test

# Check init job
kubectl logs -n mongodb-test job/mongodb-test-mongodb-ha-init-replicaset

# Test connection
kubectl run -it --rm test --image=mongo:7.0 --restart=Never -n mongodb-test -- \
  mongosh mongodb://admin:PASSWORD@mongodb-test-mongodb-ha-service:27017/admin
```

### Cleanup Commands

```bash
# Uninstall
helm uninstall mongodb-test -n mongodb-test

# Delete namespace
kubectl delete namespace mongodb-test

# Delete PVs
kubectl delete pv mongodb-pv-0 mongodb-pv-1 mongodb-pv-2
```

---

## ✅ Installation Checklist

- [ ] Helm repository added and updated
- [ ] Namespace created
- [ ] Storage prerequisites met (PVs or storageClass)
- [ ] Chart installed without errors
- [ ] All pods Running (1/1)
- [ ] Init job completed successfully
- [ ] Can connect to MongoDB
- [ ] Replica set status shows PRIMARY

**If all checked, installation is complete! 🎉**

---

**Your MongoDB HA cluster is ready to use!**

Connection string template:
```
mongodb://admin:PASSWORD@RELEASE-NAME-mongodb-ha-service.NAMESPACE.svc.cluster.local:27017/admin?replicaSet=rs0
```