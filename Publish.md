# MongoDB HA Helm Chart - Complete Guide

## 📦 PART 1: PACKAGING YOUR HELM CHART

### Step 1: Verify Your Chart Structure

First, let's make sure everything is correct:

```bash
# Check current directory
pwd
# Should show: /path/to/mongodb-ha-helm

# List all files
ls -la

# You should see:
# Chart.yaml
# values.yaml
# README.md
# .helmignore
# templates/
```

### Step 2: Validate the Chart

```bash
# Lint your chart (finds errors)
helm lint .

# Expected output:
# ==> Linting .
# [INFO] Chart.yaml: icon is recommended
# 1 chart(s) linted, 0 chart(s) failed
```

If you see errors, fix them before continuing.

### Step 3: Test with Dry Run

```bash
# Test if chart renders correctly
helm install test-mongodb . --dry-run --debug --namespace mongodb-ha

# This will show all YAML that would be created
# Scroll through and check for errors
# Press Ctrl+C when done reviewing
```

### Step 4: Package the Chart

```bash
# Create the .tgz package
helm package .

# Output:
# Successfully packaged chart and saved it to: /path/to/mongodb-ha-helm/mongodb-ha-1.0.0.tgz

# Verify the package was created
ls -lh mongodb-ha-1.0.0.tgz
```

---

## 🌐 PART 2: PUBLISH TO GITHUB PAGES

### Step 1: Create GitHub Repository

Go to GitHub and create a new repository:

1. Go to: https://github.com/new
2. **Repository name**: `mongodb-ha-helm`
3. **Description**: "Production MongoDB HA Helm Chart"
4. Make it **Public** (required for Artifact Hub)
5. **DO NOT** initialize with README, .gitignore, or license
6. Click **Create repository**

### Step 2: Prepare Your Local Repository

Still in your `mongodb-ha-helm` directory:

```bash
# Initialize git
git init

# Check what files will be added
git status

# Add all files
git add .

# Commit
git commit -m "Initial commit: MongoDB HA Helm Chart v1.0.0"
```

### Step 3: Create Helm Repository Index

This is the critical step for Helm:

```bash
# Create index.yaml (Helm repository index)
helm repo index . --url https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm

# Example: If your GitHub username is "john-doe"
helm repo index . --url https://john-doe.github.io/mongodb-ha-helm

# Verify index.yaml was created
ls -la index.yaml

# Check its content
cat index.yaml
```

**What this does**: Creates a catalog file that tells Helm where to find your chart.

### Step 4: Add the index.yaml to Git

```bash
# Add index.yaml
git add index.yaml

# Commit
git commit -m "Add Helm repository index"
```

### Step 5: Push to GitHub

Replace `YOUR-GITHUB-USERNAME` with your actual username:

```bash
# Add remote repository
git remote add origin https://github.com/YOUR-GITHUB-USERNAME/mongodb-ha-helm.git

# Example:
# git remote add origin https://github.com/john-doe/mongodb-ha-helm.git

# Push to main branch
git branch -M main
git push -u origin main
```

Enter your GitHub credentials when prompted.

### Step 6: Enable GitHub Pages

1. Go to your repository on GitHub
2. Click **Settings** (top right)
3. Scroll down to **Pages** (left sidebar)
4. Under **Source**, select:
   - **Branch**: `main`
   - **Folder**: `/ (root)`
5. Click **Save**

Wait 2-3 minutes, then you'll see:

```
✅ Your site is published at https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm/
```

### Step 7: Verify Helm Repository Works

```bash
# Add your new repository locally
helm repo add my-mongodb https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm

# Example:
# helm repo add my-mongodb https://john-doe.github.io/mongodb-ha-helm

# Update repositories
helm repo update

# Search for your chart
helm search repo my-mongodb

# Expected output:
# NAME                    CHART VERSION   APP VERSION     DESCRIPTION
# my-mongodb/mongodb-ha   1.0.0           7.0             MongoDB High Availability Replica Set for Kub...
```

✅ **If you see this, your Helm repository is live!**

---

## 🌟 PART 3: SUBMIT TO ARTIFACT HUB (Optional but Recommended)

This makes your chart discoverable on https://artifacthub.io

### Step 1: Create artifacthub-repo.yml

In your repository root:

```yaml
# artifacthub-repo.yml
repositoryID: YOUR-RANDOM-ID-HERE
owners:
  - name: Your Name
    email: your.email@example.com
```

```bash
# Create the file
nano artifacthub-repo.yml

# Paste the content above
# Replace with your actual name and email

# Add to git
git add artifacthub-repo.yml
git commit -m "Add Artifact Hub metadata"
git push
```

### Step 2: Submit to Artifact Hub

1. Go to: https://artifacthub.io
2. **Sign in** with GitHub
3. Click **Control Panel** (top right)
4. Click **Add repository**
5. Fill in:
   - **Name**: `mongodb-ha-helm`
   - **Display name**: `MongoDB HA`
   - **URL**: `https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm`
   - **Type**: Helm charts
6. Click **Add**

Wait 5-10 minutes for indexing. Your chart will appear at:

```
https://artifacthub.io/packages/helm/YOUR-REPO-NAME/mongodb-ha
```

---

## 💻 PART 4: USING ON ANOTHER MACHINE

Now let's install your chart on a completely different machine.

### Machine Requirements

```bash
# Check prerequisites
kubectl version --client
helm version

# Both should work
```

### Method A: Install from GitHub Pages (Recommended)

On the new machine:

```bash
# Step 1: Add your Helm repository
helm repo add mongodb-ha https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm

# Example:
# helm repo add mongodb-ha https://john-doe.github.io/mongodb-ha-helm

# Step 2: Update repositories
helm repo update

# Step 3: Search to verify
helm search repo mongodb-ha

# Step 4: See available values
helm show values mongodb-ha/mongodb-ha

# Step 5: Install with default values
helm install my-mongodb mongodb-ha/mongodb-ha \
  --namespace mongodb-ha \
  --create-namespace

# OR install with custom password
helm install my-mongodb mongodb-ha/mongodb-ha \
  --namespace mongodb-ha \
  --create-namespace \
  --set mongodb.auth.rootPassword="MySecurePassword123!"

# Step 6: Watch pods starting
kubectl get pods -n mongodb-ha -w
```

### Method B: Install from Package File

If you want to share the `.tgz` file directly:

**On the first machine:**

```bash
# Find your package
ls -lh mongodb-ha-1.0.0.tgz

# Copy to another machine using scp
scp mongodb-ha-1.0.0.tgz user@192.168.1.100:/tmp/

# Or upload to a file server/cloud storage
```

**On the new machine:**

```bash
# Go to where you copied the file
cd /tmp

# Verify file exists
ls -lh mongodb-ha-1.0.0.tgz

# Install directly from file
helm install my-mongodb ./mongodb-ha-1.0.0.tgz \
  --namespace mongodb-ha \
  --create-namespace

# Or with custom values
helm install my-mongodb ./mongodb-ha-1.0.0.tgz \
  --namespace mongodb-ha \
  --create-namespace \
  --set mongodb.auth.rootPassword="MyPassword123!"
```

### Method C: Install from Artifact Hub

Once your chart is on Artifact Hub:

```bash
# Users can find it by searching
helm search hub mongodb-ha

# Install directly
helm repo add mongodb-ha-repo REPO_URL_FROM_ARTIFACTHUB
helm repo update
helm install my-mongodb mongodb-ha-repo/mongodb-ha \
  --namespace mongodb-ha \
  --create-namespace
```

---

## 🔧 PART 5: COMPLETE INSTALLATION EXAMPLE

Let me show you a full end-to-end installation on a new machine:

### Before Installation (One-time setup)

```bash
# 1. Create storage on Kubernetes nodes
# SSH to each node and run:
sudo mkdir -p /mnt/mongodb-storage/mongodb-data-{0,1,2}
sudo chmod 777 /mnt/mongodb-storage/mongodb-data-*
```

### Create PersistentVolumes

Create a file `storage-setup.yaml`:

```yaml
# storage-setup.yaml
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
          - worker-node-1
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
          - worker-node-1
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
          - worker-node-1
```

```bash
# Get your node name
kubectl get nodes

# Edit storage-setup.yaml and replace "worker-node-1" with your node name

# Apply storage
kubectl apply -f storage-setup.yaml

# Verify PVs are Available
kubectl get pv
```

### Install MongoDB HA

```bash
# Add repository
helm repo add mongodb-ha https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm
helm repo update

# Install
helm install my-mongodb mongodb-ha/mongodb-ha \
  --namespace mongodb-ha \
  --create-namespace \
  --set mongodb.auth.rootPassword="Secure123!"

# Watch installation
kubectl get pods -n mongodb-ha -w

# Wait until all 3 pods are Running
```

### Verify Installation

```bash
# Check all resources
kubectl get all -n mongodb-ha

# Check initialization job logs
kubectl logs -n mongodb-ha job/my-mongodb-mongodb-ha-init-replicaset

# You should see: "✓ Initialization Complete!"
```

### Connect to MongoDB

```bash
# Get NodePort
kubectl get svc -n mongodb-ha

# Connect from outside
mongosh "mongodb://admin:Secure123!@<NODE-IP>:30017/admin?replicaSet=rs0"

# Inside cluster
kubectl run -it --rm mongo-test --image=mongo:7.0 --restart=Never -n mongodb-ha -- \
  mongosh "mongodb://admin:Secure123!@my-mongodb-mongodb-ha-service:27017/admin?replicaSet=rs0"

# Check replica set status
rs.status()
```

---

## 📝 PART 6: UPDATING YOUR CHART

When you make changes to your chart:

```bash
# 1. Edit files as needed
nano values.yaml

# 2. Bump version in Chart.yaml
nano Chart.yaml
# Change: version: 1.0.0 → version: 1.0.1

# 3. Package new version
helm package .

# 4. Update index
helm repo index . --url https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm --merge index.yaml

# 5. Commit and push
git add .
git commit -m "Release v1.0.1"
git push

# 6. Users can now upgrade
helm repo update
helm search repo mongodb-ha  # Shows new version
helm upgrade my-mongodb mongodb-ha/mongodb-ha
```

---

## 🎯 QUICK REFERENCE CARD

Save this for future use:

```bash
# PUBLISH NEW VERSION
helm lint .
helm package .
helm repo index . --url https://YOUR-USERNAME.github.io/mongodb-ha-helm --merge index.yaml
git add . && git commit -m "Release vX.X.X" && git push

# INSTALL ON NEW MACHINE
helm repo add mongodb-ha https://YOUR-USERNAME.github.io/mongodb-ha-helm
helm repo update
helm install my-mongodb mongodb-ha/mongodb-ha -n mongodb-ha --create-namespace

# VERIFY
kubectl get pods -n mongodb-ha
kubectl logs -n mongodb-ha job/my-mongodb-mongodb-ha-init-replicaset

# UNINSTALL
helm uninstall my-mongodb -n mongodb-ha
kubectl delete namespace mongodb-ha
kubectl delete pv mongodb-pv-0 mongodb-pv-1 mongodb-pv-2
```

---

## ✅ SUCCESS CHECKLIST

Before moving to another machine, verify:

- [ ] `helm lint .` passes
- [ ] Package created: `mongodb-ha-1.0.0.tgz` exists
- [ ] GitHub repository created and pushed
- [ ] GitHub Pages enabled
- [ ] `helm repo add` works from your local machine
- [ ] `helm search repo` finds your chart
- [ ] Chart installs successfully locally