# MongoDB HA Helm Chart

Production-ready MongoDB High Availability Replica Set for Kubernetes.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PersistentVolume provisioner support (local-storage or similar)
- `kubectl` configured to access your cluster

## Features

- ✅ 3-node MongoDB Replica Set
- ✅ Automatic replica set initialization
- ✅ Authentication enabled by default
- ✅ Persistent storage
- ✅ Resource limits configured
- ✅ Health checks (liveness/readiness probes)
- ✅ NodePort service for external access

---

## Quick Start

### Step 1: Prepare Storage (One-time setup)

Before installing, create persistent volumes:

```bash
# Create storage directories on each Kubernetes node
sudo mkdir -p /mnt/mongodb-storage/mongodb-data-{0,1,2}
sudo chmod 777 /mnt/mongodb-storage/mongodb-data-*

# Create PersistentVolume manifests
kubectl apply -f - <<EOF
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
    path: /mnt/mongodb-storage/mongodb-data-0
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - <your-node-name>
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
    path: /mnt/mongodb-storage/mongodb-data-1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - <your-node-name>
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
    path: /mnt/mongodb-storage/mongodb-data-2
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - <your-node-name>
EOF
```

Replace `<your-node-name>` with your actual node name from `kubectl get nodes`.

### Step 2: Install MongoDB HA

```bash
# Add the chart (if from repo)
helm repo add myrepo https://yourrepo.com/charts
helm repo update

# Or install from local directory
cd mongodb-ha-helm

# Install with default values
helm install mongodb-ha . --namespace mongodb-ha --create-namespace

# Or customize installation
helm install mongodb-ha . \
  --namespace mongodb-ha \
  --create-namespace \
  --set mongodb.auth.rootPassword="YourSecurePassword123!" \
  --set mongodb.storage.size=50Gi
```

### Step 3: Verify Installation

```bash
# Check pods status
kubectl get pods -n mongodb-ha

# Should see:
# NAME         READY   STATUS    RESTARTS   AGE
# mongodb-0    1/1     Running   0          2m
# mongodb-1    1/1     Running   0          2m
# mongodb-2    1/1     Running   0          2m

# Check replica set initialization job
kubectl get jobs -n mongodb-ha

# View initialization logs
kubectl logs -n mongodb-ha job/mongodb-ha-init-replicaset
```

### Step 4: Connect to MongoDB

```bash
# Connect from within cluster
kubectl run -it --rm mongo-client \
  --image=mongo:7.0 \
  --restart=Never \
  --namespace=mongodb-ha \
  -- mongosh "mongodb://admin:Pr0d@Mong0DB!2024@mongodb-ha-service.mongodb-ha.svc.cluster.local:27017/admin?replicaSet=rs0"

# Connect from outside cluster (NodePort)
mongosh "mongodb://admin:Pr0d@Mong0DB!2024@<node-ip>:30017/admin?replicaSet=rs0"

# Check replica set status
rs.status()
```

---

## Configuration

### Important Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `mongodb.auth.rootUsername` | Admin username | `admin` |
| `mongodb.auth.rootPassword` | Admin password | `Pr0d@Mong0DB!2024` |
| `mongodb.replicaSet.name` | Replica set name | `rs0` |
| `mongodb.replicaSet.replicas` | Number of replicas | `3` |
| `mongodb.storage.size` | PVC size per pod | `40Gi` |
| `mongodb.storage.storageClassName` | Storage class name | `local-storage` |
| `mongodb.service.nodePort` | External access port | `30017` |
| `mongodb.resources.requests.memory` | Memory request | `2Gi` |
| `mongodb.resources.limits.memory` | Memory limit | `4Gi` |

### Custom Values File

Create `my-values.yaml`:

```yaml
mongodb:
  auth:
    rootPassword: "MyCustomPassword123!"
  
  storage:
    size: 100Gi
  
  resources:
    requests:
      memory: "4Gi"
      cpu: "1000m"
    limits:
      memory: "8Gi"
      cpu: "2000m"
```

Install with custom values:

```bash
helm install mongodb-ha . -f my-values.yaml --namespace mongodb-ha --create-namespace
```

---

## Packaging and Publishing

### Package the Chart

```bash
# Lint the chart first
helm lint .

# Package the chart
helm package .

# This creates: mongodb-ha-1.0.0.tgz
```

### Push to Artifact Hub

#### Method 1: GitHub Pages (Recommended)

1. Create a GitHub repository
2. Create `index.yaml`:

```bash
helm repo index . --url https://yourusername.github.io/mongodb-ha-helm
```

3. Push to GitHub:

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/yourusername/mongodb-ha-helm.git
git push -u origin main
```

4. Enable GitHub Pages in repository settings
5. Submit to Artifact Hub: https://artifacthub.io/packages/helm/new

#### Method 2: ChartMuseum

```bash
# Install ChartMuseum
helm repo add chartmuseum https://chartmuseum.github.io/charts
helm install chartmuseum chartmuseum/chartmuseum

# Upload chart
curl --data-binary "@mongodb-ha-1.0.0.tgz" http://localhost:8080/api/charts
```

---

## Using on Another Machine

### Install from Helm Repository

```bash
# Add your repository
helm repo add myrepo https://yourusername.github.io/mongodb-ha-helm

# Update repositories
helm repo update

# Search for the chart
helm search repo mongodb-ha

# Install
helm install my-mongodb myrepo/mongodb-ha \
  --namespace mongodb-ha \
  --create-namespace \
  --set mongodb.auth.rootPassword="SecurePass123!"
```

### Install from Package File

```bash
# Copy the .tgz file to the new machine
scp mongodb-ha-1.0.0.tgz user@newmachine:/tmp/

# On new machine
helm install mongodb-ha /tmp/mongodb-ha-1.0.0.tgz \
  --namespace mongodb-ha \
  --create-namespace
```

---

## Maintenance

### Upgrade

```bash
# Upgrade with new values
helm upgrade mongodb-ha . \
  --namespace mongodb-ha \
  --set mongodb.auth.rootPassword="NewPassword123!"

# Check upgrade status
helm status mongodb-ha -n mongodb-ha
```

### Backup

```bash
# Create backup job
kubectl run mongodb-backup --image=mongo:7.0 \
  --namespace=mongodb-ha \
  --restart=Never \
  --command -- mongodump \
  --uri="mongodb://admin:Pr0d@Mong0DB!2024@mongodb-ha-service:27017/admin?replicaSet=rs0" \
  --out=/backup
```

### Scale (Not recommended for replica set)

```bash
# MongoDB replica sets should maintain 3 or 5 nodes
# Scaling requires manual replica set reconfiguration
```

### Uninstall

```bash
# Remove MongoDB
helm uninstall mongodb-ha -n mongodb-ha

# Delete namespace
kubectl delete namespace mongodb-ha

# Clean up PVs (if needed)
kubectl delete pv mongodb-pv-0 mongodb-pv-1 mongodb-pv-2

# Clean up storage (on each node)
sudo rm -rf /mnt/mongodb-storage/mongodb-data-*
```

---

## Troubleshooting

### Pods not starting

```bash
# Check pod events
kubectl describe pod mongodb-0 -n mongodb-ha

# Check logs
kubectl logs mongodb-0 -n mongodb-ha

# Common issues:
# - PV not available: Check `kubectl get pv`
# - Permission denied: Check directory permissions on nodes
```

### Replica set initialization failed

```bash
# Check init job logs
kubectl logs -n mongodb-ha job/mongodb-ha-init-replicaset

# Manually initialize (if needed)
kubectl exec -it mongodb-0 -n mongodb-ha -- mongosh
> rs.initiate()
```

### Cannot connect

```bash
# Check services
kubectl get svc -n mongodb-ha

# Test internal connectivity
kubectl run test-pod --image=mongo:7.0 --rm -it -- \
  mongosh "mongodb://mongodb-ha-service.mongodb-ha.svc.cluster.local:27017"

# Check NodePort
kubectl get svc mongodb-ha-nodeport -n mongodb-ha
```

---

## Support

- GitHub Issues: https://github.com/yourusername/mongodb-ha-helm/issues
- MongoDB Documentation: https://docs.mongodb.com/
- Kubernetes Documentation: https://kubernetes.io/docs/

## License

Apache 2.0