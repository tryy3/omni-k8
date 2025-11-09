# Longhorn Operations Guide

## Quick Reference

**Cluster:** 5 nodes (3 control-plane + 2 workers)
**Replica Count:** 2 (data stored on workers)
**Storage Class:** `longhorn` (default)
**Data Path:** `/var/lib/longhorn` on each node

---

## Accessing Longhorn UI

### Via Omni Workload Proxy
- Port: **50083**
- Label: "Longhorn UI"
- Access through Omni dashboard

### Via Port Forward
```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
# Access: http://localhost:8080
```

---

## Basic Operations

### Check System Status
```bash
# All Longhorn pods
kubectl -n longhorn-system get pods

# Storage class
kubectl get storageclass

# Active volumes
kubectl -n longhorn-system get volumes.longhorn.io
```

### Create Test Volume
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
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
kubectl get pvc test-pvc -w
```

### Delete Test Volume
```bash
kubectl delete pvc test-pvc -n default
```

---

## Monitoring

### Check Node Status
```bash
# Via UI: Nodes tab
# Or via kubectl:
kubectl -n longhorn-system get nodes.longhorn.io
```

### Check Volume Health
```bash
kubectl -n longhorn-system get volumes.longhorn.io
```

### View Longhorn Metrics (if monitoring enabled)
```bash
# Prometheus metrics endpoint
kubectl -n longhorn-system port-forward svc/longhorn-backend 9500:9500
# Access: http://localhost:9500/metrics
```

---

## Talos Node Upgrades - CRITICAL ⚠️

### The Problem
By default, Talos upgrades **wipe `/var/lib/longhorn`**, destroying all volume replicas on that node.

### With Omni
Omni should preserve `/var/lib/*` by default, but **always verify** before upgrading:

1. **Test on ONE worker node first**
2. Check Longhorn UI before upgrade - note replica locations
3. Perform upgrade on single worker
4. After upgrade, verify in Longhorn UI that replicas are intact
5. If successful, proceed with other nodes

### Monitoring During Upgrade
```bash
# Watch Longhorn volumes during upgrade
watch kubectl -n longhorn-system get volumes.longhorn.io

# Watch node status
watch kubectl get nodes
```

### If Data Gets Wiped
Longhorn will automatically rebuild replicas from remaining healthy replicas (if replica count > 1).

**Recovery steps:**
1. Wait for node to come back up
2. In Longhorn UI, go to Node → select upgraded node
3. Edit Node and Disks → Delete the disk
4. Add disk back with path `/var/lib/longhorn/`
5. Longhorn will resync replicas

---

## Adding/Removing Nodes

### Adding a Worker Node
1. Add node to cluster via Omni
2. Wait for node to be Ready
3. Longhorn will automatically detect and use it
4. Check in Longhorn UI: Node tab

### Removing a Worker Node
1. In Longhorn UI: Node tab
2. Select node → Edit Node and Disks
3. Set Scheduling to **Disable**
4. Wait for replicas to migrate (can take time)
5. Once replicas migrated, remove node from cluster

---

## Storage Management

### Check Available Space
```bash
# In Longhorn UI: Dashboard
# Or via CLI:
kubectl -n longhorn-system get nodes.longhorn.io -o json | jq '.items[] | {name: .metadata.name, allocatable: .status.diskStatus}'
```

### Adjust Replica Count (if needed)
```bash
# Edit the default settings
kubectl -n longhorn-system edit settings.longhorn.io default-replica-count

# Or in UI: Settings → General → Default Replica Count
```

### Volume Expansion
Longhorn supports volume expansion. To expand a PVC:
```bash
kubectl patch pvc <pvc-name> -n <namespace> -p '{"spec":{"resources":{"requests":{"storage":"<new-size>"}}}}'
```

---

## Troubleshooting

### Pods Stuck in Pending
```bash
# Check PVC status
kubectl get pvc -A

# Check Longhorn manager logs
kubectl -n longhorn-system logs -l app=longhorn-manager --tail=100

# Check events
kubectl get events -n longhorn-system --sort-by='.lastTimestamp'
```

### Volume Won't Attach
```bash
# Check volume status in UI
# Or via CLI:
kubectl -n longhorn-system describe volume.longhorn.io <volume-name>

# Check if iscsi is working
kubectl -n longhorn-system exec -it <longhorn-manager-pod> -- iscsiadm -m session
```

### Replica Stuck in Error
In Longhorn UI:
1. Go to Volume
2. Select volume with issue
3. Click on replica → Actions → Delete
4. Longhorn will recreate replica automatically

### Node Shows Unschedulable
```bash
# Check node in UI: Node tab
# Common causes:
# - Disk full
# - Disk not detected
# - Node cordoned

# Fix: Check disk space
df -h /var/lib/longhorn
```

---

## Backup Configuration (Future)

When you're ready to configure backups to S3:

1. In Longhorn UI: Settings → General
2. Set Backup Target:
   - S3: `s3://<bucket-name>@<region>/`
   - Credentials via secret or IAM role
3. Configure backup schedule per volume

Documentation: https://longhorn.io/docs/1.10.0/snapshots-and-backups/backup-and-restore/set-backup-target/

---

## Useful Commands

```bash
# Check Longhorn version
kubectl -n longhorn-system get deployment longhorn-ui -o jsonpath='{.spec.template.spec.containers[0].image}'

# Restart Longhorn manager (if needed)
kubectl -n longhorn-system rollout restart deployment longhorn-driver-deployer

# Get all Longhorn CRDs
kubectl get crd | grep longhorn.io

# Check iSCSI status on a node
kubectl -n longhorn-system exec -it <any-longhorn-pod> -- iscsiadm -m session

# Force delete stuck volume
kubectl -n longhorn-system patch volume.longhorn.io <volume-name> -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl -n longhorn-system delete volume.longhorn.io <volume-name>
```

---

## Performance Tips

1. **For better performance:** Use dedicated disks (not OS disk)
2. **For production:** Increase replica count to 3 when you have 3+ workers
3. **For databases:** Consider using `directio` setting in storage class
4. **Monitor:** Keep an eye on disk I/O and network bandwidth

---

## Documentation

- Longhorn Official Docs: https://longhorn.io/docs/1.10.0/
- Talos + Longhorn: https://longhorn.io/docs/1.10.0/advanced-resources/os-distro-specific/talos-linux-support/
- Storage Class Params: https://longhorn.io/docs/1.10.0/references/storage-class-parameters/

---

## Support

For issues:
1. Check Longhorn UI for errors
2. Check pod logs: `kubectl -n longhorn-system logs -l app=longhorn-manager`
3. Longhorn GitHub: https://github.com/longhorn/longhorn/issues
4. Longhorn Slack: https://slack.cncf.io/ (#longhorn channel)