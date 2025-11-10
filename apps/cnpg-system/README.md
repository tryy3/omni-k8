# CloudNativePG Setup

This directory contains the CloudNativePG operator and configuration for managing PostgreSQL clusters in the homelab Kubernetes cluster.

## Overview

CloudNativePG provides cloud-native PostgreSQL management with:
- ✅ Automatic failover and high availability
- ✅ Integrated backup and recovery
- ✅ Rolling updates and major version upgrades
- ✅ Built-in monitoring and metrics
- ✅ Declarative configuration via Kubernetes CRDs

## Architecture

- **Operator**: Manages PostgreSQL clusters across all namespaces
- **Monitoring**: Prometheus metrics + Grafana dashboards
- **Per-App Clusters**: Isolated database per application for security and flexibility
- **Naming Convention**: `<app-name>-pg` (e.g., `nextcloud-pg`, `gitea-pg`)

## Quick Start

### 1. Create a New PostgreSQL Cluster

Copy the template to your app directory:

```bash
mkdir -p apps/<namespace>/<app-name>-pg
cp apps/cnpg-system/templates/cluster-template.yaml apps/<namespace>/<app-name>-pg/cluster.yaml
```

Edit the file and replace:
- `<APP_NAME>` with your application name
- `<NAMESPACE>` with your target namespace
- Adjust storage size, resources, and PostgreSQL version as needed

Commit and push - ArgoCD will automatically deploy it.

### 2. Connecting Your Application

CloudNativePG automatically creates secrets and services for each cluster.

#### Auto-Generated Secrets

For a cluster named `myapp-pg`, these secrets are created:

| Secret Name | Purpose | Contents |
|-------------|---------|----------|
| `myapp-pg-superuser` | PostgreSQL superuser (`postgres`) | `username`, `password` |
| `myapp-pg-app` | Application user (`app`) | `username`, `password`, `dbname`, `host`, `port`, `uri` |

#### Auto-Generated Services

| Service Name | Purpose | Connects To |
|--------------|---------|-------------|
| `myapp-pg-rw` | **Read-Write** | Primary only (INSERT/UPDATE/DELETE) |
| `myapp-pg-ro` | **Read-Only** | Replicas only (SELECT queries) |
| `myapp-pg-r` | **Read** | Any instance (flexible reads) |

**Most applications should use `myapp-pg-rw`** for read-write operations.

#### Connection Examples

##### Option 1: Use Full Connection URI

```yaml
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: myapp-pg-app
        key: uri
```

This provides: `postgresql://app:password@myapp-pg-rw:5432/app`

##### Option 2: Use Individual Credentials

```yaml
env:
  - name: DB_HOST
    valueFrom:
      secretKeyRef:
        name: myapp-pg-app
        key: host  # Returns: myapp-pg-rw
  - name: DB_PORT
    valueFrom:
      secretKeyRef:
        name: myapp-pg-app
        key: port  # Returns: 5432
  - name: DB_NAME
    valueFrom:
      secretKeyRef:
        name: myapp-pg-app
        key: dbname  # Returns: app
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: myapp-pg-app
        key: username  # Returns: app
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: myapp-pg-app
        key: password  # Returns: random password
```

##### Option 3: Read-Only Connection (Analytics/Reporting)

For read-heavy workloads that don't need writes:

```yaml
env:
  - name: DATABASE_URL
    value: postgresql://app:$(DB_PASSWORD)@myapp-pg-ro:5432/app
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: myapp-pg-app
        key: password
```

### 3. Verify Cluster Health

```bash
# Check cluster status
kubectl get cluster myapp-pg -n <namespace>

# Check pods
kubectl get pods -n <namespace> -l cnpg.io/cluster=myapp-pg

# Check primary/replica status
kubectl cnpg status myapp-pg -n <namespace>

# View logs
kubectl logs -n <namespace> myapp-pg-1 -c postgres
```

## Secrets Management

### Default Pattern: Native Kubernetes Secrets

CloudNativePG automatically generates and manages secrets for each cluster. This is the **recommended approach** for homelab use:

✅ **Advantages:**
- Zero configuration required
- Operator manages lifecycle automatically
- Secrets are encrypted at rest in etcd
- RBAC controls access

❌ **Trade-offs:**
- Not centralized with external secret managers
- Basic audit capabilities

### Advanced: External Secret Integration

For production or high-security applications, you can integrate with Bao/Vault:

1. Pre-create credentials in Bao/Vault
2. Use ExternalSecret to sync to Kubernetes
3. Reference those credentials in the Cluster spec

See [CloudNativePG documentation](https://cloudnative-pg.io/documentation/) for details on custom secret management.

## Monitoring

### Prometheus Metrics

CloudNativePG exposes metrics on port 9187:
- Database health and availability
- Connection counts and usage
- Query performance statistics
- Replication lag and status
- Backup status and WAL archiving

Metrics are automatically scraped by Prometheus via ServiceMonitor.

### Grafana Dashboards

Pre-configured dashboards are available in Grafana:
- **Cluster Overview** - Health, connections, storage
- **Operator Metrics** - Operator performance
- **Backup Status** - Backup and WAL metrics
- **Query Performance** - PostgreSQL statistics

Access Grafana at: `https://monitoring.tryy3.dev`

## Backup and Recovery

See [BACKUP_SETUP.md](BACKUP_SETUP.md) for detailed backup configuration using S3-compatible storage.

**Note**: Backup configuration is optional but highly recommended for production workloads.

## Migration

See [MIGRATION_GUIDE.md](MIGRATION_GUIDE.md) for instructions on migrating existing PostgreSQL databases to CloudNativePG.

## Troubleshooting

### Cluster Not Starting

```bash
# Check operator logs
kubectl logs -n cnpg-system deployment/cloudnative-pg-controller-manager

# Check cluster events
kubectl describe cluster myapp-pg -n <namespace>

# Check pod events
kubectl describe pod myapp-pg-1 -n <namespace>
```

### Connection Issues

```bash
# Verify service exists
kubectl get svc -n <namespace> | grep myapp-pg

# Test connection from within cluster
kubectl run -it --rm debug --image=postgres:16 --restart=Never -- \
  psql postgresql://app:PASSWORD@myapp-pg-rw.<namespace>.svc.cluster.local:5432/app
```

### Replication Issues

```bash
# Check replication status
kubectl cnpg status myapp-pg -n <namespace>

# View replication lag
kubectl cnpg metrics myapp-pg -n <namespace> | grep lag
```

## Resources

- **Official Documentation**: https://cloudnative-pg.io/documentation/
- **GitHub Repository**: https://github.com/cloudnative-pg/cloudnative-pg
- **Operator Logs**: `kubectl logs -n cnpg-system deployment/cloudnative-pg-controller-manager`
- **Helm Chart**: https://cloudnative-pg.github.io/charts

## File Structure

```
apps/cnpg-system/
├── namespace/
│   └── namespace.yaml          # cnpg-system namespace
├── operator/
│   ├── Chart.yaml              # Operator Helm chart
│   └── values.yaml             # Operator configuration
├── monitoring/
│   ├── podmonitor.yaml         # Operator metrics scraping
│   └── servicemonitor.yaml     # Cluster metrics scraping
├── templates/
│   └── cluster-template.yaml   # Reusable cluster template
├── README.md                   # This file
├── BACKUP_SETUP.md            # Backup configuration guide
└── MIGRATION_GUIDE.md         # Migration procedures
```

## Examples

After deployment, you can create example applications to test the setup. Typical pattern:

```
apps/myapp/
├── myapp-pg/
│   └── cluster.yaml           # PostgreSQL cluster
└── myapp/
    ├── deployment.yaml        # Application deployment
    └── service.yaml           # Application service
```

The application's `deployment.yaml` references the auto-generated secrets as shown in the connection examples above.

