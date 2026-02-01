# Caching Strategies

> **Goal**: Understand when, where, and how to cache data to dramatically improve system performance and reduce load on databases.

---

## 1. Why Cache?

### The Problem

```
┌─────────────────────────────────────────────────┐
│  Without Caching                                │
│                                                 │
│  Every request → Database query → Response      │
│                                                 │
│  Problem:                                       │
│  • Database is slow (disk I/O, network)         │
│  • Same data queried repeatedly                 │
│  • Database becomes bottleneck                  │
│  • Latency is unpredictable                     │
└─────────────────────────────────────────────────┘
```

### The Solution

```
┌─────────────────────────────────────────────────┐
│  With Caching                                   │
│                                                 │
│  Request → Check Cache → Hit? Return instantly  │
│                       → Miss? Query DB, cache   │
│                                                 │
│  Result:                                        │
│  • Microseconds vs milliseconds                 │
│  • Reduced database load                        │
│  • Predictable latency                          │
│  • Handle more traffic                          │
└─────────────────────────────────────────────────┘
```

### Cache Performance Impact

| Operation | Latency |
|-----------|---------|
| L1 cache | 0.5 ns |
| L2 cache | 7 ns |
| RAM | 100 ns |
| Redis (local) | 0.5 ms |
| SSD read | 0.1 ms |
| Database query | 5-50 ms |
| Network round-trip | 10-100 ms |

> **Key insight**: Caching is about moving data closer to where it's needed.

---

## 2. Cache Hit Rate: The Only Metric That Matters

```
Hit Rate = Cache Hits / (Cache Hits + Cache Misses)
```

| Hit Rate | Meaning |
|----------|---------|
| 99% | Excellent — 1 in 100 requests hits DB |
| 90% | Good — 1 in 10 hits DB |
| 50% | Poor — cache may not be helping |
| < 50% | Reconsider caching strategy |

### What Affects Hit Rate?

```
┌─────────────────────────────────────────────────┐
│  High Hit Rate When:                            │
│  • Same data requested repeatedly               │
│  • Working set fits in cache                    │
│  • TTL matches data update frequency            │
│  • Cache key design matches access patterns     │
│                                                 │
│  Low Hit Rate When:                             │
│  • Random access patterns                       │
│  • Large working set                            │
│  • Frequent data updates                        │
│  • Poor cache key design                        │
└─────────────────────────────────────────────────┘
```

---

## 3. Caching Layers

### Where Can You Cache?

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CACHING LAYERS                               │
│                                                                     │
│  Browser ──→ CDN ──→ API Gateway ──→ App Server ──→ Database        │
│     ↓          ↓           ↓              ↓             ↓           │
│  Browser    Edge       Response        In-Memory    Query           │
│  Cache     Cache       Cache           Cache        Cache           │
│                                                                     │
│  ← Closer to user (faster)          Closer to data (fresher) →      │
└─────────────────────────────────────────────────────────────────────┘
```

### Layer Details

| Layer | Location | What to Cache | TTL |
|-------|----------|--------------|-----|
| **Browser** | User's device | Static assets, API responses | Hours-days |
| **CDN** | Edge servers | Static files, API responses | Minutes-hours |
| **API Gateway** | Entry point | Full responses | Seconds-minutes |
| **Application** | App server memory | Computed data, sessions | Seconds-minutes |
| **Distributed Cache** | Redis/Memcached | Hot data, sessions | Seconds-hours |
| **Database** | Query cache | Query results | Query-dependent |

---

## 4. Caching Patterns

### Pattern 1: Cache-Aside (Lazy Loading)

**Most common pattern.** Application manages cache explicitly.

```
┌─────────────────────────────────────────────────┐
│  READ:                                          │
│  1. Check cache                                 │
│  2. If hit → return                             │
│  3. If miss → query DB → store in cache         │
│                                                 │
│  WRITE:                                         │
│  1. Update database                             │
│  2. Invalidate/delete cache                     │
└─────────────────────────────────────────────────┘
```

```python
def get_user(user_id):
    # Check cache
    cached = cache.get(f"user:{user_id}")
    if cached:
        return cached
    
    # Cache miss - query database
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # Store in cache
    cache.set(f"user:{user_id}", user, ttl=300)
    return user

def update_user(user_id, data):
    # Update database
    db.execute("UPDATE users SET ... WHERE id = ?", data, user_id)
    
    # Invalidate cache
    cache.delete(f"user:{user_id}")
```

**Pros:**
- Simple to implement
- Only caches data that's actually requested
- Cache failures don't break reads

**Cons:**
- First request always slow (cache miss)
- Potential for stale data between write and cache invalidation

---

### Pattern 2: Write-Through

**Cache is updated synchronously with database.**

```
┌─────────────────────────────────────────────────┐
│  WRITE:                                         │
│  1. Write to cache                              │
│  2. Cache writes to database                    │
│  3. Return success                              │
│                                                 │
│  READ:                                          │
│  1. Always read from cache                      │
└─────────────────────────────────────────────────┘
```

**Pros:**
- Cache always consistent with DB
- Reads always fast

**Cons:**
- Write latency (must update both)
- Caches data that may never be read

---

### Pattern 3: Write-Behind (Write-Back)

**Cache updated immediately, DB updated asynchronously.**

```
┌─────────────────────────────────────────────────┐
│  WRITE:                                         │
│  1. Write to cache                              │
│  2. Return success immediately                  │
│  3. Asynchronously write to database            │
│                                                 │
│  Risk: Data loss if cache fails before DB write │
└─────────────────────────────────────────────────┘
```

**Pros:**
- Very fast writes
- Batch DB writes for efficiency

**Cons:**
- Risk of data loss
- Complex to implement correctly

---

### Pattern 4: Read-Through

**Cache fetches from database on miss automatically.**

```
┌─────────────────────────────────────────────────┐
│  Application → Cache → (miss) → Database        │
│                                                 │
│  Cache handles DB fetching transparently        │
└─────────────────────────────────────────────────┘
```

Similar to cache-aside, but the cache manages DB fetching.

---

### Pattern 5: Refresh-Ahead

**Proactively refresh cache before expiration.**

```
┌─────────────────────────────────────────────────┐
│  TTL = 5 minutes                                │
│  Refresh at 80% of TTL (4 minutes)              │
│                                                 │
│  If accessed when TTL < 1 min:                  │
│    → Return cached value                        │
│    → Async refresh from DB                      │
└─────────────────────────────────────────────────┘
```

**Pros:**
- Avoids cache miss latency
- Always fresh data for hot keys

**Cons:**
- More complex
- Refreshes data that may not be needed

---

## 5. Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### Strategies

#### Time-Based (TTL)

```python
cache.set("user:123", user_data, ttl=300)  # Expires in 5 minutes
```

**Simple but imprecise.** Data may be stale for up to TTL duration.

#### Event-Based Invalidation

```python
def update_user(user_id, data):
    db.update(user_id, data)
    cache.delete(f"user:{user_id}")           # Direct invalidation
    events.publish("user.updated", user_id)   # Event for other caches
```

**Precise but complex.** Requires tracking dependencies.

#### Version-Based

```python
# Include version in cache key
version = db.get_version("users")
cache_key = f"user:{user_id}:v{version}"

# Increment version to invalidate all user caches
db.increment_version("users")
```

**Bulk invalidation without touching each key.**

---

## 6. Cache Eviction Policies

When cache is full, what gets removed?

| Policy | Description | Best For |
|--------|-------------|----------|
| **LRU** (Least Recently Used) | Remove oldest accessed | General purpose |
| **LFU** (Least Frequently Used) | Remove least accessed | Hot data patterns |
| **FIFO** (First In First Out) | Remove oldest added | Simple, predictable |
| **TTL** | Remove expired first | Time-sensitive data |
| **Random** | Remove randomly | Simple, surprisingly effective |

> **Redis default**: LRU approximation (samples keys, evicts least recent)

---

## 7. Common Caching Technologies

### Redis

```python
import redis

r = redis.Redis(host='redis', port=6379)

# Basic operations
r.set("key", "value", ex=300)  # 5 min TTL
r.get("key")

# Data structures
r.hset("user:123", mapping={"name": "Alice", "email": "a@b.com"})
r.lpush("queue", "task1", "task2")
r.sadd("tags:article:1", "python", "caching")
```

**Strengths:**
- Rich data structures (strings, hashes, lists, sets, sorted sets)
- Pub/sub messaging
- Lua scripting
- Persistence options
- Clustering support

### Memcached

```python
import memcache

mc = memcache.Client(['memcached:11211'])
mc.set("key", "value", time=300)
mc.get("key")
```

**Strengths:**
- Simple key-value
- Multi-threaded (scales on single node)
- Memory efficient

**When to use:**
- Simple caching needs
- High memory efficiency required

---

## 8. CDN Caching

### What CDNs Cache

```
┌─────────────────────────────────────────────────────────────────────┐
│  User (London) ──→ CDN Edge (London) ──→ Origin (US-East)           │
│                          ↓                                          │
│                    Cached Response                                  │
│                    (no origin hit)                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Controlling CDN Cache

```http
# Response headers from origin
Cache-Control: public, max-age=3600     # Cache for 1 hour
Cache-Control: private, no-cache        # Don't cache
Cache-Control: public, s-maxage=86400   # CDN caches 1 day
```

### What to Cache at CDN

| Content | Cache? | TTL |
|---------|--------|-----|
| Static assets (JS, CSS, images) | ✅ | Long (days/weeks) |
| Public API responses | ✅ | Short (seconds/minutes) |
| User-specific data | ❌ | Don't cache publicly |
| HTML pages | Depends | Vary by needs |

---

## 9. Caching Challenges

### Challenge 1: Cache Stampede

```
┌─────────────────────────────────────────────────┐
│  Popular key expires                            │
│       ↓                                         │
│  1000 requests simultaneously miss              │
│       ↓                                         │
│  1000 database queries at once                  │
│       ↓                                         │
│  Database overwhelmed                           │
└─────────────────────────────────────────────────┘
```

**Solutions:**

```python
# Solution 1: Lock/Mutex
def get_with_lock(key):
    value = cache.get(key)
    if value:
        return value
    
    if cache.acquire_lock(f"lock:{key}"):
        try:
            value = db.query(key)
            cache.set(key, value)
            return value
        finally:
            cache.release_lock(f"lock:{key}")
    else:
        # Wait for other request to populate cache
        time.sleep(0.1)
        return cache.get(key)

# Solution 2: Stale-while-revalidate
# Return stale data immediately, refresh in background
```

### Challenge 2: Hot Keys

```
┌─────────────────────────────────────────────────┐
│  Celebrity tweet → Millions read same key       │
│  Single Redis node overloaded                   │
└─────────────────────────────────────────────────┘
```

**Solutions:**
- Local cache in front of distributed cache
- Replicate hot keys across multiple nodes
- Shard by adding random suffix: `key:1`, `key:2`, `key:3`

### Challenge 3: Cache Penetration

```
┌─────────────────────────────────────────────────┐
│  Request for non-existent data                  │
│       ↓                                         │
│  Cache miss (doesn't exist)                     │
│       ↓                                         │
│  DB query (returns nothing)                     │
│       ↓                                         │
│  Nothing to cache, next request repeats         │
└─────────────────────────────────────────────────┘
```

**Solutions:**

```python
# Cache negative results
def get_user(user_id):
    cached = cache.get(f"user:{user_id}")
    if cached == "NOT_FOUND":
        return None
    if cached:
        return cached
    
    user = db.query(user_id)
    if user:
        cache.set(f"user:{user_id}", user)
    else:
        cache.set(f"user:{user_id}", "NOT_FOUND", ttl=60)
    return user
```

Or use **Bloom filters** to check if key might exist before querying.

---

## 10. Practical Guidelines

### What to Cache

| Cache | Don't Cache |
|-------|-------------|
| Frequently read data | Frequently changing data |
| Expensive computations | Cheap operations |
| Stable data | Highly personalized data |
| Data with clear access patterns | Random access data |

### Cache Key Design

```python
# ✅ Good: Clear, hierarchical, versioned
f"v1:users:{user_id}"
f"v1:users:{user_id}:posts"
f"v1:products:{product_id}:price:USD"

# ❌ Bad: Ambiguous, collision-prone
f"{user_id}"
f"user_data"
f"cache_key"
```

### TTL Guidelines

| Data Type | Suggested TTL |
|-----------|---------------|
| User sessions | 30 min - 24 hours |
| API responses | 1 - 5 minutes |
| Database query results | 5 - 60 minutes |
| Static config | 1 - 24 hours |
| Product catalog | 5 - 30 minutes |

---

## 11. Redis in Production

### Basic Setup

```python
import redis
from contextlib import contextmanager

# Connection pool (reuse connections)
pool = redis.ConnectionPool(
    host='redis',
    port=6379,
    max_connections=100,
    decode_responses=True
)

def get_redis():
    return redis.Redis(connection_pool=pool)

# Usage
def get_cached_user(user_id):
    r = get_redis()
    key = f"user:{user_id}"
    
    # Try cache
    cached = r.get(key)
    if cached:
        return json.loads(cached)
    
    # Miss - get from DB
    user = db.get_user(user_id)
    r.setex(key, 300, json.dumps(user))  # Cache for 5 min
    return user
```

### Useful Redis Commands

```bash
# Monitor
redis-cli monitor           # Watch all commands
redis-cli info stats       # Cache stats

# Memory
redis-cli info memory      # Memory usage
redis-cli memory usage key # Size of specific key

# Keys
redis-cli keys "user:*"    # Find keys (don't use in production!)
redis-cli scan 0 match "user:*"  # Safe iteration
```

---

## TL;DR

| Pattern | Use When |
|---------|----------|
| **Cache-Aside** | Most cases, simple implementation |
| **Write-Through** | Strong consistency needed |
| **Write-Behind** | Write performance critical |
| **Refresh-Ahead** | Can't afford cache miss latency |

**Key Principles:**
- Cache what's read often and changes rarely
- Hit rate is the critical metric
- TTL is simplest invalidation, event-based is most accurate
- Watch for stampede, hot keys, and penetration
- Start simple, measure, then optimize

---

## Quick Reference

### Redis Cache-Aside

```python
def get_or_set(key, fetch_fn, ttl=300):
    value = cache.get(key)
    if value is None:
        value = fetch_fn()
        cache.setex(key, ttl, json.dumps(value))
    return json.loads(value) if isinstance(value, str) else value
```

### Cache Headers

```http
Cache-Control: public, max-age=3600
Cache-Control: private, no-store
ETag: "abc123"
```

### Interview Talking Points

1. "Cache-aside is the most common pattern—app checks cache, fetches on miss"
2. "Cache stampede happens when popular keys expire; solve with locks or stale-while-revalidate"
3. "CDN caching moves data to edge servers, reducing latency for global users"
4. "Cache invalidation is hard—TTL is simple but imprecise, event-based is precise but complex"
5. "Always measure hit rate—below 90% means your caching strategy needs work"
