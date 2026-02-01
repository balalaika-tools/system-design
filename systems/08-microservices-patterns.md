# Microservices Patterns

> **Goal**: Understand the essential patterns that make microservices work in production, from communication to resilience to data management.

---

## 1. Service Communication Patterns

### Synchronous Communication

```
┌─────────────────────────────────────────────────────────────────────┐
│  REQUEST/RESPONSE                                                   │
│                                                                     │
│  Service A ──HTTP/gRPC──→ Service B ──response──→ Service A         │
│                                                                     │
│  ✅ Simple mental model ✅                                         │
│  ✅ Immediate response  ✅                                         │
│  ❌ Tight coupling ❌                                              │
│  ❌ Cascading failures ❌                                          │
│  ❌ Both services must be up ❌                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Asynchronous Communication

```
┌─────────────────────────────────────────────────────────────────────┐
│  MESSAGING                                                          │
│                                                                     │
│  Service A ──publish──→ [Message Broker] ──subscribe──→ Service B   │
│                                                                     │
│  ✅ Loose coupling  ✅                                             │
│  ✅ Resilience (queue buffers) ✅                                  │
│  ✅ Services can be down temporarily ✅                            │
│  ❌ Eventual consistency ❌                                        │
│  ❌ More complex ❌                                                │
│  ❌ More complex ❌                                                │
│  ❌ Debugging harder ❌                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### Choosing Communication Style

| Scenario | Recommended |
|----------|-------------|
| Need immediate response | Sync (HTTP/gRPC) |
| Fire and forget | Async (events) |
| Long-running process | Async (events) |
| Query data from another service | Sync or local cache |
| Notify multiple services | Async (pub/sub) |

---

## 2. API Gateway Pattern

### What It Is

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Clients                    API Gateway                Services     │
│                                                                     │
│  Mobile ───┐                ┌──────────┐             ┌──────────┐   │
│            │                │          │             │  Users   │   │
│  Web ──────┼───────────────→│ Gateway  │────────────→│  Orders  │   │
│            │                │          │             │  Payment │   │
│  Partners ─┘                └──────────┘             └──────────┘   │
│                                                                     │
│  Gateway handles:                                                   │
│  • Routing                 • Authentication                         │
│  • Rate limiting           • Request/Response transformation        │
│  • SSL termination         • Load balancing                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Backend for Frontend (BFF)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Different clients, different needs                                 │
│                                                                     │
│  Mobile App ──→ Mobile BFF ──→ Services                             │
│  Web App ────→ Web BFF ────→ Services                               │
│  Partners ───→ Partner API ──→ Services                             │
│                                                                     │
│  Each BFF optimized for its client's needs                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Service Discovery

### The Problem

```
┌─────────────────────────────────────────────────────────────────────┐
│  In microservices, services scale up/down dynamically.              │
│                                                                     │
│  Order Service needs to call User Service.                          │
│  But User Service has 5 instances with different IPs.               │
│  IPs change when instances restart.                                 │
│                                                                     │
│  How does Order Service know where to send requests?                │
└─────────────────────────────────────────────────────────────────────┘
```

### Client-Side Discovery

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  ┌──────────┐     1. Query     ┌──────────────┐                     │
│  │  Order   │ ───────────────→ │   Service    │                     │
│  │ Service  │ ←─────────────── │   Registry   │                     │
│  └──────────┘   2. Return IPs  │  (Consul,    │                     │
│       │                        │   Eureka)    │                     │
│       │ 3. Direct call         └──────────────┘                     │
│       ↓                              ↑                              │
│  ┌──────────┐                        │ Register                     │
│  │  User    │ ───────────────────────┘                              │
│  │ Service  │                                                       │
│  └──────────┘                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Server-Side Discovery

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  ┌──────────┐               ┌──────────────┐      ┌──────────┐      │
│  │  Order   │ ────────────→ │    Load      │ ───→ │  User    │      │
│  │ Service  │               │   Balancer   │      │ Service  │      │
│  └──────────┘               └──────────────┘      └──────────┘      │
│                                    ↑                    │           │
│                                    │     Register       │           │
│                              ┌─────┴─────┐              │           │
│                              │  Service  │ ←────────────┘           │
│                              │  Registry │                          │
│                              └───────────┘                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Kubernetes Service Discovery

```yaml
# Services are automatically discoverable via DNS
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
    - port: 80
      targetPort: 8080

# Other services call: http://user-service/api/users
# Kubernetes DNS resolves the name
```

---

## 4. Circuit Breaker Pattern

### The Problem

```
┌─────────────────────────────────────────────────────────────────────┐
│  Cascading Failure                                                  │
│                                                                     │
│  Order Service → User Service (down)                                │
│       ↓                                                             │
│  Order Service hangs waiting                                        │
│       ↓                                                             │
│  Order Service thread pool exhausted                                │
│       ↓                                                             │
│  Order Service becomes unresponsive                                 │
│       ↓                                                             │
│  Callers of Order Service fail too                                  │
│       ↓                                                             │
│  System-wide outage                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

### The Solution

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CIRCUIT BREAKER STATES                           │
│                                                                     │
│  ┌─────────┐    Failure threshold    ┌─────────┐                    │
│  │ CLOSED  │ ─────────reached──────→ │  OPEN   │                    │
│  │ (normal)│                         │ (fail   │                    │
│  └─────────┘                         │  fast)  │                    │
│       ↑                              └────┬────┘                    │
│       │                                   │                         │
│       │      Success                      │ Timeout                 │
│       │                                   ↓                         │
│       │                            ┌───────────┐                    │
│       └────────────────────────────│HALF-OPEN  │                    │
│                                    │(test with │                    │
│                                    │ one call) │                    │
│                                    └───────────┘                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementation

```python
import circuitbreaker

@circuitbreaker.circuit(failure_threshold=5, recovery_timeout=30)
def call_user_service(user_id):
    response = requests.get(f"http://user-service/users/{user_id}")
    return response.json()

# Usage
try:
    user = call_user_service(123)
except circuitbreaker.CircuitBreakerError:
    # Circuit is open - return cached/default data
    user = get_cached_user(123) or {"id": 123, "name": "Unknown"}
```

### States Explained

| State | Behavior |
|-------|----------|
| **Closed** | Normal operation, requests pass through |
| **Open** | Fail fast, no requests sent (return fallback) |
| **Half-Open** | Allow one test request to check recovery |

---

## 5. Retry Pattern

### Basic Retry

```python
import tenacity

@tenacity.retry(
    stop=tenacity.stop_after_attempt(3),
    wait=tenacity.wait_exponential(multiplier=1, min=1, max=10),
    retry=tenacity.retry_if_exception_type(requests.RequestException)
)
def call_external_api():
    response = requests.get("http://external-api/resource")
    response.raise_for_status()
    return response.json()
```

### Exponential Backoff

```
Attempt 1: wait 1s
Attempt 2: wait 2s
Attempt 3: wait 4s
Attempt 4: wait 8s (capped at max)
```

### With Jitter

```
Add randomness to prevent thundering herd:

Attempt 1: wait 1s + random(0, 0.5s)
Attempt 2: wait 2s + random(0, 1s)
...
```

### Retry Guidelines

```
✅ Retry on: Network errors, 5xx, 429 (rate limit)
❌ Don't retry on: 4xx (client error), validation errors
⚠️ Make operations idempotent before retrying
```

---

## 6. Bulkhead Pattern

### The Problem

```
┌─────────────────────────────────────────────────────────────────────┐
│  Shared thread pool                                                 │
│                                                                     │
│  ┌────────────────────────────────────────────┐                     │
│  │             Thread Pool (100)              │                     │
│  │  Service A calls ████████████████████████ │ ← All consumed!      │
│  │  Service B calls                           │ ← Can't run         │
│  │  Service C calls                           │ ← Can't run         │
│  └────────────────────────────────────────────┘                     │
│                                                                     │
│  Slow Service A blocks everything                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### The Solution

```
┌─────────────────────────────────────────────────────────────────────┐
│  Isolated thread pools (bulkheads)                                  │
│                                                                     │
│  ┌──────────────────┐                                               │
│  │ Service A Pool   │ ████████████████████ (saturated)              │
│  │     (30)         │                                               │
│  └──────────────────┘                                               │
│  ┌──────────────────┐                                               │
│  │ Service B Pool   │ ████                 (works fine)             │
│  │     (40)         │                                               │
│  └──────────────────┘                                               │
│  ┌──────────────────┐                                               │
│  │ Service C Pool   │ ██████               (works fine)             │
│  │     (30)         │                                               │
│  └──────────────────┘                                               │
│                                                                     │
│  Failure in one doesn't affect others                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Saga Pattern

### The Problem

```
┌─────────────────────────────────────────────────────────────────────┐
│  Distributed Transaction                                            │
│                                                                     │
│  Create Order:                                                      │
│  1. Reserve inventory (Inventory Service)                           │
│  2. Process payment (Payment Service)                               │
│  3. Create order (Order Service)                                    │
│  4. Send notification (Notification Service)                        │
│                                                                     │
│  If step 3 fails, need to:                                          │
│  • Refund payment                                                   │
│  • Release inventory                                                │
│                                                                     │
│  No distributed ACID transactions!                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Choreography-Based Saga

```
┌─────────────────────────────────────────────────────────────────────┐
│  Services coordinate via events                                     │
│                                                                     │
│  1. Order Service ──publishes──→ OrderCreated                       │
│  2. Inventory Service ←──listens, reserves, publishes──→ Reserved   │
│  3. Payment Service ←──listens, charges, publishes──→ PaymentDone   │
│  4. Order Service ←──listens──→ OrderConfirmed                      │
│                                                                     │
│  On failure:                                                        │
│  Payment fails ──publishes──→ PaymentFailed                         │
│  Inventory Service ←──listens──→ releases inventory                 │
│                                                                     │
│  ✅ Decoupled ✅                                                   │
│  ❌ Hard to track flow ❌                                          │
│  ❌ Complex debugging  ❌                                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Orchestration-Based Saga

```
┌─────────────────────────────────────────────────────────────────────┐
│  Central orchestrator coordinates                                   │
│                                                                     │
│                    ┌──────────────┐                                 │
│                    │ Orchestrator │                                 │
│                    └──────┬───────┘                                 │
│          ┌────────────────┼────────────────┐                        │
│          ↓                ↓                ↓                        │
│    ┌──────────┐    ┌──────────┐    ┌──────────┐                     │
│    │Inventory │    │ Payment  │    │  Order   │                     │
│    │ Service  │    │ Service  │    │ Service  │                     │
│    └──────────┘    └──────────┘    └──────────┘                     │
│                                                                     │
│  Orchestrator:                                                      │
│  1. Call Inventory → Reserve                                        │
│  2. Call Payment → Charge                                           │
│  3. Call Order → Create                                             │
│  4. If any fails → Call compensating actions                        │
│                                                                     │
│  ✅ Clear flow ✅                                                  │
│  ✅ Easier debugging ✅                                            │
│  ❌ Orchestrator is single point ❌                                │
│  ❌ Tighter coupling ❌                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### Compensating Transactions

```python
class CreateOrderSaga:
    def execute(self, order):
        try:
            # Step 1
            inventory_reservation = inventory_service.reserve(order.items)
            
            # Step 2
            payment = payment_service.charge(order.total)
            
            # Step 3
            order_record = order_service.create(order)
            
        except PaymentError:
            # Compensate step 1
            inventory_service.release(inventory_reservation.id)
            raise
            
        except OrderCreationError:
            # Compensate step 2
            payment_service.refund(payment.id)
            # Compensate step 1
            inventory_service.release(inventory_reservation.id)
            raise
```

---

## 8. Strangler Fig Pattern

### What It Is

Gradually replace a legacy system by routing traffic to new services.

```
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 1: Initial                                                   │
│                                                                     │
│  All traffic ──→ Legacy Monolith                                    │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  PHASE 2: Partial Migration                                         │
│                                                                     │
│  Traffic ──→ Router ──/users──→ New User Service                    │
│                     ──/other──→ Legacy Monolith                     │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  PHASE 3: More Migration                                            │
│                                                                     │
│  Traffic ──→ Router ──/users──→ User Service                        │
│                     ──/orders─→ Order Service                       │
│                     ──/other──→ Legacy (shrinking)                  │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  PHASE 4: Complete                                                  │
│                                                                     │
│  Traffic ──→ Router ──→ Microservices (Legacy decommissioned)       │
└─────────────────────────────────────────────────────────────────────┘
```

### Benefits

```
✅ Incremental migration
✅ Lower risk (can rollback)
✅ Continuous delivery
✅ Learn as you go
```

---

## 9. Sidecar Pattern

### What It Is

```
┌─────────────────────────────────────────────────────────────────────┐
│  Pod / Container Group                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  ┌──────────────┐    ┌──────────────┐                       │    │
│  │  │    Main      │    │   Sidecar    │                       │    │
│  │  │ Application  │←──→│   (Envoy)    │←──→ Network           │    │
│  │  └──────────────┘    └──────────────┘                       │    │
│  │                                                             │    │
│  │  Sidecar handles:                                           │    │
│  │  • Service discovery                                        │    │
│  │  • Load balancing                                           │    │
│  │  • TLS / mTLS                                               │    │
│  │  • Retries, circuit breaking                                │    │
│  │  • Observability (metrics, traces)                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Service Mesh

When every service has a sidecar proxy, you get a **service mesh**.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SERVICE MESH                                 │
│                                                                     │
│  ┌─────────────────┐         ┌─────────────────┐                    │
│  │ Service A       │         │ Service B       │                    │
│  │ ┌───┐  ┌─────┐  │         │ ┌─────┐  ┌───┐  │                    │
│  │ │App│←→│Proxy│ ←┼─────────┼→│Proxy│←→│App│  │                    │
│  │ └───┘  └─────┘  │         │ └─────┘  └───┘  │                    │
│  └─────────────────┘         └─────────────────┘                    │
│                                     ↑                               │
│                              ┌──────┴──────────┐                    │
│                              │ Control Plane   │                    │
│                              │ (Istio/Linkerd) │                    │
│                              └─────────────────┘                    │
│                                                                     │
│  Control Plane manages:                                             │
│  • Configuration                                                    │
│  • Certificates                                                     │
│  • Policy                                                           │
│  • Telemetry                                                        │
└─────────────────────────────────────────────────────────────────────┘
```

### Popular Service Meshes

| Mesh | Notes |
|------|-------|
| **Istio** | Feature-rich, complex |
| **Linkerd** | Simple, lightweight |
| **Consul Connect** | HashiCorp ecosystem |

---

## 10. Database per Service

### The Pattern

```
┌─────────────────────────────────────────────────────────────────────┐
│  Each service owns its data                                         │
│                                                                     │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                         │
│  │  Order   │   │  User    │   │ Payment  │                         │
│  │ Service  │   │ Service  │   │ Service  │                         │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘                         │
│       │              │              │                               │
│       ↓              ↓              ↓                               │
│  [Order DB]     [User DB]     [Payment DB]                          │
│  PostgreSQL      MongoDB        PostgreSQL                          │
│                                                                     │
│  ✅ Loose coupling ✅                                              │
│  ✅ Right database for each service ✅                             │
│  ❌ No joins across services ❌                                    │
│  ❌ Distributed transactions are hard ❌                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Querying Across Services

```
Option 1: API Composition
Order Service calls User Service API to get user details

Option 2: Data Replication
User events → replicate to Order Service's read model

Option 3: Shared Read Database
Event-sourced write, materialized views for reads
```

---

## 11. Health Checks

### Liveness vs Readiness

```
┌─────────────────────────────────────────────────────────────────────┐
│  LIVENESS: "Is the process running?"                                │
│  • Failed → Restart container                                       │
│  • Check: Process not deadlocked                                    │
│                                                                     │
│  READINESS: "Can it handle traffic?"                                │
│  • Failed → Remove from load balancer                               │
│  • Check: Dependencies available                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementation

```python
@app.get("/health/live")
def liveness():
    return {"status": "alive"}

@app.get("/health/ready")
def readiness():
    db_ok = check_database_connection()
    redis_ok = check_redis_connection()
    
    if db_ok and redis_ok:
        return {"status": "ready"}
    else:
        raise HTTPException(status_code=503, detail="Not ready")
```

---

## 12. Observability

### Three Pillars

```
┌─────────────────────────────────────────────────────────────────────┐
│                       OBSERVABILITY                                 │
│                                                                     │
│  LOGS                METRICS              TRACES                    │
│  What happened       How much             Where did it go           │
│                                                                     │
│  • Structured JSON   • Counters           • Distributed tracing     │
│  • Log aggregation   • Gauges             • Request flow            │
│  • Searchable        • Histograms         • Latency breakdown       │
│                                                                     │
│  Tools:              Tools:               Tools:                    │
│  ELK, Loki          Prometheus, Grafana  Jaeger, Zipkin             │
└─────────────────────────────────────────────────────────────────────┘
```

### Correlation IDs

```python
# Pass correlation ID through all services
@app.middleware("http")
async def add_correlation_id(request, call_next):
    correlation_id = request.headers.get("X-Correlation-ID", str(uuid.uuid4()))
    
    # Add to context
    request.state.correlation_id = correlation_id
    
    # Include in response
    response = await call_next(request)
    response.headers["X-Correlation-ID"] = correlation_id
    
    return response
```

---

## TL;DR

| Pattern | Purpose |
|---------|---------|
| **API Gateway** | Single entry point, cross-cutting concerns |
| **Service Discovery** | Find service instances dynamically |
| **Circuit Breaker** | Prevent cascading failures |
| **Retry + Backoff** | Handle transient failures |
| **Bulkhead** | Isolate failures |
| **Saga** | Distributed transactions |
| **Strangler Fig** | Incremental migration |
| **Sidecar / Service Mesh** | Infrastructure concerns out of app |
| **Database per Service** | Loose coupling, data ownership |

**Key Principles:**
- Design for failure (things will break)
- Async communication when possible
- Circuit breakers prevent cascade
- Sagas for distributed transactions
- Observe everything (logs, metrics, traces)

---

## Quick Reference

### Circuit Breaker Config

```python
@circuitbreaker.circuit(
    failure_threshold=5,    # Open after 5 failures
    recovery_timeout=30,    # Try again after 30s
    expected_exception=Exception
)
```

### Retry Config

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(min=1, max=10),
    retry=retry_if_exception_type(RequestException)
)
```

### Interview Talking Points

1. "Circuit breakers prevent cascading failures by failing fast when a service is down"
2. "Sagas manage distributed transactions through compensating actions"
3. "Service mesh moves infrastructure concerns (retries, TLS, tracing) out of application code"
4. "Each microservice should own its data to maintain loose coupling"
5. "Correlation IDs are essential for tracing requests across services"
