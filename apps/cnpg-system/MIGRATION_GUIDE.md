# PostgreSQL Migration Guide

This guide covers migrating existing PostgreSQL databases to CloudNativePG clusters.

## Migration Strategy

**Recommended Approach**: Migrate to the same PostgreSQL version first, then upgrade separately.

✅ **Benefits:**
- Lower risk of issues
- Easier troubleshooting
- Clear separation of migration vs upgrade problems
- Can roll back upgrade independently

## Overview

```
┌─────────────────┐
│ External PG 16  │
│ (Old Database)  │
└────────┬────────┘
         │
         │ 1. Export (pg_dump)
         ▼
┌─────────────────┐
│   Dump File     │
└────────┬────────┘
         │
         │ 2. Import (pg_restore)
         ▼
┌─────────────────┐
│  CNPG PG 16     │
│ (New Cluster)   │
└────────┬────────┘
         │
         │ 3. Later: Upgrade
         ▼
┌─────────────────┐
│  CNPG PG 17     │
│ (Upgraded)      │
└─────────────────┘
```

## Pre-Migration Checklist

Before starting the migration:

### 1. Document Current Setup

```bash
# PostgreSQL version
psql -h old-host -U postgres -c "SELECT version();"

# List all databases
psql -h old-host -U postgres -c "\l"

# List extensions per database
psql -h old-host -U postgres -d yourdb -c "\dx"

# Database size
psql -h old-host -U postgres -d yourdb -c "SELECT pg_size_pretty(pg_database_size('yourdb'));"

# Current configuration
psql -h old-host -U postgres -c "SHOW ALL;" > old-config.txt
```

### 2. Test Export on Non-Production Data

Always test the export/import process on a copy or subset of data first.

### 3. Plan Maintenance Window

Estimate downtime:
- Small database (<1 GB): 5-15 minutes
- Medium database (1-10 GB): 15-60 minutes  
- Large database (10-100 GB): 1-4 hours
- Very large database (>100 GB): Consider alternatives (streaming replication)

### 4. Backup Source Database

```bash
# Create a full backup before migration
pg_dump -h old-host -U postgres -Fc yourdb > backup-before-migration.dump
```

### 5. Verify CloudNativePG Cluster is Ready

```bash
# Check operator is running
kubectl get pods -n cnpg-system

# Verify cluster is healthy
kubectl get cluster myapp-pg -n myapp
kubectl cnpg status myapp-pg -n myapp
```

## Same-Version Migration Steps

### Step 1: Prepare CloudNativePG Cluster

Create a cluster with the **same major version** as your source database.

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: myapp-pg
  namespace: myapp
spec:
  instances: 2
  
  # IMPORTANT: Match source PostgreSQL version
  imageName: ghcr.io/cloudnative-pg/postgresql:16.6  # Same as source
  
  bootstrap:
    initdb:
      database: app  # Your database name
      owner: app
      # Install extensions if needed
      postInitSQL:
        - CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
        - CREATE EXTENSION IF NOT EXISTS pgcrypto;
  
  storage:
    size: 20Gi  # Size based on source database + growth
    storageClass: longhorn
  
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 2Gi
```

Apply and wait for cluster to be ready:

```bash
kubectl apply -f cluster.yaml
kubectl wait --for=condition=Ready cluster/myapp-pg -n myapp --timeout=5m
```

### Step 2: Export Source Database

Use `pg_dump` with custom format for flexibility:

```bash
# Export specific database
pg_dump -h old-host -U postgres -Fc -v yourdb > yourdb.dump

# Or export all databases (including roles)
pg_dumpall -h old-host -U postgres > all-databases.sql
```

**Options:**
- `-Fc` - Custom format (recommended, allows selective restore)
- `-Fd` - Directory format (parallel dump for large databases)
- `-Fp` - Plain SQL (human-readable, less flexible)
- `-v` - Verbose output
- `-j 4` - Use 4 parallel jobs (only with `-Fd`)

**For very large databases:**

```bash
# Parallel export (faster)
pg_dump -h old-host -U postgres -Fd -j 4 -v yourdb -f yourdb.dir/
```

### Step 3: Get CNPG Cluster Connection Details

```bash
# Get the app password
kubectl get secret myapp-pg-app -n myapp -o jsonpath='{.data.password}' | base64 -d

# Get service name
kubectl get svc -n myapp | grep myapp-pg-rw
```

### Step 4: Import to CloudNativePG

#### Option A: From Local Machine

Port-forward to the cluster:

```bash
# Terminal 1: Port forward
kubectl port-forward -n myapp svc/myapp-pg-rw 5432:5432
```

```bash
# Terminal 2: Restore
pg_restore -h localhost -p 5432 -U app -d app -v --no-owner --no-acl yourdb.dump
```

#### Option B: From Within Cluster (Faster)

Copy dump file into a pod and restore:

```bash
# Create a temporary pod with PostgreSQL client
kubectl run -n myapp pgclient --image=postgres:16 --rm -it --restart=Never -- bash

# Inside the pod, install and use pg_restore
# (Copy dump file via kubectl cp or download from S3/NFS)
```

#### Option C: Using kubectl cp

```bash
# Copy dump to pod
kubectl cp yourdb.dump myapp/myapp-pg-1:/tmp/yourdb.dump

# Exec into pod and restore
kubectl exec -it -n myapp myapp-pg-1 -- \
  pg_restore -U postgres -d app -v --no-owner --no-acl /tmp/yourdb.dump
```

**Common pg_restore options:**
- `--no-owner` - Don't restore ownership (use cluster's default user)
- `--no-acl` - Don't restore access privileges
- `--clean` - Drop existing objects before recreating (careful!)
- `-j 4` - Use 4 parallel jobs (only with `-Fc` or `-Fd` format)
- `--if-exists` - Use IF EXISTS when dropping objects

### Step 5: Verify Data Integrity

```bash
# Connect to new cluster
kubectl port-forward -n myapp svc/myapp-pg-rw 5432:5432

# In another terminal
psql -h localhost -p 5432 -U app -d app
```

**Check:**
```sql
-- Row counts match
SELECT schemaname, tablename, n_live_tup 
FROM pg_stat_user_tables 
ORDER BY n_live_tup DESC;

-- Extensions installed
\dx

-- Sequences are correct (important!)
SELECT sequence_name, last_value FROM information_schema.sequences;

-- Test critical queries
SELECT COUNT(*) FROM important_table;
```

### Step 6: Update Application Configuration

Update your application to use the new database:

```yaml
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: myapp-pg-app
        key: uri
```

Or individual variables:

```yaml
env:
  - name: DB_HOST
    value: myapp-pg-rw.myapp.svc.cluster.local
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: myapp-pg-app
        key: password
```

### Step 7: Test Application

Deploy application and verify:
- Database connections work
- All features function correctly
- Performance is acceptable
- No errors in application logs

### Step 8: Monitor and Validate

Monitor for 24-48 hours:

```bash
# Check cluster health
kubectl cnpg status myapp-pg -n myapp

# Watch logs
kubectl logs -n myapp myapp-pg-1 -f

# Check Grafana dashboard
# https://monitoring.tryy3.dev
```

### Step 9: Decommission Old Database

Once confident the migration was successful:
1. Keep old database running in read-only mode for a few days
2. Take a final backup
3. Shut down old database
4. Archive final backup for retention period

## Post-Migration: Version Upgrade

After migration is stable and verified, you can upgrade PostgreSQL versions.

### Automatic Upgrade with CloudNativePG

CloudNativePG handles major version upgrades automatically:

```yaml
# Current cluster
spec:
  imageName: ghcr.io/cloudnative-pg/postgresql:16.6

# Change to new version
spec:
  imageName: ghcr.io/cloudnative-pg/postgresql:17.2
```

Apply the change:

```bash
kubectl apply -f cluster.yaml
```

**What happens:**
1. Operator detects major version change
2. Creates new primary pod with new version
3. Runs `pg_upgrade` automatically
4. Performs switchover to new primary
5. Rebuilds replicas with new version
6. Cleans up old pods

**Downtime:** Brief (seconds to minutes) during primary switchover.

### Monitor the Upgrade

```bash
# Watch the upgrade process
kubectl get pods -n myapp -w

# Check operator logs
kubectl logs -n cnpg-system deployment/cloudnative-pg-controller-manager -f

# Verify cluster status
kubectl cnpg status myapp-pg -n myapp
```

### Post-Upgrade Tasks

```sql
-- Connect to upgraded cluster
psql -h localhost -p 5432 -U app -d app

-- Rebuild statistics (important!)
ANALYZE VERBOSE;

-- Check for deprecated features
-- See PostgreSQL release notes for version-specific checks

-- Update extensions if needed
ALTER EXTENSION pg_stat_statements UPDATE;
```

## Version Compatibility Notes

### PostgreSQL 15 → 16

**Breaking Changes:**
- `pg_walinspect` requires superuser by default (was granted to `pg_monitor`)
- `COPY FROM` with `DEFAULT` keyword behavior changed
- Some deprecated server parameters removed

**Extensions:**
- Verify extension compatibility
- Most extensions work without changes

**Upgrade considerations:**
- Test query plans (optimizer improvements may change plans)
- Review application for deprecated SQL syntax

### PostgreSQL 16 → 17

**Breaking Changes:**
- `psql` meta-command changes
- Some replication protocol changes
- Permission changes for certain system views

**New Features:**
- Improved incremental backup
- Better VACUUM performance
- JSON improvements

**Upgrade considerations:**
- Test connection libraries (ensure pg protocol compatibility)
- Review query performance (optimizer changes)

### PostgreSQL 15 → 17 (Skipping 16)

**Allowed** but more risky:
- More breaking changes to consider
- Larger optimizer differences
- More thorough testing required

**Recommendation:** Upgrade 15 → 16, then 16 → 17 if possible.

## Troubleshooting

### Export Fails: "Permission Denied"

```bash
# Use superuser or database owner
pg_dump -h old-host -U postgres yourdb > yourdb.sql

# Or grant necessary permissions
GRANT SELECT ON ALL TABLES IN SCHEMA public TO backup_user;
```

### Import Fails: "Role does not exist"

Use `--no-owner` flag:

```bash
pg_restore --no-owner --no-acl -d app yourdb.dump
```

### Import Fails: "Extension not available"

Install extensions in target cluster first:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS postgis;  -- if needed
```

Or add to cluster bootstrap:

```yaml
bootstrap:
  initdb:
    postInitSQL:
      - CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### Sequence Values Wrong After Import

Reset sequences:

```sql
-- For all sequences in database
SELECT 'SELECT SETVAL(' ||
       quote_literal(quote_ident(sequence_namespace::regnamespace::text) || '.' ||
                     quote_ident(sequence_name)) ||
       ', COALESCE(MAX(' || quote_ident(column_name) || '), 1)) FROM ' ||
       quote_ident(table_schema) || '.' || quote_ident(table_name) || ';'
FROM information_schema.sequences;

-- Execute output queries
```

### Performance Degradation After Migration

```sql
-- Rebuild statistics
ANALYZE VERBOSE;

-- Rebuild indexes
REINDEX DATABASE app;

-- Check query plans
EXPLAIN ANALYZE SELECT ...;
```

### Upgrade Fails or Stalls

```bash
# Check operator logs
kubectl logs -n cnpg-system deployment/cloudnative-pg-controller-manager

# Check pod events
kubectl describe pod myapp-pg-1 -n myapp

# Check cluster status
kubectl cnpg status myapp-pg -n myapp
```

If upgrade fails, the operator will keep the cluster running on the old version. Investigate the issue and retry.

## Rollback Procedures

### Before Cutover

Simply don't point applications to the new cluster. Keep using the old database.

### After Cutover (Data Written to New Cluster)

Rollback is complex because data has diverged. Options:

1. **Keep old database running** until fully confident (recommended)
2. **Export new data back to old** (if new writes are minimal)
3. **Accept data loss** and restore from old database backup

**Prevention:** Always keep old database running for at least 48 hours after cutover.

## Large Database Considerations

For databases >100 GB, consider:

### Option 1: Streaming Replication

Set up PostgreSQL streaming replication to CNPG:

1. Configure old database as replica source
2. Bootstrap CNPG cluster from streaming replication
3. Promote CNPG cluster when caught up
4. Minimal downtime (seconds)

See: https://cloudnative-pg.io/documentation/current/bootstrap/

### Option 2: Parallel pg_dump/restore

```bash
# Export with parallel jobs
pg_dump -Fd -j 8 -f dumpdir/ yourdb

# Import with parallel jobs
pg_restore -j 8 -d app dumpdir/
```

### Option 3: Physical Copy

For very large databases (TB+), consider:
- Backup/restore via Barman
- Filesystem-level copy (requires downtime)
- Cloud provider database migration tools

## Checklist Summary

### Pre-Migration
- [ ] Document source database (version, size, extensions)
- [ ] Test export on non-prod data
- [ ] Plan maintenance window
- [ ] Backup source database
- [ ] Create CNPG cluster (same version)
- [ ] Verify CNPG cluster is healthy

### Migration
- [ ] Export source database (`pg_dump`)
- [ ] Import to CNPG cluster (`pg_restore`)
- [ ] Verify data integrity (row counts, sequences)
- [ ] Test critical queries
- [ ] Update application configuration
- [ ] Deploy and test application
- [ ] Monitor for 24-48 hours

### Post-Migration
- [ ] Keep old database running (read-only)
- [ ] Monitor new cluster in Grafana
- [ ] Take final backup of old database
- [ ] Plan version upgrade (separate from migration)
- [ ] Document migration experience
- [ ] Decommission old database

### Optional: Version Upgrade
- [ ] Change `imageName` in cluster spec
- [ ] Apply changes
- [ ] Monitor upgrade process
- [ ] Run `ANALYZE` on all tables
- [ ] Verify application compatibility
- [ ] Check query performance

## Best Practices

1. ✅ **Always migrate to same version first** - Separate migration from upgrade
2. ✅ **Test on non-production first** - Practice the migration process
3. ✅ **Keep old database running** - Safety net for rollback
4. ✅ **Verify data integrity** - Row counts, sequences, constraints
5. ✅ **Monitor closely** - Watch for issues in first 48 hours
6. ✅ **Document everything** - Capture actual times, issues, solutions
7. ✅ **Plan for rollback** - Know how to revert if needed
8. ✅ **Upgrade later** - After migration is proven stable

## Resources

- **pg_dump documentation**: https://www.postgresql.org/docs/current/app-pgdump.html
- **pg_restore documentation**: https://www.postgresql.org/docs/current/app-pgrestore.html
- **CloudNativePG Bootstrap**: https://cloudnative-pg.io/documentation/current/bootstrap/
- **PostgreSQL Upgrade Notes**: https://www.postgresql.org/docs/release/
- **Major Version Upgrade**: https://cloudnative-pg.io/documentation/current/rolling_update/

