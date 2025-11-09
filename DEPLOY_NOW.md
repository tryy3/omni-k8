# 🚀 Deploy Longhorn NOW

## Quick Deploy
```bash
cd /home/tryy3/Projects/omni-k8
git add .
git commit -m "Replace Rook/Ceph with Longhorn v1.10.0"
git push
```

## Watch Progress
```bash
# ArgoCD sync
kubectl -n argocd get applications -w

# Longhorn deployment (wait for this)
watch kubectl -n longhorn-system get pods

# Verify storage class
kubectl get storageclass

# Check monitoring
kubectl get pvc -n monitoring
kubectl -n monitoring get pods | grep prometheus
```

## Access Longhorn UI
- **Omni Proxy:** Port 50083
- **Port Forward:** `kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80`

## If Rook Gets Stuck
```bash
# Force remove
kubectl patch cephcluster rook-ceph -n rook-ceph -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl delete namespace rook-ceph --force --grace-period=0
kubectl get crd | grep rook.io | awk '{print $1}' | xargs kubectl delete crd
```

## If Old PVC Stays Pending
```bash
# Delete it (ArgoCD will recreate with Longhorn)
kubectl delete pvc prometheus-monitoring-kube-prometheus-prometheus-db-prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring
```

## Success Checklist
- [ ] Rook namespace gone: `kubectl get ns | grep rook`
- [ ] Longhorn pods running: `kubectl -n longhorn-system get pods`
- [ ] Storage class exists: `kubectl get storageclass longhorn`
- [ ] Prometheus PVC bound: `kubectl get pvc -n monitoring`
- [ ] Prometheus pod running: `kubectl -n monitoring get pods | grep prometheus`

## Timeline
- Git push: 1 min
- Rook removal: 5-10 min
- Omni applies patch: 5-10 min (nodes may restart)
- Longhorn deploys: 5-10 min
- **Total: 20-40 min**

## Full Docs
- **DEPLOYMENT_SUMMARY.md** - Complete steps
- **LONGHORN_OPERATIONS.md** - Operations guide