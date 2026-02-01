# Database Scaling and Data Management

> **Goal**: Understand how databases scale differently from stateless services, the trade-offs of each approach, and how to choose the right strategy.

---

## 1. Stateless vs Stateful: Why Databases Are Hard

### Stateless Services (Easy to Scale)

```
┌─────────────────────────────────────────────────┐
│  FastAPI / Node.js / Go services                │
│                                                 │
│  • No persistent data                           │
│  • Any instance handles any request             │
│  • Scale: add instances + load balancer         │
│  • Fail: restart, no data loss                  │
└─────────────────────────────────────────────────┘

Request → Load Balancer → Any Instance → Response
```

### Stateful Services (Hard to Scale)

```
┌─────────────────────────────────────────────────┐
│  Databases                                      │
│                                                 │
│  • Own persistent data                          │
│  • Requests depend on data location             │
│  • Scale: requires data partitioning            │
│  • Fail: risk of data loss/inconsistency        │
└─────────────────────────────────────────────────┘

Request → Where is this data? → Correct Node → Response
```

> **Core principle**: Stateless services scale with load balancers. Stateful services scale with data partitioning.

---

## 2. Scaling Strategies Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SCALING LADDER                                 │
│                                                                     │
│  ┌──────────────────┐                                               │
│  │ VERTICAL SCALING │ ← Start here                                  │
│  │ Bigger machine   │                                               │
│  └────────┬─────────┘                                               │
│           ↓                                                         │
│  ┌──────────────────┐                                               │
│  │ READ REPLICAS    │ ← Read-heavy workloads                        │
│  │ Scale reads      │                                               │
│  └────────┬─────────┘                                               │
│           ↓                                                         │
│  ┌──────────────────┐                                               │
│  │ SHARDING         │ ← Large datasets, high writes                 │
│  │ Partition data   │                                               │
│  └────────┬─────────┘                                               │
│           ↓                                                         │
│  ┌──────────────────┐                                               │
│  │ DISTRIBUTED DB   │ ← Global scale                                │
│  │ Built for scale  │                                               │
│  └──────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Vertical Scaling (Scale Up)

### What It Is

Make your single database **bigger and faster**:
- More CPU cores
- More RAM (fit more data in memory)
- Faster disks (NVMe SSDs)
- Better network

### When It Works

| Database | Good Vertical Scaling? |
|----------|----------------------|
| PostgreSQL | ✅ Excellent |
| MySQL | ✅ Excellent |
| MongoDB (single) | ✅ Good |
| Redis (single) | ✅ Good |

### Limits

```
┌─────────────────────────────────────────────────┐
│  Vertical Scaling Ceiling                       │
│                                                 │
│  • Hardware has limits (max 128 cores, 4TB RAM) │
│  • Diminishing returns                          │
│  • Single point of failure                      │
│  • Expensive at the top                         │
└─────────────────────────────────────────────────┘
```

### Pro Tip

> **Vertical scaling should be your first step.** A properly tuned PostgreSQL on a beefy machine handles more than most people think. Don't prematurely shard.

---

## 4. Read Replicas (Scale Reads)

### How It Works

```
                    ┌─────────────┐
                    │   Primary   │ ← All WRITES
                    │   (Leader)  │
                    └──────┬──────┘
                           │ replication
            ┌──────────────┼──────────────┐
            ↓              ↓              ↓
     ┌──────────┐   ┌──────────┐   ┌──────────┐
     │ Replica  │   │ Replica  │   │ Replica  │
     │    A     │   │    B     │   │    C     │
     └──────────┘   └──────────┘   └──────────┘
          ↑              ↑              ↑
        READS          READS          READS
```

### Benefits

| Benefit | How |
|---------|-----|
| **Read throughput** | Spread reads across replicas |
| **Availability** | Promote replica if primary fails |
| **Geographic distribution** | Replicas near users |
| **Analytics isolation** | Run reports on replica |

### Limitations

| Limitation | Why |
|------------|-----|
| Writes still bottleneck | Only one primary accepts writes |
| Replication lag | Replicas may be seconds behind |
| Storage not scaled | All nodes store all data |
| Complexity | Application must know read/write routing |

### Replication Lag: The Hidden Gotcha

```python
# This can fail!
def create_and_show_user(data):
    user = db.primary.insert(data)      # Write to primary
    return db.replica.get(user.id)      # Read from replica
                                        # Might not exist yet!
```

**Solutions:**
- Read-your-writes: route to primary for user's own data
- Causal consistency: track replication position
- Accept eventual consistency where possible

---

## 5. Sharding (True Horizontal Scaling)

### What It Is

**Split data across multiple independent nodes.** Each node stores a subset.

```
┌───────────────────────────────────────────────────┐
│  SHARDING BY USER ID                              │
│                                                   │
│  Users 1-1M      → Shard A (Node 1)               │
│  Users 1M-2M     → Shard B (Node 2)               │
│  Users 2M-3M     → Shard C (Node 3)               │
│                                                   │
│  Each shard has its own:                          │
│  • CPU, RAM, Disk                                 │
│  • Independent storage                            │
│  • Can have its own replicas                      │
└───────────────────────────────────────────────────┘
```

### What Sharding Solves

| Problem | How Sharding Helps |
|---------|-------------------|
| Storage limits | Data split across nodes |
| Write throughput | Writes distributed |
| Memory limits | Each node caches its subset |
| Query load | Queries route to relevant shard |

### Sharding Strategies

#### Range-Based Sharding

```
A-M → Shard 1
N-Z → Shard 2
```
**Pro:** Simple range queries  
**Con:** Hotspots (if most names start with A-M)

#### Hash-Based Sharding

```
hash(user_id) % num_shards → Shard N
```
**Pro:** Even distribution  
**Con:** Range queries hit all shards

#### Directory-Based Sharding

```
Lookup table: user_id → shard_location
```
**Pro:** Flexible, can rebalance  
**Con:** Lookup table is single point of failure

### The Shard Key: Critical Decision

```
Choosing a shard key:

✅ Good: user_id, tenant_id, account_id
   - Even distribution
   - Queries typically filter by this
   - Clear ownership

❌ Bad: timestamp, status, created_at
   - Hot spots (recent data)
   - Uneven distribution
   - Doesn't match query patterns
```

> **The shard key determines everything.** Choose based on your access patterns, not just data distribution.

---

## 6. Who Routes to the Correct Shard?

### Option A: Database-Managed (Recommended)

```
┌─────────────────────────────────────────────────┐
│  Application                                    │
│      ↓ query                                    │
│  Database Driver / Router                       │
│      ↓ routes internally                        │
│  Correct Shard                                  │
│                                                 │
│  Examples: MongoDB, Redis Cluster, CockroachDB, │
│           Cassandra, Vitess (MySQL)             │
└─────────────────────────────────────────────────┘
```

**Application is unaware of shard placement.** Database handles routing, rebalancing, and failover.

### Option B: Middleware / Proxy

```
Application → Proxy → Shard A/B/C
```

The proxy knows shard mapping. More complex but gives control.

### Option C: Application-Level (Usually Wrong)

```python
# ❌ Don't do this
def get_db_connection(user_id):
    if user_id < 1_000_000:
        return db_shard_1
    elif user_id < 2_000_000:
        return db_shard_2
    else:
        return db_shard_3
```

**Problems:**
- Every service needs routing logic
- Resharding requires code changes
- Tight coupling to data layout

---

## 7. Database Type Comparison

### Relational Databases (PostgreSQL, MySQL)

```
Scaling approach:
1. Vertical scaling ← Start here
2. Read replicas
3. Application-level partitioning
4. Vitess/Citus extensions
5. Migrate to distributed DB

Not distributed by design, but excellent for most workloads.
```

### Document Databases (MongoDB)

```
Built-in sharding:
• Shard key selection during setup
• Automatic balancing
• Driver-aware routing

Good for: variable schemas, horizontal scale needs
```

### Wide-Column (Cassandra, ScyllaDB)

```
Designed for scale:
• Consistent hashing
• No single primary
• Tunable consistency

Good for: write-heavy, time-series, massive scale
```

### NewSQL (CockroachDB, TiDB, Spanner)

```
Best of both worlds:
• SQL interface
• Distributed by design
• ACID transactions across shards

Good for: need SQL + horizontal scale
```

---

## 8. Cloud-Managed Databases

### What the Cloud Handles

| Concern | Managed by Cloud |
|---------|-----------------|
| Hardware provisioning | ✅ |
| Backups | ✅ |
| Patching | ✅ |
| Failover | ✅ |
| Replication | ✅ |
| Monitoring | ✅ |

### What YOU Still Handle

| Concern | Your Responsibility |
|---------|-------------------|
| Schema design | ✅ |
| Query optimization | ✅ |
| Shard key selection | ✅ |
| Access patterns | ✅ |
| Data modeling | ✅ |
| Capacity planning | ✅ |

> **The cloud reduces operational burden, not architectural decisions.**

---

## 9. Choosing the Right Approach

### Decision Matrix

| Situation | Recommended Approach |
|-----------|---------------------|
| < 100GB, moderate traffic | Vertical scaling |
| Read-heavy, < 1TB | Read replicas |
| Write-heavy, < 1TB | Vertical + write optimization |
| > 1TB or high write throughput | Sharding or distributed DB |
| Global users, low latency required | Multi-region distributed DB |
| Simple queries, massive scale | NoSQL (Cassandra, DynamoDB) |
| Complex queries, ACID required | NewSQL (CockroachDB, Spanner) |

### Cost vs Complexity

```
        Complexity
            ↑
            │     ┌─────────────┐
            │     │ Distributed │
            │     │     DB      │
            │     └─────────────┘
            │           ↑
            │     ┌─────────────┐
            │     │  Sharding   │
            │     └─────────────┘
            │           ↑
            │     ┌─────────────┐
            │     │  Replicas   │
            │     └─────────────┘
            │           ↑
            │     ┌─────────────┐
            │     │  Vertical   │
            │     └─────────────┘
            └─────────────────────→ Scale
```

> **Start simple. Scale up before scaling out. Add complexity only when needed.**

---

## 10. Practical Patterns

### Connection Pooling

```python
# ✅ Good: Connection pool
from sqlalchemy import create_engine
engine = create_engine(
    DATABASE_URL,
    pool_size=20,
    max_overflow=30,
    pool_pre_ping=True
)

# ❌ Bad: New connection per request
def get_user(id):
    conn = psycopg2.connect(DATABASE_URL)  # Expensive!
    ...
```

### Read/Write Splitting

```python
class Database:
    def __init__(self):
        self.primary = create_engine(PRIMARY_URL)
        self.replica = create_engine(REPLICA_URL)
    
    def write(self, query):
        return self.primary.execute(query)
    
    def read(self, query, use_primary=False):
        engine = self.primary if use_primary else self.replica
        return engine.execute(query)
```

### Graceful Degradation

```python
def get_user_profile(user_id):
    try:
        return db.get(f"user:{user_id}")
    except DatabaseUnavailable:
        # Return cached version or minimal data
        return cache.get(f"user:{user_id}") or {"id": user_id, "status": "limited"}
```

---

## 11. Anti-Patterns

### ❌ Premature Sharding

```
Don't shard because you "might need it."
Sharding adds complexity:
• Cross-shard queries
• Distributed transactions
• Operational overhead
```

### ❌ Wrong Shard Key

```
Sharding by timestamp creates hot shards.
Recent data = hot shard, old data = cold shards.
Choose keys that match your query patterns.
```

### ❌ Ignoring Connection Limits

```
PostgreSQL default: 100 connections
50 app instances × 20 connections each = 1000 connections
Use connection pooling (PgBouncer, ProxySQL).
```

### ❌ No Read Replica for Analytics

```
Running analytics on production primary:
• Locks tables
• Consumes resources
• Impacts users

Always use replicas for reporting/analytics.
```

---

## 12. Interview Deep-Dives

### "How would you scale a database?"

```
1. First, profile: What's the bottleneck?
   - CPU? Memory? Disk? Network? Connections?

2. Optimize before scaling:
   - Indexes, query optimization, caching

3. Scale vertically first:
   - Bigger machine is simpler than distributed

4. Add read replicas if read-heavy

5. Consider sharding only for:
   - Dataset too large for one node
   - Write throughput exceeds single node

6. Evaluate distributed databases if:
   - Need horizontal scale with SQL
   - Global distribution required
```

### "Explain replication lag"

```
Primary commits write at T=0
Replica receives write at T=100ms

If user reads from replica at T=50ms,
they see stale data.

Solutions:
• Read from primary for user's own data
• Synchronous replication (hurts latency)
• Track replication position
• Design for eventual consistency
```

### "How do you choose a shard key?"

```
Criteria:
1. High cardinality (many unique values)
2. Even distribution (no hotspots)
3. Matches query patterns (queries include shard key)
4. Stable (doesn't change)

Example for e-commerce:
• user_id for user data ✅
• order_id for orders ✅  
• timestamp ❌ (hotspots)
• status ❌ (low cardinality)
```

---

## TL;DR

| Strategy | When to Use | Scales |
|----------|-------------|--------|
| **Vertical** | First step, always | CPU, RAM, Disk |
| **Replicas** | Read-heavy workloads | Read throughput |
| **Sharding** | Large data, high writes | Storage, Writes |
| **Distributed DB** | Global scale, SQL needs | Everything |

**Key Principles:**
- Stateless scales with load balancers; stateful scales with data partitioning
- Vertical scaling first, horizontal when necessary
- Replication helps reads, not writes or storage
- The shard key is the most critical decision
- Cloud handles operations, not architecture

---

## Quick Reference

### PostgreSQL Connection String

```
postgresql://user:password@host:5432/dbname?sslmode=require
```

### Read Replica Pattern

```python
WRITE_DB = os.getenv("PRIMARY_DATABASE_URL")
READ_DB = os.getenv("REPLICA_DATABASE_URL")
```

### Interview Talking Points

1. "Always start with vertical scaling—most systems never need sharding"
2. "Read replicas scale reads, not writes or storage"
3. "The shard key determines query routing and data distribution"
4. "Replication lag can cause read-your-writes inconsistency"
5. "Managed databases reduce ops burden, not architectural decisions"
