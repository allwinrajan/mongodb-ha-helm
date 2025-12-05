# MongoDB HA Helm Chart - SIMPLIFIED Build & Publish Guide

## 🚨 CRITICAL FIX FIRST: Remove Namespace Creation

Your chart is creating namespaces which causes conflicts. Let's fix this:

### Step 1: Edit values.yaml

```bash
cd /home/administrator/mongodb-ha-helm
nano values.yaml
```

**Change this:**
```yaml
namespace:
  name: mongodb-ha
  create: true          # ← CHANGE THIS TO FALSE
```

**To this:**
```yaml
namespace:
  name: mongodb-ha
  create: false         # ← MUST BE FALSE!
```

**Do the same for ALL values files:**

```bash
# Edit values-test.yaml
nano values-test.yaml
# Change: create: true → create: false

# Edit values-prod.yaml
nano values-prod.yaml
# Change: create: true → create: false

# Edit values-dev.yaml (if you have it)
nano values-dev.yaml
# Change: create: true → create: false
```

### Step 2: Update Chart.yaml

```bash
nano Chart.yaml
```

**Replace with this:**

```yaml
apiVersion: v2
name: mongodb-ha
description: MongoDB High Availability Replica Set for Kubernetes
type: application
version: 1.0.0
appVersion: "7.0"
keywords:
  - mongodb
  - database
  - replicaset
  - high-availability
maintainers:
  - name: Your Name
    email: your.email@example.com
```

Save and exit (Ctrl+O, Enter, Ctrl+X).

---

## 📦 BUILD YOUR HELM CHART

### Step 1: Clean Up Old Installation

```bash
# Delete old test installation
helm uninstall mongodb-test -n mongodb-test 2>/dev/null || true

# Wait for namespace to fully delete
kubectl delete namespace mongodb-test --wait=true --timeout=60s

# Clean up Released PVs
kubectl delete pv pvc-1e98c99e-8c7c-4792-89bf-eccd568caf83 \
                  pvc-6fc0df8c-bb23-4695-841b-72591c4c4c29 \
                  pvc-9c5e444a-d9a7-40ae-be51-d582bf408146 \
                  pvc-bd36c118-9a29-4e8a-a286-f325b1c5238c
```

### Step 2: Validate Chart

```bash
cd /home/administrator/mongodb-ha-helm

# Check for errors
helm lint .

# Test rendering
helm template test-release . \
  -f values-test.yaml \
  --namespace mongodb-test | less

# Press 'q' to exit
```

**Expected output:**
```
==> Linting .
[INFO] Chart.yaml: icon is recommended
1 chart(s) linted, 0 chart(s) failed
```

### Step 3: Test Installation Locally

```bash
# Create namespace manually (this is the correct way!)
kubectl create namespace mongodb-test

# Install chart
helm install mongodb-test . \
  -f values-test.yaml \
  --namespace mongodb-test \
  --timeout 10m

# Watch pods starting
kubectl get pods -n mongodb-test -w
```

**Press Ctrl+C when all pods are Running (1/1).**

### Step 4: Verify It Works

```bash
# Check everything is running
kubectl get all -n mongodb-test

# Check init job completed
kubectl logs -n mongodb-test job/mongodb-test-mongodb-ha-init-replicaset

# Should see: "✓ INITIALIZATION COMPLETE!"
```

### Step 5: Clean Up Test

```bash
# Uninstall
helm uninstall mongodb-test -n mongodb-test

# Delete namespace
kubectl delete namespace mongodb-test
```

---

## 📤 PUBLISH TO GITHUB

### Step 1: Package Chart

```bash
cd /home/administrator/mongodb-ha-helm

# Remove old packages
rm -f mongodb-ha-*.tgz

# Create new package
helm package .

# Verify
ls -lh mongodb-ha-1.0.0.tgz
```

**Expected output:**
```
Successfully packaged chart and saved it to: /home/administrator/mongodb-ha-helm/mongodb-ha-1.0.0.tgz
```

### Step 2: Initialize Git (First Time Only)

```bash
# Initialize git repository
git init

# Set your identity
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Add all files
git add .

# Create first commit
git commit -m "Initial release: MongoDB HA v1.0.0"
```

### Step 3: Create GitHub Repository

1. **Go to:** https://github.com/new
2. **Repository name:** `mongodb-ha-helm`
3. **Description:** `Production MongoDB HA Helm Chart`
4. **Public** (must be public for GitHub Pages)
5. **DO NOT** check any initialization boxes
6. **Click:** "Create repository"

### Step 4: Push to GitHub

**Replace `YOUR-GITHUB-USERNAME` below:**

```bash
# Add remote repository
git remote add origin https://github.com/YOUR-GITHUB-USERNAME/mongodb-ha-helm.git

# Example:
# git remote add origin https://github.com/john-doe/mongodb-ha-helm.git

# Push to GitHub
git branch -M main
git push -u origin main
```

**Enter your GitHub username and password when prompted.**

### Step 5: Create Helm Repository Index

This is the **MOST IMPORTANT** step:

```bash
# Create index.yaml (replace YOUR-GITHUB-USERNAME)
helm repo index . --url https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm

# Example:
# helm repo index . --url https://john-doe.github.io/mongodb-ha-helm

# Verify index.yaml exists
cat index.yaml
```

**You should see:**
```yaml
apiVersion: v1
entries:
  mongodb-ha:
  - apiVersion: v2
    appVersion: "7.0"
    created: "2024-12-04T..."
    description: MongoDB High Availability Replica Set for Kubernetes
    name: mongodb-ha
    urls:
    - https://YOUR-USERNAME.github.io/mongodb-ha-helm/mongodb-ha-1.0.0.tgz
    version: 1.0.0
```

### Step 6: Push Package and Index

```bash
# Add the package and index
git add mongodb-ha-1.0.0.tgz index.yaml

# Commit
git commit -m "Add packaged chart and Helm repository index"

# Push
git push
```

### Step 7: Enable GitHub Pages

1. Go to your repository on GitHub
2. Click **Settings** (top menu)
3. Click **Pages** (left sidebar)
4. Under **Source**:
   - **Branch:** main
   - **Folder:** / (root)
5. Click **Save**

**Wait 2-3 minutes**, then you'll see:
```
✅ Your site is published at https://YOUR-USERNAME.github.io/mongodb-ha-helm/
```

---

## ✅ VERIFY PUBLICATION

### Step 1: Test from Your Machine

```bash
# Add your Helm repository
helm repo add my-mongodb https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm

# Example:
# helm repo add my-mongodb https://john-doe.github.io/mongodb-ha-helm

# Update repository cache
helm repo update

# Search for chart
helm search repo my-mongodb
```

**Expected output:**
```
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
my-mongodb/mongodb-ha   1.0.0           7.0             MongoDB High Availability...
```

### Step 2: Test Chart Values

```bash
# View chart values
helm show values my-mongodb/mongodb-ha

# View chart info
helm show chart my-mongodb/mongodb-ha
```

### Step 3: Test Installation from Repository

```bash
# Create test namespace
kubectl create namespace helm-test

# Install from repository
helm install test-from-repo my-mongodb/mongodb-ha \
  --namespace helm-test \
  --set mongodb.replicaSet.replicas=1 \
  --set mongodb.storage.size=5Gi \
  --set mongodb.auth.rootPassword="TestPass123"

# Watch installation
kubectl get pods -n helm-test -w
```

**If this works, YOUR CHART IS PUBLISHED SUCCESSFULLY! 🎉**

Clean up:
```bash
helm uninstall test-from-repo -n helm-test
kubectl delete namespace helm-test
```

---

## 🔄 UPDATING YOUR CHART (Future Changes)

When you need to update your chart:

### Step 1: Make Changes

```bash
cd /home/administrator/mongodb-ha-helm

# Make your changes
nano values.yaml
```

### Step 2: Bump Version

```bash
# Edit Chart.yaml
nano Chart.yaml

# Change version:
# OLD: version: 1.0.0
# NEW: version: 1.0.1
```

### Step 3: Package New Version

```bash
# Package new version
helm package .

# This creates: mongodb-ha-1.0.1.tgz
```

### Step 4: Update Repository Index

**IMPORTANT: Use --merge flag!**

```bash
# Update index.yaml with new version
helm repo index . --url https://YOUR-USERNAME.github.io/mongodb-ha-helm --merge index.yaml
```

### Step 5: Commit and Push

```bash
# Add new files
git add mongodb-ha-1.0.1.tgz index.yaml Chart.yaml

# Commit
git commit -m "Release v1.0.1: Fixed namespace handling"

# Push
git push
```

**Wait 2-3 minutes**, then:

```bash
# Update your local cache
helm repo update

# Check new version
helm search repo my-mongodb --versions
```

---

## 📝 COMPLETE WORKFLOW SUMMARY

Here's the **EXACT** process you should follow:

### First Time Setup

```bash
# 1. Fix namespace creation in values files
sed -i 's/create: true/create: false/g' values*.yaml

# 2. Validate chart
helm lint .

# 3. Package chart
helm package .

# 4. Create Git repository
git init
git add .
git commit -m "Initial release v1.0.0"

# 5. Push to GitHub
git remote add origin https://github.com/YOUR-USERNAME/mongodb-ha-helm.git
git branch -M main
git push -u origin main

# 6. Create Helm repository index
helm repo index . --url https://YOUR-USERNAME.github.io/mongodb-ha-helm

# 7. Push index
git add mongodb-ha-1.0.0.tgz index.yaml
git commit -m "Add Helm repository index"
git push

# 8. Enable GitHub Pages (in GitHub Settings)

# 9. Test from local machine
helm repo add my-mongodb https://YOUR-USERNAME.github.io/mongodb-ha-helm
helm repo update
helm search repo my-mongodb
```

### For Updates

```bash
# 1. Make changes
nano values.yaml

# 2. Bump version in Chart.yaml
nano Chart.yaml  # version: 1.0.0 → 1.0.1

# 3. Package
helm package .

# 4. Update index (IMPORTANT: --merge flag!)
helm repo index . --url https://YOUR-USERNAME.github.io/mongodb-ha-helm --merge index.yaml

# 5. Push
git add .
git commit -m "Release v1.0.1"
git push

# 6. Users update
helm repo update
```

---

## 🎯 QUICK COMMANDS REFERENCE

```bash
# VALIDATE
helm lint .
helm template test . -n test | less

# PACKAGE
helm package .

# CREATE INDEX
helm repo index . --url https://YOUR-USERNAME.github.io/mongodb-ha-helm

# UPDATE INDEX (for new versions)
helm repo index . --url https://YOUR-USERNAME.github.io/mongodb-ha-helm --merge index.yaml

# GIT WORKFLOW
git add .
git commit -m "Release vX.X.X"
git push

# TEST REPOSITORY
helm repo add my-mongodb https://YOUR-USERNAME.github.io/mongodb-ha-helm
helm repo update
helm search repo my-mongodb
```

---

## 🚀 YOUR REPOSITORY URL

After publishing, share this with others:

```
https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm
```

They can install with:

```bash
helm repo add mongodb-ha https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm
helm repo update
helm install mongodb mongodb-ha/mongodb-ha -n mongodb-ns --create-namespace
```

**That's it! Your chart is now published and reproducible! 🎉**