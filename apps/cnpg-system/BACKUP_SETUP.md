# CloudNativePG Backup Setup Guide

This guide covers setting up S3-compatible backup and recovery for CloudNativePG clusters.

## Overview

CloudNativePG supports continuous backup and Point-In-Time Recovery (PITR) using Barman Cloud. Backups include:
- **Base backups** - Full database snapshots (scheduled)
- **WAL archiving** - Continuous incremental backups
- **PITR** - Restore to any point in time

## Prerequisites

Before configuring backups, you need:
1. **S3-compatible storage** (MinIO, Backblaze B2, Wasabi, AWS S3, etc.)
2. **Access credentials** (access key ID and secret access key)
3. **Bucket and path** for storing backups

## S3-Compatible Storage Options

### Option 1: Self-Hosted MinIO

**Pros:**
- ✅ Complete control
- ✅ Free and open source
- ✅ Runs in your cluster or on NAS
- ✅ S3-compatible API

**Cons:**
- ❌ You manage storage and reliability
- ❌ Requires additional resources

**Setup Example:**
```bash
# Deploy MinIO via Helm (example)
helm repo add minio https://charts.min.io/
helm install minio minio/minio \
  --namespace minio-system \
  --create-namespace \
  --set rootUser=admin \
  --set rootPassword=changeme123 \
  --set persistence.size=100Gi
```

Create bucket: `cnpg-backups`

### Option 2: Backblaze B2

**Pros:**
- ✅ Affordable ($6/TB/month)
- ✅ No egress fees for first 3x stored data
- ✅ S3-compatible API
- ✅ Managed service

**Cons:**
- ❌ External dependency
- ❌ Requires internet connectivity

**Setup:**
1. Create account at backblaze.com
2. Create bucket (e.g., `my-homelab-postgres-backups`)
3. Generate application key (access key + secret)
4. Note endpoint: `s3.us-west-004.backblazeb2.com` (varies by region)

### Option 3: Wasabi

**Pros:**
- ✅ Flat pricing ($6.99/TB/month)
- ✅ No egress fees
- ✅ S3-compatible
- ✅ Fast performance

**Cons:**
- ❌ 90-day minimum storage commitment
- ❌ External dependency

### Option 4: AWS S3

**Pros:**
- ✅ Industry standard
- ✅ Highly reliable
- ✅ Global availability

**Cons:**
- ❌ More expensive
- ❌ Egress fees can add up

## Backup Configuration

### Step 1: Create S3 Credentials Secret

Store your S3 credentials as a Kubernetes secret:

```bash
kubectl create secret generic s3-backup-credentials \
  --namespace=<namespace> \
  --from-literal=ACCESS_KEY_ID='your-access-key' \
  --from-literal=SECRET_ACCESS_KEY='your-secret-key'
```

**With ExternalSecrets (if using Bao/Vault):**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: s3-backup-credentials
  namespace: <namespace>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: s3-backup-credentials
    creationPolicy: Owner
  data:
    - secretKey: ACCESS_KEY_ID
      remoteRef:
        key: postgres/s3-backup
        property: access_key_id
    - secretKey: SECRET_ACCESS_KEY
      remoteRef:
        key: postgres/s3-backup
        property: secret_access_key
```

### Step 2: Configure Cluster with Backup

Add backup configuration to your cluster manifest:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: myapp-pg
  namespace: myapp
spec:
  instances: 2
  imageName: ghcr.io/cloudnative-pg/postgresql:16.6
  
  storage:
    size: 10Gi
    storageClass: longhorn

  # Backup configuration
  backup:
    # Barman Object Store (S3-compatible)
    barmanObjectStore:
      # S3 destination path
      destinationPath: s3://my-bucket/postgres/myapp-pg
      
      # S3 endpoint (omit for AWS S3, required for others)
      endpointURL: https://s3.us-west-004.backblazeb2.com  # Example: Backblaze B2
      
      # S3 credentials reference
      s3Credentials:
        accessKeyId:
          name: s3-backup-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: s3-backup-credentials
          key: SECRET_ACCESS_KEY
      
      # WAL (Write-Ahead Log) archiving
      wal:
        compression: gzip
        encryption: AES256
        maxParallel: 2
      
      # Data backup settings
      data:
        compression: gzip
        encryption: AES256
        jobs: 2  # Parallel compression jobs
        immediateCheckpoint: true
    
    # Retention policy
    retentionPolicy: "30d"  # Keep backups for 30 days
    
    # Additional barman options
    barmanObjectStore:
      serverName: myapp-pg  # Logical server name in backups
      tags:
        environment: homelab
        application: myapp

  # Optional: Configure WAL archiving
  postgresql:
    parameters:
      # WAL settings for better PITR
      wal_level: replica
      archive_timeout: "5min"
```

### Step 3: Create Scheduled Backup

Set up automated base backups:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: myapp-pg-daily-backup
  namespace: myapp
spec:
  # Cron schedule (daily at 2 AM)
  schedule: "0 2 * * *"
  
  # Backup method
  backupOwnerReference: self
  cluster:
    name: myapp-pg
  
  # Use Barman object store
  method: barmanObjectStore
  
  # Take immediate backup on creation
  immediate: true
  
  # Optional: Suspend backups temporarily
  suspend: false
```

### Step 4: Verify Backup Configuration

```bash
# Check backup status
kubectl get backup -n myapp

# View backup details
kubectl describe backup myapp-pg-<timestamp> -n myapp

# Check WAL archiving status
kubectl cnpg status myapp-pg -n myapp

# View backup list via Barman
kubectl cnpg backup list myapp-pg -n myapp
```

## Recovery Procedures

### Restore to Latest Backup

Create a new cluster from the most recent backup:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: myapp-pg-restored
  namespace: myapp
spec:
  instances: 2
  imageName: ghcr.io/cloudnative-pg/postgresql:16.6
  
  # Bootstrap from backup
  bootstrap:
    recovery:
      source: myapp-pg-backup
  
  # External cluster reference for recovery
  externalClusters:
    - name: myapp-pg-backup
      barmanObjectStore:
        destinationPath: s3://my-bucket/postgres/myapp-pg
        endpointURL: https://s3.us-west-004.backblazeb2.com
        s3Credentials:
          accessKeyId:
            name: s3-backup-credentials
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: s3-backup-credentials
            key: SECRET_ACCESS_KEY
  
  storage:
    size: 10Gi
    storageClass: longhorn
```

### Point-In-Time Recovery (PITR)

Restore to a specific timestamp:

```yaml
bootstrap:
  recovery:
    source: myapp-pg-backup
    recoveryTarget:
      targetTime: "2025-11-10 14:30:00.00000+00"
      # Or use targetLSN, targetXID, targetName
```

### Recovery Options

| Parameter | Description | Example |
|-----------|-------------|---------|
| `targetTime` | Restore to specific timestamp | `"2025-11-10 14:30:00+00"` |
| `targetLSN` | Restore to Log Sequence Number | `"0/3000000"` |
| `targetXID` | Restore to transaction ID | `"12345"` |
| `targetName` | Restore to named restore point | `"before-migration"` |

## Backup Strategies

### Strategy 1: Daily Full Backups (Recommended for Homelab)

```yaml
schedule: "0 2 * * *"  # 2 AM daily
retentionPolicy: "30d"
```

**Pros:**
- Simple and predictable
- Low overhead for small databases
- 30-day retention provides good safety net

**Cons:**
- Uses more storage than incremental

### Strategy 2: Weekly Full + Daily Incremental

```yaml
# Full backup weekly
schedule: "0 2 * * 0"  # Sunday 2 AM
retentionPolicy: "90d"

# Incremental handled by WAL archiving (automatic)
```

**Pros:**
- More efficient storage usage
- Longer retention possible
- PITR available via WAL

**Cons:**
- Longer recovery time (replay WAL)

### Strategy 3: Continuous Backup Only

Rely on WAL archiving without scheduled base backups.

**Pros:**
- Minimal storage overhead
- Continuous protection

**Cons:**
- Longer recovery times
- First recovery requires manual base backup

**Recommendation**: Start with Strategy 1 (daily full backups) for simplicity.

## Monitoring Backups

### Check Backup Status in Grafana

The CloudNativePG Grafana dashboard includes:
- Last successful backup timestamp
- Backup failures
- WAL archiving status
- Backup size and duration

### Prometheus Alerts (Optional)

Create PrometheusRule for backup monitoring:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cnpg-backup-alerts
  namespace: cnpg-system
spec:
  groups:
    - name: postgresql-backup
      interval: 60s
      rules:
        - alert: PostgreSQLBackupFailed
          expr: cnpg_pg_backup_last_failed > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "PostgreSQL backup failed for {{ $labels.cluster }}"
            description: "Cluster {{ $labels.cluster }} backup has failed."
        
        - alert: PostgreSQLBackupTooOld
          expr: time() - cnpg_pg_backup_last_time > 86400 * 2
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "PostgreSQL backup is too old for {{ $labels.cluster }}"
            description: "Last backup for {{ $labels.cluster }} was over 2 days ago."
```

## Estimated Storage Requirements

| Database Size | WAL/Day | Daily Backup | 30-Day Storage |
|---------------|---------|--------------|----------------|
| 1 GB | ~100 MB | ~500 MB | ~15 GB |
| 10 GB | ~500 MB | ~3 GB | ~90 GB |
| 100 GB | ~5 GB | ~30 GB | ~900 GB |

*Estimates assume 50% compression and moderate write activity*

## Troubleshooting

### Backup Fails with "Access Denied"

Check:
- S3 credentials are correct
- Bucket permissions allow read/write
- Endpoint URL is correct

```bash
# Test S3 access manually
kubectl run -it --rm s3-test --image=amazon/aws-cli --restart=Never -- \
  s3 ls s3://my-bucket/postgres/ \
  --endpoint-url=https://s3.endpoint.com
```

### WAL Archiving Lag

```bash
# Check WAL status
kubectl cnpg status myapp-pg -n myapp

# View operator logs
kubectl logs -n cnpg-system deployment/cloudnative-pg-controller-manager
```

### High Backup Storage Usage

- Review retention policy (reduce from 30d to 7d)
- Enable compression (should be default)
- Consider incremental backup strategy
- Clean up old backups manually if needed

## Best Practices

1. ✅ **Test restores regularly** - Verify backups actually work
2. ✅ **Monitor backup health** - Use Grafana dashboards and alerts
3. ✅ **Secure credentials** - Use ExternalSecrets or RBAC
4. ✅ **Geographic redundancy** - Consider cross-region S3 replication for critical data
5. ✅ **Document recovery procedures** - Practice disaster recovery
6. ✅ **Retain backups offsite** - S3 provides off-cluster storage
7. ✅ **Encrypt backups** - Enable AES256 encryption

## Next Steps

1. Choose an S3-compatible storage provider
2. Create bucket and credentials
3. Test backup configuration with a non-production cluster
4. Set up monitoring and alerts
5. Document and practice recovery procedures
6. Roll out to production clusters

## Resources

- **Barman Documentation**: https://pgbarman.org/
- **CloudNativePG Backup Guide**: https://cloudnative-pg.io/documentation/current/backup_recovery/
- **S3 API Compatibility**: Most providers support AWS S3 API standards

