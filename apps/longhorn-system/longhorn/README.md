# Longhorn Storage Classes

Quick reference guide for Longhorn storage classes in this cluster.

## Available Storage Classes

### `longhorn-fast` (Write-Intensive)

**Purpose:** High-performance storage for write-intensive workloads

**Configuration:**
- Replicas: 1
- Data Locality: best-effort
- Reclaim Policy: Delete
- Volume Expansion: Enabled

**Use Cases:**
- Databases (PostgreSQL, MySQL, MariaDB)
- Key-value stores (Redis, etcd)
- Time-series databases (InfluxDB, TimescaleDB)
- Message queues (RabbitMQ, Kafka)
- Any application doing frequent writes

**Why single replica?**
- Databases handle consistency at application level
- Eliminates storage replication overhead (3-5x faster writes)
- Data locality keeps I/O on same node (no network latency)
- Use application-level replication (e.g., CNPG for PostgreSQL)

### `longhorn-reliable` (Default, General Purpose)

**Purpose:** Balanced storage for general workloads with redundancy

**Configuration:**
- Replicas: 2
- Data Locality: best-effort
- Reclaim Policy: Retain
- Volume Expansion: Enabled
- Default Class: Yes

**Use Cases:**
- Configuration files and secrets
- Media storage (photos, videos)
- Static assets
- Application logs
- General application data
- Any workload without specific performance requirements

**Why two replicas?**
- Protects against single node/disk failure
- Acceptable write overhead for non-database workloads
- Retain policy prevents accidental data loss

## Quick Decision Guide

```
┌─────────────────────────────────────────┐
│ Is this a database or write-intensive?  │
└────────────┬────────────────────────────┘
             │
      ┌──────┴──────┐
      │             │
     Yes            No
      │             │
      ▼             ▼
 longhorn-fast  longhorn-reliable
 (1 replica)    (2 replicas)
 + app-level    + backups
   replication
```

## Usage Examples

### Database with Application Replication

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-database
spec:
  instances: 3  # CNPG handles replication
  storage:
    storageClass: longhorn-fast  # Fast single replica
    size: 20Gi
```

### General Application

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  # storageClassName not specified = uses default (longhorn-reliable)
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### Explicit Storage Class

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: media-storage
spec:
  storageClassName: longhorn-reliable
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

## Important Notes

### Backups
- Storage replicas are NOT backups
- Configure Longhorn backups to S3/NFS for disaster recovery
- Use application-level backups (e.g., CNPG backups) for databases

### Existing Volumes
- Changing storage class settings only affects NEW volumes
- Existing PVCs retain their original configuration
- To migrate, you must backup → delete → recreate → restore

### Performance Tips
- For databases: Use `longhorn-fast` + application replication
- For critical data: Use 2 replicas + regular backups
- Monitor disk space: Longhorn needs room for snapshots

### Reclaim Policies
- `longhorn-fast` (Delete): Volumes auto-delete with PVC
- `longhorn-reliable` (Retain): Volumes persist after PVC deletion (safer)

## Global Longhorn Settings

Key settings configured in `values.yaml`:
- Replica soft anti-affinity: Spreads replicas across nodes
- Auto-balance: Rebalances replicas when adding nodes
- Node drain policy: Prevents losing last replica during maintenance
- Storage thresholds: Keeps 10% disk space free
- Auto-cleanup: Removes orphaned snapshots and data

