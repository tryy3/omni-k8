# Envoy Gateway Setup Guide

This directory contains the Envoy Gateway configuration for exposing services via TLS-secured HTTP routes.

## Architecture Overview

The setup consists of several interconnected components:

### 1. **EnvoyProxy & GatewayClass** (`proxy.yaml`)
- **EnvoyProxy**: Defines how Envoy Proxy behaves (runs as a LoadBalancer service via Tailscale)
- **GatewayClass**: Links the Gateway to the Envoy controller
- **Status**: ✅ **SET UP ONCE** - You don't need to modify this for new endpoints

### 2. **Gateway** (`gateway.yaml`)
- **Purpose**: Listens on a specific hostname and port (e.g., `test.tryy3.dev:443`)
- **TLS Configuration**: Terminates HTTPS and uses a certificate secret
- **Status**: ✅ **SET UP ONCE** - You typically only need one Gateway per domain
- **Note**: One Gateway can have multiple listeners for different hostnames

### 3. **HTTPRoute** (`gateway.yaml`)
- **Purpose**: Routes HTTP traffic to backend services
- **Parent**: References a Gateway listener by hostname
- **Backend**: Points to the actual Kubernetes Service that serves the traffic
- **Status**: ⭐ **CREATED FOR EACH NEW ENDPOINT** - Add one HTTPRoute per service you want to expose

### 4. **Certificate** (`gateway.yaml`)
- **Purpose**: Holds the TLS certificate for the domain (auto-managed by cert-manager)
- **Issuer**: References `letsencrypt-prod` ClusterIssuer for DNS-01 validation via DNSimple
- **Status**: ⭐ **CREATED FOR EACH NEW DOMAIN** - One Certificate per unique domain/hostname

---

## Quick Answer: What Do I Need for a New Endpoint?

### Scenario 1: Same Domain, Different Service
**Example**: You want `test.tryy3.dev/monitoring` → Prometheus, but already have `test.tryy3.dev` → ArgoCD

**What you need:**
- ✅ **HTTPRoute** - Add a new one in `gateway.yaml`
- ❌ **Gateway** - Reuse the existing one
- ❌ **Certificate** - Reuse the existing one (same domain)
- ❌ **EnvoyProxy/GatewayClass** - Already set up

**Changes needed:** Add 1 HTTPRoute resource

### Scenario 2: Different Domain, Different Service
**Example**: You want `monitoring.tryy3.dev` → Prometheus (new domain)

**What you need:**
- ✅ **HTTPRoute** - Add a new one
- ✅ **Gateway** - Add a new listener to the existing Gateway resource
- ✅ **Certificate** - Create a new one for the new domain
- ❌ **EnvoyProxy/GatewayClass** - Already set up

**Changes needed:** Add 1 HTTPRoute + 1 Gateway listener + 1 Certificate resource

---

## Step-by-Step: Adding a New Endpoint

### Example: Expose Prometheus at `monitoring.tryy3.dev`

#### Step 1: Create a Certificate (if new domain)

Add to `gateway.yaml`:

```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: monitoring-tryy3-dev
  namespace: envoy
spec:
  secretName: monitoring-tryy3-dev-tls
  duration: 2160h # 90d
  renewBefore: 720h # 30d
  commonName: monitoring.tryy3.dev
  dnsNames:
    - monitoring.tryy3.dev
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

**Key points:**
- `secretName`: Must match the cert reference in the Gateway listener (below)
- `commonName` and `dnsNames`: Must match the domain you're securing
- `namespace: envoy`: Must be in the same namespace as the Gateway

#### Step 2: Add a Gateway Listener (if new domain)

Update the `custom-domain-gateway` Gateway resource in `gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: custom-domain-gateway
  labels:
    external-dns: enabled
  annotations:
    cert-manager.io/cluster-issuer: cert-manager-webhook-dnsimple-production
spec:
  gatewayClassName: tailscale
  listeners:
    # Existing listener
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "test.tryy3.dev"
      allowedRoutes:
        namespaces:
          from: All
      tls:
        mode: Terminate
        certificateRefs:
          - group: ""
            kind: Secret
            name: test-tryy3-dev-tls
    
    # NEW LISTENER for monitoring domain
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

**Key points:**
- Each listener needs a unique `name`
- `hostname`: The domain this listener responds to
- `certificateRefs`: Points to the secret created by the Certificate resource
- Port `443` is reused across listeners (different hostnames)

#### Step 3: Create an HTTPRoute

Add to `gateway.yaml`:

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: monitoring-route
  namespace: monitoring  # Create this in the namespace of your service
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: https-monitoring  # Match the listener name
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

**Key points:**
- `namespace`: Should be in the namespace where the service lives (for better organization)
- `parentRefs.sectionName`: Must match the Gateway listener `name` (if multiple listeners)
- `hostnames`: Should match the Gateway listener's hostname
- `backendRefs`: Points to your actual Kubernetes Service
- `port`: The port your service listens on (check with `kubectl get svc -n monitoring`)

---

## Complete Example: Adding Three Endpoints

### File: `gateway.yaml`

```yaml
# ===== CERTIFICATES =====
# Create TLS certificates for each unique domain

apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-tryy3-dev
  namespace: envoy
spec:
  secretName: test-tryy3-dev-tls
  commonName: test.tryy3.dev
  dnsNames:
    - test.tryy3.dev
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: monitoring-tryy3-dev
  namespace: envoy
spec:
  secretName: monitoring-tryy3-dev-tls
  commonName: monitoring.tryy3.dev
  dnsNames:
    - monitoring.tryy3.dev
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

---
# ===== GATEWAY =====
# Single gateway with multiple listeners (one per domain)

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: custom-domain-gateway
  labels:
    external-dns: enabled
  annotations:
    cert-manager.io/cluster-issuer: cert-manager-webhook-dnsimple-production
spec:
  gatewayClassName: tailscale
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "test.tryy3.dev"
      allowedRoutes:
        namespaces:
          from: All
      tls:
        mode: Terminate
        certificateRefs:
          - group: ""
            kind: Secret
            name: test-tryy3-dev-tls

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

---
# ===== HTTP ROUTES =====
# Route traffic from each domain to services

# Route 1: test.tryy3.dev → argocd-server
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
  namespace: argocd
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
            value: /
      backendRefs:
        - group: ""
          kind: Service
          name: argocd-server
          namespace: argocd
          port: 80

---
# Route 2: monitoring.tryy3.dev/ → prometheus
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

---
# Route 3: monitoring.tryy3.dev/alertmanager → alertmanager
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: alertmanager-route
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

## Troubleshooting

### Certificate not being issued?
```bash
# Check certificate status
kubectl describe certificate test-tryy3-dev -n envoy

# Check certificate request
kubectl describe certificaterequest -n envoy

# Check ACME challenge
kubectl describe challenge -n envoy
```

### HTTPRoute not working?
```bash
# Check HTTPRoute status
kubectl describe httproute monitoring-route -n monitoring

# Check if it's attached to the gateway
kubectl get httproute -A

# Verify the parent gateway exists
kubectl get gateway -n envoy
```

### Gateway not programmed?
```bash
# Check gateway status
kubectl describe gateway custom-domain-gateway -n envoy

# Should show "Programmed: True"
```

### Service not reachable?
```bash
# Verify service exists and is ready
kubectl get svc -n monitoring
kubectl describe svc prometheus -n monitoring

# Check service endpoints
kubectl get endpoints prometheus -n monitoring
```

---

## Key Takeaways

| Component | Created Once? | Why? |
|-----------|---------------|------|
| **EnvoyProxy** | ✅ Yes | Infrastructure setup - reused for all endpoints |
| **GatewayClass** | ✅ Yes | Connects Gateway to Envoy controller |
| **Gateway** | ✅ Yes (mostly) | One gateway per load balancer; add listeners for new domains |
| **Gateway Listener** | ⭐ Per domain | Add a new listener for each unique hostname |
| **Certificate** | ⭐ Per domain | Each unique domain needs its own cert |
| **HTTPRoute** | ⭐ Per service | Add one for each backend service you want to expose |

---

## Common Patterns

### Pattern 1: Same domain, path-based routing
```yaml
# monitoring.tryy3.dev/prometheus → prometheus service
# monitoring.tryy3.dev/alertmanager → alertmanager service
# Use ONE HTTPRoute with multiple rules and path matches
```

### Pattern 2: Multiple domains, one per service
```yaml
# test.tryy3.dev → argocd-server
# monitoring.tryy3.dev → prometheus
# Use separate HTTPRoute, Listener, and Certificate for each
```

### Pattern 3: Wildcard domain (advanced)
```yaml
# *.tryy3.dev → different services based on subdomain
# Uses dynamic routing (requires more complex configuration)
```

---

## Next Steps

1. Verify current setup: `kubectl get gateway,httproute,certificate -A`
2. Add your new endpoint following the examples above
3. Commit changes to git (ArgoCD will auto-sync)
4. Monitor with: `kubectl describe httproute <name> -n <namespace>`
