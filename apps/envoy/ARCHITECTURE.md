# Envoy Gateway Architecture

## System Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     Internet / Tailscale                         │
│                    (test.tryy3.dev:443)                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Tailscale LoadBalancer                        │
│                  (gateway-envoy.*.ts.net)                        │
│                   [EnvoyProxy Service]                           │
└────────────────────────────┬────────────────────────────────────┘
                             │
                    ┌────────┴────────┐
                    ▼                 ▼
          ┌──────────────────┐  ┌──────────────────┐
          │  HTTPS:443       │  │  HTTPS:443       │
          │ test.tryy3.dev   │  │ monitoring.dev   │
          │ [Gateway         │  │ [Gateway         │
          │  Listener]       │  │  Listener]       │
          └────────┬─────────┘  └────────┬─────────┘
                   │                     │
                   │ (TLS terminated)    │ (TLS terminated)
                   │                     │
        ┌──────────▼──────────┐ ┌───────▼──────────┐
        │  HTTP routing       │ │  HTTP routing    │
        │  [HTTPRoute]        │ │  [HTTPRoute]     │
        │  rules:             │ │  rules:          │
        │  /                  │ │  /               │
        │  /admin             │ │  /alertmanager   │
        └──────────┬──────────┘ └───────┬──────────┘
                   │                    │
        ┌──────────▼──────────┐ ┌──────▼──────────┐
        │ Kubernetes Service  │ │ Kubernetes Svc  │
        │ argocd-server:80    │ │ prometheus:9090 │
        │ [argocd ns]         │ │ [monitoring ns] │
        └──────────┬──────────┘ └────────┬────────┘
                   │                     │
        ┌──────────▼──────────┐ ┌──────▼──────────┐
        │ Application Pod     │ │ Application Pod │
        │ argocd-server       │ │ prometheus      │
        └─────────────────────┘ └─────────────────┘
```

## Component Relationships

```
┌─────────────────────────────────────────────────────────────┐
│                      Envoy Namespace                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  GatewayClass: "tailscale"                           │  │
│  │  ├─ References EnvoyProxy                            │  │
│  │  └─ Connects to Envoy Controller                     │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                     │
│  ┌────────────────────▼─────────────────────────────────┐  │
│  │  EnvoyProxy: "tailscale-proxy"                       │  │
│  │  ├─ Provider: Kubernetes                             │  │
│  │  ├─ Service Type: LoadBalancer                       │  │
│  │  ├─ LoadBalancerClass: tailscale                     │  │
│  │  └─ Hostname: gateway-envoy                          │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                     │
│  ┌────────────────────▼─────────────────────────────────┐  │
│  │  Gateway: "custom-domain-gateway"                    │  │
│  │  ├─ GatewayClassName: tailscale                      │  │
│  │  └─ Listeners:                                       │  │
│  │     ├─ https (hostname: test.tryy3.dev)            │  │
│  │     ├─ https-monitoring (hostname: monitoring...)  │  │
│  │     └─ [more listeners for other domains]          │  │
│  │     └─ Each references a Certificate Secret         │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                     │
│  ┌────────────────────▼─────────────────────────────────┐  │
│  │  Certificates (in envoy namespace)                   │  │
│  │  ├─ Certificate: test-tryy3-dev                      │  │
│  │  │  └─ Creates Secret: test-tryy3-dev-tls          │  │
│  │  ├─ Certificate: monitoring-tryy3-dev               │  │
│  │  │  └─ Creates Secret: monitoring-tryy3-dev-tls    │  │
│  │  └─ [more certificates for other domains]           │  │
│  │     Issued by: letsencrypt-prod ClusterIssuer      │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   Other Namespaces                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Namespace: argocd                                   │  │
│  │  HTTPRoute: "app-route"                              │  │
│  │  ├─ Parent: Gateway "custom-domain-gateway" (envoy) │  │
│  │  ├─ Hostname: test.tryy3.dev                        │  │
│  │  └─ Backend: Service "argocd-server" port 80        │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Namespace: monitoring                               │  │
│  │  HTTPRoute: "prometheus-route"                        │  │
│  │  ├─ Parent: Gateway "custom-domain-gateway" (envoy) │  │
│  │  ├─ Hostname: monitoring.tryy3.dev                  │  │
│  │  └─ Backend: Service "prometheus" port 9090         │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  [Services and Pods in each namespace...]                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Resource Dependencies

```
Certificate (envoy namespace)
    ├─ Issuer: letsencrypt-prod (ClusterIssuer)
    │   └─ Solves with: DNSimple webhook (acme.dnsimple.com)
    │       └─ Uses Secret: cert-dnsimple-secret (cert-manager namespace)
    └─ Creates Secret: *-tls (envoy namespace)
           │
           ▼
    Gateway (envoy namespace)
    ├─ Listener references → Secret: *-tls
    └─ Connected to HTTPRoutes via parentRefs
           │
           ├─ HTTPRoute in argocd namespace
           │   └─ Backend: Service argocd-server (argocd ns)
           │
           ├─ HTTPRoute in monitoring namespace
           │   └─ Backend: Service prometheus (monitoring ns)
           │
           └─ [more HTTPRoutes in various namespaces]

GatewayClass (envoy namespace)
    └─ References: EnvoyProxy "tailscale-proxy"
           └─ Creates: LoadBalancer Service (Tailscale)
                  └─ Assigns: Public Tailscale IP/Hostname
```

## Request Flow Example

```
1. User visits https://test.tryy3.dev
   │
   ▼
2. DNS resolves to Tailscale IP (100.x.x.x / *.ts.net)
   │
   ▼
3. TLS connection to LoadBalancer Service
   │
   ▼
4. Envoy Proxy receives HTTPS traffic
   │
   ▼
5. Gateway listener matches hostname "test.tryy3.dev"
   │
   ▼
6. TLS is terminated using certificate from "test-tryy3-dev-tls" Secret
   │
   ▼
7. HTTP traffic passed to Gateway routing layer
   │
   ▼
8. HTTPRoute in argocd namespace matches
   ├─ Hostname matches: test.tryy3.dev ✓
   └─ Path matches: / ✓
   │
   ▼
9. Traffic routed to Service backend
   argocd-server:80 (in argocd namespace)
   │
   ▼
10. Kubernetes Service load-balances to Pod
    │
    ▼
11. Request reaches ArgoCD application
    │
    ▼
12. Response sent back through reverse path
```

## Certificate Lifecycle

```
1. Certificate resource created in envoy namespace
   │
   ▼
2. cert-manager sees Certificate
   │
   ▼
3. cert-manager creates CertificateRequest
   │
   ▼
4. cert-manager sends ACME order to Let's Encrypt
   │
   ▼
5. Let's Encrypt asks: "Prove you own test.tryy3.dev"
   │
   ▼
6. DNSimple webhook creates DNS TXT record
   (via cert-dnsimple-secret token)
   │
   ▼
7. Let's Encrypt validates DNS record
   │
   ▼
8. Let's Encrypt issues certificate
   │
   ▼
9. cert-manager stores certificate in Secret: "test-tryy3-dev-tls"
   │
   ▼
10. Gateway reads Secret and uses certificate
    │
    ▼
11. HTTPS requests to test.tryy3.dev are now secure
```

## Adding a New Endpoint: What Needs to Change?

```
Scenario 1: New service on existing domain
─────────────────────────────────────────

BEFORE: test.tryy3.dev → argocd-server
AFTER:  test.tryy3.dev → argocd-server OR prometheus

Changes needed:
  ✓ Add 1 HTTPRoute (in prometheus namespace)
  ✗ No Gateway changes
  ✗ No Certificate changes
  ✗ No EnvoyProxy changes

Result: 1 new resource


Scenario 2: New domain with new service
──────────────────────────────────────

BEFORE: test.tryy3.dev → argocd-server
AFTER:  test.tryy3.dev → argocd-server
        monitoring.tryy3.dev → prometheus

Changes needed:
  ✓ Add 1 Certificate (creates TLS secret for monitoring.tryy3.dev)
  ✓ Add 1 Gateway Listener (adds listener for monitoring.tryy3.dev to existing Gateway)
  ✓ Add 1 HTTPRoute (routes monitoring.tryy3.dev traffic)
  ✗ No EnvoyProxy changes

Result: 3 new/modified resources


Scenario 3: Different port on same backend
────────────────────────────────────────

BEFORE: test.tryy3.dev/ → argocd-server:80
AFTER:  test.tryy3.dev/admin → argocd-server:8080

Changes needed:
  ✓ Add 1 HTTPRoute (with path match /admin and port 8080)
  ✗ No Gateway changes
  ✗ No Certificate changes
  ✗ No EnvoyProxy changes

Result: 1 new resource
```

## Namespace Organization

```
envoy namespace (infrastructure)
├── EnvoyProxy
├── GatewayClass
├── Gateway (with all listeners)
├── Certificates (one per domain)
└── TLS Secrets (auto-generated)

cert-manager namespace
├── cert-manager controller
├── webhook pods
└── cert-dnsimple-secret (DNSimple credentials)

argocd namespace (application)
├── argocd-server Service/Pod
└── HTTPRoute: app-route

monitoring namespace (application)
├── prometheus Service/Pod
├── alertmanager Service/Pod
└── HTTPRoutes: prometheus-route, alertmanager-route

other app namespaces...
└── HTTPRoutes pointing to services
```

## Summary: The Minimal Setup

To expose ONE service on ONE domain, you need:

```
┌─────────────────┐      (set up once)
│  EnvoyProxy +   │
│  GatewayClass   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐      (per domain)
│  Certificate    │  ◄─── Creates TLS Secret
└────────┬────────┘
         │
         ▼
┌─────────────────┐      (per domain)
│  Gateway +      │  ◄─── One listener per domain
│  Listener       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐      (per service)
│  HTTPRoute      │  ◄─── Routes to backend Service
└─────────────────┘
```

That's it! Everything else is configuration details.