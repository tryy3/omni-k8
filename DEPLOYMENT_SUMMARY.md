# Longhorn Deployment Summary

## What Was Done

Successfully prepared migration from **Rook/Ceph → Longhorn** for your 5-node Talos cluster (3 control-plane + 2 workers).

---

## Files Created ✅

### Longhorn Application
```
apps/longhorn-system/
├── namespace/
│   └── namespace.yaml          # Namespace with privileged pod security
├── longhorn/
│   ├── Chart.yaml              # Helm chart v1.10.0
│   └── values.yaml             # Minimal config: 2 replicas, Omni UI integration
```

### Talos Configuration
```
infra/patches/
└── longhorn.yaml               # Kubelet extraMounts for /var/lib/longhorn
```

### Documentation
```
LONGHORN_OPERATIONS.md          # Complete operations guide
LONGHORN_MIGRATION_PLAN.md      # Full migration plan (reference)
```

---

## Files Modified ✅

- **infra/cluster-template.yaml** - Added Longhorn patch
- **README.md** - Updated to reference Longhorn instead of Rook/Ceph

---

## Files Removed ✅

- **apps/rook-ceph/** (entire directory)
- **ROOK_*.md** documentation files
- **DEBUG_SUMMARY.md**

---

## Configuration Details

### Longhorn Settings
- **Version:** 1.10.0
- **Replica Count:** 2 (matches your 2 worker nodes)
- **Storage Class:** `longhorn` (set as default)
- **Data Path:** `/var/lib/longhorn`
- **UI Access:** Omni Workload Proxy port 50083

### Talos Extensions (Already Installed)
- ✅ iscsi-tools
- ✅ util-linux-tools
- ✅ udev
- ✅ lvm2
- ✅ xfsprogs
- ✅ glibc

---

## Next Steps - DEPLOYMENT

### Step 1: Review Changes
```bash
cd /home/tryy3/Projects/omni-k8
git status
git diff
```

### Step 2: Commit and Push
```bash
git add .
git commit -m "Replace Rook/Ceph with Longhorn v1.10.0"
git push
```

### Step 3: Monitor ArgoCD Sync
```bash
# Watch ArgoCD applications
kubectl -n argocd get applications -w

# Watch Rook removal (should happen automatically)
watch kubectl -n rook-ceph get pods
```

**Expected:** Rook namespace should be cleaned up by ArgoCD when it syncs.

### Step 4: Monitor Cluster Update
Omni will detect the cluster-template.yaml change and apply the Longhorn patch (kubelet extraMounts).

**Note:** Nodes may restart to apply the kubelet configuration change. This is normal.

```bash
# Watch node status
watch kubectl get nodes
```

### Step 5: Monitor Longhorn Deployment
Once ArgoCD syncs the new longhorn-system application:

```bash
# Watch Longhorn pods
watch kubectl -n longhorn-system get pods

# Should see:
# - longhorn-manager (DaemonSet)
# - longhorn-driver-deployer
# - longhorn-ui
# - csi-provisioner
# - csi-attacher
# - csi-resizer
# - csi-snapshotter
# - engine-image (DaemonSet)
```

**Timeline:** 5-10 minutes for all pods to be ready.

### Step 6: Verify Storage Class
```bash
kubectl get storageclass

# Should show "longhorn" as default (with *)
# NAME       PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
# longhorn   driver.longhorn.io   Delete          Immediate           true                   XXm
```

### Step 7: Check Prometheus PVC
The old pending Prometheus PVC should be recreated with Longhorn:

```bash
# Watch PVCs in monitoring namespace
kubectl get pvc -n monitoring -w

# If old PVC still pending, delete it (ArgoCD will recreate):
kubectl delete pvc prometheus-monitoring-kube-prometheus-prometheus-db-prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring
```

### Step 8: Verify Monitoring Stack
```bash
# Check Prometheus pod
kubectl -n monitoring get pods | grep prometheus

# Should be Running with PVC bound
```

### Step 9: Access Longhorn UI
Via Omni Workload Proxy:
- Port: **50083**
- Label: "Longhorn UI"

Or via port-forward:
```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
# Visit: http://localhost:8080
```

In the UI, verify:
- All 5 nodes are detected
- 2 nodes are schedulable (the 2 workers)
- No errors on dashboard

---

## If Rook Won't Remove Cleanly

If ArgoCD doesn't remove Rook (due to finalizers), you may need to manually clean up:

```bash
# Delete Ceph cluster
kubectl delete cephcluster -n rook-ceph rook-ceph --wait=false

# Remove finalizers if stuck
kubectl patch cephcluster rook-ceph -n rook-ceph -p '{"metadata":{"finalizers":[]}}' --type=merge

# Force delete namespace
kubectl delete namespace rook-ceph --force --grace-period=0

# Remove Rook CRDs (optional but recommended)
kubectl get crd | grep rook.io | awk '{print $1}' | xargs kubectl delete crd
kubectl get crd | grep ceph.rook.io | awk '{print $1}' | xargs kubectl delete crd
```

---

## Verification Checklist

After deployment completes:

- [ ] Rook namespace deleted (`kubectl get ns`)
- [ ] Longhorn namespace exists (`kubectl get ns longhorn-system`)
- [ ] All Longhorn pods running (`kubectl -n longhorn-system get pods`)
- [ ] Storage class exists and is default (`kubectl get storageclass`)
- [ ] Longhorn UI accessible (Omni proxy port 50083)
- [ ] Prometheus PVC bound (`kubectl get pvc -n monitoring`)
- [ ] Prometheus pod running (`kubectl -n monitoring get pods | grep prometheus`)
- [ ] Grafana accessible (Omni proxy port 50082)
- [ ] Metrics showing in Grafana

---

## Timeline Estimate

| Phase | Duration | Notes |
|-------|----------|-------|
| Git push | 1 min | - |
| ArgoCD sync | 2-5 min | Detects changes |
| Rook removal | 5-10 min | May need manual cleanup |
| Omni applies cluster patch | 5-10 min | Nodes may restart |
| Longhorn deployment | 5-10 min | All pods starting |
| PVC recreation | 1-2 min | Monitoring PVCs bind |
| Total | **20-40 min** | |

---

## Rollback (If Needed)

If something goes wrong:

1. Restore from Git:
   ```bash
   git revert HEAD
   git push
   ```

2. Or use simple local-path-provisioner temporarily:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
   ```

---

## Post-Deployment

Once everything is working:

1. **Test volume creation:**
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: test-longhorn
     namespace: default
   spec:
     accessModes: [ReadWriteOnce]
     storageClassName: longhorn
     resources:
       requests:
         storage: 1Gi
   EOF
   
   # Verify it binds
   kubectl get pvc test-longhorn
   
   # Clean up
   kubectl delete pvc test-longhorn
   ```

2. **Document Talos upgrade procedure** in your runbook

3. **Consider setting up backups** (S3) when ready

4. **Monitor Longhorn health** regularly via UI

---

## Support Resources

- **Operations Guide:** LONGHORN_OPERATIONS.md
- **Longhorn Docs:** https://longhorn.io/docs/1.10.0/
- **Talos + Longhorn:** https://longhorn.io/docs/1.10.0/advanced-resources/os-distro-specific/talos-linux-support/

---

## Questions or Issues?

Refer to LONGHORN_OPERATIONS.md for:
- Troubleshooting steps
- Common issues and solutions
- Talos upgrade procedures
- Adding/removing nodes
- Performance tuning

---

**Status:** Ready for deployment! 🚀

**Action Required:** Review changes, commit, and push to trigger deployment.