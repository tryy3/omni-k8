# Envoy Gateway Templates

Copy and paste these templates, then fill in the `[PLACEHOLDERS]` with your actual values.

## Template 1: New Service on Existing Domain

**Use this when:** You want to expose a new service on `test.tryy3.dev`

**Fill in these placeholders:**
- `[ROUTE_NAME]` - Unique name for this route (e.g., `prometheus-route`)
- `[NAMESPACE]` - Namespace where your service lives
- `[SERVICE_NAME]` - Name of the Kubernetes Service
- `[SERVICE_PORT]` - Port the service listens on
- `[PATH]` - Optional path prefix (e.g., `/prometheus` or `/` for all)
- `[LISTENER_NAME]` - The Gateway listener name (use `https` for existing domain)

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: [ROUTE_NAME]
  namespace: [NAMESPACE]
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: [LISTENER_NAME]
  hostnames:
    - "test.tryy3.dev"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: [PATH]
      backendRefs:
        - group: ""
          kind: Service
          name: [SERVICE_NAME]
          namespace: [NAMESPACE]
          port: [SERVICE_PORT]
```

**Example: Prometheus on test.tryy3.dev/prometheus**

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: prometheus-route
  namespace: monitoring
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: https
  hostnames:
    - "test.tryy3.dev"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /prometheus
      backendRefs:
        - group: ""
          kind: Service
          name: prometheus
          namespace: monitoring
          port: 9090
```

---

## Template 2: New Domain with New Service

**Use this when:** You want to add a completely new domain like `monitoring.tryy3.dev`

**You need to add:**
1. Certificate resource
2. Gateway listener (modify existing Gateway)
3. HTTPRoute

### Step 1: Add Certificate

**Fill in these placeholders:**
- `[CERT_NAME]` - Unique name (e.g., `monitoring-tryy3-dev`)
- `[DOMAIN]` - Your domain (e.g., `monitoring.tryy3.dev`)

```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: [CERT_NAME]
  namespace: envoy
spec:
  secretName: [CERT_NAME]-tls
  duration: 2160h # 90d
  renewBefore: 720h # 30d
  commonName: [DOMAIN]
  dnsNames:
    - [DOMAIN]
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

**Example: monitoring.tryy3.dev**

```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: monitoring-tryy3-dev
  namespace: envoy
spec:
  secretName: monitoring-tryy3-dev-tls
  duration: 2160h
  renewBefore: 720h
  commonName: monitoring.tryy3.dev
  dnsNames:
    - monitoring.tryy3.dev
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

### Step 2: Add Gateway Listener

**Find the existing Gateway in gateway.yaml and add this listener to the `listeners:` array**

**Fill in these placeholders:**
- `[LISTENER_NAME]` - Unique name (e.g., `https-monitoring`)
- `[DOMAIN]` - Your domain
- `[CERT_NAME]` - Must match Certificate `secretName` above

```yaml
    - name: [LISTENER_NAME]
      protocol: HTTPS
      port: 443
      hostname: "[DOMAIN]"
      allowedRoutes:
        namespaces:
          from: All
      tls:
        mode: Terminate
        certificateRefs:
          - group: ""
            kind: Secret
            name: [CERT_NAME]-tls
```

**Example: monitoring.tryy3.dev listener**

```yaml
    - name: https-monitoring
      protocol: HTTPS
      port: 443
      hostname: "monitoring.tryy3.dev"
      allowedRoutes:
        namespaces:
          from: All
      tls:
        mode: Terminate
        certificateRefs:
          - group: ""
            kind: Secret
            name: monitoring-tryy3-dev-tls
```

### Step 3: Add HTTPRoute

**Fill in these placeholders:**
- `[ROUTE_NAME]` - Unique name (e.g., `monitoring-route`)
- `[NAMESPACE]` - Service namespace
- `[LISTENER_NAME]` - Match the listener name from Step 2
- `[DOMAIN]` - Your domain
- `[SERVICE_NAME]` - Service to route to
- `[SERVICE_PORT]` - Service port
- `[PATH]` - Path prefix

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: [ROUTE_NAME]
  namespace: [NAMESPACE]
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: [LISTENER_NAME]
  hostnames:
    - "[DOMAIN]"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: [PATH]
      backendRefs:
        - group: ""
          kind: Service
          name: [SERVICE_NAME]
          namespace: [NAMESPACE]
          port: [SERVICE_PORT]
```

**Example: monitoring.tryy3.dev → prometheus**

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: monitoring-route
  namespace: monitoring
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: https-monitoring
  hostnames:
    - "monitoring.tryy3.dev"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - group: ""
          kind: Service
          name: prometheus
          namespace: monitoring
          port: 9090
```

---

## Template 3: Multiple Routes on Same Domain (Path-Based)

**Use this when:** Multiple services on the same domain with different paths

**Example: monitoring.tryy3.dev/prometheus and monitoring.tryy3.dev/alertmanager**

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: monitoring-prometheus-route
  namespace: monitoring
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: https-monitoring
  hostnames:
    - "monitoring.tryy3.dev"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /prometheus
      backendRefs:
        - group: ""
          kind: Service
          name: prometheus
          namespace: monitoring
          port: 9090

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: monitoring-alertmanager-route
  namespace: monitoring
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: https-monitoring
  hostnames:
    - "monitoring.tryy3.dev"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /alertmanager
      backendRefs:
        - group: ""
          kind: Service
          name: alertmanager
          namespace: monitoring
          port: 9093
```

---

## Template 4: Using Different Backend Ports

**Use this when:** Your service uses a non-standard port

**Common ports:**
- `80` - HTTP
- `443` - HTTPS
- `8080` - Web UI (common alternative)
- `9090` - Prometheus
- `9093` - Alertmanager
- `3000` - Grafana

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: [ROUTE_NAME]
  namespace: [NAMESPACE]
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: [LISTENER_NAME]
  hostnames:
    - "[DOMAIN]"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - group: ""
          kind: Service
          name: [SERVICE_NAME]
          namespace: [NAMESPACE]
          port: [SERVICE_PORT]
```

---

## Template 5: Multiple HTTPRoutes in Single File

**Use this when:** Adding multiple routes at once

```yaml
---
# Route 1
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: route-one
  namespace: namespace-one
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: https
  hostnames:
    - "test.tryy3.dev"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /service-one
      backendRefs:
        - group: ""
          kind: Service
          name: service-one
          namespace: namespace-one
          port: 8080

---
# Route 2
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: route-two
  namespace: namespace-two
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: https
  hostnames:
    - "test.tryy3.dev"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /service-two
      backendRefs:
        - group: ""
          kind: Service
          name: service-two
          namespace: namespace-two
          port: 3000
```

---

## Quick Reference: Fill-in Values

### Finding Your Service Details

```bash
# List all services in a namespace
kubectl get svc -n [NAMESPACE]

# Get specific service details
kubectl get svc [SERVICE_NAME] -n [NAMESPACE] -o yaml

# Check service port
kubectl get svc [SERVICE_NAME] -n [NAMESPACE] -o jsonpath='{.spec.ports[*].port}'

# Verify service is running
kubectl get endpoints [SERVICE_NAME] -n [NAMESPACE]
```

### Service Port Examples

```bash
# Get ArgoCD port
kubectl get svc argocd-server -n argocd -o jsonpath='{.spec.ports[?(@.name=="http")].port}'
# Output: 80

# Get Prometheus port
kubectl get svc prometheus -n monitoring -o jsonpath='{.spec.ports[?(@.name=="web")].port}'
# Output: 9090
```

---

## Common Scenarios Reference

| Need | Copy Template | Modify |
|------|--------------|--------|
| New service, same domain | Template 1 | Fill placeholders |
| New domain | Templates 2+3 | Fill placeholders + modify Gateway |
| Path-based routing | Template 3 | Fill placeholders |
| Non-standard port | Template 4 | Fill placeholders |
| Multiple routes | Template 5 | Repeat section per route |

---

## Checklist Before Committing

For each HTTPRoute, verify:

- [ ] `metadata.name` - Unique name
- [ ] `metadata.namespace` - Matches service namespace
- [ ] `parentRefs.sectionName` - Matches Gateway listener name
- [ ] `hostnames` - Matches domain
- [ ] `backendRefs.name` - Matches actual Service name
- [ ] `backendRefs.namespace` - Matches service namespace
- [ ] `backendRefs.port` - Correct port number

For each Certificate (if new domain):

- [ ] `metadata.name` - Unique name
- [ ] `metadata.namespace` - Must be `envoy`
- [ ] `spec.secretName` - Matches Gateway `certificateRefs.name`
- [ ] `spec.commonName` - Matches domain
- [ ] `spec.dnsNames` - Includes domain

For each Gateway Listener (if new domain):

- [ ] `name` - Unique listener name
- [ ] `hostname` - Your domain
- [ ] `certificateRefs.name` - Matches Certificate `secretName`
