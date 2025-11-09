# Envoy Gateway Documentation Index

Welcome! This directory contains everything you need to expose services via TLS-secured HTTP routes using Envoy Gateway.

## 📚 Documentation Files

### For First-Time Setup
- **[README.md](README.md)** - Comprehensive guide explaining how the entire system works
  - Architecture overview
  - Component relationships
  - Step-by-step setup instructions
  - Troubleshooting guide

### For Quick Reference
- **[QUICKSTART.md](QUICKSTART.md)** - Fast guide for common tasks
  - TL;DR for adding endpoints
  - Common mistakes and fixes
  - Checklist before pushing

### For Understanding the System
- **[ARCHITECTURE.md](ARCHITECTURE.md)** - Visual diagrams and system flow
  - System flow diagrams
  - Component relationships
  - Request flow walkthrough
  - Certificate lifecycle
  - Namespace organization

### For Copy-Paste Configuration
- **[TEMPLATES.md](TEMPLATES.md)** - Ready-to-use YAML templates
  - Template 1: New service on existing domain
  - Template 2: New domain with new service
  - Template 3: Multiple routes with path-based routing
  - Template 4: Different backend ports
  - Complete examples
  - Service discovery commands

## 🚀 Quick Start

### I want to...

**...expose a new service on an existing domain (e.g., `test.tryy3.dev`)**
1. Go to [QUICKSTART.md](QUICKSTART.md) - "Same domain" section
2. Copy template from [TEMPLATES.md](TEMPLATES.md) - Template 1
3. Fill in placeholders
4. Add to `gateway.yaml`
5. Commit and push

**...add a brand new domain (e.g., `monitoring.tryy3.dev`)**
1. Go to [QUICKSTART.md](QUICKSTART.md) - "New domain" section
2. Follow the 3-step process:
   - Use Template 2a (Certificate) from [TEMPLATES.md](TEMPLATES.md)
   - Use Template 2b (Gateway Listener) from [TEMPLATES.md](TEMPLATES.md)
   - Use Template 2c (HTTPRoute) from [TEMPLATES.md](TEMPLATES.md)
3. Modify Gateway resource
4. Add new resources to `gateway.yaml`
5. Commit and push

**...understand how everything works**
1. Start with [README.md](README.md) - "Architecture Overview"
2. Look at diagrams in [ARCHITECTURE.md](ARCHITECTURE.md)
3. Review current [gateway.yaml](extra/gateway.yaml) example

**...troubleshoot an issue**
1. Check [QUICKSTART.md](QUICKSTART.md) - "Verification" section
2. Review [README.md](README.md) - "Troubleshooting" section

## 📋 File Structure

```
omni-k8/apps/envoy/
├── INDEX.md                    ← You are here
├── README.md                   ← Comprehensive guide
├── QUICKSTART.md               ← Quick reference
├── ARCHITECTURE.md             ← Diagrams and flows
├── TEMPLATES.md                ← Copy-paste templates
├── namespace/
│   └── namespace.yaml          ← Envoy namespace definition
├── helm/
│   ├── Chart.yaml              ← Helm chart reference
│   └── values.yaml             ← Helm configuration
└── extra/
    ├── proxy.yaml              ← EnvoyProxy & GatewayClass (set once)
    └── gateway.yaml            ← Gateway, HTTPRoutes, Certificates (edit here!)
```

## 🎯 The Answer to "What Do I Need?"

| What You Want | What You Add |
|---|---|
| New service on existing domain | 1 HTTPRoute |
| New domain with service | 1 Certificate + 1 Gateway listener + 1 HTTPRoute |
| Path-based routing | 1 HTTPRoute (with multiple rules) |
| Different port number | Update HTTPRoute `port` field |

## ✅ Standard Setup Overview

**SET UP ONCE (Already Done):**
- ✅ EnvoyProxy (`proxy.yaml`)
- ✅ GatewayClass (`proxy.yaml`)
- ✅ Gateway base resource (`gateway.yaml`)
- ✅ cert-manager integration

**ADD FOR EACH NEW DOMAIN:**
- ⭐ Certificate resource (TLS certificate)
- ⭐ Gateway listener (tell Envoy about the domain)

**ADD FOR EACH NEW SERVICE:**
- ⭐ HTTPRoute resource (route traffic)

## 🔍 Finding Service Details

Need to know what port your service uses?

```bash
# List services
kubectl get svc -n [namespace]

# Get service port
kubectl get svc [service-name] -n [namespace] -o jsonpath='{.spec.ports[*].port}'

# Check service is running
kubectl get endpoints [service-name] -n [namespace]
```

## 📖 Reading Guide by Use Case

### "I just want to add one endpoint quickly"
1. Read: [QUICKSTART.md](QUICKSTART.md) (2 min)
2. Copy: Appropriate template from [TEMPLATES.md](TEMPLATES.md) (1 min)
3. Edit and commit (5 min)

### "I need to understand everything before making changes"
1. Read: [README.md](README.md) (15 min)
2. Review: [ARCHITECTURE.md](ARCHITECTURE.md) diagrams (10 min)
3. Study: Current [gateway.yaml](extra/gateway.yaml) example (5 min)

### "Something's not working"
1. Check: [QUICKSTART.md](QUICKSTART.md) troubleshooting section
2. Review: [README.md](README.md) troubleshooting guide
3. Run: Kubectl commands from [TEMPLATES.md](TEMPLATES.md) "Service Discovery"

### "I need a template but I'm not sure which one"
1. Go to: [TEMPLATES.md](TEMPLATES.md) "Common Scenarios Reference" table
2. Find your scenario
3. Copy the recommended template

## 🔗 Key Concepts Map

- **Gateway** - Listens on a domain:port, terminates TLS
- **HTTPRoute** - Routes traffic to backend services
- **Certificate** - TLS certificate (auto-managed by cert-manager)
- **Service** - Your actual Kubernetes service to expose
- **Listener** - Part of Gateway that handles one domain

## 💡 Pro Tips

1. **One HTTPRoute per service** - This keeps things organized
2. **Reuse domains** - Add more HTTPRoutes instead of new Certificates
3. **Test locally first** - Use `kubectl port-forward` to test services
4. **Check ports** - Most service issues are wrong port numbers
5. **Use path prefixes** - Route different paths to different services on same domain

## 📞 Common Commands

```bash
# Check current Gateway status
kubectl get gateway -n envoy
kubectl describe gateway custom-domain-gateway -n envoy

# Check Certificate status
kubectl get certificate -n envoy
kubectl describe certificate [cert-name] -n envoy

# Check HTTPRoutes
kubectl get httproute -A
kubectl describe httproute [route-name] -n [namespace]

# Check if TLS secret exists
kubectl get secret -n envoy | grep tls

# Verify service exists
kubectl get svc -n [namespace]
kubectl get endpoints [service-name] -n [namespace]
```

## 🎓 Learning Path

1. **Day 1**: Read [QUICKSTART.md](QUICKSTART.md) + add your first endpoint
2. **Day 2**: Read [README.md](README.md) to understand the system
3. **Day 3**: Review [ARCHITECTURE.md](ARCHITECTURE.md) for deeper understanding
4. **Future**: Use [TEMPLATES.md](TEMPLATES.md) for quick reference

## 📝 Before You Edit

Always check:
- [ ] You've read the relevant guide
- [ ] You're editing the right file (`gateway.yaml`)
- [ ] Service exists: `kubectl get svc -n [namespace]`
- [ ] Service is running: `kubectl get endpoints [service-name] -n [namespace]`
- [ ] You're using correct port number

## 🚨 Most Common Mistakes

1. ❌ Wrong port number → ✅ Check with `kubectl get svc`
2. ❌ Forgot to add Gateway listener for new domain → ✅ Add listener to Gateway spec
3. ❌ HTTPRoute in wrong namespace → ✅ Use same namespace as service
4. ❌ Wrong sectionName → ✅ Match listener name exactly
5. ❌ New Certificate not in `envoy` namespace → ✅ Always use `namespace: envoy`

## 📚 File Descriptions

### README.md
Deep dive into how the system works. Explains every component, relationships, and troubleshooting. Start here if you want to understand the "why" behind everything.

### QUICKSTART.md
Fast reference guide with TL;DR sections. Use this when you know what you want to do but need a quick refresher. Includes common mistakes and fixes.

### ARCHITECTURE.md
Visual diagrams showing system flow, dependencies, and how requests flow through the system. Great for visual learners and understanding the big picture.

### TEMPLATES.md
Copy-paste ready YAML templates for common scenarios. Fill in placeholders and you're done. Also includes service discovery commands and checklists.

### gateway.yaml
The actual configuration file where you add new endpoints. Split into three sections: Certificates, Gateway, and HTTPRoutes.

### proxy.yaml
Infrastructure setup (EnvoyProxy + GatewayClass). Set up once, rarely modified.

---

**Ready to add your first endpoint?** → Go to [QUICKSTART.md](QUICKSTART.md)

**Want to understand it all?** → Start with [README.md](README.md)

**Need a template?** → Find it in [TEMPLATES.md](TEMPLATES.md)

**Confused about the system?** → Check [ARCHITECTURE.md](ARCHITECTURE.md)