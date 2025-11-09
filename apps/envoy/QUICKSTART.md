# Envoy Gateway Quick Start Guide

## TL;DR: Adding a New Endpoint

### I want to expose a new service on an **existing domain** (e.g., `test.tryy3.dev`)

**Add ONE HTTPRoute to `gateway.yaml`:**

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-service-route
  namespace: my-namespace
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: https  # Match the listener name
  hostnames:
    - "test.tryy3.dev"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /my-path  # Optional: route specific paths
      backendRefs:
        - group: ""
          kind: Service
          name: my-service
          namespace: my-namespace
          port: 8080
```

**That's it!** Commit and push, ArgoCD will deploy it.

---

### I want to expose a new service on a **new domain** (e.g., `new.tryy3.dev`)

**Add THREE resources to `gateway.yaml`:**

#### 1. Certificate (for TLS)
```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: new-tryy3-dev
  namespace: envoy
spec:
  secretName: new-tryy3-dev-tls
  commonName: new.tryy3.dev
  dnsNames:
    - new.tryy3.dev
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

#### 2. Gateway Listener (tell Envoy about the new domain)
Add to the existing `custom-domain-gateway` Gateway resource:
```yaml
    - name: https-new
      protocol: HTTPS
      port: 443
      hostname: "new.tryy3.dev"
      allowedRoutes:
        namespaces:
          from: All
      tls:
        mode: Terminate
        certificateRefs:
          - group: ""
            kind: Secret
            name: new-tryy3-dev-tls
```

#### 3. HTTPRoute (route traffic to your service)
```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-service-route
  namespace: my-namespace
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: custom-domain-gateway
      namespace: envoy
      sectionName: https-new
  hostnames:
    - "new.tryy3.dev"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - group: ""
          kind: Service
          name: my-service
          namespace: my-namespace
          port: 8080
```

**Commit and push!**

---

## The Moving Pieces Explained

### What you DON'T need to change:
- ❌ `proxy.yaml` - The infrastructure setup
- ❌ `namespace.yaml` - Envoy namespace
- ❌ `helm/` - Helm chart

### What you DO need to change:
- ⭐ `gateway.yaml` - Add Certificates, Gateway listeners, and HTTPRoutes here

---

## Reference Table

| Want to do | Add | Modify |
|-----------|-----|--------|
| New service, same domain | 1 HTTPRoute | Nothing |
| New domain | 1 Certificate + 1 Gateway listener + 1 HTTPRoute | Gateway spec |
| Path-based routing | 1 HTTPRoute | Nothing |
| Different backend port | 1 HTTPRoute | Nothing |

---

## Finding Your Service Details

```bash
# List services in a namespace
kubectl get svc -n my-namespace

# Get service port
kubectl get svc my-service -n my-namespace -o yaml | grep -A 5 "ports:"

# Test service is running
kubectl get endpoints my-service -n my-namespace
```

---

## Checklist Before Pushing

- [ ] Service name is correct (matches `kubectl get svc`)
- [ ] Service namespace is correct
- [ ] Service port matches (8080, 9090, 80, etc.)
- [ ] Domain matches in Certificate, Gateway, and HTTPRoute
- [ ] Certificate namespace is `envoy`
- [ ] Gateway listener `name` matches HTTPRoute `sectionName`
- [ ] HTTPRoute `namespace` matches service namespace

---

## Verify It's Working

```bash
# Wait for certificate to be issued
kubectl get certificate -n envoy
# Should show READY: True

# Check gateway is programmed
kubectl get gateway -n envoy
# Should show PROGRAMMED: True

# Check HTTPRoute is accepted
kubectl get httproute -A
# Should show all routes in SYNCED state

# Test it works
curl -k https://your-domain.tryy3.dev
```

---

## Common Mistakes

❌ **Mistake**: Using wrong port number
```yaml
port: 80  # ArgoCD uses port 80, but Prometheus uses 9090
```
✅ **Fix**: Check with `kubectl get svc -n namespace`

---

❌ **Mistake**: Forgetting to add Gateway listener for new domain
```yaml
# Only added HTTPRoute, but Gateway has no listener for this hostname
hostnames:
  - "forgot-to-add-listener.tryy3.dev"
```
✅ **Fix**: Add the listener to the Gateway spec

---

❌ **Mistake**: Wrong sectionName in HTTPRoute
```yaml
sectionName: https  # But created listener named "https-new"
```
✅ **Fix**: Match the listener name exactly

---

❌ **Mistake**: HTTPRoute in wrong namespace
```yaml
metadata:
  namespace: kube-system  # Should be same as the service
```
✅ **Fix**: Use the same namespace as your service

---

## Need More Details?

See `README.md` in this directory for:
- Deep dive into how it all works
- Troubleshooting guide
- Advanced patterns (path-based routing, etc.)