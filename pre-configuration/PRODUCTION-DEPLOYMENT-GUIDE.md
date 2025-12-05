# MongoDB HA - Production Quick Start Guide

## 🚀 5-MINUTE PRODUCTION SETUP

This is the **fastest way** to deploy MongoDB HA in production with local storage.

---

## STEP 1: PREPARE NODES (2 minutes)

SSH to each Kubernetes worker node:

```bash
# On k8s-worker-1
sudo mkdir -p /mnt/mongodb-storage/mongodb-data-0
sudo chmod 777 /mnt/mongodb-storage/mongodb-data-0

# On k8s-worker-2
sudo mkdir -p /mnt/mongodb-storage/mongodb-data-1
sudo chmod 777 /mnt/mongodb-storage/mongodb-data-1

# On k8s-worker-3
sudo mkdir -p /mnt/mongodb-storage/mongodb-data-2
sudo chmod 777 /mnt/mongodb-storage/mongodb-data-2
```

**Single node?** Run all three commands on one node.

---

## STEP 2: CREATE PVs (1 minute)

Get your node names:

```bash
kubectl get nodes
```

Download and edit the PV file:

```bash
# Download the template
nano mongodb-pv-prod.yaml

# Replace ALL 'YOUR-NODE-NAME-HERE' with your actual node names
# Save and exit (Ctrl+X, Y, Enter)

# Apply
kubectl apply -f mongodb-pv-prod.yaml

# Verify (all should show "Available")
kubectl get pv
```

Expected output:
```
NAME           STATUS      STORAGECLASS    CAPACITY
mongodb-pv-0   Available   local-storage   40Gi
mongodb-pv-1   Available   local-storage   40Gi
mongodb-pv-2   Available   local-storage   40Gi
```

---

## STEP 3: INSTALL MONGODB HA (2 minutes)

```bash
cd /path/to/mongodb-ha-helm

# Validate
helm lint .

# Install
helm install mongodb-prod . \
  -f values-prod.yaml \
  --namespace mongodb-ha \
  --create-namespace \
  --timeout 15m

# Watch deployment
kubectl get pods -n mongodb-ha -w
```

Wait for: **All 3 pods show 1/1 Running** (~2-3 minutes)

---

## STEP 4: VERIFY (30 seconds)

```bash
# Check replica set
kubectl exec -it mongodb-prod-mongodb-ha-0 -n mongodb-ha -- \
  mongosh -u admin -p 'Pr0d@MongoDB!Secure2024' --authenticationDatabase admin --eval "rs.status()"

# Should show:
# ✅ 1 PRIMARY (pod-0)
# ✅ 2 SECONDARY (pod-1, pod-2)
# ✅ All health: 1
```

---

## ✅ DONE! YOU NOW HAVE:

- 3 MongoDB replicas with automatic failover
- Persistent storage (40GB per replica)
- Authentication enabled
- NodePort access on port 30017

---

## 🔗 CONNECTION STRINGS

**Internal (from pods in cluster):**
```
mongodb://admin:Pr0d@MongoDB!Secure2024@mongodb-prod-mongodb-ha-service.mongodb-ha.svc.cluster.local:27017/admin?replicaSet=rs0
```

**External (from outside cluster):**
```
mongodb://admin:Pr0d@MongoDB!Secure2024@<NODE-IP>:30017/admin?replicaSet=rs0
```

Get NODE-IP with: `kubectl get nodes -o wide`

---

## 🎯 NEXT STEPS

1. **Change default password** in values-prod.yaml
2. **Create application users** (don't use admin in apps!)
3. **Set up backups**
4. **Configure monitoring**

---

## 📖 FULL DOCUMENTATION

For detailed information, see:
- `PRODUCTION-DEPLOYMENT-GUIDE.md` - Complete production setup
- `PACKAGING-GUIDE.md` - How to share your chart
- Chart README - Configuration options

---

## 🆘 TROUBLESHOOTING

**Pods stuck in Pending?**
```bash
kubectl describe pod mongodb-prod-mongodb-ha-0 -n mongodb-ha
# Check PVC binding and node affinity
```

**Init job failed?**
```bash
kubectl logs -n mongodb-ha job/mongodb-prod-mongodb-ha-init-replicaset
```

**Can't connect?**
```bash
# Check NodePort service
kubectl get svc -n mongodb-ha

# Check firewall
sudo ufw allow 30017/tcp
```

---

## 🗑️ UNINSTALL

⚠️ **WARNING: This deletes all data!**

```bash
helm uninstall mongodb-prod -n mongodb-ha
kubectl delete namespace mongodb-ha
kubectl delete pv mongodb-pv-0 mongodb-pv-1 mongodb-pv-2

# On each node:
sudo rm -rf /mnt/mongodb-storage/mongodb-data-*
```

---

## ✨ SUCCESS INDICATORS

You're good to go when:
- ✅ All 3 pods: `1/1 Running`
- ✅ Init job: `COMPLETIONS 1/1`
- ✅ All PVCs: `Bound`
- ✅ `rs.status()` shows 1 PRIMARY + 2 SECONDARY
- ✅ Authentication works

**Your MongoDB HA cluster is production-ready! 🎉**