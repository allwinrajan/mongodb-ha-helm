# MongoDB HA Helm Chart - Packaging & Publishing Guide

## 📦 READY TO PACKAGE!

✅ Your MongoDB HA Helm chart is **production-ready** and tested!

---

## 🎯 QUICK PACKAGING STEPS

### Step 1: Final Validation

```bash
cd /home/administrator/mongodb-ha-helm

# Lint the chart
helm lint .

# Expected output:
# ==> Linting .
# [INFO] Chart.yaml: icon is recommended
# 1 chart(s) linted, 0 chart(s) failed
```

### Step 2: Package the Chart

```bash
# Create .tgz package
helm package .

# Output:
# Successfully packaged chart and saved it to: /home/administrator/mongodb-ha-helm/mongodb-ha-1.0.0.tgz

# Verify
ls -lh mongodb-ha-1.0.0.tgz
```

✅ **You now have: `mongodb-ha-1.0.0.tgz`**

---

## 🌐 OPTION 1: USE ON ANOTHER MACHINE (SIMPLE)

### Transfer the Package

```bash
# Copy to another machine via SCP
scp mongodb-ha-1.0.0.tgz user@192.168.1.100:/tmp/

# Or upload to shared storage
# Or commit to Git repository
```

### Install on Target Machine

```bash
# On the target machine
cd /tmp

# Install directly from package
helm install mongodb-prod ./mongodb-ha-1.0.0.tgz \
  -f custom-values.yaml \
  --namespace mongodb-ha \
  --create-namespace \
  --timeout 15m
```

---

## 🌍 OPTION 2: PUBLISH TO GITHUB PAGES (RECOMMENDED)

This allows anyone to install with: `helm repo add ...`

### Step 1: Create GitHub Repository

1. Go to: https://github.com/new
2. Repository name: `mongodb-ha-helm`
3. Make it **Public**
4. Click **Create repository**

### Step 2: Initialize Git

```bash
cd /home/administrator/mongodb-ha-helm

# Initialize git
git init

# Add all files
git add .

# Commit
git commit -m "Initial release: MongoDB HA Helm Chart v1.0.0"
```

### Step 3: Create Helm Repository Index

```bash
# Replace YOUR-GITHUB-USERNAME with your actual username
helm repo index . --url https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm

# Example:
# helm repo index . --url https://john-doe.github.io/mongodb-ha-helm

# Add index to git
git add index.yaml
git commit -m "Add Helm repository index"
```

### Step 4: Push to GitHub

```bash
# Add remote (replace YOUR-GITHUB-USERNAME)
git remote add origin https://github.com/YOUR-GITHUB-USERNAME/mongodb-ha-helm.git

# Push
git branch -M main
git push -u origin main
```

### Step 5: Enable GitHub Pages

1. Go to your repository on GitHub
2. Click **Settings** → **Pages**
3. Source: **Branch: main** / **/ (root)**
4. Click **Save**

Wait 2-3 minutes, then your chart is published at:
```
https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm/
```

### Step 6: Test Installation

On any machine with Helm:

```bash
# Add your Helm repository
helm repo add my-mongodb https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm

# Update
helm repo update

# Search
helm search repo my-mongodb

# Install
helm install mongodb-prod my-mongodb/mongodb-ha \
  --namespace mongodb-ha \
  --create-namespace \
  --set mongodb.auth.rootPassword="YourPassword123!"
```

---

## 🔄 UPDATE YOUR CHART (FUTURE RELEASES)

When you make changes:

```bash
# 1. Update version in Chart.yaml
nano Chart.yaml
# Change: version: 1.0.0 → version: 1.0.1

# 2. Package new version
helm package .

# 3. Update index
helm repo index . --url https://YOUR-GITHUB-USERNAME.github.io/mongodb-ha-helm --merge index.yaml

# 4. Commit and push
git add .
git commit -m "Release v1.0.1: Bug fixes"
git push

# Users can now update:
helm repo update
helm upgrade mongodb-prod my-mongodb/mongodb-ha
```

---

## 📝 QUICK REFERENCE

### Package Only
```bash
helm lint .
helm package .
# Result: mongodb-ha-1.0.0.tgz
```

### Publish to GitHub
```bash
helm lint .
helm package .
helm repo index . --url https://YOUR-USERNAME.github.io/mongodb-ha-helm
git add . && git commit -m "Release vX.X.X" && git push
```

### Install from GitHub
```bash
helm repo add mongodb-ha https://YOUR-USERNAME.github.io/mongodb-ha-helm
helm repo update
helm install mongodb-prod mongodb-ha/mongodb-ha -n mongodb-ha --create-namespace
```

### Install from File
```bash
helm install mongodb-prod ./mongodb-ha-1.0.0.tgz -n mongodb-ha --create-namespace
```

---

## ✅ YOU'RE READY!

Your chart is:
- ✅ Tested and working
- ✅ Production-ready
- ✅ Ready to package
- ✅ Ready to share

**Choose your distribution method:**
- 📦 Package file (`.tgz`) - Simple, direct transfer
- 🌐 GitHub Pages - Professional, easy updates
- 🌟 Artifact Hub - Public discoverability (optional)

---

## 🎯 RECOMMENDED WORKFLOW FOR YOUR CASE

Since you're deploying to production with local storage:

1. **Package the chart:**
   ```bash
   helm package .
   ```

2. **Transfer to production machines:**
   ```bash
   scp mongodb-ha-1.0.0.tgz prod-machine:/opt/helm-charts/
   ```

3. **On production, follow PRODUCTION-DEPLOYMENT-GUIDE.md:**
   - Create storage directories
   - Create PVs with local-storage
   - Install with: `helm install mongodb-prod ./mongodb-ha-1.0.0.tgz -f values-prod.yaml`

4. **Verify deployment**
   - Check pods, PVCs, replica set status
   - Test authentication and connections
   - Create application users

**That's it! You're production-ready! 🚀**