# Redis: Complete Production Guide for Java/Spring Boot Engineers

> For backend developers with 3+ years Java experience transitioning from PostgreSQL-only stacks.

---

## Table of Contents

1. [Redis Fundamentals](#1-redis-fundamentals)
2. [Architecture & Internals](#2-architecture--internals)
3. [Data Structures Deep Dive](#3-data-structures-deep-dive)
4. [Redis in System Design](#4-redis-in-system-design)
5. [Production Best Practices](#5-production-best-practices)
6. [Redis with Spring Boot](#6-redis-with-spring-boot)
7. [Advanced Topics](#7-advanced-topics)
8. [Real-World Scenarios](#8-real-world-scenarios)
9. [Comparisons](#9-comparisons)
10. [Production-Ready Checklist](#10-production-ready-checklist)

---

# 1. Redis Fundamentals

## What Redis Really Is

Redis (Remote Dictionary Server) is not just a cache. It is an **in-memory data structure store** that can act as:

- A **cache** (most common use)
- A **database** (with persistence)
- A **message broker** (Pub/Sub, Streams)
- A **session store**
- A **distributed coordination primitive** (locks, semaphores)
- A **rate limiter**
- A **leaderboard engine** (sorted sets)

The fundamental difference from PostgreSQL: Redis keeps **all data in RAM**. Disk is secondary. This gives sub-millisecond latency at the cost of data volume limits and durability trade-offs.

## When to Use Redis

| Use Case | Why Redis Wins |
|---|---|
| Read-heavy workloads with repeated queries | Avoid hitting DB for the same data |
| Session storage | Fast random access, built-in TTL |
| Rate limiting | Atomic increment + expiry in one operation |
| Leaderboards | Sorted Sets with O(log N) ranking |
| Real-time pub/sub | Built-in messaging with no setup |
| Distributed locks | Atomic SET NX EX pattern |
| Job queues | Lists as FIFO queues, Streams for durable queues |
| Counters / analytics | Atomic INCR, HyperLogLog for cardinality |

## When NOT to Use Redis

- **Primary store for complex relational data** — no joins, no foreign keys, no ACID transactions across multiple keys by default.
- **Large blob storage** — Redis is RAM. Storing 10MB images per user kills memory fast.
- **Long-term durable storage without care** — data loss on crash is possible without proper persistence config.
- **Full-text search** — use Elasticsearch or PostgreSQL FTS.
- **Analytical queries** — no GROUP BY, aggregations, or SQL-style queries.
- **Budgets too tight for RAM** — RAM is 10–50x more expensive than disk.

## Key Properties

### In-Memory Model

All data lives in RAM. Reads and writes never touch disk during normal operation (unless persistence is configured). This is why Redis can do **1 million ops/sec** on commodity hardware while PostgreSQL peaks at tens of thousands.

**Implication**: Your Redis data set size is bounded by available RAM. Rule of thumb: plan for your data to use 2–3x the raw byte size due to overhead (pointers, metadata, encoding).

### Single-Threaded Design

Redis processes commands on a **single thread** (the event loop). This means:

- **No locking** needed internally — commands are inherently atomic.
- **No race conditions** on single commands.
- **No parallelism** for CPU-bound tasks — but Redis is I/O bound, not CPU bound.
- One slow command (e.g., `KEYS *`, `SMEMBERS` on a huge set) **blocks everything**.

Since Redis 6.0, I/O is threaded (reading/writing to sockets happens in parallel) but command execution remains single-threaded.

### Latency vs Durability Trade-offs

| Config | Latency | Durability |
|---|---|---|
| No persistence | ~0.1ms | Data lost on restart |
| RDB every 60s | ~0.1ms | Up to 60s of data loss |
| AOF fsync every second | ~0.2ms | Up to 1s of data loss |
| AOF fsync always | ~1–2ms | Near-zero data loss |
| Redis + PostgreSQL dual write | Variable | Full durability |

For caching: no persistence is fine. For session storage: AOF every second is usually acceptable. For financial data as primary store: use PostgreSQL.

---

# 2. Architecture & Internals

## Event Loop Model

Redis uses an **event-driven, non-blocking I/O** model similar to Node.js but in C.

```
Client 1 ──┐
Client 2 ──┤──► Event Loop (epoll/kqueue) ──► Command Processor ──► Response
Client N ──┘         (single thread)
```

The event loop:
1. Accepts new connections (non-blocking)
2. Reads available data from client sockets
3. Parses the Redis Serialization Protocol (RESP)
4. Executes the command (single-threaded, no locks needed)
5. Writes the response back

Since Redis 6.0, steps 2 and 5 are handled by I/O threads, but step 4 remains single-threaded.

**Why this matters for you**: A single `O(N)` command on a large dataset can cause latency spikes for **all** other clients. Never run `KEYS *`, `SMEMBERS` on large sets, or `HGETALL` on large hashes in production.

## Memory Model & Internal Data Structures

Redis uses several internal encodings per data type, switching automatically:

| Data Type | Small Encoding | Large Encoding |
|---|---|---|
| String | embstr (≤44 bytes), int | raw (SDS) |
| Hash | listpack (≤128 fields, ≤64 bytes/value) | hashtable |
| List | listpack (≤128 elements, ≤64 bytes) | quicklist |
| Set | listpack / intset | hashtable |
| Sorted Set | listpack (≤128 members) | skiplist + hashtable |

**SDS (Simple Dynamic String)**: Redis's own string implementation. Unlike C strings, SDS stores length explicitly (O(1) `STRLEN`) and is binary-safe (can store null bytes). Memory is pre-allocated to avoid reallocation on every append.

**Ziplist/Listpack**: A compact, contiguous memory block for small collections. CPU cache-friendly but O(N) for random access. Redis uses this for small hashes, lists, and sorted sets to save memory.

**Skiplist**: Used for sorted sets. O(log N) for rank queries. A probabilistic data structure — multiple levels of linked lists where each level skips ahead farther. Allows range queries efficiently.

## Persistence

### RDB (Redis Database Backup)

Point-in-time snapshots. Redis **forks** the process, and the child writes the entire dataset to disk as a compact binary file.

```
# redis.conf
save 900 1      # snapshot if 1+ key changed in 900s
save 300 10     # snapshot if 10+ keys changed in 300s
save 60 10000   # snapshot if 10000+ keys changed in 60s
dbfilename dump.rdb
dir /var/lib/redis
```

**Pros**:
- Compact binary format — fast restarts
- Single file — easy to backup/move
- Minimal performance impact (child process does the work)

**Cons**:
- Data loss between snapshots (up to minutes)
- Fork can be slow on large datasets — causes latency spike
- On huge datasets (100GB+), fork + copy-on-write uses lots of memory

**When to use**: When you can tolerate some data loss and want fast restarts. Good for cache-only setups where losing a few minutes of cache is fine.

### AOF (Append-Only File)

Every write command is appended to a log file. On restart, Redis replays all commands.

```
# redis.conf
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec   # fsync every second (recommended)
# appendfsync always   # fsync every write (safest, slowest)
# appendfsync no       # let OS decide (fastest, least safe)

auto-aof-rewrite-percentage 100  # rewrite when AOF doubles
auto-aof-rewrite-min-size 64mb
```

**Pros**:
- Much lower data loss (1 second max with `everysec`)
- Human-readable log (can replay/audit commands)
- Automatic rewrite compacts the file

**Cons**:
- AOF files are larger than RDB
- Slower restart (must replay all commands vs load binary)
- AOF rewrite still requires fork

**When to use**: When data loss matters. Session storage, distributed locks, rate limit counters.

### Hybrid: RDB + AOF

```
# redis.conf
save 900 1
appendonly yes
aof-use-rdb-preamble yes  # AOF starts with RDB snapshot, then appends
```

Best of both: fast restart from RDB portion, minimal data loss from AOF tail. Recommended for most production setups where Redis is not purely a cache.

## Eviction Policies

When Redis runs out of memory (`maxmemory` reached):

```
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru
```

| Policy | Behavior | Use When |
|---|---|---|
| `noeviction` | Return error on writes | You must never lose data |
| `allkeys-lru` | Evict least recently used from all keys | General caching |
| `volatile-lru` | Evict LRU from keys with TTL only | Mix of cache + persistent data |
| `allkeys-lfu` | Evict least frequently used | Skewed access patterns |
| `volatile-lfu` | Evict LFU from TTL keys | Same as above but safer |
| `allkeys-random` | Evict random key | Uniform access, rare |
| `volatile-random` | Evict random TTL key | Rare |
| `volatile-ttl` | Evict key with nearest expiry | Prioritize short-lived data |

**LRU vs LFU**:
- LRU evicts keys not accessed recently — good for temporal locality
- LFU evicts keys accessed least overall — better for skewed workloads where some keys are always hot

**Production recommendation**: `allkeys-lru` for pure cache, `volatile-lru` if you mix cached and persistent data. Always set `maxmemory`.

## Replication

Redis uses **async primary-replica replication**:

```
# On replica's redis.conf
replicaof <master-ip> 6379
replica-read-only yes
```

Flow:
1. Replica connects to primary
2. Primary forks, sends RDB snapshot to replica (full sync)
3. Primary buffers commands during snapshot transfer
4. After snapshot loaded, replica catches up via replication buffer
5. Ongoing: primary sends commands to replicas asynchronously

**Key trade-off**: Async means replica can lag behind primary. If primary crashes before replication, that data is lost. `WAIT numreplicas timeout` command blocks until N replicas acknowledge — use for stronger durability guarantees.

## Redis Sentinel

High-availability solution for **non-cluster** Redis setups:

```
# sentinel.conf
sentinel monitor mymaster 127.0.0.1 6379 2  # quorum = 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

Sentinel provides:
- **Monitoring**: Detects primary failure via heartbeats
- **Notification**: Alerts admins via pub/sub
- **Automatic failover**: Promotes a replica to primary
- **Configuration provider**: Clients query Sentinel for current primary address

**Quorum**: Minimum Sentinels that must agree primary is down before failover. Use 3 Sentinels with quorum 2 for production.

**Limitation**: Sentinel is not a distributed system — it doesn't shard data. One primary, one dataset.

## Redis Cluster

Horizontal scaling: data is **sharded** across multiple primary nodes.

```
# Hash slots: 16384 total (0–16383)
# Node A: slots 0–5460
# Node B: slots 5461–10922
# Node C: slots 10923–16383

# Key → slot: CRC16(key) % 16384
```

**Hash tags**: Force multiple keys to the same slot:
```
{user:123}:profile    # both go to same slot
{user:123}:sessions   # based on {user:123}
```

**Why this matters**: Multi-key commands (`MSET`, `SUNION`, transactions) require all keys in the same slot. Without hash tags, cross-slot operations fail.

**Cluster setup requirements**:
- Minimum 3 primaries (for proper quorum)
- Each primary should have 1+ replicas
- Clients must be cluster-aware (use `Lettuce` or `Redisson` in Java)

**Failure handling**: If a primary fails, its replica is promoted. If a primary has no replica and fails, the cluster can become unavailable for those slots (configurable).

## Why Redis Is Fast

1. **RAM access**: ~100ns vs ~10ms for SSD, ~100ms for spinning disk
2. **No query planning**: Commands map directly to data structures
3. **Single-threaded**: No lock contention, no context switching
4. **Event loop**: No thread-per-connection overhead
5. **Efficient protocols**: RESP is simple to parse
6. **Optimized data structures**: Ziplist/listpack uses CPU cache lines efficiently
7. **No marshalling overhead**: Data stays in-process memory

---

# 3. Data Structures Deep Dive

## String

**Internal representation**: SDS (Simple Dynamic String) for most values, `int` encoding for integers, `embstr` for strings ≤44 bytes (single allocation).

**Commands and complexity**:

| Command | Complexity | Notes |
|---|---|---|
| `GET` / `SET` | O(1) | Basic |
| `INCR` / `DECR` | O(1) | Atomic counter |
| `APPEND` | O(1) amortized | SDS pre-allocates |
| `GETRANGE` | O(N) | N = length of returned string |
| `SETEX` / `SET EX` | O(1) | Set with TTL |
| `SETNX` / `SET NX` | O(1) | Set if not exists |

**Real-world usage**:
- Caching serialized JSON objects: `SET user:123 "{...}" EX 3600`
- Distributed counters: `INCR page:views:home`
- Flags/feature toggles: `SET feature:dark-mode "true" EX 86400`
- Distributed locks: `SET lock:resource uuid NX EX 30`

**Anti-patterns**:
- Storing huge JSON blobs (>1MB) — use compression or split into Hash
- Using String as a poor man's Hash (`SET user:123:name "Alice"`) — creates millions of keys, wastes memory; use Hash instead
- Storing binary data without considering memory (Redis adds ~90 bytes overhead per key)

## Hash

**Internal representation**: `listpack` for small hashes (≤128 fields, field+value ≤64 bytes), `hashtable` for larger ones.

**Commands and complexity**:

| Command | Complexity | Notes |
|---|---|---|
| `HSET` / `HGET` | O(1) | Single field |
| `HMSET` / `HMGET` | O(N) | N fields |
| `HGETALL` | O(N) | **Danger on large hashes** |
| `HDEL` | O(N) | N fields deleted |
| `HLEN` | O(1) | |
| `HSCAN` | O(1) per call | Use for large hashes |

**Real-world usage**:
- User profile: `HSET user:123 name "Alice" email "a@b.com" age 30`
- Shopping cart: `HSET cart:session123 product:456 2 product:789 1`
- Configuration objects: `HSET config:app max_connections 100 timeout 30`

**Memory advantage**: A Hash with 100 fields uses far less memory than 100 separate String keys due to listpack encoding and shared key overhead.

**Anti-patterns**:
- `HGETALL` on a Hash with thousands of fields — blocks Redis
- Using Hash field values as lists (comma-separated) — use List instead
- Nesting by encoding JSON as a field value — you lose the ability to query sub-fields

## List

**Internal representation**: `listpack` for small lists (≤128 elements, ≤64 bytes each), `quicklist` (linked list of listpack nodes) for larger ones.

**Commands and complexity**:

| Command | Complexity | Notes |
|---|---|---|
| `LPUSH` / `RPUSH` | O(1) | Push to head/tail |
| `LPOP` / `RPOP` | O(1) | Pop from head/tail |
| `LRANGE` | O(N) | N = elements returned |
| `LLEN` | O(1) | |
| `LINDEX` | O(N) | N = index position |
| `LINSERT` | O(N) | Must find pivot element |
| `BLPOP` / `BRPOP` | O(1) | **Blocking pop — great for queues** |

**Real-world usage**:
- **FIFO queue**: `RPUSH queue:email task1` → `BLPOP queue:email 0` (blocking, waits for items)
- **Recent activity feed**: `LPUSH activity:user:123 event1` → `LTRIM activity:user:123 0 99` (keep last 100)
- **Pagination cache**: Store IDs in list, fetch pages via `LRANGE`

**Anti-patterns**:
- Using `LINDEX` repeatedly for random access — O(N) each time; use Hash or Sorted Set instead
- Huge lists with `LRANGE 0 -1` (return all) — same as `HGETALL`
- Not trimming lists — they grow unbounded

## Set

**Internal representation**: `intset` for small integer-only sets, `listpack` for small mixed sets, `hashtable` for large sets.

**Commands and complexity**:

| Command | Complexity | Notes |
|---|---|---|
| `SADD` / `SREM` | O(1) | |
| `SISMEMBER` | O(1) | Membership check |
| `SMEMBERS` | O(N) | **Danger on large sets** |
| `SUNION` / `SINTER` | O(N+M) | Across sets |
| `SCARD` | O(1) | Cardinality |
| `SSCAN` | O(1) per call | Safe for large sets |

**Real-world usage**:
- Tags on articles: `SADD article:123:tags "redis" "backend" "caching"`
- Online users: `SADD online:users user:456` with TTL refresh
- Unique visitors per day: `SADD visitors:2024-01-15 user:789`
- Permissions: `SADD user:123:permissions "read" "write"`

**Anti-patterns**:
- `SMEMBERS` on a large set (>10k members) — blocks Redis; use `SSCAN`
- Using Set when order matters — use Sorted Set
- Storing huge objects as members — members should be IDs, not full objects

## Sorted Set (ZSet)

**Internal representation**: `listpack` for small sorted sets (≤128 members), `skiplist + hashtable` for larger ones. The hashtable provides O(1) score lookup by member; the skiplist provides O(log N) rank queries.

**Commands and complexity**:

| Command | Complexity | Notes |
|---|---|---|
| `ZADD` | O(log N) | |
| `ZRANGE` / `ZREVRANGE` | O(log N + M) | M = elements returned |
| `ZRANGEBYSCORE` | O(log N + M) | Score range |
| `ZRANK` / `ZREVRANK` | O(log N) | Get rank of member |
| `ZSCORE` | O(1) | Get score of member |
| `ZCARD` | O(1) | |
| `ZINCRBY` | O(log N) | Atomic score increment |

**Real-world usage**:
- **Leaderboard**: `ZADD leaderboard 9500 "user:123"` → `ZREVRANGE leaderboard 0 9 WITHSCORES` (top 10)
- **Rate limiting**: `ZADD requests:user:123 <timestamp> <request_id>`, then `ZCOUNT` in window
- **Priority queue**: Lower score = higher priority
- **Time-series index**: Score = timestamp, member = event ID
- **Autocomplete**: Store prefixes as members with score 0

**Anti-patterns**:
- Using `ZRANGE 0 -1` on a huge sorted set — use pagination
- Storing full objects as members (should be IDs)
- Using when you don't need ordering — overhead vs Set

## HyperLogLog

**What it is**: A probabilistic data structure for counting **unique elements** with ~0.81% standard error. Uses only **12KB of memory** regardless of cardinality.

**Commands**:
```
PFADD unique:visitors:2024-01-15 user:1 user:2 user:3
PFCOUNT unique:visitors:2024-01-15   # → approximate unique count
PFMERGE weekly:visitors daily:mon daily:tue daily:wed
```

**Real-world usage**:
- Unique page views per day
- Unique active users (DAU/MAU)
- A/B test reach counting

**When NOT to use**:
- When you need exact counts
- When you need to check membership of a specific element (not possible)
- When your cardinality is <1000 (just use a Set — it's exact and uses less memory at that scale)

## Bitmap

**What it is**: Redis Strings used as arrays of bits. Each bit is addressed by offset.

```
SETBIT user:active:2024-01-15 userId 1   # mark user active on this day
GETBIT user:active:2024-01-15 userId     # check if active
BITCOUNT user:active:2024-01-15          # count active users
BITOP AND result bitmap1 bitmap2         # users active on BOTH days
```

**Real-world usage**:
- Daily active user tracking (bit per user per day)
- Feature flag per user
- User has completed onboarding steps (bit per step)
- Bloom filter implementation

**Memory calculation**: Tracking 1 million users = 1 million bits = 125KB. Incredibly efficient.

**Anti-patterns**:
- Very sparse bitmaps with huge offsets (userId = 100,000,000 → 12MB bitmap for one user)
- Using when userId space is not compact integers

## Streams

**What it is**: A log-like data structure. Append-only, entries have auto-generated IDs (timestamp-sequence). Supports consumer groups for parallel processing.

```
# Producer
XADD orders * product_id 123 quantity 2 user_id 456
# Returns: "1705312800000-0" (timestamp-sequence)

# Consumer Group
XGROUP CREATE orders processing $ MKSTREAM
XREADGROUP GROUP processing worker1 COUNT 10 BLOCK 5000 STREAMS orders >
XACK orders processing 1705312800000-0
```

**Real-world usage**:
- Event sourcing
- Activity feeds with guaranteed delivery
- Microservice event bus
- Audit logs

**Advantage over List**: Consumer groups allow multiple workers to process in parallel without duplicate processing. Messages are acknowledged, so failures can be reprocessed.

**Anti-patterns**:
- Using Streams when you just need simple pub/sub (use Pub/Sub — simpler)
- Not setting `MAXLEN` — Streams grow forever, consuming memory

---

# 4. Redis in System Design

## Cache-Aside Pattern

The most common caching pattern. Application manages cache explicitly.

```
Application Logic:
1. Try to get data from Redis
2. If hit → return cached data
3. If miss → query database → store in Redis → return data
```

**Pros**:
- Application controls what goes in cache
- Cache only contains data that's actually requested
- Cache failure doesn't break the system (fall through to DB)

**Cons**:
- First request always hits DB (cold start)
- Potential stale data between DB update and cache invalidation
- Thundering herd on cache miss (many requests simultaneously miss and all hit DB)

**Mitigation — Cache Stampede**:
```java
// Probabilistic early expiration (PER) / mutex lock pattern
String value = redis.get(key);
if (value == null) {
    boolean locked = redis.set(lockKey, "1", SetArgs.Builder.nx().ex(5));
    if (locked) {
        try {
            value = database.query();
            redis.setex(key, 3600, value);
        } finally {
            redis.del(lockKey);
        }
    } else {
        // Wait briefly and retry, or return stale data
        Thread.sleep(50);
        return redis.get(key);  // second attempt
    }
}
```

## Write-Through

Write to cache AND database synchronously on every write.

```
Write request → Application → Redis (write) → Database (write) → Response
```

**Pros**:
- Cache always consistent with DB
- No stale data

**Cons**:
- Write latency includes both Redis AND DB
- Cache fills with data that may never be read

**When to use**: When read/write ratio is balanced and consistency is critical.

## Write-Behind (Write-Back)

Write to cache immediately, asynchronously write to database.

```
Write request → Application → Redis (write) → Response
                                     ↓ (async)
                                   Database
```

**Pros**:
- Very fast writes (only Redis, not DB)
- Can batch DB writes

**Cons**:
- Data loss if Redis fails before DB write
- More complex implementation

**When to use**: High write throughput scenarios where some data loss is acceptable.

## Distributed Locking

Critical for preventing race conditions in distributed systems.

### The Problem Without Locks

```
Thread A: reads inventory = 1
Thread B: reads inventory = 1
Thread A: decrements → sets inventory = 0
Thread B: decrements → sets inventory = 0  ← WRONG, should be -1 or error
```

### The Solution: SET NX EX

```
SET lock:order:payment:123 <unique-value> NX EX 30
```

- `NX`: Set only if Not eXists (atomic check-and-set)
- `EX 30`: Auto-expire after 30 seconds (prevents deadlock if holder crashes)
- `<unique-value>`: UUID so only the lock holder can release it

**Release with Lua script (atomic)**:
```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

### Pitfalls

1. **Clock drift**: If lock TTL is too short, lock expires while holder is still working. Solution: extend lock TTL with watchdog (Redisson does this automatically).

2. **GC pause**: JVM GC pause can cause lock to expire. The lock holder resumes after GC and thinks it still holds the lock. Solution: validate lock still held before committing.

3. **Network partition**: In Redis Cluster/Sentinel, a failover can cause two nodes to think they hold the same lock (split-brain). Solution: **Redlock algorithm** uses 5+ independent Redis nodes — acquire lock on majority.

4. **Single point of failure**: Single Redis node crash loses all locks. Solution: Sentinel failover or Redlock.

### Redlock Algorithm

For strong guarantees, use Redlock with 5 independent Redis instances:
1. Get current timestamp T1
2. Try to acquire lock on all 5 instances with TTL
3. If acquired on majority (3+) and elapsed time < TTL → lock acquired
4. Validity time = TTL - (T2 - T1) - clock drift
5. Release lock on all instances

**Controversy**: Martin Kleppmann argued Redlock is unsafe under certain failure modes. For critical systems, use ZooKeeper or etcd. For most practical use cases, single-node Redis lock is acceptable.

## Rate Limiting Design

### Fixed Window Counter

```
Key: ratelimit:user:123:2024011515  (hour window)
INCR → if result > limit, reject
EXPIRE key 3600
```

**Problem**: User can make 2x limit requests by sending at end of window + start of next.

### Sliding Window Log

```
ZADD requests:user:123 <now_ms> <request_uuid>
ZREMRANGEBYSCORE requests:user:123 0 <now_ms - window_ms>
count = ZCARD requests:user:123
if count >= limit: reject
```

**Problem**: Stores one entry per request. Memory-intensive for high-traffic users.

### Sliding Window Counter (Token Bucket approximation)

```lua
local current_window = math.floor(now / window_size)
local prev_window = current_window - 1

local prev_count = redis.call("GET", key .. ":" .. prev_window) or 0
local curr_count = redis.call("GET", key .. ":" .. current_window) or 0

local elapsed = (now % window_size) / window_size
local approx = prev_count * (1 - elapsed) + curr_count

if approx >= limit then return 0 end

redis.call("INCR", key .. ":" .. current_window)
redis.call("EXPIRE", key .. ":" .. current_window, window_size * 2)
return 1
```

Best of both worlds: low memory, accurate smoothing.

## Session Storage

Why Redis over database for sessions:

1. **Read performance**: Sessions are read on every authenticated request
2. **TTL**: Built-in expiry without a cleanup job
3. **Horizontal scaling**: All app servers share one Redis, no sticky sessions needed
4. **Simple invalidation**: `DEL session:<id>` instantly logs out user

```
Key:   session:<session_uuid>
Value: Hash { userId, email, roles, created_at, last_active }
TTL:   30 minutes (reset on each request)
```

**Failure scenario**: Redis goes down → all users are logged out. Mitigation: Redis Sentinel or Cluster, and design for graceful degradation (temporary maintenance page).

## Event Streaming (Redis Streams)

```
Producer (Order Service) → XADD orders * ...
                                        ↓
                              Consumer Group: payment-workers
                              - worker1: XREADGROUP ... XACK
                              - worker2: XREADGROUP ... XACK

Dead letter: XPENDING + XCLAIM for messages unacknowledged > N seconds
```

**Failure scenario**: Worker crashes mid-processing. Message stays as pending. Use `XPENDING` to find stuck messages and `XCLAIM` to reassign to another worker.

---

# 5. Production Best Practices

## Memory Management

### Set maxmemory Always

```
# redis.conf
maxmemory 6gb                    # Leave 2GB for OS on 8GB machine
maxmemory-policy allkeys-lru
```

Without `maxmemory`, Redis grows until OOM killer kills the process. This is never what you want.

### Monitor memory fragmentation ratio

```bash
redis-cli INFO memory | grep mem_fragmentation_ratio
```

- `1.0–1.5`: Normal
- `>1.5`: High fragmentation — consider `MEMORY PURGE` or restart
- `<1.0`: Redis is using swap — severe performance degradation

### Use OBJECT ENCODING to verify memory savings

```bash
redis-cli OBJECT ENCODING mykey
# → listpack, hashtable, skiplist, embstr, etc.
```

Tune thresholds to maximize listpack usage:
```
hash-max-listpack-entries 128
hash-max-listpack-value 64
zset-max-listpack-entries 128
zset-max-listpack-value 64
```

## TTL Strategies

**Always set TTLs** on cache entries. Memory-filled-with-stale-data is a common production failure.

```
# Short TTL (seconds to minutes): volatile/computed data
SET session:abc token EX 1800

# Medium TTL (hours): user-specific cached data  
SET user:123:profile json EX 3600

# Long TTL (days): rarely changing reference data
SET product:456:details json EX 86400

# No TTL only for: configuration that Redis owns, locks (always have TTL), 
#                  persistent data you're treating Redis as primary store
```

**TTL jitter**: To prevent cache stampede on expiry, add random jitter:
```java
int ttl = 3600 + random.nextInt(300);  // 3600 ± 5 minutes
redis.setex(key, ttl, value);
```

## Key Naming Conventions

Use a consistent namespace pattern: `{namespace}:{entity_type}:{identifier}:{sub_field}`

```
user:123:profile          ✓
user:123:sessions         ✓
order:456:items           ✓
ratelimit:api:user:789    ✓
lock:payment:order:456    ✓
cache:product:789:price   ✓

u123p                     ✗  (cryptic)
USER_PROFILE_123          ✗  (no nesting convention)
user123profile            ✗  (no delimiter)
```

**Rules**:
- Use `:` as delimiter
- Lowercase
- Most-general to most-specific (like DNS, reversed)
- Include entity type to avoid collisions
- Document your naming scheme

## Avoiding Hot Keys

A hot key is one that receives a disproportionate share of traffic. Since Redis is single-threaded, one client hammering one key can saturate the instance.

**Detection**:
```bash
redis-cli --hotkeys          # Redis 4.0+, requires LFU eviction
redis-cli monitor | grep KEY # Watch live traffic (expensive, dev only)
```

**Solutions**:
1. **Local in-process cache**: Cache hot values in JVM heap (Caffeine) for 1–5 seconds
2. **Key splitting**: `product:123:stock` → `product:123:stock:shard:3` (read from random shard, write to all)
3. **Read replicas**: Route reads to replicas for read-heavy hot keys

## Handling Large Values

Large values (>10KB) are problematic:
- Slow to serialize/deserialize
- Block the network
- Cause memory fragmentation

**Strategies**:
- Compress before storing: `gzip`, `snappy`, or `lz4`
- Store large blobs in S3/object storage, cache only the URL or metadata in Redis
- Break large objects into chunks
- Use Hashes instead of serialized JSON for structured data

## Monitoring & Metrics

Key metrics to watch in production:

| Metric | Command | Alert Threshold |
|---|---|---|
| Memory usage | `INFO memory` → `used_memory` | >80% of maxmemory |
| Hit rate | `INFO stats` → keyspace_hits / (hits+misses) | <90% for cache |
| Connected clients | `INFO clients` | >80% of maxclients |
| Blocked clients | `INFO clients` → `blocked_clients` | >0 for extended time |
| Replication lag | `INFO replication` → `master_repl_offset` | >1MB |
| Evicted keys | `INFO stats` → `evicted_keys` | Should be ~0 if data matters |
| Command latency | `LATENCY HISTORY` | p99 >10ms |
| Slow log | `SLOWLOG GET 10` | Any command >10ms |

```bash
# Check slow commands
redis-cli SLOWLOG GET 10

# Reset slow log
redis-cli SLOWLOG RESET

# Configure threshold (microseconds)
redis-cli CONFIG SET slowlog-log-slower-than 10000  # 10ms
```

## Common Production Mistakes

1. **No maxmemory set**: Redis OOMs and gets killed by the OS.
2. **Using KEYS \* in production**: O(N) over entire keyspace, blocks everything.
3. **No TTLs on cache**: Cache fills, eviction starts, unpredictable behavior.
4. **Large numbers of small keys**: >50M keys → memory overhead dominates.
5. **HGETALL / SMEMBERS on large collections**: Single O(N) command blocks all clients.
6. **Storing sessions without Sentinel/Cluster**: Single Redis failure logs everyone out.
7. **Not accounting for serialization cost**: Java object serialization is slow; benchmark it.
8. **Sync operations in Redis callbacks**: Never call Redis from within a Redis callback.
9. **No connection pooling**: Creating a new connection per operation is extremely slow.
10. **Ignoring replication lag in read-from-replica scenarios**: Reading from replica can return stale data.

---

# 6. Redis with Spring Boot

## Setup

### Maven Dependencies

```xml
<dependencies>
    <!-- Spring Data Redis (includes Lettuce by default) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    
    <!-- Spring Cache abstraction -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
    
    <!-- Redisson for distributed primitives (locks, rate limiters) -->
    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson-spring-boot-starter</artifactId>
        <version>3.27.0</version>
    </dependency>
    
    <!-- Jackson for JSON serialization -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD:}
      database: 0
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 20       # Max connections in pool
          max-idle: 10         # Max idle connections
          min-idle: 5          # Min idle connections kept alive
          max-wait: 1000ms     # Max time to wait for connection
        shutdown-timeout: 100ms

  cache:
    type: redis
    redis:
      time-to-live: 3600000    # 1 hour in milliseconds
      cache-null-values: false  # Don't cache null results
      key-prefix: "myapp:"
      use-key-prefix: true

# Redisson config (if using redisson-spring-boot-starter)
# Creates redisson.yaml in resources, or configure programmatically
```

### Redis Configuration Bean

```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        
        // Key serializer: plain strings
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        
        // Value serializer: JSON
        Jackson2JsonRedisSerializer<Object> jsonSerializer = 
            new Jackson2JsonRedisSerializer<>(Object.class);
        
        ObjectMapper mapper = new ObjectMapper();
        mapper.activateDefaultTyping(
            mapper.getPolymorphicTypeValidator(),
            ObjectMapper.DefaultTyping.NON_FINAL,
            JsonTypeInfo.As.PROPERTY
        );
        jsonSerializer.setObjectMapper(mapper);
        
        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);
        template.afterPropertiesSet();
        
        return template;
    }
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer())
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new Jackson2JsonRedisSerializer<>(Object.class))
            )
            .disableCachingNullValues();
        
        // Per-cache TTL configuration
        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();
        cacheConfigs.put("users", config.entryTtl(Duration.ofMinutes(30)));
        cacheConfigs.put("products", config.entryTtl(Duration.ofHours(6)));
        cacheConfigs.put("sessions", config.entryTtl(Duration.ofMinutes(30)));
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
    
    // StringRedisTemplate is pre-configured for String key/value
    // It's registered automatically by Spring Boot
}
```

---

## Example 1: Basic Redis Operations

Use case: Manual key-value operations when you need full control.

```java
@Service
@RequiredArgsConstructor
public class RedisBasicService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final StringRedisTemplate stringTemplate;

    // String operations
    public void cacheUserProfile(Long userId, UserProfile profile) {
        String key = "user:" + userId + ":profile";
        redisTemplate.opsForValue().set(key, profile, Duration.ofHours(1));
    }
    
    public Optional<UserProfile> getUserProfile(Long userId) {
        String key = "user:" + userId + ":profile";
        Object value = redisTemplate.opsForValue().get(key);
        return Optional.ofNullable((UserProfile) value);
    }
    
    // Hash operations - more memory-efficient than one key per field
    public void cacheUserFields(Long userId, Map<String, String> fields) {
        String key = "user:" + userId + ":meta";
        redisTemplate.opsForHash().putAll(key, fields);
        redisTemplate.expire(key, Duration.ofHours(1));
    }
    
    public String getUserField(Long userId, String field) {
        String key = "user:" + userId + ":meta";
        return (String) redisTemplate.opsForHash().get(key, field);
    }
    
    // Counter (atomic)
    public Long incrementPageView(String pageSlug) {
        String key = "pageviews:" + pageSlug + ":" + LocalDate.now();
        Long count = redisTemplate.opsForValue().increment(key);
        // Set TTL only on first increment
        if (count == 1) {
            redisTemplate.expire(key, Duration.ofDays(7));
        }
        return count;
    }
    
    // Set operations - track unique visitors
    public void trackVisitor(String page, String userId) {
        String key = "visitors:" + page + ":" + LocalDate.now();
        redisTemplate.opsForSet().add(key, userId);
        redisTemplate.expire(key, Duration.ofDays(30));
    }
    
    public Long getUniqueVisitorCount(String page) {
        String key = "visitors:" + page + ":" + LocalDate.now();
        return redisTemplate.opsForSet().size(key);
    }
    
    // Sorted Set - leaderboard
    public void updateScore(String leaderboard, String userId, double score) {
        redisTemplate.opsForZSet().add(leaderboard, userId, score);
    }
    
    public Set<ZSetOperations.TypedTuple<Object>> getTopN(String leaderboard, int n) {
        return redisTemplate.opsForZSet()
            .reverseRangeWithScores(leaderboard, 0, n - 1);
    }
    
    // List - recent activity feed
    public void addActivity(Long userId, String activity) {
        String key = "activity:" + userId;
        redisTemplate.opsForList().leftPush(key, activity);
        redisTemplate.opsForList().trim(key, 0, 99);  // Keep last 100
        redisTemplate.expire(key, Duration.ofDays(30));
    }
    
    public List<Object> getRecentActivity(Long userId, int count) {
        String key = "activity:" + userId;
        return redisTemplate.opsForList().range(key, 0, count - 1);
    }
    
    // Delete
    public void evictUser(Long userId) {
        // Pattern-based deletion — use SCAN, never KEYS
        Set<String> keys = scanKeys("user:" + userId + ":*");
        if (!keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }
    
    private Set<String> scanKeys(String pattern) {
        return redisTemplate.execute((RedisCallback<Set<String>>) connection -> {
            Set<String> keys = new HashSet<>();
            Cursor<byte[]> cursor = connection.keyCommands().scan(
                ScanOptions.scanOptions().match(pattern).count(100).build()
            );
            while (cursor.hasNext()) {
                keys.add(new String(cursor.next()));
            }
            return keys;
        });
    }
}
```

**Why this works**: `RedisTemplate` wraps Lettuce connections with Spring's connection pool. All operations use pooled connections. `opsForValue()`, `opsForHash()`, etc. return operation objects that execute on the same connection.

---

## Example 2: Spring Cache with Redis (`@Cacheable`)

Use case: Declarative caching. Best for methods with stable results and clear invalidation events.

```java
@Service
@RequiredArgsConstructor
@CacheConfig(cacheNames = "products")
public class ProductService {

    private final ProductRepository productRepository;

    // Cache result. Key = "products::123"
    // On cache miss: executes method, stores result
    // On cache hit: returns cached value, method NOT called
    @Cacheable(key = "#productId")
    public ProductDto getProduct(Long productId) {
        return productRepository.findById(productId)
            .map(this::toDto)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }
    
    // Condition: only cache if product is active
    @Cacheable(key = "#productId", condition = "#result != null && #result.active()")
    public ProductDto getActiveProduct(Long productId) {
        return productRepository.findActiveById(productId)
            .map(this::toDto)
            .orElse(null);
    }
    
    // Cache multiple at once
    @Cacheable(key = "'category:' + #categoryId")
    public List<ProductDto> getProductsByCategory(Long categoryId) {
        return productRepository.findByCategoryId(categoryId)
            .stream().map(this::toDto).toList();
    }
    
    // Update cache on write (cache-put pattern)
    @CachePut(key = "#product.id()")
    public ProductDto updateProduct(ProductDto product) {
        Product saved = productRepository.save(toEntity(product));
        return toDto(saved);
    }
    
    // Evict on delete
    @CacheEvict(key = "#productId")
    public void deleteProduct(Long productId) {
        productRepository.deleteById(productId);
    }
    
    // Evict entire cache (careful in production — thundering herd)
    @CacheEvict(allEntries = true)
    @Scheduled(cron = "0 0 2 * * *")  // 2 AM daily
    public void clearProductCache() {
        // Just the annotation is enough — method body optional
    }
    
    // Evict multiple caches
    @Caching(evict = {
        @CacheEvict(cacheNames = "products", key = "#productId"),
        @CacheEvict(cacheNames = "product-lists", allEntries = true)
    })
    public void deleteProductAndLists(Long productId) {
        productRepository.deleteById(productId);
    }
}
```

**When to use `@Cacheable` vs manual**: 

- `@Cacheable`: Great for read-heavy service methods, clean code, standard TTL
- Manual: When you need TTL per-item, conditional caching logic, or non-method cache operations (e.g., cache warming, bulk operations)

**Pitfall — self-invocation**: `@Cacheable` uses Spring AOP proxy. Calling a `@Cacheable` method from within the **same class** bypasses the proxy and the cache. Always inject the service and call it through the injected reference, or use `@EnableCaching(proxyTargetClass = true)`.

---

## Example 3: Manual Caching with Fallback

Use case: Complex caching logic not expressible with annotations.

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final UserRepository userRepository;
    private static final String USER_CACHE_PREFIX = "user:";
    private static final Duration USER_CACHE_TTL = Duration.ofHours(1);
    
    public UserDto getUser(Long userId) {
        String key = USER_CACHE_PREFIX + userId;
        
        // 1. Try cache
        UserDto cached = (UserDto) redisTemplate.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // 2. Fetch from DB
        UserDto user = userRepository.findById(userId)
            .map(this::toDto)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        // 3. Store with jitter to prevent stampede
        int ttlSeconds = (int) USER_CACHE_TTL.toSeconds() + new Random().nextInt(300);
        redisTemplate.opsForValue().set(key, user, Duration.ofSeconds(ttlSeconds));
        
        return user;
    }
    
    // Cache-aside with null caching (prevents cache penetration)
    public UserDto getUserWithNullCaching(Long userId) {
        String key = USER_CACHE_PREFIX + userId;
        String nullSentinel = "NULL_SENTINEL";
        
        Object cached = redisTemplate.opsForValue().get(key);
        
        if (nullSentinel.equals(cached)) {
            return null;  // Cached as null — user doesn't exist
        }
        if (cached instanceof UserDto dto) {
            return dto;
        }
        
        Optional<User> user = userRepository.findById(userId);
        if (user.isEmpty()) {
            // Cache the "not found" result for a short time
            redisTemplate.opsForValue().set(key, nullSentinel, Duration.ofMinutes(5));
            return null;
        }
        
        UserDto dto = toDto(user.get());
        redisTemplate.opsForValue().set(key, dto, USER_CACHE_TTL);
        return dto;
    }
    
    // Bulk cache loading
    public Map<Long, UserDto> getUsers(List<Long> userIds) {
        List<String> keys = userIds.stream()
            .map(id -> USER_CACHE_PREFIX + id)
            .toList();
        
        // Redis MGET — fetch all in one round trip
        List<Object> cached = redisTemplate.opsForValue().multiGet(keys);
        
        Map<Long, UserDto> result = new HashMap<>();
        List<Long> missedIds = new ArrayList<>();
        
        for (int i = 0; i < userIds.size(); i++) {
            Long userId = userIds.get(i);
            Object value = cached.get(i);
            if (value instanceof UserDto dto) {
                result.put(userId, dto);
            } else {
                missedIds.add(userId);
            }
        }
        
        if (!missedIds.isEmpty()) {
            // Batch fetch from DB
            List<User> dbUsers = userRepository.findAllById(missedIds);
            
            // Pipeline writes back to Redis (one network round trip)
            redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
                for (User user : dbUsers) {
                    UserDto dto = toDto(user);
                    result.put(user.getId(), dto);
                    String key = USER_CACHE_PREFIX + user.getId();
                    // Note: in pipeline, we can't use higher-level ops; use raw connection
                }
                return null;
            });
        }
        
        return result;
    }
    
    // Invalidate on update
    @Transactional
    public UserDto updateUser(Long userId, UpdateUserRequest request) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        user.setName(request.name());
        user.setEmail(request.email());
        User saved = userRepository.save(user);
        
        // Invalidate cache AFTER successful DB write
        redisTemplate.delete(USER_CACHE_PREFIX + userId);
        
        return toDto(saved);
    }
}
```

---

## Example 4: Distributed Lock

Use case: Prevent concurrent processing of the same resource (payment processing, inventory deduction, scheduled job deduplication).

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class DistributedLockService {

    private final RedissonClient redissonClient;
    
    /**
     * Execute a task with a distributed lock.
     * Uses Redisson's RLock which includes:
     * - Automatic lock renewal (watchdog)
     * - Fair lock option
     * - Safe release (only lock holder can release)
     */
    public <T> T executeWithLock(String lockName, long waitSeconds, 
                                  long leaseSeconds, Callable<T> task) {
        RLock lock = redissonClient.getLock("lock:" + lockName);
        boolean acquired = false;
        
        try {
            acquired = lock.tryLock(waitSeconds, leaseSeconds, TimeUnit.SECONDS);
            
            if (!acquired) {
                throw new LockAcquisitionException("Could not acquire lock: " + lockName);
            }
            
            log.debug("Lock acquired: {}", lockName);
            return task.call();
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new LockAcquisitionException("Interrupted while waiting for lock", e);
        } catch (Exception e) {
            throw new RuntimeException("Error executing locked task", e);
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
                log.debug("Lock released: {}", lockName);
            }
        }
    }
}

// Usage in payment service
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final DistributedLockService lockService;
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    
    public PaymentResult processPayment(Long orderId, PaymentRequest request) {
        String lockName = "order:payment:" + orderId;
        
        return lockService.executeWithLock(lockName, 5, 30, () -> {
            Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));
            
            if (order.isPaid()) {
                return PaymentResult.alreadyPaid();
            }
            
            PaymentResult result = paymentGateway.charge(request);
            
            if (result.isSuccess()) {
                order.markAsPaid(result.transactionId());
                orderRepository.save(order);
            }
            
            return result;
        });
    }
}

// Manual lock with RedisTemplate (no Redisson)
@Service
@RequiredArgsConstructor
public class ManualLockService {

    private final StringRedisTemplate redisTemplate;
    
    public boolean tryLock(String resource, String ownerId, Duration ttl) {
        String key = "lock:" + resource;
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(key, ownerId, ttl);
        return Boolean.TRUE.equals(result);
    }
    
    public boolean releaseLock(String resource, String ownerId) {
        String key = "lock:" + resource;
        
        // Atomic check-and-delete via Lua script
        String script = """
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('DEL', KEYS[1])
            else
                return 0
            end
            """;
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(key),
            ownerId
        );
        
        return Long.valueOf(1).equals(result);
    }
    
    public boolean extendLock(String resource, String ownerId, Duration extension) {
        String key = "lock:" + resource;
        
        String script = """
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('PEXPIRE', KEYS[1], ARGV[2])
            else
                return 0
            end
            """;
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(key),
            ownerId,
            String.valueOf(extension.toMillis())
        );
        
        return Long.valueOf(1).equals(result);
    }
}
```

**When to use Redisson vs manual**: Use Redisson for production — it handles watchdog (auto-renewal), handles node failure, and has been battle-tested. Use manual implementation only when you can't add a dependency.

---

## Example 5: Rate Limiter

Use case: API rate limiting per user/IP, preventing abuse.

```java
@Component
@RequiredArgsConstructor
public class RedisRateLimiter {

    private final StringRedisTemplate redisTemplate;
    
    // Sliding window using Sorted Set
    private static final String RATE_LIMIT_SCRIPT = """
        local key = KEYS[1]
        local now = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local limit = tonumber(ARGV[3])
        local request_id = ARGV[4]
        
        -- Remove old entries outside the window
        redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
        
        -- Count current requests in window
        local count = redis.call('ZCARD', key)
        
        if count >= limit then
            return {0, count, window}  -- rejected
        end
        
        -- Add current request
        redis.call('ZADD', key, now, request_id)
        redis.call('EXPIRE', key, math.ceil(window / 1000) + 1)
        
        return {1, count + 1, limit - count - 1}  -- allowed, current count, remaining
        """;
    
    public RateLimitResult isAllowed(String identifier, int limitPerMinute) {
        String key = "ratelimit:" + identifier;
        long now = System.currentTimeMillis();
        long windowMs = 60_000L;  // 1 minute
        String requestId = UUID.randomUUID().toString();
        
        List<Long> result = redisTemplate.execute(
            new DefaultRedisScript<>(RATE_LIMIT_SCRIPT, List.class),
            Collections.singletonList(key),
            String.valueOf(now),
            String.valueOf(windowMs),
            String.valueOf(limitPerMinute),
            requestId
        );
        
        boolean allowed = result.get(0) == 1L;
        long currentCount = result.get(1);
        long remaining = result.get(2);
        
        return new RateLimitResult(allowed, currentCount, remaining, limitPerMinute);
    }
    
    public record RateLimitResult(boolean allowed, long currentCount, 
                                   long remaining, int limit) {}
}

// Spring MVC integration via HandlerInterceptor
@Component
@RequiredArgsConstructor
public class RateLimitInterceptor implements HandlerInterceptor {

    private final RedisRateLimiter rateLimiter;
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                              HttpServletResponse response, Object handler) throws Exception {
        
        String identifier = resolveIdentifier(request);
        int limit = resolveLimit(handler);
        
        RedisRateLimiter.RateLimitResult result = rateLimiter.isAllowed(identifier, limit);
        
        // Always add rate limit headers
        response.setHeader("X-RateLimit-Limit", String.valueOf(result.limit()));
        response.setHeader("X-RateLimit-Remaining", String.valueOf(result.remaining()));
        response.setHeader("X-RateLimit-Current", String.valueOf(result.currentCount()));
        
        if (!result.allowed()) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("""
                {"error": "Rate limit exceeded", "retryAfter": 60}
                """);
            return false;
        }
        
        return true;
    }
    
    private String resolveIdentifier(HttpServletRequest request) {
        // Rate limit by user if authenticated, otherwise by IP
        String userId = (String) request.getAttribute("userId");
        if (userId != null) return "user:" + userId;
        
        String forwarded = request.getHeader("X-Forwarded-For");
        return "ip:" + (forwarded != null ? forwarded.split(",")[0].trim() 
                                          : request.getRemoteAddr());
    }
    
    private int resolveLimit(Object handler) {
        if (handler instanceof HandlerMethod method) {
            RateLimit annotation = method.getMethodAnnotation(RateLimit.class);
            if (annotation != null) return annotation.value();
        }
        return 100;  // default: 100 requests per minute
    }
}

// Custom annotation
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int value() default 100;
}
```

---

## Example 6: Pub/Sub

Use case: Real-time notifications, cache invalidation across instances, lightweight event broadcasting. **Not suitable for durable messaging** — if subscriber is down, messages are lost.

```java
// Message publisher
@Service
@RequiredArgsConstructor
public class NotificationPublisher {

    private final RedisTemplate<String, Object> redisTemplate;
    
    public void publishUserEvent(String eventType, Long userId, Object payload) {
        UserEvent event = new UserEvent(eventType, userId, payload, Instant.now());
        redisTemplate.convertAndSend("events:user:" + userId, event);
    }
    
    public void publishSystemAlert(String message) {
        redisTemplate.convertAndSend("alerts:system", message);
    }
    
    // Targeted: specific user channel
    public void notifyUser(Long userId, Notification notification) {
        String channel = "notifications:user:" + userId;
        redisTemplate.convertAndSend(channel, notification);
    }
}

// Message listener configuration
@Configuration
public class RedisPubSubConfig {

    @Bean
    public RedisMessageListenerContainer messageListenerContainer(
            RedisConnectionFactory factory,
            UserEventListener userEventListener) {
        
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);
        
        // Subscribe to specific channel
        container.addMessageListener(userEventListener, 
            new ChannelTopic("events:user:*"));
        
        // Pattern subscription
        container.addMessageListener(userEventListener,
            new PatternTopic("events:*"));
        
        return container;
    }
    
    @Bean
    public MessageConverter messageConverter() {
        return new GenericJackson2JsonMessageConverter();
    }
}

// Message listener implementation
@Component
@RequiredArgsConstructor
@Slf4j
public class UserEventListener implements MessageListener {

    private final SimpMessagingTemplate websocketTemplate;  // For WebSocket forwarding
    private final ObjectMapper objectMapper;
    
    @Override
    public void onMessage(Message message, byte[] pattern) {
        try {
            String channel = new String(message.getChannel());
            String body = new String(message.getBody());
            
            log.debug("Received message on channel: {}", channel);
            
            UserEvent event = objectMapper.readValue(body, UserEvent.class);
            
            // Forward to WebSocket
            websocketTemplate.convertAndSendToUser(
                String.valueOf(event.userId()),
                "/queue/events",
                event
            );
            
        } catch (Exception e) {
            log.error("Error processing pub/sub message", e);
        }
    }
}

// Cache invalidation across multiple app instances
@Service
@RequiredArgsConstructor
public class CacheInvalidationService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final CacheManager cacheManager;
    
    // Publish invalidation event to all instances
    public void invalidateUserCacheGlobally(Long userId) {
        redisTemplate.convertAndSend("cache:invalidate", "user:" + userId);
    }
    
    // Each instance listens and clears its local cache
    @Bean
    public MessageListener cacheInvalidationListener() {
        return (message, pattern) -> {
            String cacheKey = new String(message.getBody());
            Cache cache = cacheManager.getCache("users");
            if (cache != null) {
                cache.evict(cacheKey.replace("user:", ""));
            }
        };
    }
}
```

**Pub/Sub limitations**:
- No persistence: subscriber down = messages lost
- No consumer groups: all subscribers get every message
- No backpressure: fast publisher can overwhelm slow subscriber
- Use Redis Streams for durable, reliable messaging

---

## Example 7: Redis Streams

Use case: Durable event streaming with consumer groups, where guaranteed delivery matters.

```java
// Stream producer
@Service
@RequiredArgsConstructor
public class OrderEventProducer {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final String ORDER_STREAM = "stream:orders";
    
    public String publishOrderCreated(Order order) {
        Map<String, Object> message = Map.of(
            "event_type", "ORDER_CREATED",
            "order_id", order.getId(),
            "user_id", order.getUserId(),
            "total", order.getTotal(),
            "timestamp", Instant.now().toEpochMilli()
        );
        
        RecordId recordId = redisTemplate.opsForStream()
            .add(ObjectRecord.create(ORDER_STREAM, message));
        
        // Trim stream to prevent unbounded growth (keep last 10000 messages)
        redisTemplate.opsForStream().trim(ORDER_STREAM, 10000, true);  // true = approximate
        
        return recordId.getValue();
    }
}

// Stream consumer with consumer group
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderEventConsumer {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final String STREAM = "stream:orders";
    private static final String GROUP = "payment-service";
    private static final String CONSUMER = "payment-worker-" + UUID.randomUUID();
    
    @PostConstruct
    public void init() {
        // Create consumer group if not exists
        try {
            redisTemplate.opsForStream().createGroup(STREAM, 
                ReadOffset.from("0"),  // Read from beginning
                GROUP);
        } catch (Exception e) {
            // Group already exists — normal on restart
            log.debug("Consumer group already exists: {}", GROUP);
        }
    }
    
    @Scheduled(fixedDelay = 100)  // Poll every 100ms
    public void consume() {
        List<MapRecord<String, Object, Object>> messages = redisTemplate.opsForStream()
            .read(Consumer.from(GROUP, CONSUMER),
                  StreamReadOptions.empty().count(10).block(Duration.ofSeconds(1)),
                  StreamOffset.create(STREAM, ReadOffset.lastConsumed()));
        
        if (messages == null || messages.isEmpty()) return;
        
        for (MapRecord<String, Object, Object> record : messages) {
            try {
                processRecord(record);
                // Acknowledge successful processing
                redisTemplate.opsForStream().acknowledge(STREAM, GROUP, record.getId());
            } catch (Exception e) {
                log.error("Failed to process record: {}", record.getId(), e);
                // Don't acknowledge — will be reprocessed (check XPENDING)
            }
        }
    }
    
    // Recover stuck/failed messages
    @Scheduled(fixedDelay = 60000)  // Every minute
    public void recoverPendingMessages() {
        // Find messages pending for more than 5 minutes
        PendingMessages pending = redisTemplate.opsForStream()
            .pending(STREAM, Consumer.from(GROUP, CONSUMER), 
                     Range.unbounded(), 100);
        
        for (PendingMessage msg : pending) {
            if (msg.getElapsedTimeSinceLastDelivery().toMinutes() > 5) {
                // Claim the message
                List<MapRecord<String, Object, Object>> claimed = 
                    redisTemplate.opsForStream().claim(
                        STREAM, GROUP, CONSUMER,
                        Duration.ofMinutes(5),
                        msg.getId()
                    );
                
                for (MapRecord<String, Object, Object> record : claimed) {
                    try {
                        processRecord(record);
                        redisTemplate.opsForStream().acknowledge(STREAM, GROUP, record.getId());
                    } catch (Exception e) {
                        log.error("Dead letter: {}", record.getId(), e);
                        // Move to dead letter stream after N attempts
                    }
                }
            }
        }
    }
    
    private void processRecord(MapRecord<String, Object, Object> record) {
        Map<Object, Object> body = record.getValue();
        String eventType = (String) body.get("event_type");
        Long orderId = Long.parseLong(body.get("order_id").toString());
        
        log.info("Processing {} for order {}", eventType, orderId);
        // ... business logic
    }
}
```

---

# 7. Advanced Topics

## Pipelines

Pipelines batch multiple commands into one network round trip. Dramatically reduces latency for bulk operations.

```java
@Service
@RequiredArgsConstructor
public class PipelineService {

    private final RedisTemplate<String, Object> redisTemplate;
    
    // Without pipeline: N network round trips
    // With pipeline: 1 network round trip
    public void bulkCache(Map<String, Object> data) {
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            data.forEach((key, value) -> {
                byte[] keyBytes = key.getBytes();
                byte[] valueBytes = serialize(value);
                connection.stringCommands().setEx(keyBytes, 3600, valueBytes);
            });
            return null;
        });
    }
    
    // Pipeline for read operations
    public List<Object> bulkGet(List<String> keys) {
        return redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (String key : keys) {
                connection.stringCommands().get(key.getBytes());
            }
            return null;
        });
        // Returns list of results in same order as commands
    }
}
```

**Pipeline vs MGET**: For the same operation on multiple keys, MGET is simpler. Pipelines are for **mixed operations** (e.g., set some keys, expire others, increment counters — all in one batch).

**Gotcha**: In pipelines, commands are sent and results are buffered — you can't use the result of one command as input to the next within the same pipeline. For that, use Lua scripts.

## Transactions

Redis transactions (`MULTI/EXEC`) guarantee atomic execution of a block of commands, but they differ from SQL transactions:

- **No rollback**: If a command fails mid-transaction, the others still execute
- **No read-your-writes**: You can't read a value written in the same `MULTI/EXEC` block
- **Optimistic locking via WATCH**: A watched key's change causes transaction to fail

```java
// WATCH + MULTI/EXEC for optimistic locking
public boolean transferPoints(String fromUser, String toUser, int amount) {
    return redisTemplate.execute(new SessionCallback<Boolean>() {
        @Override
        public Boolean execute(RedisOperations operations) throws DataAccessException {
            String fromKey = "points:" + fromUser;
            String toKey = "points:" + toUser;
            
            operations.watch(fromKey);  // Watch for concurrent modification
            
            Integer fromBalance = (Integer) operations.opsForValue().get(fromKey);
            if (fromBalance == null || fromBalance < amount) {
                operations.unwatch();
                return false;
            }
            
            operations.multi();  // Start transaction
            operations.opsForValue().decrement(fromKey, amount);
            operations.opsForValue().increment(toKey, amount);
            
            List<Object> results = operations.exec();  // Execute or null if WATCH key changed
            return results != null;  // null = transaction aborted (concurrent modification)
        }
    });
}
```

**When transactions aren't the right tool**: For most atomicity needs, Lua scripts are simpler and more powerful. Use `MULTI/EXEC` primarily when you need WATCH-based optimistic concurrency.

## Lua Scripting

Lua scripts execute atomically on Redis's single thread. The script and all its operations run without interruption — more powerful than MULTI/EXEC.

```java
// Token bucket rate limiter — more accurate than sliding window
private static final String TOKEN_BUCKET_SCRIPT = """
    local key = KEYS[1]
    local capacity = tonumber(ARGV[1])
    local refill_rate = tonumber(ARGV[2])   -- tokens per second
    local now = tonumber(ARGV[3])           -- current time in milliseconds
    local requested = tonumber(ARGV[4])     -- tokens requested
    
    local data = redis.call('HMGET', key, 'tokens', 'last_refill')
    local tokens = tonumber(data[1]) or capacity
    local last_refill = tonumber(data[2]) or now
    
    -- Calculate tokens to add since last refill
    local elapsed = (now - last_refill) / 1000  -- convert to seconds
    local new_tokens = math.min(capacity, tokens + elapsed * refill_rate)
    
    if new_tokens < requested then
        -- Not enough tokens
        redis.call('HMSET', key, 'tokens', new_tokens, 'last_refill', now)
        redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) + 1)
        return {0, math.floor(new_tokens)}
    end
    
    -- Consume tokens
    local remaining = new_tokens - requested
    redis.call('HMSET', key, 'tokens', remaining, 'last_refill', now)
    redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) + 1)
    return {1, math.floor(remaining)}
    """;

@Service
@RequiredArgsConstructor
public class TokenBucketRateLimiter {

    private final RedisTemplate<String, Object> redisTemplate;
    private final DefaultRedisScript<List> script = new DefaultRedisScript<>(TOKEN_BUCKET_SCRIPT, List.class);
    
    public boolean consume(String identifier, int capacity, double refillRatePerSecond) {
        String key = "tokenbucket:" + identifier;
        long now = System.currentTimeMillis();
        
        List<Long> result = redisTemplate.execute(script,
            Collections.singletonList(key),
            String.valueOf(capacity),
            String.valueOf(refillRatePerSecond),
            String.valueOf(now),
            "1"  // requesting 1 token
        );
        
        return result.get(0) == 1L;
    }
}
```

**Rules for Lua in Redis**:
- All keys must be declared in `KEYS` array (for cluster routing)
- Lua scripts have a 5-second execution limit by default
- No I/O operations (no HTTP calls inside Lua)
- Time is available via `redis.call('TIME')` — returns [seconds, microseconds]

## Performance Tuning

```bash
# Disable Transparent Huge Pages (major latency cause)
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Increase somaxconn for more connection backlog
sysctl -w net.core.somaxconn=65535

# Disable swap (Redis in swap = disaster)
swapoff -a

# Check for slow commands
redis-cli SLOWLOG GET 10
redis-cli SLOWLOG RESET
redis-cli CONFIG SET slowlog-log-slower-than 10000  # 10ms threshold

# Latency monitoring
redis-cli LATENCY HISTORY event
redis-cli LATENCY RESET

# Memory analysis
redis-cli MEMORY DOCTOR
redis-cli MEMORY USAGE key [SAMPLES N]
```

```
# redis.conf tuning
hz 20                          # Event loop frequency (default 10, increase for more precision)
dynamic-hz yes                 # Adaptive hz under load
lazyfree-lazy-eviction yes     # Async eviction (reduces latency spikes)
lazyfree-lazy-expire yes       # Async expiry
lazyfree-lazy-server-del yes   # Async DEL for large objects
replica-lazy-flush yes         # Async FLUSHDB on replica sync
no-appendfsync-on-rewrite yes  # Skip fsync during AOF rewrite (reduces latency)
```

## Benchmarking

```bash
# Basic benchmark
redis-benchmark -h localhost -p 6379 -n 100000 -c 50

# Benchmark specific commands
redis-benchmark -h localhost -p 6379 -n 100000 -c 50 -t set,get

# With pipelining (simulate batched ops)
redis-benchmark -h localhost -p 6379 -n 100000 -c 50 -P 16

# Benchmark with payload size
redis-benchmark -h localhost -p 6379 -n 100000 -d 1024 -t set,get
```

**Typical numbers on commodity hardware**:
- Single instance: 100k–1M ops/sec (command-dependent)
- GET/SET: ~100k ops/sec without pipelining
- GET/SET: ~1M ops/sec with pipelining (P=16)
- p99 latency: <1ms for GET/SET

---

# 8. Real-World Scenarios

## Scenario 1: E-Commerce Product Catalog & Cart

### Architecture

```
User Request
     │
     ▼
API Gateway
     │
     ├──► ProductService ──► Redis (product cache) ──► PostgreSQL
     │                                                     ↑ miss
     └──► CartService ──► Redis (cart data) ──────────────┘
```

### Product Caching

```java
@Service
@RequiredArgsConstructor
public class ProductCacheService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final ProductRepository productRepository;
    
    // Product details: 6 hour TTL (rarely changes)
    public ProductDto getProduct(Long id) {
        String key = "product:" + id;
        ProductDto cached = (ProductDto) redisTemplate.opsForValue().get(key);
        if (cached != null) return cached;
        
        ProductDto product = productRepository.findById(id)
            .map(this::toDto)
            .orElseThrow();
        
        redisTemplate.opsForValue().set(key, product, Duration.ofHours(6));
        return product;
    }
    
    // Inventory (stock): 30 second TTL (changes frequently)
    public int getStock(Long productId) {
        String key = "product:" + productId + ":stock";
        String cached = (String) redisTemplate.opsForValue().get(key);
        if (cached != null) return Integer.parseInt(cached);
        
        int stock = productRepository.getStock(productId);
        redisTemplate.opsForValue().set(key, String.valueOf(stock), Duration.ofSeconds(30));
        return stock;
    }
    
    // Price: 1 hour TTL
    // Flash sale prices: 5 minute TTL with explicit override
    public void setFlashSalePrice(Long productId, BigDecimal price, Duration duration) {
        redisTemplate.opsForValue().set("product:" + productId + ":price", price, duration);
    }
}

// Shopping cart — stored entirely in Redis
@Service
@RequiredArgsConstructor
public class CartService {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final Duration CART_TTL = Duration.ofDays(7);
    
    // Cart as a Hash: field = productId, value = quantity
    public void addItem(String sessionId, Long productId, int quantity) {
        String key = "cart:" + sessionId;
        redisTemplate.opsForHash().increment(key, productId.toString(), quantity);
        redisTemplate.expire(key, CART_TTL);
    }
    
    public void removeItem(String sessionId, Long productId) {
        redisTemplate.opsForHash().delete("cart:" + sessionId, productId.toString());
    }
    
    public Map<Object, Object> getCart(String sessionId) {
        return redisTemplate.opsForHash().entries("cart:" + sessionId);
    }
    
    // Merge guest cart into user cart on login
    public void mergeCarts(String guestSessionId, Long userId) {
        String guestKey = "cart:" + guestSessionId;
        String userKey = "cart:user:" + userId;
        
        Map<Object, Object> guestCart = redisTemplate.opsForHash().entries(guestKey);
        guestCart.forEach((productId, qty) -> {
            redisTemplate.opsForHash().increment(userKey, productId, (Long) qty);
        });
        
        redisTemplate.delete(guestKey);
        redisTemplate.expire(userKey, CART_TTL);
    }
}
```

### Redis Usage Summary
- Product details: Cache-aside, 6h TTL, JSON value
- Inventory counts: Cache-aside, 30s TTL, String value
- Shopping carts: Redis as primary store (Hash), 7-day TTL
- Leaderboard/popular products: Sorted Set by view count

---

## Scenario 2: Authentication & Session Management

### Architecture

```
Login ──► Auth Service ──► Validate credentials (PostgreSQL)
                 │
                 └──► Create session (Redis) ──► Return session token
                 
Request ──► API Gateway ──► Validate token (Redis) ──► Route to service
```

```java
@Service
@RequiredArgsConstructor
public class SessionService {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final String SESSION_PREFIX = "session:";
    private static final Duration SESSION_TTL = Duration.ofMinutes(30);
    private static final int MAX_SESSIONS_PER_USER = 5;
    
    public String createSession(Long userId, String email, Set<String> roles, 
                                 String deviceId) {
        String sessionId = UUID.randomUUID().toString();
        String sessionKey = SESSION_PREFIX + sessionId;
        String userSessionsKey = "user:" + userId + ":sessions";
        
        Map<String, Object> sessionData = Map.of(
            "userId", userId,
            "email", email,
            "roles", roles,
            "deviceId", deviceId,
            "createdAt", Instant.now().toEpochMilli(),
            "lastActive", Instant.now().toEpochMilli()
        );
        
        // Store session
        redisTemplate.opsForHash().putAll(sessionKey, sessionData);
        redisTemplate.expire(sessionKey, SESSION_TTL);
        
        // Track user's sessions (for "log out all devices")
        redisTemplate.opsForSet().add(userSessionsKey, sessionId);
        redisTemplate.expire(userSessionsKey, Duration.ofDays(7));
        
        // Enforce max sessions per user
        Long sessionCount = redisTemplate.opsForSet().size(userSessionsKey);
        if (sessionCount > MAX_SESSIONS_PER_USER) {
            evictOldestSession(userId);
        }
        
        return sessionId;
    }
    
    public Optional<SessionData> validateAndRefresh(String sessionId) {
        String key = SESSION_PREFIX + sessionId;
        Map<Object, Object> data = redisTemplate.opsForHash().entries(key);
        
        if (data.isEmpty()) return Optional.empty();
        
        // Sliding expiration: reset TTL on each valid request
        redisTemplate.opsForHash().put(key, "lastActive", Instant.now().toEpochMilli());
        redisTemplate.expire(key, SESSION_TTL);
        
        return Optional.of(SessionData.fromMap(data));
    }
    
    public void invalidateSession(String sessionId) {
        Map<Object, Object> data = redisTemplate.opsForHash().entries(SESSION_PREFIX + sessionId);
        if (!data.isEmpty()) {
            Long userId = (Long) data.get("userId");
            redisTemplate.opsForSet().remove("user:" + userId + ":sessions", sessionId);
        }
        redisTemplate.delete(SESSION_PREFIX + sessionId);
    }
    
    // "Log out all devices" feature
    public void invalidateAllUserSessions(Long userId) {
        String userSessionsKey = "user:" + userId + ":sessions";
        Set<Object> sessions = redisTemplate.opsForSet().members(userSessionsKey);
        
        if (sessions != null) {
            List<String> keys = sessions.stream()
                .map(s -> SESSION_PREFIX + s)
                .toList();
            keys.add(userSessionsKey);
            redisTemplate.delete(keys);
        }
    }
    
    private void evictOldestSession(Long userId) {
        // Find and remove oldest session
        // Implementation: store sessions in Sorted Set with creation timestamp as score
    }
}
```

---

## Scenario 3: High-Load API Rate Limiting

### Architecture: Multi-tier rate limiting

```
Request ──► Nginx (IP-level limit) ──► API Gateway ──► Rate Limiter Service (Redis)
                                                              │
                                       ┌─────────────────────┼─────────────────────┐
                                       ▼                     ▼                     ▼
                                  Global limit          Per-user limit        Per-endpoint limit
                                  (100k req/min)       (100 req/min)         (10 req/min)
```

```java
@Service
@RequiredArgsConstructor
public class MultiTierRateLimiter {

    private final RedissonClient redissonClient;
    
    public RateLimitDecision check(RateLimitContext context) {
        // Check all tiers, most restrictive wins
        List<RateLimitTier> tiers = List.of(
            new RateLimitTier("global", 100_000, Duration.ofMinutes(1)),
            new RateLimitTier("user:" + context.userId(), 100, Duration.ofMinutes(1)),
            new RateLimitTier("endpoint:" + context.endpoint() + ":user:" + context.userId(), 
                               10, Duration.ofMinutes(1))
        );
        
        for (RateLimitTier tier : tiers) {
            RRateLimiter limiter = redissonClient.getRateLimiter("ratelimit:" + tier.key());
            limiter.trySetRate(RateType.OVERALL, tier.limit(), 
                               tier.window().toSeconds(), RateIntervalUnit.SECONDS);
            
            if (!limiter.tryAcquire()) {
                return RateLimitDecision.rejected(tier.key(), tier.limit());
            }
        }
        
        return RateLimitDecision.allowed();
    }
    
    record RateLimitTier(String key, long limit, Duration window) {}
    record RateLimitContext(String userId, String endpoint, String ip) {}
    record RateLimitDecision(boolean allowed, String blockedBy, long limit) {
        static RateLimitDecision allowed() { return new RateLimitDecision(true, null, 0); }
        static RateLimitDecision rejected(String by, long limit) { 
            return new RateLimitDecision(false, by, limit); 
        }
    }
}
```

---

# 9. Comparisons

## Redis vs PostgreSQL

| Aspect | Redis | PostgreSQL |
|---|---|---|
| Speed | Sub-millisecond | 1–100ms |
| Data size | RAM-limited | Disk-limited |
| Queries | Key-based, simple | Full SQL |
| Joins | Not possible | Full support |
| Transactions | Limited (no rollback) | Full ACID |
| Durability | Configurable (can lose data) | Full durability |
| Schema | Schema-less | Strongly typed |
| Aggregations | Limited | Full GROUP BY, window functions |
| Best for | Speed, ephemeral data | Relational data, complex queries |

**When to use both**: This is the norm. PostgreSQL is your source of truth; Redis is your speed layer. Read from Redis, fall through to PostgreSQL on miss, write to PostgreSQL first, invalidate Redis.

## Redis vs Memcached

| Aspect | Redis | Memcached |
|---|---|---|
| Data structures | Rich (10+ types) | Strings only |
| Persistence | Yes (RDB/AOF) | No |
| Pub/Sub | Yes | No |
| Cluster | Native cluster | Client-side sharding |
| Replication | Built-in | No |
| Lua scripting | Yes | No |
| Memory efficiency | Slightly less | Slightly better for simple strings |
| Threading | Single (I/O multithreaded since 6.0) | Multi-threaded |

**When to choose Memcached**: Almost never for new projects. Redis is strictly more capable. Memcached only wins on raw string caching throughput when you have multiple CPU cores available — an increasingly niche advantage.

## When Redis Is a Bad Choice

1. **Your data is relational** — you need joins, foreign keys, or complex queries. Redis can't replace PostgreSQL.

2. **Data must never be lost** — if you can't afford even 1 second of data loss, Redis (even with AOF always) has failure scenarios. Use PostgreSQL with synchronous replication.

3. **Data exceeds available RAM** — if your dataset is 500GB and you have 64GB RAM, Redis won't work without aggressive eviction and redesign.

4. **Full-text search** — use Elasticsearch or PostgreSQL `pg_trgm`/`tsvector`. Redis has no text search capability.

5. **Complex analytics** — Redis has no GROUP BY, window functions, or SQL aggregations. Use PostgreSQL, ClickHouse, or BigQuery.

6. **Team has no Redis expertise** — Redis failure modes (OOM, eviction, replication lag) require understanding to handle. Don't add it unless the team can operate it.

7. **Your bottleneck isn't database reads** — adding Redis for a write-heavy workload or one that's bottlenecked on business logic doesn't help. Profile before adding infrastructure.

---

# 10. Production-Ready Checklist

**You are production-ready with Redis if you can:**

## Fundamentals
- [ ] Explain why Redis is fast (in-memory, single-threaded event loop, efficient data structures)
- [ ] Choose the right data structure for a given problem (not just String for everything)
- [ ] Explain when Redis is the wrong tool
- [ ] Describe the difference between RDB and AOF persistence and choose appropriately

## Architecture
- [ ] Design a cache-aside pattern with cache stampede prevention
- [ ] Explain replication lag and its implications for read-from-replica
- [ ] Set up Redis Sentinel for HA vs Redis Cluster for scale
- [ ] Choose between single node, Sentinel, and Cluster for a given use case

## Operations
- [ ] Set `maxmemory` and choose an appropriate eviction policy
- [ ] Configure TTLs with jitter to prevent expiry storms
- [ ] Use `SCAN` instead of `KEYS *` for key iteration
- [ ] Use `HSCAN`, `SSCAN`, `ZSCAN` instead of `HGETALL`, `SMEMBERS`, `ZRANGE 0 -1` on large collections
- [ ] Interpret `INFO memory` output and detect fragmentation
- [ ] Use `SLOWLOG` to identify problematic commands
- [ ] Monitor hit rate, eviction rate, and connection count

## Spring Boot Integration
- [ ] Configure `RedisTemplate` with proper serializers (not Java serialization)
- [ ] Use `@Cacheable`, `@CachePut`, `@CacheEvict` appropriately
- [ ] Avoid self-invocation with Spring Cache annotations
- [ ] Configure Lettuce connection pooling correctly
- [ ] Use `executePipelined` for bulk operations

## Distributed Systems
- [ ] Implement distributed lock with Redisson (with watchdog/auto-renewal)
- [ ] Explain why a naive lock (without unique owner value) is broken
- [ ] Implement sliding window rate limiting with Lua for atomicity
- [ ] Design session storage with sliding TTL and multi-device logout

## Advanced
- [ ] Write and execute a Lua script for atomic multi-step operations
- [ ] Use pipelines to batch N operations into one round trip
- [ ] Set up Redis Streams with consumer groups and handle pending message recovery
- [ ] Implement cache penetration prevention (null caching)
- [ ] Implement cache avalanche prevention (TTL jitter, circuit breaker)
- [ ] Detect and mitigate hot key issues (local cache, key splitting)

## Failure Scenarios — Know What Breaks and How
- [ ] **Cache stampede**: Many requests miss at once → all hit DB → DB overload. Mitigation: mutex lock, probabilistic early expiry.
- [ ] **Cache penetration**: Requests for non-existent keys → always miss → always hit DB. Mitigation: null caching, Bloom filter.
- [ ] **Cache avalanche**: Many keys expire simultaneously → wave of DB hits. Mitigation: TTL jitter.
- [ ] **OOM kill**: `maxmemory` not set → Redis uses all RAM → OOM killer kills process. Mitigation: always set `maxmemory`.
- [ ] **Hot key**: One key gets 100k req/s → single Redis thread saturated. Mitigation: local cache, key splitting.
- [ ] **Replication lag**: Replica is behind primary → stale reads. Mitigation: route to primary for consistency-sensitive reads.
- [ ] **Lock expiry before task completes**: Holder crashes or GC pauses → lock expires → two holders. Mitigation: Redisson watchdog, validate lock before committing.
- [ ] **AOF rewrite latency spike**: AOF rewrite forks process → copy-on-write → memory pressure → latency. Mitigation: `no-appendfsync-on-rewrite yes`, schedule rewrites off-peak.

---

*Last reviewed: 2024. Redis version 7.x.*
