# Networking Fundamentals for System Design

> **Goal**: Understand how web traffic flows from client to server, how services communicate, and the mental models that make distributed systems intuitive.

---

## 1. The Request Journey

When you type `https://api.example.com/users` in your browser, here's what happens:

```
┌─────────────────────────────────────────────────────────────────┐
│  1. DNS Resolution                                              │
│     api.example.com → 203.0.113.10                              │
│                                                                 │
│  2. TCP Connection                                              │
│     Connect to 203.0.113.10:443                                 │
│                                                                 │
│  3. TLS Handshake (SNI: "api.example.com")                      │
│     Secure the connection, select certificate                   │
│                                                                 │
│  4. HTTP Request                                                │
│     GET /users HTTP/1.1                                         │
│     Host: api.example.com                                       │
│                                                                 │
│  5. Server Processing → Response                                │
└─────────────────────────────────────────────────────────────────┘
```

**Key insight**: The hostname is sent twice:
1. **During TLS** (SNI) — to select the right certificate
2. **In HTTP** (Host header) — to route to the right application

---

## 2. IP Addresses, Ports, and Protocols

### The Mental Model

| Component | Identifies | Analogy |
|-----------|-----------|---------|
| **IP Address** | Which machine | Street address |
| **Port** | Which service | Apartment number |
| **Protocol** | How to communicate | Language spoken |

### Default Ports (Convention, Not Law)

| Protocol | Default Port | Can Change? |
|----------|-------------|-------------|
| HTTP | 80 | Yes |
| HTTPS | 443 | Yes |
| PostgreSQL | 5432 | Yes |
| Redis | 6379 | Yes |
| MongoDB | 27017 | Yes |

> **Pro tip**: Default ports are just conventions. HTTP works on any port. The port identifies the service, not the protocol.

### Special IP Addresses

```
127.0.0.1 / localhost  → This machine only (loopback)
0.0.0.0                → Listen on ALL interfaces
192.168.x.x / 10.x.x.x → Private network ranges (not routable on internet)
```

---

## 3. Web Servers: What They Actually Do

A **web server** is a program that:
1. **Listens** on an IP + port
2. **Receives** network requests
3. **Responds** with data

### The Zoo of Web Servers

| Type | Examples | Role |
|------|----------|------|
| **HTTP Servers** | Nginx, Apache | Static files, reverse proxy, TLS |
| **Application Servers** | Uvicorn, Gunicorn, Puma | Run your code |
| **Frameworks** | Express, FastAPI, Rails | Your application logic |

### Production Architecture

```
Internet → Nginx (reverse proxy) → Uvicorn (app server) → FastAPI (your code)
```

**Why this layering?**
- TLS termination at Nginx
- Static file serving without hitting Python
- Connection management and buffering
- Security hardening
- Load balancing

---

## 4. SNI: How HTTPS Hosts Multiple Sites

**The Problem**: Server must select a TLS certificate *before* reading the HTTP request. But which certificate?

**The Solution**: SNI (Server Name Indication)

```
┌─────────────────────────────────────────────────┐
│  TLS Handshake                                  │
│  Client: "I want to connect to api.example.com" │
│  Server: "Here's the cert for api.example.com"  │
└─────────────────────────────────────────────────┘
```

**Result**: One IP, one port (443), many websites.

```
IP:443
 ├── SNI: api.example.com    → Cert A → API backend
 ├── SNI: www.example.com    → Cert B → Website frontend
 └── SNI: admin.example.com  → Cert C → Admin panel
```

> **Historical note**: Before SNI, each HTTPS site needed its own IP address. SNI made shared hosting possible for HTTPS.

---

## 5. Containers and Network Isolation

A container has its own:
- Network namespace
- IP address
- Port space

```
┌─────────────────────────────────────────────────┐
│  HOST MACHINE                                   │
│  ┌─────────────┐  ┌─────────────┐               │
│  │ Container A │  │ Container B │               │
│  │ IP: 172.17.0.2│  │ IP: 172.17.0.3│           │
│  │ Port: 8000  │  │ Port: 8000  │               │
│  └─────────────┘  └─────────────┘               │
│                                                 │
│  Both use port 8000 — no conflict!              │
└─────────────────────────────────────────────────┘
```

### Port Mapping

Container ports are **internal**. To expose them:

```bash
docker run -p 80:8000 myapp
#            ↑    ↑
#          host  container
```

> **Common gotcha**: `localhost` inside a container means *that container*, not the host machine.

---

## 6. Service Discovery: Names Over IPs

### Why Not Use IP Addresses?

IPs are:
- **Ephemeral** — containers restart with new IPs
- **Environment-specific** — different in dev/staging/prod
- **Incompatible with scaling** — multiple instances = multiple IPs

### The Solution: DNS-Based Service Discovery

```python
# ✅ Good — uses service name
redis = Redis.from_url("redis://redis:6379")

# ❌ Bad — hardcoded IP
redis = Redis.from_url("redis://172.18.0.5:6379")
```

### How It Works

```
┌─────────────────────────────────────────────────┐
│  Your Code: connect to "redis"                  │
│       ↓                                         │
│  Internal DNS: redis → 172.18.0.5               │
│       ↓                                         │
│  Connection established                         │
│                                                 │
│  When Redis restarts with new IP:               │
│  Internal DNS: redis → 172.18.0.9 (updated!)    │
│                                                 │
│  Your code: unchanged   ✔️✔️                   │
└─────────────────────────────────────────────────┘
```

Docker Compose, Kubernetes, and cloud platforms all provide automatic DNS for service names.

---

## 7. URLs Decoded

```
postgresql://user:pass@postgres:5432/mydb
└────┬────┘ └───┬───┘ └──┬───┘ └─┬─┘ └─┬┘
  scheme    auth     host    port  path
```

| Part | Purpose | Who Uses It |
|------|---------|-------------|
| **Scheme** | Protocol to speak | Client library |
| **Auth** | Credentials | Server authentication |
| **Host** | Where to connect | DNS resolution |
| **Port** | Which service | TCP connection |
| **Path** | Resource identifier | Application routing |

### Scheme ≠ HTTP

```
http://     → HTTP protocol
https://    → HTTP over TLS
redis://    → Redis protocol (not HTTP!)
postgresql://→ PostgreSQL wire protocol
amqp://     → Message queue protocol
```

> **Key insight**: The scheme tells the *client library* how to communicate, not the network.

---

## 8. APIs vs Websites: Same Protocol, Different Content

From the network's perspective, there is **no difference**:

```
Both are just:  HTTP Request → Server → HTTP Response
```

| Aspect | Website | API |
|--------|---------|-----|
| Returns | HTML, CSS, JS | JSON, XML |
| Consumer | Browser (human) | Code (machine) |
| Rendering | Visual | Parsed |

A modern website typically:
1. Browser loads HTML/JS
2. JS calls the API
3. API returns JSON
4. JS renders the UI

---

## 9. Virtual Networks and Subnets

### What's a Subnet?

A **subnet** is a range of IP addresses that form a private network.

```
Subnet: 172.18.0.0/16
        └──┬──┘ └┬┘
      network  mask (65,536 addresses)

Available IPs: 172.18.0.1 to 172.18.255.254
```

### Container Communication

Containers on the **same network** can:
- Communicate directly via names
- Reach each other on any port
- Not need port mapping for internal traffic

```
┌─────────────────────────────────────────────────┐
│  Docker Network: my-app-network (172.18.0.0/16) │
│                                                 │
│  frontend (172.18.0.2) ──→ backend:8000 ✔️✔️   │
│  backend  (172.18.0.3) ──→ postgres:5432 ✔️✔️  │
│  postgres (172.18.0.4)                          │
└─────────────────────────────────────────────────┘
```

---

## 10. Practical Patterns

### Pattern: Environment-Based Configuration

```python
import os

DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://localhost:5432/dev")
REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379")
```

- **Development**: localhost (services on host machine)
- **Docker Compose**: service names (redis, postgres)
- **Kubernetes**: service names (redis.default.svc.cluster.local)
- **Production**: managed service URLs

### Pattern: Health Checks

```python
@app.get("/health")
def health():
    return {"status": "ok", "service": "user-api"}

@app.get("/ready")
def ready():
    # Check dependencies
    db_ok = check_database()
    redis_ok = check_redis()
    return {"ready": db_ok and redis_ok}
```

- `/health` — "Am I running?" (for restarts)
- `/ready` — "Can I serve traffic?" (for load balancers)

---

## 11. Mental Model Summary

```
┌─────────────────────────────────────────────────┐
│  IP Address    → Which machine                  │
│  Port          → Which service                  │
│  Protocol      → How to communicate             │
│  DNS           → Names to IPs                   │
│  SNI           → Which certificate (TLS)        │
│  Host Header   → Which application (HTTP)       │
│  Container     → Isolated network namespace     │
│  Service Name  → Logical destination            │
└─────────────────────────────────────────────────┘
```

---

## TL;DR

- **Port** identifies a service, not a page or website
- **SNI + Host header** allow multiple sites on one IP:443
- **Containers** have isolated networks; use service names, not IPs
- **DNS** enables scaling without code changes
- **URLs** encode protocol + location + resource in one string
- **Websites and APIs** are both HTTP — different content, same protocol

---

## Quick Reference

### Common Debugging Commands

```bash
# DNS lookup
nslookup api.example.com
dig api.example.com

# Test TCP connectivity
telnet api.example.com 443
nc -zv api.example.com 443

# HTTP request
curl -v https://api.example.com/health

# Check listening ports
netstat -tlnp
ss -tlnp

# Docker network inspection
docker network ls
docker network inspect my-network
```

### Interview Talking Points

1. "A port identifies a service, not a specific page or application"
2. "SNI allows multiple HTTPS sites to share one IP address"
3. "In containerized environments, always use service names, never IPs"
4. "The scheme in a URL tells the client library which protocol to speak"
5. "From the network's perspective, APIs and websites are identical — just HTTP"
