# Proxies, Load Balancers, and Traffic Management

> **Goal**: Understand the different types of traffic intermediaries, when to use each, and how they combine in real production systems.

---

## The Big Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TRAFFIC FLOW                                 │
│                                                                     │
│ Client → [Forward Proxy] → Internet → [Firewall] → [Load Balancer]  │
│                                             ↓                       │
│                                     [Reverse Proxy / Ingress]       │
│                                             ↓                       │
│                                        [API Gateway]                │
│                                             ↓                       │
│                                      Backend Services               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Forward Proxy

### What It Is

A forward proxy sits **between clients and the internet**. The client **knows** it's using a proxy.

```
Client → Forward Proxy → Internet → Destination
```

### What It Does

| Function | How |
|----------|-----|
| Hide client IP | Destination sees proxy IP |
| Access control | Allow/block websites |
| Caching | Store frequently accessed content |
| Logging | Track all outbound requests |
| Bypass restrictions | Route through different locations |

### Use Cases

- Corporate networks (control employee internet access)
- Schools and universities
- Privacy tools
- Web scraping at scale

### Mental Model

> "I don't talk to the internet directly — I send everything through my middleman."

---

## 2. Reverse Proxy

### What It Is

A reverse proxy sits **in front of your servers**. The client **doesn't know** what's behind it.

```
Client → Reverse Proxy → Backend Server(s)
```

### What It Does

| Function | How |
|----------|-----|
| Hide infrastructure | Client only sees proxy |
| TLS termination | Handle HTTPS, backends use HTTP |
| Routing | Direct traffic to correct backend |
| Security | WAF, rate limiting, header filtering |
| Caching | Store responses |
| Compression | Gzip responses |

### The Key Distinction

```
Forward Proxy:  Protects CLIENTS
Reverse Proxy:  Protects SERVERS
```

### Common Reverse Proxies

- **Nginx** — Most popular, excellent performance
- **HAProxy** — High performance, great for TCP/HTTP
- **Traefik** — Container-native, auto-discovery
- **Caddy** — Auto HTTPS, simple config
- **Envoy** — Service mesh, advanced features

### Mental Model

> "You talk to me. I decide what server actually handles your request."

---

## 3. Load Balancer

### What It Is

A load balancer **distributes traffic** across multiple backend servers. It's a **specialized reverse proxy**.

```
Client → Load Balancer → Server A
                      → Server B
                      → Server C
```

### Why Load Balance?

| Goal | How Load Balancer Helps |
|------|------------------------|
| **Scale** | Spread load across instances |
| **Availability** | Remove failed instances |
| **Performance** | Route to fastest/nearest server |
| **Zero-downtime deploys** | Drain old, add new instances |

### Load Balancing Algorithms

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| **Round Robin** | Rotate through servers | Equal servers, stateless |
| **Least Connections** | Pick server with fewest active | Varying request duration |
| **IP Hash** | Same client → same server | Session affinity needed |
| **Weighted** | More traffic to beefier servers | Heterogeneous fleet |
| **Random** | Pick randomly | Simple, surprisingly effective |

### Layer 4 vs Layer 7 Load Balancing

```
Layer 4 (Transport):
  - Sees: IP, Port, TCP/UDP
  - Fast, simple
  - No HTTP awareness
  - Example: AWS NLB

Layer 7 (Application):
  - Sees: URLs, headers, cookies
  - Can route by path (/api vs /web)
  - Can modify requests
  - Example: AWS ALB, Nginx
```

### Key Truth

```
✅ All load balancers are reverse proxies
❌ Not all reverse proxies are load balancers
```

### Mental Model

> "Who's free right now? Cool — you take this request."

---

## 4. Firewall

### What It Is

A firewall controls **network-level access**. It operates at **Layer 3-4** (IP addresses, ports, protocols).

### What It Sees

```
✅ IP addresses (source and destination)
✅ Ports
✅ Protocols (TCP, UDP, ICMP)
✅ Connection state

❌ URLs
❌ HTTP headers
❌ Request body
❌ Application logic
```

### Firewall Rules Examples

```
# Allow HTTPS from anywhere
ALLOW TCP from 0.0.0.0/0 to :443

# Allow SSH only from office
ALLOW TCP from 203.0.113.0/24 to :22

# Block everything else
DENY ALL
```

### Firewall vs Reverse Proxy

|  | Firewall | Reverse Proxy |
|--|----------|---------------|
| OSI Layer | L3-L4 | L7 |
| Sees URLs | ❌ | ✅ |
| Sees headers | ❌ | ✅ |
| Routing | ❌ | ✅ |
| Load balancing | ❌ | ✅ |
| IP blocking | ✅ | ⚠️ Possible but not primary |
| TLS termination | ❌ | ✅ |

### Common Misconception

> "If I have a reverse proxy, I don't need a firewall"

**Wrong.** A reverse proxy:
- Does NOT close ports
- Does NOT protect at network level
- Does NOT replace perimeter security

### Proper Layered Setup

```
Internet
   ↓
Firewall (allow only 80, 443)
   ↓
Reverse Proxy (handle HTTP)
   ↓
Backend Services
```

This is **defense in depth**.

### Mental Model

> "Firewall decides YES/NO to the connection. Reverse proxy decides WHAT TO DO with the request."

---

## 5. API Gateway

### What It Is

An API Gateway is a **reverse proxy specialized for APIs**. It adds **intelligence** to traffic management.

```
Client → API Gateway → Service A
                    → Service B
                    → Service C
```

### What Makes It Different

| Feature | Reverse Proxy | API Gateway |
|---------|--------------|-------------|
| Routing | ✅ | ✅ |
| Load balancing | ✅ | ✅ |
| TLS termination | ✅ | ✅ |
| **Authentication** | Basic | JWT, OAuth2, API Keys |
| **Rate limiting** | Basic | Per-user, per-endpoint |
| **API versioning** | ❌ | ✅ `/v1`, `/v2` |
| **Request transformation** | ❌ | ✅ |
| **Response aggregation** | ❌ | ✅ |
| **Analytics** | Basic | Detailed API metrics |
| **Developer portal** | ❌ | ✅ |

### API Gateway Responsibilities

```
┌─────────────────────────────────────────────────┐
│  CROSS-CUTTING CONCERNS                         │
│                                                 │
│  • Authentication & Authorization               │
│  • Rate limiting & Quotas                       │
│  • Request/Response transformation              │
│  • Protocol translation (REST → gRPC)           │
│  • Caching                                      │
│  • Logging & Analytics                          │
│  • Circuit breaking                             │
└─────────────────────────────────────────────────┘
```

### Common API Gateways

- **Kong** — Open source, plugin ecosystem
- **AWS API Gateway** — Serverless, managed
- **Apigee** — Enterprise, full lifecycle
- **Tyk** — Open source, performance focused
- **Azure API Management** — Microsoft ecosystem

### When Do You Need One?

| Scenario | Need API Gateway? |
|----------|------------------|
| Single monolith | Probably not |
| Few microservices, internal | Maybe |
| Public API | Yes |
| Multiple consumers (web, mobile, partners) | Yes |
| Complex auth requirements | Yes |
| Rate limiting per customer | Yes |

### Mental Model

> "I am the API brain 🧠 — I enforce rules, contracts, and behavior."

---

## 6. Ingress (Kubernetes)

### What It Is

**Ingress is NOT a server.** It's a Kubernetes **resource** (configuration).

The actual traffic is handled by an **Ingress Controller** (which IS a reverse proxy).

### The Relationship

```
Ingress Resource (YAML config)
        ↓ read by
Ingress Controller (Nginx, Traefik, etc.)
        ↓ routes traffic to
Kubernetes Services
        ↓ forward to
Pods (your containers)
```

### Ingress Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
```

### What Ingress Does

- **Host-based routing**: `api.example.com` vs `admin.example.com`
- **Path-based routing**: `/users` vs `/orders`
- **TLS termination**: HTTPS certificates
- **Basic load balancing**: Round-robin to pods

### What Ingress Does NOT Do

- Authentication (use API Gateway or service mesh)
- Rate limiting (basic, use API Gateway)
- Request transformation
- API versioning

### Common Ingress Controllers

- **nginx-ingress** — Most common, battle-tested
- **Traefik** — Auto-discovery, Let's Encrypt
- **HAProxy Ingress** — High performance
- **Kong Ingress** — API Gateway features
- **Istio Gateway** — Service mesh integration

### Mental Model

> "Ingress is the rulebook. The Ingress Controller is the bouncer."

---

## 7. How They Fit Together

### Classic Backend

```
Internet
   ↓
Load Balancer (AWS ALB)
   ↓
Reverse Proxy (Nginx)
   ↓
Application Servers
```

### Microservices (non-Kubernetes)

```
Internet
   ↓
Load Balancer
   ↓
API Gateway
   ↓
Service A ←→ Service B ←→ Service C
```

### Kubernetes Stack

```
Internet
   ↓
Cloud Load Balancer (L4)
   ↓
Ingress Controller (L7)
   ↓
[API Gateway - optional]
   ↓
Services → Pods
```

### Enterprise Stack

```
Internet
   ↓
CDN (CloudFlare, Akamai)
   ↓
WAF (Web Application Firewall)
   ↓
Cloud Load Balancer
   ↓
API Gateway
   ↓
Service Mesh (Istio, Linkerd)
   ↓
Services
```

---

## 8. Decision Guide

### When to Use What

| Need | Solution |
|------|----------|
| Distribute traffic to multiple instances | Load Balancer |
| Hide backend infrastructure | Reverse Proxy |
| Route by URL path | Reverse Proxy or Ingress |
| TLS termination | Reverse Proxy or Load Balancer |
| Authentication for APIs | API Gateway |
| Rate limiting per user | API Gateway |
| Block IPs/ports | Firewall |
| Kubernetes HTTP routing | Ingress |
| Control outbound traffic | Forward Proxy |

### Overlap Is Normal

These components **overlap** and **stack**. A real system might have:

```
Firewall → Load Balancer → Reverse Proxy → API Gateway → Services
```

Each layer adds specific capabilities.

---

## 9. Anti-Patterns

### ❌ Opening All Ports "Because We Have Nginx"

Nginx handles HTTP but doesn't close network access. Use a firewall.

### ❌ Putting Business Logic in Reverse Proxy

Keep routing rules simple. Complex logic belongs in your application.

### ❌ API Gateway as the Only Security Layer

Defense in depth. Don't rely on a single component.

### ❌ Load Balancing Stateful Services Like Databases

Databases need leader election, not round-robin. Use proper clustering.

### ❌ Skipping Health Checks

Load balancers need `/health` endpoints to route traffic correctly.

---

## 10. Comparison Table

| Component | Position | Layer | Primary Job |
|-----------|----------|-------|-------------|
| Forward Proxy | Client side | L7 | Control outbound |
| Firewall | Network edge | L3-4 | Block/allow connections |
| Load Balancer | Server side | L4/L7 | Distribute traffic |
| Reverse Proxy | Server side | L7 | Route and protect |
| Ingress | K8s cluster edge | L7 | HTTP routing rules |
| API Gateway | Service edge | L7 | API policies and security |

---

## TL;DR

| Component | One-Line Summary |
|-----------|------------------|
| **Forward Proxy** | Protects clients |
| **Reverse Proxy** | Protects servers |
| **Load Balancer** | Spreads traffic |
| **Firewall** | Network security |
| **Ingress** | Kubernetes routing config |
| **API Gateway** | API rules + security + observability |

Or even shorter:

> **Firewall blocks.
> Load balancers spread.
> Reverse proxies route.
> API Gateways think.**

---

## Quick Reference

### Nginx Load Balancer Config

```nginx
upstream backend {
    least_conn;
    server backend1:8000 weight=3;
    server backend2:8000;
    server backend3:8000;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Health Check Endpoint

```python
@app.get("/health")
def health():
    return {"status": "healthy"}
```

### Interview Talking Points

1. "A load balancer is a specialized reverse proxy focused on distributing traffic"
2. "Firewalls operate at L3-4 (IPs, ports); reverse proxies at L7 (HTTP)"
3. "API Gateways handle cross-cutting concerns like auth, rate limiting, and transformation"
4. "In Kubernetes, Ingress is configuration; the Ingress Controller is the actual proxy"
5. "Defense in depth: layer firewall, proxy, and gateway for comprehensive security"
