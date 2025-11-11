# CloudNativePG Grafana Dashboard

This directory contains the CloudNativePG dashboard as a ConfigMap for automatic deployment via Grafana sidecar.

## Dashboard Source

The dashboard is sourced from the [official CloudNativePG Grafana Dashboards repository](https://github.com/cloudnative-pg/grafana-dashboards/blob/main/charts/cluster/grafana-dashboard.json).

## How It Works

The dashboard JSON has been pre-processed to:
- Remove `__inputs` and `__requires` placeholders
- Replace all datasource references with the actual Prometheus datasource (uid: `prometheus`)
- Package as a ConfigMap with label `grafana_dashboard: "1"`

The Grafana sidecar automatically discovers and imports this dashboard.

## Updating the Dashboard

To update to the latest version:

```bash
# Download latest dashboard
curl -sL https://raw.githubusercontent.com/cloudnative-pg/grafana-dashboards/main/charts/cluster/grafana-dashboard.json -o /tmp/cnpg-dashboard-raw.json

# Process it (replace datasources, remove __inputs)
python3 << 'EOF'
import json

with open('/tmp/cnpg-dashboard-raw.json', 'r') as f:
    dashboard = json.load(f)

# Remove metadata
dashboard.pop('__inputs', None)
dashboard.pop('__requires', None)
dashboard.pop('__elements', None)

# Replace datasource references
def replace_datasource(obj):
    if isinstance(obj, dict):
        if 'datasource' in obj and isinstance(obj['datasource'], dict):
            if obj['datasource'].get('uid') == '${DS_PROMETHEUS}':
                obj['datasource']['uid'] = 'prometheus'
        for key, value in obj.items():
            obj[key] = replace_datasource(value)
    elif isinstance(obj, list):
        return [replace_datasource(item) for item in obj]
    elif isinstance(obj, str):
        return obj.replace('${DS_PROMETHEUS}', 'prometheus')
    return obj

dashboard = replace_datasource(dashboard)

with open('/tmp/cnpg-dashboard-processed.json', 'w') as f:
    json.dump(dashboard, f, indent=2)
EOF

# Recreate ConfigMap
cat > dashboard.yaml << 'YAML'
apiVersion: v1
kind: ConfigMap
metadata:
  name: cnpg-grafana-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  cloudnativepg-dashboard.json: |
YAML
cat /tmp/cnpg-dashboard-processed.json | sed 's/^/    /' >> dashboard.yaml

# Cleanup
rm /tmp/cnpg-dashboard-raw.json /tmp/cnpg-dashboard-processed.json
```

## Manual Import (Alternative)

If you prefer to manually import the dashboard:

1. Go to Grafana → Dashboards → Import
2. Upload the JSON from: https://raw.githubusercontent.com/cloudnative-pg/grafana-dashboards/main/charts/cluster/grafana-dashboard.json
3. Select "Prometheus" as the datasource
4. Click Import

