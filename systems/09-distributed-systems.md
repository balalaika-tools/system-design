# Distributed Systems Fundamentals

> **Goal**: Understand the core concepts, trade-offs, and guarantees in distributed systems—CAP theorem, consistency models, consensus, and failure modes.

---

## 1. Why Distributed Systems?

### Single Server Limits

```
┌─────────────────────────────────────────────────┐
│  Single Server                                  │
│                                                 │
│  ❌ Hardware limits (CPU, RAM, Disk) ❌        │
│  ❌ Single point of failure ❌                 │
│  ❌ Geographic latency ❌                      │
│  ❌ Maintenance requires downtime ❌           │
└─────────────────────────────────────────────────┘
```

### Distributed Benefits

```
┌─────────────────────────────────────────────────┐
│  Distributed System                             │
│                                                 │
│  ✅ Horizontal scaling ✅                      │
│  ✅ Fault tolerance ✅                         │
│  ✅ Geographic distribution ✅                 │
│  ✅ Zero-downtime operations ✅                │
│                                                 │
│  But: Complexity, consistency challenges        │
└─────────────────────────────────────────────────┘
```

---

## 2. The CAP Theorem

### What It Says

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CAP THEOREM                                  │
│                                                                     │
│  In a distributed system, during a network partition,               │
│  you can only guarantee TWO of:                                     │
│                                                                     │
│                     Consistency                                     │
│                        /\                                           │
│                       /  \                                          │
│                      /    \                                         │
│                     /      \                                        │
│         Availability ────── Partition Tolerance                     │
│                                                                     │
│  C: Every read receives the most recent write                       │
│  A: Every request receives a response (not error)                   │
│  P: System continues despite network failures                       │
└─────────────────────────────────────────────────────────────────────┘
```

### The Reality

```
┌─────────────────────────────────────────────────────────────────────┐
│  Network partitions WILL happen.                                    │
│  So you must choose between C and A during partitions.              │
│                                                                     │
│  CP: Consistent, Partition-tolerant                                 │
│      During partition → refuse some requests                        │
│      Example: Banking, inventory                                    │
│                                                                     │
│  AP: Available, Partition-tolerant                                  │
│      During partition → allow stale reads                           │
│      Example: Social media feeds, caching                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Database Classifications

| Database | Type | Trade-off |
|----------|------|-----------|
| PostgreSQL (single) | CA | No partition tolerance |
| MongoDB | CP (default) | Rejects writes without majority |
| Cassandra | AP | Eventual consistency |
| Redis Cluster | AP | Eventual consistency |
| Zookeeper | CP | Rejects if no quorum |
| DynamoDB | AP (configurable) | Tunable consistency |

> **Important**: CAP is about behavior during partitions. Most of the time, systems aren't partitioned.

---

## 3. Consistency Models

### Strong Consistency

```
┌─────────────────────────────────────────────────┐
│  Every read sees the most recent write          │
│                                                 │
│  Write X=5 ──→ All reads return 5               │
│                                                 │
│  ✅ Simple mental model ✅                     │
│  ❌ Higher latency ❌                          │
│  ❌ Lower availability ❌                      │
│                                                 │
│  Example: Bank balance, inventory count         │
└─────────────────────────────────────────────────┘
```

### Eventual Consistency

```
┌─────────────────────────────────────────────────┐
│  Reads eventually return the latest write       │
│                                                 │
│  Write X=5                                      │
│  Read (might return old value)                  │
│  ...time passes...                              │
│  Read (returns 5)                               │
│                                                 │
│  ✅ Higher availability ✅                     │
│  ✅ Lower latency ✅                           │
│  ❌ Complex application logic ❌               │
│                                                 │
│  Example: Social media likes, DNS               │
└─────────────────────────────────────────────────┘
```

### Consistency Spectrum

```
Strong ────────────────────────────────────→ Eventual

Linearizability
    Sequential Consistency
        Causal Consistency
            Read-your-writes
                Monotonic reads
                    Eventual Consistency

← Stronger guarantees          Weaker guarantees →
← Higher latency               Lower latency →
← Lower availability           Higher availability →
```

### Practical Consistency Levels

| Level | Guarantee | Use Case |
|-------|-----------|----------|
| **Linearizable** | Real-time ordering | Distributed locks |
| **Sequential** | Global order (not real-time) | Consistent snapshots |
| **Causal** | Cause-effect ordering | Social feeds, comments |
| **Read-your-writes** | See your own writes | User profile updates |
| **Eventual** | Will converge eventually | Caches, metrics |

---

## 4. Consensus

### The Problem

```
┌─────────────────────────────────────────────────────────────────────┐
│  Multiple nodes need to agree on a value                            │
│                                                                     │
│  Examples:                                                          │
│  • Who is the database leader?                                      │
│  • Was this transaction committed?                                  │
│  • What is the next sequence number?                                │
│                                                                     │
│  Challenge: Nodes can fail, messages can be lost/delayed            │
└─────────────────────────────────────────────────────────────────────┘
```

### Consensus Algorithms

| Algorithm | Used By | Notes |
|-----------|---------|-------|
| **Paxos** | Google Chubby | Theoretical foundation, complex |
| **Raft** | etcd, Consul | Understandable, widely used |
| **ZAB** | Zookeeper | Zookeeper's protocol |
| **PBFT** | Blockchain | Byzantine fault tolerant |

### Raft Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         RAFT                                        │
│                                                                     │
│  Leader Election:                                                   │
│  • One leader, multiple followers                                   │
│  • Leader handles all writes                                        │
│  • If leader fails, followers elect new leader                      │
│                                                                     │
│  Log Replication:                                                   │
│  • Leader appends to log                                            │
│  • Replicates to followers                                          │
│  • Commits when majority acknowledges                               │
│                                                                     │
│  Safety:                                                            │
│  • Only nodes with complete logs can become leader                  │
│  • Committed entries are never lost                                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Quorum

```
┌─────────────────────────────────────────────────────────────────────┐
│  QUORUM = Majority of nodes must agree                              │
│                                                                     │
│  5 nodes → need 3 for quorum (can tolerate 2 failures)              │
│  3 nodes → need 2 for quorum (can tolerate 1 failure)               │
│                                                                     │
│  Formula: Quorum = (N / 2) + 1                                      │
│  Fault tolerance = N - Quorum                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Replication

### Leader-Follower (Primary-Replica)

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Writes ──→ [Leader] ──replication──→ [Follower 1]                  │
│                    ──replication──→ [Follower 2]                    │
│                                                                     │
│  Reads can go to followers (may be stale)                           │
│                                                                     │
│  If leader fails:                                                   │
│  • Detect failure                                                   │
│  • Elect new leader from followers                                  │
│  • Reconfigure clients                                              │
└─────────────────────────────────────────────────────────────────────┘
```

### Synchronous vs Asynchronous Replication

```
Synchronous:
  Write ──→ Leader ──→ Wait for follower ack ──→ Confirm
  
  ✅ No data loss
  ❌ Higher latency
  ❌ Unavailable if follower down

Asynchronous:
  Write ──→ Leader ──→ Confirm immediately
                  ──→ Replicate in background
  
  ✅ Lower latency
  ✅ Available if followers down
  ❌ Potential data loss
```

### Multi-Leader Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│  Multiple data centers, each with a leader                          │
│                                                                     │
│  [DC US-East]          [DC EU-West]                                 │
│   Leader A  ←──sync──→  Leader B                                    │
│      ↓                     ↓                                        │
│  Followers             Followers                                    │
│                                                                     │
│  ✅ Low latency writes in each region ✅                           │
│  ❌ Write conflicts possible ❌                                    │
│  ❌ Complex conflict resolution ❌                                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Conflict Resolution

```
When two leaders accept conflicting writes:

1. Last-Write-Wins (LWW)
   Use timestamp, latest wins (may lose data)

2. Custom resolution
   Application-specific merge logic

3. Conflict-free Replicated Data Types (CRDTs)
   Data structures that merge automatically
```

---

## 6. Partitioning (Sharding)

### Why Partition?

```
Data too big for one node → split across multiple nodes
```

### Partitioning Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│  KEY RANGE PARTITIONING                                             │
│                                                                     │
│  A-M → Partition 1                                                  │
│  N-Z → Partition 2                                                  │
│                                                                     │
│  ✅ Range queries efficient ✅                                     │
│  ❌ Hotspots if access patterns skewed ❌                          │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  HASH PARTITIONING                                                  │
│                                                                     │
│  hash(key) mod N → Partition number                                 │
│                                                                     │
│  ✅ Even distribution ✅                                           │
│  ❌ Range queries hit all partitions ❌                            │
└─────────────────────────────────────────────────────────────────────┘
```

### Consistent Hashing

```
┌─────────────────────────────────────────────────────────────────────┐
│  Adding/removing nodes only affects neighboring keys                │
│                                                                     │
│           Node A                                                    │
│              │                                                      │
│    ┌─────────●─────────┐                                            │
│    │         │         │                                            │
│ Node D      Ring     Node B                                         │
│    │         │         │                                            │
│    └─────────●─────────┘                                            │
│              │                                                      │
│           Node C                                                    │
│                                                                     │
│  Keys are assigned to the next node clockwise                       │
│  Adding Node E only moves keys between neighbors                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Failure Modes

### Types of Failures

| Failure | Description | Example |
|---------|-------------|---------|
| **Crash** | Node stops working | Server dies |
| **Omission** | Messages lost | Network drop |
| **Timing** | Response too slow | Timeout |
| **Byzantine** | Node behaves arbitrarily wrong | Corrupted data, malicious |

### Handling Failures

```
┌─────────────────────────────────────────────────────────────────────┐
│  FAILURE DETECTION                                                  │
│                                                                     │
│  Heartbeats:                                                        │
│  • Nodes send periodic "I'm alive" messages                         │
│  • No heartbeat for X seconds → suspected failure                   │
│                                                                     │
│  Challenge: Distinguish slow from dead (network vs crash)           │
│                                                                     │
│  Phi Accrual Failure Detector:                                      │
│  • Adaptive threshold based on historical latency                   │
│  • Returns probability of failure, not binary                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Split Brain

```
┌─────────────────────────────────────────────────────────────────────┐
│  Network partition creates two groups                               │
│  Each group thinks the other is dead                                │
│  Both elect a leader → TWO LEADERS (split brain)                    │
│                                                                     │
│  [Node A]  [Node B]  ||  [Node C]  [Node D]  [Node E]               │
│  Leader 1            ||  Leader 2                                   │
│                                                                     │
│  Prevention:                                                        │
│  • Require quorum to operate                                        │
│  • Fencing: Old leader prevented from writing                       │
│  • STONITH: Shoot The Other Node In The Head                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. Time in Distributed Systems

### The Problem

```
┌─────────────────────────────────────────────────────────────────────┐
│  Physical clocks are unreliable:                                    │
│                                                                     │
│  • Clocks drift (different rates)                                   │
│  • NTP synchronization has errors                                   │
│  • Leap seconds                                                     │
│                                                                     │
│  "Event A happened before Event B"                                  │
│  Hard to determine across nodes!                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Logical Clocks

```
┌─────────────────────────────────────────────────────────────────────┐
│  LAMPORT CLOCKS                                                     │
│                                                                     │
│  • Each node has a counter                                          │
│  • Increment on local event                                         │
│  • On message send: include counter                                 │
│  • On receive: max(local, received) + 1                             │
│                                                                     │
│  Gives partial ordering (not total)                                 │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  VECTOR CLOCKS                                                      │
│                                                                     │
│  • Each node tracks all nodes' counters                             │
│  • Can detect concurrent events                                     │
│  • Enables causal consistency                                       │
│                                                                     │
│  Node A: [A:1, B:0, C:0]                                            │
│  Node B: [A:1, B:2, C:0]                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### Hybrid Logical Clocks (HLC)

```
Combines physical and logical time:
• Uses physical time when possible
• Falls back to logical when physical is unreliable
• Used in CockroachDB, MongoDB
```

---

## 9. Distributed Transactions

### Two-Phase Commit (2PC)

```
┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 1: PREPARE                                                   │
│                                                                     │
│  Coordinator: "Can everyone commit?"                                │
│  Participants: "Yes" / "No"                                         │
│                                                                     │
│  PHASE 2: COMMIT / ABORT                                            │
│                                                                     │
│  If all yes: Coordinator: "Commit"                                  │
│  If any no:  Coordinator: "Abort"                                   │
│                                                                     │
│  ❌ Blocking: If coordinator dies after Phase 1, all block ❌      │
│  ❌ Not partition tolerant ❌                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Three-Phase Commit (3PC)

Adds a pre-commit phase to reduce blocking, but still has issues.

### Consensus-Based Commit

```
Use Raft/Paxos for atomic commit:
• Non-blocking
• Partition tolerant
• Used in modern distributed databases (Spanner, CockroachDB)
```

---

## 10. PACELC Theorem

### Beyond CAP

```
┌─────────────────────────────────────────────────────────────────────┐
│  PACELC                                                             │
│                                                                     │
│  If Partition:                                                      │
│    Choose between Availability and Consistency                      │
│  Else (normal operation):                                           │
│    Choose between Latency and Consistency                           │
│                                                                     │
│  PA/EL: Prioritize availability and latency (Cassandra, DynamoDB)   │
│  PC/EC: Prioritize consistency always (traditional RDBMS)           │
│  PA/EC: Available during partition, consistent otherwise            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 11. Practical Guidelines

### Designing for Distribution

```
✅ Assume failures will happen
✅ Use idempotent operations
✅ Implement retry with backoff
✅ Use correlation IDs for tracing
✅ Design for eventual consistency where possible
✅ Use consensus for coordination (not roll your own)
✅ Monitor everything

❌ Don't assume network is reliable
❌ Don't assume clocks are synchronized
❌ Don't assume failures are obvious
❌ Don't try to hide distribution from developers
```

### Choosing Consistency Level

| Use Case | Recommended |
|----------|-------------|
| Banking, inventory | Strong consistency |
| User sessions | Read-your-writes |
| Social feeds | Eventual consistency |
| Analytics | Eventual consistency |
| Caching | Eventual consistency |
| Distributed locks | Strong consistency |

---

## TL;DR

| Concept | Summary |
|---------|---------|
| **CAP** | During partition, choose Consistency or Availability |
| **Strong Consistency** | Every read sees latest write |
| **Eventual Consistency** | Reads converge to latest write over time |
| **Consensus** | Agreement among nodes (Raft, Paxos) |
| **Quorum** | Majority agreement (N/2 + 1) |
| **Replication** | Copies of data for fault tolerance |
| **Partitioning** | Splitting data across nodes |
| **2PC** | Atomic commit across nodes (blocking) |

**Key Principles:**
- CAP is about partition behavior; most of the time you're not partitioned
- Consistency is a spectrum, not binary
- Use consensus algorithms, don't invent your own
- Design for failure—it will happen
- Eventual consistency is often good enough

---

## Quick Reference

### CAP Decision

```
During partition:
  Need consistency? → CP (reject some requests)
  Need availability? → AP (allow stale data)
```

### Consistency Level Selection

```
Strong: Critical data, correctness > performance
Eventual: High throughput, staleness acceptable
Causal: Preserve cause-effect relationships
```

### Interview Talking Points

1. "CAP says during a network partition, you must choose between consistency and availability"
2. "Most real-world systems choose eventual consistency for better availability and latency"
3. "Consensus algorithms like Raft ensure nodes agree on values despite failures"
4. "Quorum (majority) ensures any two quorums overlap, preserving consistency"
5. "Two-phase commit is blocking; modern systems use consensus-based approaches"
