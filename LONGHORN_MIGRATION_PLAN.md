# Migration Plan: Rook/Ceph → Longhorn

## Current Situation Assessment

**Rook/Ceph Status:**
- ❌ Ceph cluster in `HEALTH_ERR` state (has been for 48 days)
- ❌ Prometheus PVC stuck in `Pending` (48 days)
- ✅ Rook operator and CSI drivers running
- ⚠️ Monitoring stack cannot function without storage

**Good News:**
- You already have udev, lvm2, xfsprogs extensions (from Rook setup)
- No data to migrate (Prometheus PVC never bound)
- Clean slate migration possible

---

## Critical Questions to Answer First

### Q1: Do you have any data in Rook/Ceph that needs to be preserved?
**Answer:** Based on cluster state, likely NO. The PVC is Pending, so no data exists.

**Verification command:**
```bash
kubectl get pv -A
kubectl get pvc -A
```

If ANY PVC shows `Bound` status with `ceph-block` storage class → you have data to consider.

### Q2: What Talos version are you running?
**Check with:**
```bash
talosctl version
# or via Omni dashboard
```

**Why this matters:** Longhorn needs iscsi-tools which is available in Talos 1.6+

### Q3: How many nodes in your cluster and their roles?
**Check with:**
```bash
kubectl get nodes -o wide
```

**Why this matters:** Longhorn replica count should match node availability

### Q4: Do you want to keep Rook namespace/CRDs for potential rollback?
**Recommendation:** Remove cleanly since Rook isn't working anyway.

---

## Prerequisites Check

### ✅ Already Have (from Rook setup)
- [x] udev extension
- [x] lvm2 extension  
- [x] xfsprogs extension

### ⚠️ Still Need for Longhorn
- [ ] iscsi-tools extension
- [ ] util-linux-tools extension
- [ ] Pod Security configuration (privileged namespace)
- [ ] Kubelet extraMounts for /var/lib/longhorn
- [ ] Updated Talos machine config patches

---

## Omni + Talos Upgrade Considerations

### About the `--preserve` Flag

**The Issue:**
When upgrading Talos nodes, the default behavior wipes `/var/lib/longhorn`, destroying all volume replicas.

**With talosctl:**
```bash
talosctl upgrade --nodes <ip> --image <image> --preserve
```

**With Omni:**
Omni handles node upgrades through its UI/API. The `--preserve` flag equivalent depends on how you trigger the upgrade:

#### Option 1: Omni automatically handles preservation
- Omni's cluster management **should** preserve `/var/lib/*` by default
- Verify by checking Omni's upgrade settings in the dashboard

#### Option 2: Manual control via machine config
Add to your Talos machine config:
```yaml
machine:
  install:
    extraOptions:
      - --preserve
```

#### Option 3: Via Omni API/CLI
```bash
omnictl cluster machine update <machine-id> --preserve-data
```

**IMPORTANT:** Test this on ONE worker node first before cluster-wide upgrades!

### Recommendation for Your Setup:
1. Document current Talos version
2. Before ANY Talos upgrades, test on one worker node
3. Verify `/var/lib/longhorn` data persists after upgrade
4. Add warning to your cluster documentation

---

## Migration Plan

### Phase 0: Pre-Migration Preparation (15 minutes)

**0.1 Backup Current Configuration**
```bash
cd /home/tryy3/Projects/omni-k8
git add .
git commit -m "Backup before Longhorn migration"
git push
```

**0.2 Document Current State**
```bash
# Save current state
kubectl get pvc -A > /tmp/pre-migration-pvcs.txt
kubectl get pods -n rook-ceph > /tmp/rook-pods.txt
kubectl get cephcluster -A -o yaml > /tmp/ceph-cluster-backup.yaml
```

**0.3 Verify No Critical Data**
```bash
# Check for any bound PVCs
kubectl get pvc -A | grep -v Pending
```

If anything shows `Bound` → **STOP** and reconsider data migration strategy.

---

### Phase 1: Remove Rook/Ceph (30 minutes)

**1.1 Remove Rook Application Manifests**
```bash
cd /home/tryy3/Projects/omni-k8

# Remove Rook directories
rm -rf apps/rook-ceph/operator
rm -rf apps/rook-ceph/cluster
rm -rf apps/rook-ceph/namespace

# Keep the parent directory if ArgoCD expects it, or remove entirely:
rm -rf apps/rook-ceph
```

**1.2 Remove Rook Extensions Patch (if it exists)**
```bash
# Check if patch exists
ls infra/patches/rook-extensions.yaml

# If exists, remove it
rm infra/patches/rook-extensions.yaml

# Update cluster-template.yaml to remove reference
# (manually edit to remove rook-extensions patch)
```

**1.3 Commit Changes**
```bash
git add .
git commit -m "Remove Rook/Ceph configuration"
git push
```

**1.4 Wait for ArgoCD to Remove Rook**
```bash
# Watch ArgoCD sync
kubectl -n argocd get applications

# Watch Rook namespace
watch kubectl -n rook-ceph get pods
```

**Expected:** Rook namespace should be cleaned up by ArgoCD.

**If ArgoCD doesn't remove it (may be stuck due to finalizers):**
```bash
# Manually clean Rook (only if ArgoCD sync fails)
kubectl delete cephcluster -n rook-ceph rook-ceph
kubectl delete cephblockpool -n rook-ceph --all
kubectl delete deployment -n rook-ceph --all
kubectl delete daemonset -n rook-ceph --all

# Force remove namespace if stuck
kubectl patch cephcluster rook-ceph -n rook-ceph -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl delete namespace rook-ceph --force --grace-period=0
```

**1.5 Remove Rook CRDs (Optional but Recommended)**
```bash
kubectl get crd | grep rook.io | awk '{print $1}' | xargs kubectl delete crd
kubectl get crd | grep ceph.rook.io | awk '{print $1}' | xargs kubectl delete crd
```

---

### Phase 2: Update Talos Configuration for Longhorn (45 minutes)

**2.1 Create Longhorn Extensions Patch**

Create: `infra/patches/longhorn-extensions.yaml`
```yaml
machine:
  install:
    extensions:
      # Existing extensions (keep these)
      - image: ghcr.io/siderolabs/udev:v1.11.1
      - image: ghcr.io/siderolabs/lvm2:v1.11.1
      - image: ghcr.io/siderolabs/xfsprogs:v1.11.1
      # New for Longhorn
      - image: ghcr.io/siderolabs/iscsi-tools:v1.11.1
      - image: ghcr.io/siderolabs/util-linux-tools:v1.11.1
  
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options:
          - bind
          - rshared
          - rw
```

**Note:** Adjust version `v1.11.1` to match your Talos version.

**2.2 Create Pod Security Patch**

Create: `infra/patches/longhorn-pod-security.yaml`
```yaml
cluster:
  inlineManifests:
    - name: longhorn-namespace
      contents: |-
        apiVersion: v1
        kind: Namespace
        metadata:
          name: longhorn-system
          labels:
            pod-security.kubernetes.io/enforce: privileged
            pod-security.kubernetes.io/audit: privileged
            pod-security.kubernetes.io/warn: privileged
```

**2.3 Update Cluster Template**

Edit: `infra/cluster-template.yaml`
```yaml
kind: Cluster
name: talos-default
talos:
  version: v1.11.1
kubernetes:
  version: 1.34.1
features:
  enableWorkloadProxy: true
patches:
  - name: cni
    file: patches/cni.yaml
  - name: longhorn-extensions  # NEW
    file: patches/longhorn-extensions.yaml  # NEW
  - name: longhorn-pod-security  # NEW
    file: patches/longhorn-pod-security.yaml  # NEW
  - name: tailscale
    file: patches/tailscale.yaml
---
kind: ControlPlane
machineClass:
  name: omni-controlplanes
  size: 2
patches:
  - name: cilium
    file: patches/cilium.yaml
  - name: argocd
    file: patches/argocd.yaml
  - name: monitoring
    file: patches/monitoring.yaml
  - name: longhorn-extensions  # NEW - apply to control plane too
    file: patches/longhorn-extensions.yaml  # NEW
---
kind: Workers
name: workers
machineClass:
  name: omni-workers
  size: unlimited
patches:
  - name: longhorn-extensions  # NEW - apply to workers
    file: patches/longhorn-extensions.yaml  # NEW
```

**2.4 Commit Talos Configuration Changes**
```bash
git add .
git commit -m "Add Longhorn Talos configuration"
git push
```

**2.5 Monitor Omni Sync and Node Updates**
```bash
# Watch in Omni dashboard
# Nodes will likely reboot to apply extensions

# Or via CLI
omnictl cluster watch

# Check node readiness
watch kubectl get nodes
```

**Expected timeline:** 10-15 minutes for all nodes to update and become ready.

---

### Phase 3: Install Longhorn (20 minutes)

**3.1 Create Longhorn Application Structure**
```bash
mkdir -p apps/longhorn-system/longhorn
```

**3.2 Create Longhorn Helm Chart**

Create: `apps/longhorn-system/longhorn/Chart.yaml`
```yaml
apiVersion: v2
name: longhorn
version: 1.9.0
dependencies:
  - name: longhorn
    version: 1.9.0
    repository: https://charts.longhorn.io
```

**3.3 Create Longhorn Values**

Create: `apps/longhorn-system/longhorn/values.yaml`
```yaml
longhorn:
  defaultSettings:
    defaultReplicaCount: 3
    defaultDataPath: /var/lib/longhorn
    
  persistence:
    defaultClass: true
    defaultClassReplicaCount: 3
    reclaimPolicy: Delete
  
  ingress:
    enabled: true
    ingressClassName: cilium
    host: longhorn.local
    annotations:
      omni-kube-service-exposer.sidero.dev/port: "50083"
      omni-kube-service-exposer.sidero.dev/label: "Longhorn UI"
```

**3.4 Create Namespace**

Create: `apps/longhorn-system/namespace/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: longhorn-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

**3.5 Commit Longhorn Application**
```bash
git add .
git commit -m "Add Longhorn installation"
git push
```

**3.6 Wait for ArgoCD to Deploy Longhorn**
```bash
# Watch ArgoCD
kubectl -n argocd get applications

# Watch Longhorn deployment
watch kubectl -n longhorn-system get pods

# Check Longhorn manager
kubectl -n longhorn-system logs -l app=longhorn-manager -f
```

**Expected:** 5-10 minutes for all Longhorn components to be ready.

---

### Phase 4: Verification (15 minutes)

**4.1 Check Longhorn System Health**
```bash
# All pods running
kubectl -n longhorn-system get pods

# Check storage class
kubectl get storageclass

# Should show "longhorn" as default (*)
```

**4.2 Verify Nodes and Disks**
```bash
# Access Longhorn UI via Omni proxy (port 50083)
# Or port-forward:
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80

# Check in UI:
# - All nodes are schedulable
# - Disks are detected
# - No errors in dashboard
```

**4.3 Test PVC Creation**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-longhorn-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
EOF

# Watch it bind
kubectl get pvc test-longhorn-pvc -w

# Should go to "Bound" status within 30 seconds
```

**4.4 Check Monitoring PVC**
```bash
# The old pending PVC should be removed by now
kubectl get pvc -n monitoring

# If old PVC still exists and is Pending, delete it:
kubectl delete pvc -n monitoring prometheus-monitoring-kube-prometheus-prometheus-db-prometheus-monitoring-kube-prometheus-prometheus-0

# ArgoCD should recreate it with Longhorn
# Watch for new PVC to bind:
kubectl get pvc -n monitoring -w
```

**4.5 Verify Prometheus is Running**
```bash
# Check Prometheus pod
kubectl -n monitoring get pods | grep prometheus

# Should be in Running state now
```

---

### Phase 5: Update Documentation (10 minutes)

**5.1 Update README**

Add to `README.md` section about storage:
```markdown
## Storage

This cluster uses **Longhorn** for persistent volume management.

- Storage class: `longhorn` (default)
- Replica count: 3
- Data path: `/var/lib/longhorn` on each node
- Dashboard: Accessible via Omni Workload Proxy on port 50083

### Important: Talos Node Upgrades

When upgrading Talos nodes via Omni, ensure data preservation:
- Omni should preserve `/var/lib/longhorn` automatically
- Verify in Omni upgrade settings before proceeding
- Test on one worker node first
- Monitor Longhorn UI during upgrades for replica health

See LONGHORN_OPERATIONS.md for detailed procedures.
```

**5.2 Create Operations Guide**

Create: `LONGHORN_OPERATIONS.md` with:
- How to access Longhorn UI
- How to upgrade Talos nodes safely
- How to add/remove nodes
- How to backup/restore
- Troubleshooting common issues

**5.3 Clean Up Old Documentation**
```bash
# Remove Rook documentation if desired
rm ROOK_*.md
# Or keep for reference
```

---

## Timeline Summary

| Phase | Duration | Can Fail? | Impact if Fails |
|-------|----------|-----------|-----------------|
| 0: Prep | 15 min | No | None |
| 1: Remove Rook | 30 min | Yes | Old broken system remains |
| 2: Update Talos | 45 min | Yes | Nodes may reboot, temporary downtime |
| 3: Install Longhorn | 20 min | Yes | No storage available |
| 4: Verification | 15 min | Yes | Storage may not work |
| 5: Documentation | 10 min | No | Just docs |
| **Total** | **~2.5 hours** | | |

**Best time to perform:** During maintenance window when monitoring downtime is acceptable.

---

## Rollback Plan

### If Longhorn Fails to Install

**Option 1: Fix Longhorn Issues**
- Check pod logs: `kubectl -n longhorn-system logs -l app=longhorn-manager`
- Check node compatibility
- Verify extensions loaded: `talosctl get extensions`

**Option 2: Attempt Rook Reinstall**
1. Remove Longhorn application from Git
2. Re-add Rook applications from Git history
3. Push and wait for ArgoCD sync

**Note:** Since Rook wasn't working before, this may not be viable.

**Option 3: Use Local Path Provisioner (Temporary)**
Deploy simple local-path-provisioner for immediate storage:
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## Risk Assessment

### Low Risk ✅
- Removing Rook (it's already broken)
- Adding Talos extensions (battle-tested)
- Installing Longhorn (widely used on Talos)

### Medium Risk ⚠️
- Node reboots during extension installation (temporary unavailability)
- Pod security configuration (might need adjustment)

### High Risk ❌
- **None identified** - you have no existing data to lose

---

## Post-Migration Checklist

- [ ] Rook namespace deleted
- [ ] Rook CRDs removed
- [ ] Longhorn namespace created with privileged PSP
- [ ] Longhorn extensions loaded on all nodes
- [ ] All Longhorn pods running
- [ ] Storage class "longhorn" is default
- [ ] Test PVC binds successfully
- [ ] Prometheus PVC bound and pod running
- [ ] Grafana accessible and showing metrics
- [ ] Longhorn UI accessible via Omni proxy
- [ ] Documentation updated
- [ ] Talos upgrade procedure documented

---

## Questions Before Starting?

1. **Do you want me to proceed with creating all the files?**
2. **Should I create a rollback script just in case?**
3. **Do you want detailed troubleshooting steps included?**
4. **Any specific Longhorn settings you want configured?**
   - Replica count (default: 3)
   - Backup target (S3, NFS, etc.)
   - Snapshot policies
5. **When do you plan to execute this migration?**

---

## Next Steps

If you approve this plan, I will:
1. Create all the Longhorn configuration files
2. Create the Talos patches for extensions and pod security
3. Update the cluster template
4. Create a LONGHORN_OPERATIONS.md guide
5. Create a simple rollback script
6. Update the README

**Estimated time to create files:** 10 minutes
**Your review time:** 15 minutes
**Migration execution:** 2.5 hours

Let me know when you're ready to proceed! 🚀