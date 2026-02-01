# Architectural Patterns

> **Goal**: Understand the major architectural styles, their trade-offs, and when to use each pattern for system design.

---

## 1. Architecture Evolution

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE EVOLUTION                           │
│                                                                     │
│  Monolith ──→ Modular Monolith ──→ Microservices ──→ Serverless     │
│                                                                     │
│  Simple                                                Complex      │
│  Coupled                                              Decoupled     │
│  Easy deployment                                    Complex ops     │
│  Vertical scale                                   Horizontal scale  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Monolithic Architecture

### What It Is

```
┌─────────────────────────────────────────────────┐
│              MONOLITH                           │
│  ┌─────────────────────────────────────────┐    │
│  │  UI Layer                               │    │
│  ├─────────────────────────────────────────┤    │
│  │  Business Logic                         │    │
│  │  - User Service                         │    │
│  │  - Order Service                        │    │
│  │  - Payment Service                      │    │
│  │  - Inventory Service                    │    │
│  ├─────────────────────────────────────────┤    │
│  │  Data Access Layer                      │    │
│  └─────────────────────────────────────────┘    │
│                    ↓                            │
│              [Database]                         │
└─────────────────────────────────────────────────┘
```

### Characteristics

| Aspect | Description |
|--------|-------------|
| **Deployment** | Single unit |
| **Database** | Usually shared |
| **Communication** | In-process function calls |
| **Scaling** | Entire application scales together |
| **Team structure** | One team, one codebase |

### Advantages

```
✅ Simple to develop initially
✅ Simple to test (everything in one place)
✅ Simple to deploy (one artifact)
✅ Easy debugging (single process)
✅ No network latency between components
✅ ACID transactions trivial
```

### Disadvantages

```
❌ Grows into "big ball of mud"
❌ Hard to scale specific components
❌ One bug can crash everything
❌ Long deployment cycles
❌ Technology lock-in
❌ Team coordination becomes bottleneck
```

### When to Use

- Early-stage startups
- Small teams (< 10 engineers)
- Well-understood domain
- Simple scaling needs

> **Pro tip**: Start with a monolith. Extract microservices when you have clear bounded contexts and team scaling needs.

---

## 3. Modular Monolith

### What It Is

A monolith with **clear module boundaries** that could become microservices.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MODULAR MONOLITH                                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                 │
│  │    User      │ │    Order     │ │   Payment    │                 │
│  │   Module     │ │   Module     │ │   Module     │                 │
│  │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │                 │
│  │ │  API     │ │ │ │  API     │ │ │ │  API     │ │                 │
│  │ │ Service  │ │ │ │ Service  │ │ │ │ Service  │ │                 │
│  │ │ Domain   │ │ │ │ Domain   │ │ │ │ Domain   │ │                 │
│  │ │ Data     │ │ │ │ Data     │ │ │ │ Data     │ │                 │
│  │ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │                 │
│  └──────────────┘ └──────────────┘ └──────────────┘                 │
│         ↓                ↓                ↓                         │
│  ┌─────────────────────────────────────────────────┐                │
│  │               Shared Database                    │               │
│  └─────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
```

### Rules

```
1. Modules communicate via well-defined interfaces (not direct DB queries)
2. Each module has its own tables (no cross-module table access)
3. Shared kernel for common types only
4. Can be extracted to microservice with minimal changes
```

### Best of Both Worlds

| Monolith Benefit | Modular Benefit |
|-----------------|-----------------|
| Simple deployment | Clear boundaries |
| No network latency | Independent modules |
| Easy debugging | Prepared for extraction |
| ACID transactions | Team autonomy |

---

## 4. Microservices Architecture

### What It Is

```
┌─────────────────────────────────────────────────────────────────────┐
│                      MICROSERVICES                                  │
│                                                                     │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐           │
│  │  User   │    │  Order  │    │ Payment │    │Inventory│           │
│  │ Service │    │ Service │    │ Service │    │ Service │           │
│  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘           │
│       │              │              │              │                │
│       ↓              ↓              ↓              ↓                │
│  [User DB]      [Order DB]    [Payment DB]   [Inventory DB]         │
│                                                                     │
│  Each service:                                                      │
│  • Independently deployable                                         │
│  • Owns its data                                                    │
│  • Communicates via network                                         │
│  • Can use different technology                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Principles

```
┌─────────────────────────────────────────────────┐
│  MICROSERVICES PRINCIPLES                       │
│                                                 │
│  1. Single Responsibility                       │
│     Each service does one thing well            │
│                                                 │
│  2. Autonomous                                  │
│     Deploy, scale, fail independently           │
│                                                 │
│  3. Owns Its Data                               │
│     No shared databases                         │
│                                                 │
│  4. Decentralized                               │
│     No central point of control                 │
│                                                 │
│  5. Observable                                  │
│     Logging, tracing, metrics built-in          │
└─────────────────────────────────────────────────┘
```

### Communication Patterns

```
Synchronous (Request/Response):
  Service A ─HTTP/gRPC─→ Service B

Asynchronous (Events):
  Service A ─publish─→ Message Queue ─subscribe─→ Service B
```

### Advantages

```
✅ Independent deployment
✅ Technology flexibility
✅ Scalability per service
✅ Fault isolation
✅ Team autonomy
✅ Easier to understand (small codebase)
```

### Disadvantages

```
❌ Distributed system complexity
❌ Network latency
❌ Data consistency challenges
❌ Operational overhead
❌ Testing complexity
❌ Debugging across services
```

### When to Use

- Large teams (Conway's Law alignment)
- Different scaling requirements per component
- Need for technology diversity
- Complex domain with clear boundaries
- Mature DevOps practices

### When NOT to Use

- Small teams
- Early-stage products
- Unclear domain boundaries
- No DevOps maturity

---

## 5. Service-Oriented Architecture (SOA)

### SOA vs Microservices

```
┌─────────────────────────────────────────────────────────────────────┐
│  SOA                           │  Microservices                     │
│                                │                                    │
│  • Larger, coarser services    │  • Small, fine-grained services    │
│  • Enterprise Service Bus      │  • Direct communication            │
│  • Shared data model           │  • Service owns its data           │
│  • SOAP/XML common             │  • REST/gRPC/JSON common           │
│  • Centralized governance      │  • Decentralized governance        │
│  • Reuse-focused               │  • Replace-focused                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Event-Driven Architecture

### What It Is

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EVENT-DRIVEN                                     │
│                                                                     │
│  ┌──────────┐                                                       │
│  │  Order   │ ──publish──→ OrderCreated                             │
│  │ Service  │                   │                                   │
│  └──────────┘                   ↓                                   │
│                         [Event Broker]                              │
│                               │                                     │
│           ┌───────────────────┼───────────────────┐                 │
│           ↓                   ↓                   ↓                 │
│     ┌──────────┐       ┌──────────┐       ┌──────────┐              │
│     │ Inventory│       │  Email   │       │Analytics │              │
│     │ Service  │       │ Service  │       │ Service  │              │
│     └──────────┘       └──────────┘       └──────────┘              │
│                                                                     │
│  Services react to events, not direct calls                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Event Types

| Type | Description | Example |
|------|-------------|---------|
| **Domain Event** | Business fact | OrderPlaced, PaymentReceived |
| **Integration Event** | Cross-service | UserCreated (for other services) |
| **Event Notification** | Signal something happened | Minimal data |
| **Event-Carried State Transfer** | Full state in event | Complete order details |

### Advantages

```
✅ Loose coupling
✅ Scalability (add consumers easily)
✅ Resilience (async processing)
✅ Auditability (event log)
✅ Temporal decoupling
```

### Disadvantages

```
❌ Eventual consistency
❌ Event ordering challenges
❌ Debugging complexity
❌ Event schema evolution
❌ Duplicate handling required
```

---

## 7. CQRS (Command Query Responsibility Segregation)

### What It Is

Separate models for **reading** and **writing** data.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CQRS                                        │
│                                                                     │
│  Commands (Write)                    Queries (Read)                 │
│  ┌──────────────┐                    ┌──────────────┐               │
│  │ CreateOrder  │                    │ GetOrders    │               │
│  │ UpdateUser   │                    │ SearchUsers  │               │
│  └──────┬───────┘                    └──────┬───────┘               │
│         ↓                                   ↓                       │
│  ┌──────────────┐                    ┌──────────────┐               │
│  │ Write Model  │ ───sync/async───→  │ Read Model   │               │
│  │ (normalized) │                    │ (denormalized)│              │
│  └──────┬───────┘                    └──────┬───────┘               │
│         ↓                                   ↓                       │
│  [Write Database]                    [Read Database]                │
│  (OLTP optimized)                    (Query optimized)              │
└─────────────────────────────────────────────────────────────────────┘
```

### Why CQRS?

```
Traditional: Same model for reads and writes

Problem:
- Read patterns ≠ Write patterns
- Optimizing for one hurts the other
- Complex queries hit write database

CQRS Solution:
- Write model: Normalized, consistent
- Read model: Denormalized, fast queries
- Different scaling strategies
```

### When to Use

- Complex domains
- Different read/write workloads
- Need for different scaling
- Event-sourced systems

### When NOT to Use

- Simple CRUD applications
- Read/write patterns are similar
- Team unfamiliar with pattern

---

## 8. Event Sourcing

### What It Is

Instead of storing **current state**, store **all events** that led to current state.

```
┌─────────────────────────────────────────────────────────────────────┐
│  TRADITIONAL                      │  EVENT SOURCING                 │
│                                   │                                 │
│  Account Table                    │  Event Log                      │
│  ┌──────────────────┐            │  ┌──────────────────────────┐    │
│  │ id: 123          │            │  │ 1. AccountOpened(123)    │    │
│  │ balance: 100     │            │  │ 2. MoneyDeposited(150)   │    │
│  │ name: Alice      │            │  │ 3. MoneyWithdrawn(50)    │    │
│  └──────────────────┘            │  │ 4. NameChanged("Alice")  │    │
│                                   │  └──────────────────────────┘   │
│  Current state stored             │  Replay events = current state  │
└─────────────────────────────────────────────────────────────────────┘
```

### Advantages

```
✅ Complete audit trail
✅ Temporal queries ("What was balance on Jan 1?")
✅ Debugging (replay events)
✅ Rebuild projections from events
✅ Natural fit with event-driven
```

### Disadvantages

```
❌ Complex querying (need projections)
❌ Event versioning
❌ Storage growth
❌ Eventual consistency
❌ Learning curve
```

### Event Store

```python
# Events are immutable, append-only
events = [
    {"type": "AccountOpened", "data": {"id": "123", "name": "Bob"}},
    {"type": "MoneyDeposited", "data": {"amount": 100}},
    {"type": "MoneyWithdrawn", "data": {"amount": 30}},
]

# Rebuild state by replaying
def rebuild_account(events):
    account = {"balance": 0}
    for event in events:
        if event["type"] == "AccountOpened":
            account["id"] = event["data"]["id"]
            account["name"] = event["data"]["name"]
        elif event["type"] == "MoneyDeposited":
            account["balance"] += event["data"]["amount"]
        elif event["type"] == "MoneyWithdrawn":
            account["balance"] -= event["data"]["amount"]
    return account
```

---

## 9. Serverless Architecture

### What It Is

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SERVERLESS                                     │
│                                                                     │
│  Event ──→ [Cloud Function] ──→ Response                            │
│                                                                     │
│  • No server management                                             │
│  • Pay per execution                                                │
│  • Auto-scaling                                                     │
│  • Event-triggered                                                  │
│                                                                     │
│  AWS Lambda, Google Cloud Functions, Azure Functions                │
└─────────────────────────────────────────────────────────────────────┘
```

### Architecture Pattern

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  HTTP ──→ API Gateway ──→ Lambda ──→ DynamoDB                       │
│                                                                     │
│  S3 Upload ──→ Lambda ──→ Process Image ──→ S3                      │
│                                                                     │
│  Schedule ──→ Lambda ──→ Send Reports                               │
│                                                                     │
│  Queue ──→ Lambda ──→ Process Message                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Advantages

```
✅ No infrastructure management
✅ Auto-scaling to zero
✅ Pay per use
✅ Quick to deploy
✅ Event-driven by nature
```

### Disadvantages

```
❌ Cold starts (latency)
❌ Execution time limits
❌ Vendor lock-in
❌ Debugging complexity
❌ State management challenges
❌ Cost unpredictability at scale
```

### When to Use

- Event processing
- APIs with variable traffic
- Scheduled tasks
- Prototypes/MVPs
- Glue code between services

---

## 10. Hexagonal Architecture (Ports & Adapters)

### What It Is

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│                    ┌─────────────────────┐                          │
│    REST API ──────→│                     │←────── Database          │
│    (Adapter)       │                     │        (Adapter)         │
│                    │     DOMAIN CORE     │                          │
│    CLI ───────────→│                     │←────── Message Queue     │
│    (Adapter)       │  (Business Logic)   │        (Adapter)         │
│                    │                     │                          │
│    gRPC ──────────→│                     │←────── Cache             │
│    (Adapter)       │                     │        (Adapter)         │
│                    └─────────────────────┘                          │
│                                                                     │
│  Ports: Interfaces defined by the core                              │
│  Adapters: Implementations that connect to the outside world        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Principles

```
1. Domain core has no external dependencies
2. External systems connect via adapters
3. Ports define contracts (interfaces)
4. Easy to swap implementations
5. Highly testable
```

### Example Structure

```
src/
  domain/           # Pure business logic (no frameworks)
    models/
    services/
  ports/            # Interfaces
    inbound/        # How outside calls us
    outbound/       # How we call outside
  adapters/         # Implementations
    inbound/
      rest_api.py
      grpc_api.py
    outbound/
      postgres_repo.py
      redis_cache.py
```

---

## 11. Architecture Decision Guide

### Decision Matrix

| Factor | Monolith | Microservices | Serverless |
|--------|----------|---------------|------------|
| Team size | Small | Large | Any |
| Complexity | Low-Medium | High | Medium |
| Scaling needs | Uniform | Per-service | Per-function |
| Time to market | Fast | Slow initially | Fastest |
| Operational overhead | Low | High | Very low |
| Cost at low scale | Fixed | Higher | Lowest |
| Cost at high scale | Lower | Optimized | Variable |

### Start Simple, Evolve

```
1. Start: Monolith (or modular monolith)
2. When needed: Extract clear bounded contexts
3. Add: Event-driven communication for decoupling
4. Consider: CQRS for complex read/write patterns
5. Scale: Microservices for independent scaling
```

---

## 12. Anti-Patterns

### Distributed Monolith

```
❌ Microservices that:
   - Share a database
   - Are deployed together
   - Have tight coupling
   - Can't fail independently

This is worse than a monolith (distributed complexity, no benefits)
```

### Nano-Services

```
❌ Too many tiny services
   - Overwhelming operational overhead
   - Network calls for trivial operations
   - Hard to understand the system
```

### Big Ball of Mud

```
❌ No clear architecture
   - Everything depends on everything
   - No boundaries
   - Impossible to change safely
```

---

## TL;DR

| Architecture | Best For |
|--------------|----------|
| **Monolith** | Starting out, small teams, simple domains |
| **Modular Monolith** | Growing teams, preparing for microservices |
| **Microservices** | Large teams, complex domains, scaling needs |
| **Event-Driven** | Decoupled systems, async workflows |
| **CQRS** | Different read/write patterns, complex queries |
| **Event Sourcing** | Audit requirements, temporal queries |
| **Serverless** | Variable workloads, event processing |
| **Hexagonal** | Testability, swappable components |

**Key Principles:**
- Start simple, add complexity when needed
- Match architecture to team structure (Conway's Law)
- Clear boundaries prevent the "big ball of mud"
- Events enable loose coupling
- Every architecture has trade-offs—choose consciously

---

## Quick Reference

### Bounded Context Questions

```
To identify microservice boundaries, ask:
1. Does this concept mean the same in both contexts?
2. Would different teams own this?
3. Could this scale independently?
4. Does this have its own data lifecycle?
```

### Interview Talking Points

1. "Start with a monolith; extract microservices when you have clear boundaries and scaling needs"
2. "Microservices trade simplicity for independent scaling and deployment"
3. "Event-driven architecture enables loose coupling but introduces eventual consistency"
4. "CQRS separates read and write models for different optimization strategies"
5. "The worst outcome is a distributed monolith—distributed complexity with none of the benefits"
