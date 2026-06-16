# Redis Interview Questions & Study Guide

## Overview

Redis (Remote Dictionary Server) is an in-memory data structure store used as a cache, message broker, and database. It is one of the most commonly asked topics in backend interviews — especially for system design, caching strategies, and high-performance applications.

---

## Table of Contents

1. [What is Redis?](#what-is-redis)
2. [Data Structures](#data-structures)
3. [Caching Patterns](#caching-patterns)
4. [Expiry & Eviction](#expiry--eviction)
5. [Persistence](#persistence)
6. [Pub/Sub & Streams](#pubsub--streams)
7. [Transactions](#transactions)
8. [Lua Scripting](#lua-scripting)
9. [Redis Cluster & High Availability](#redis-cluster--high-availability)
10. [Redis vs Memcached](#redis-vs-memcached)
11. [Common Use Cases](#common-use-cases)
12. [Common Interview Questions](#common-interview-questions)
13. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What is Redis?

Redis is an **in-memory, key-value data store** that is:
- **Blazing fast**: sub-millisecond read/write (100k+ operations/sec)
- **Single-threaded**: all commands execute atomically in sequence (no race conditions on single operations)
- **Versatile**: supports 5+ complex data structures, not just strings
- **Persistent**: can write data to disk (RDB snapshots, AOF logs)
- **Distributed**: supports Replication, Sentinel, and Cluster modes

```
Application ──► Redis (in-memory) ──► Response in < 1ms
Application ──► Database (disk)   ──► Response in 10–100ms
```

---

## Data Structures

### 1. String

The most basic type. Stores text, numbers, or binary data (up to 512 MB).

```bash
SET key "value"
GET key
MSET k1 v1 k2 v2
MGET k1 k2

# Atomic increment/decrement (great for counters)
SET counter 0
INCR counter        # 1
INCRBY counter 5    # 6
DECR counter        # 5
DECRBY counter 2    # 3

# Set with expiry
SET session:user123 "data" EX 3600   # expires in 3600 seconds
SET lock:resource "1" NX EX 30       # NX = only set if NOT exists (distributed lock)

STRLEN key          # length of string
APPEND key " more"  # append to string
```

**Use cases**: Session storage, counters, rate limiting, feature flags, distributed locks.

---

### 2. List

Ordered collection of strings. Works as a doubly-linked list — O(1) push/pop from both ends.

```bash
# Push (L = left/head, R = right/tail)
LPUSH mylist "a"     # ["a"]
RPUSH mylist "b" "c" # ["a", "b", "c"]

# Pop
LPOP mylist          # "a" → ["b", "c"]
RPOP mylist          # "c" → ["b"]

# Blocking pop (waits for element — useful for job queues)
BLPOP queue 30       # block up to 30s waiting for an element

LLEN mylist          # length
LRANGE mylist 0 -1   # get all elements (0 to end)
LINDEX mylist 0      # get element at index
LSET mylist 0 "new"  # set element at index
```

**Use cases**: Message queues, activity feeds, recent items list, task queues.

---

### 3. Set

Unordered collection of **unique** strings.

```bash
SADD tags "java" "redis" "java"  # {"java", "redis"} — duplicates ignored
SMEMBERS tags                    # all members
SISMEMBER tags "java"            # 1 (true)
SCARD tags                       # cardinality (count)
SREM tags "redis"

# Set operations
SUNION set1 set2      # union
SINTER set1 set2      # intersection
SDIFF set1 set2       # difference (in set1 but not set2)

SPOP tags              # remove and return random member
SRANDMEMBER tags 2     # random 2 members (without removing)
```

**Use cases**: Tags, unique visitors, friend lists, permissions, social graph intersections.

---

### 4. Sorted Set (ZSet)

Like a Set, but each member has a **score** (float). Members are always sorted by score.

```bash
ZADD leaderboard 1000 "alice"
ZADD leaderboard 850  "bob"
ZADD leaderboard 1200 "charlie"

# Range by rank (0 = lowest score)
ZRANGE leaderboard 0 -1              # all, low to high
ZREVRANGE leaderboard 0 2            # top 3, high to low
ZRANGE leaderboard 0 -1 WITHSCORES  # include scores

# Range by score
ZRANGEBYSCORE leaderboard 800 1100   # members with score 800–1100

# Rank of a member
ZRANK leaderboard "alice"            # 1 (0-indexed from lowest)
ZREVRANK leaderboard "alice"         # position from highest

# Score operations
ZSCORE leaderboard "alice"           # 1000
ZINCRBY leaderboard 50 "alice"       # 1050

ZCARD leaderboard                    # count
ZREM leaderboard "bob"               # remove member
```

**Use cases**: Leaderboards, priority queues, rate limiting, time-series data with timestamps as scores.

---

### 5. Hash

Key-value pairs within a key — like a mini object/map.

```bash
HSET user:1 name "Alice" age 30 email "alice@example.com"
HGET user:1 name              # "Alice"
HMGET user:1 name email       # ["Alice", "alice@example.com"]
HGETALL user:1                # all fields and values
HKEYS user:1                  # ["name", "age", "email"]
HVALS user:1                  # ["Alice", "30", "alice@example.com"]

HDEL user:1 age
HEXISTS user:1 name           # 1 (true)
HLEN user:1                   # 2 (field count)
HINCRBY user:1 loginCount 1   # atomic increment a field
```

**Use cases**: User profiles, product data, shopping carts, session attributes.

---

### 6. Bitmap

Bit-level operations on a string. Extremely memory-efficient for boolean tracking.

```bash
# Track daily active users by user ID
SETBIT active_users:2024-01-15 1001 1   # user 1001 was active
SETBIT active_users:2024-01-15 1002 1   # user 1002 was active

GETBIT active_users:2024-01-15 1001     # 1
GETBIT active_users:2024-01-15 9999     # 0

BITCOUNT active_users:2024-01-15        # total active users (count of 1s)

# Bitwise operations across multiple days
BITOP AND result key1 key2              # active on BOTH days
BITOP OR result key1 key2              # active on EITHER day
```

**Use cases**: Daily active users, feature flag per user, login streaks, presence tracking.

---

### 7. HyperLogLog

Probabilistic data structure for **approximate cardinality counting** with very low memory (~12KB regardless of data size). Error rate ~0.81%.

```bash
PFADD unique_visitors "user:1" "user:2" "user:3"
PFCOUNT unique_visitors    # approximately 3

PFMERGE total visitors_day1 visitors_day2   # merge counts
```

**Use cases**: Count unique visitors, unique searches, approximate user counts at massive scale.

---

### Data Structures Summary

| Type | Use Case | Key Operations |
|---|---|---|
| String | Cache, counters, sessions | GET, SET, INCR |
| List | Queues, feeds, history | LPUSH, RPUSH, LPOP, BLPOP |
| Set | Tags, unique users, ACL | SADD, SISMEMBER, SINTER |
| Sorted Set | Leaderboards, ranking | ZADD, ZRANGE, ZREVRANK |
| Hash | Objects, profiles | HSET, HGET, HGETALL |
| Bitmap | Daily flags, tracking | SETBIT, BITCOUNT |
| HyperLogLog | Approx unique counts | PFADD, PFCOUNT |

---

## Caching Patterns

### Cache-Aside (Lazy Loading) — Most Common

Application checks cache first; loads from DB on miss and populates cache.

```
Application ──► Redis HIT  ──► return cached data
Application ──► Redis MISS ──► query DB ──► write to Redis ──► return data
```

```java
public String getUserProfile(String userId) {
    String cached = redis.get("user:" + userId);
    if (cached != null) return cached;  // cache hit

    String data = database.query(userId); // cache miss
    redis.setex("user:" + userId, 3600, data); // cache for 1 hour
    return data;
}
```

**Pros**: Cache only what's needed; resilient (app works if Redis is down)
**Cons**: Cache miss penalty; potential for stale data

---

### Write-Through

Write to cache and DB simultaneously on every write.

```
Application ──► Write to Redis + Write to DB ──► always in sync
```

**Pros**: Cache always up to date
**Cons**: Write latency; cache may hold data never read

---

### Write-Behind (Write-Back)

Write to cache immediately; DB write is async.

```
Application ──► Write to Redis ──► async write to DB (background)
```

**Pros**: Very fast writes
**Cons**: Risk of data loss if Redis crashes before DB write

---

### Read-Through

Cache sits in front of DB; cache handles DB loading transparently.

```
Application ──► Cache ──► (on miss) DB ──► populated cache ──► return
```

**Pros**: Application code is simpler
**Cons**: First request always slow (cold cache)

---

### Refresh-Ahead

Proactively refresh cache before it expires for hot keys.

```
Background job: if TTL < threshold → reload from DB before expiry
```

**Pros**: No cache miss latency for hot data
**Cons**: May load data not needed after refresh

---

## Expiry & Eviction

### Setting Expiry

```bash
SET key "value" EX 60        # expire in 60 seconds
SET key "value" PX 60000     # expire in 60000 milliseconds
EXPIRE key 60                # set expiry on existing key
EXPIREAT key 1735689600      # expire at Unix timestamp
TTL key                      # seconds remaining (-1 = no expiry, -2 = doesn't exist)
PTTL key                     # milliseconds remaining
PERSIST key                  # remove expiry
```

### Eviction Policies

When Redis reaches `maxmemory`, it uses the configured eviction policy:

| Policy | Description |
|---|---|
| `noeviction` | Return error when memory full (default) |
| `allkeys-lru` | Evict least recently used key from all keys |
| `allkeys-lfu` | Evict least frequently used key from all keys |
| `allkeys-random` | Evict random key from all keys |
| `volatile-lru` | Evict LRU from keys with TTL set |
| `volatile-lfu` | Evict LFU from keys with TTL set |
| `volatile-random` | Evict random key with TTL set |
| `volatile-ttl` | Evict key with shortest TTL first |

```bash
# Set in redis.conf or at runtime
CONFIG SET maxmemory 256mb
CONFIG SET maxmemory-policy allkeys-lru
```

> **Interview Tip**: For a pure cache (all data is reproducible from DB), use `allkeys-lru`. For mixed cache+persistent data, use `volatile-lru` to only evict keys with TTL.

---

## Persistence

### RDB (Redis Database) — Snapshots

Saves a point-in-time snapshot of data to disk at configurable intervals.

```bash
# redis.conf
save 900 1     # save if at least 1 key changed in 900 seconds
save 300 10    # save if at least 10 keys changed in 300 seconds
save 60 10000  # save if at least 10000 keys changed in 60 seconds

# Manual snapshot
BGSAVE    # async snapshot in background
SAVE      # sync snapshot (blocks Redis)
LASTSAVE  # timestamp of last successful save
```

**Pros**: Compact file, fast restart
**Cons**: Data loss between snapshots (up to minutes of data)

---

### AOF (Append-Only File) — Write Log

Logs every write operation. Replays the log on restart to rebuild data.

```bash
# redis.conf
appendonly yes
appendfsync everysec   # flush to disk every second (recommended)
# appendfsync always   # flush every write (safest, slowest)
# appendfsync no       # let OS decide (fastest, least safe)

# AOF rewrite (compact the log)
BGREWRITEAOF
```

**Pros**: Minimal data loss (up to 1 second with `everysec`)
**Cons**: Larger file, slower restart

---

### RDB + AOF (Recommended for production)

Use both: RDB for fast restart, AOF for minimal data loss.

| | RDB | AOF |
|---|---|---|
| Data loss on crash | Up to minutes | Up to 1 second |
| Restart speed | Fast (load snapshot) | Slow (replay all writes) |
| File size | Compact | Large (can be compacted) |
| Best for | Backups, fast restart | Data durability |

---

## Pub/Sub & Streams

### Pub/Sub

Simple publish-subscribe messaging. Messages are fire-and-forget (not persisted).

```bash
# Subscriber (in one connection)
SUBSCRIBE channel:notifications
PSUBSCRIBE order:*             # pattern subscribe

# Publisher (in another connection)
PUBLISH channel:notifications '{"type":"alert","msg":"Server down"}'
PUBLISH order:created '{"orderId":"123"}'
```

**Limitations**: Messages lost if no subscriber is listening; no message history; no consumer groups.

---

### Redis Streams (Redis 5.0+)

A persistent, append-only log with consumer groups — similar to Kafka but simpler.

```bash
# Produce messages
XADD orders "*" orderId 123 status "placed" amount 99.99
# * = auto-generate timestamp-based ID

# Read messages
XREAD COUNT 10 STREAMS orders 0    # from beginning
XREAD COUNT 10 STREAMS orders $    # only new messages

# Consumer groups (multiple consumers, each gets different messages)
XGROUP CREATE orders mygroup $ MKSTREAM
XREADGROUP GROUP mygroup consumer1 COUNT 5 STREAMS orders >
XACK orders mygroup <message-id>   # acknowledge processed message

# Stream info
XLEN orders                        # number of messages
XRANGE orders - +                  # all messages
```

---

## Transactions

Redis transactions with `MULTI`/`EXEC` are atomic — all commands run or none do (no partial execution). But there is no rollback on runtime errors.

```bash
MULTI               # start transaction
SET balance 100
INCR counter
SET status "active"
EXEC                # execute all atomically

# Discard transaction
DISCARD
```

### Optimistic Locking with WATCH

```bash
WATCH balance           # watch key for changes
MULTI
DECRBY balance 50       # conditional on balance not being modified
EXEC                    # returns nil if balance was changed (transaction cancelled)
```

> **Interview Tip**: Redis transactions do NOT support rollback on errors (unlike SQL). If one command fails at runtime, the others still execute. Use Lua scripts for true atomicity with conditional logic.

---

## Lua Scripting

Lua scripts run atomically on the Redis server — no other command executes during the script. Used for complex atomic operations.

```bash
# Run inline Lua script
EVAL "return redis.call('GET', KEYS[1])" 1 mykey

# Example: atomic check-and-set
EVAL "
  local val = redis.call('GET', KEYS[1])
  if val == ARGV[1] then
    return redis.call('SET', KEYS[1], ARGV[2])
  end
  return 0
" 1 mykey expectedValue newValue

# Load script for reuse (returns SHA1 hash)
SCRIPT LOAD "return redis.call('INCR', KEYS[1])"
EVALSHA <sha1> 1 counter
```

**Use cases**: Distributed locks, atomic rate limiters, conditional updates, complex transactions.

---

## Redis Cluster & High Availability

### Replication (Master–Replica)

```
[Master] ──── async replication ──► [Replica 1]
         └─────────────────────────► [Replica 2]
```

- Master handles all writes; replicas handle reads (read scaling)
- If master fails, manual failover required (or use Sentinel)

```bash
# redis.conf on replica
replicaof 192.168.1.100 6379
```

---

### Redis Sentinel

Provides **automatic failover** and **monitoring** for single-instance Redis.

```
[Sentinel 1]    [Sentinel 2]    [Sentinel 3]
     └──────────────┴─────────────────┘
              monitor + vote
                    │
              ┌─────┴─────┐
           [Master]    [Replica]
          (if master fails, Sentinel promotes replica)
```

- At least 3 Sentinel instances for quorum
- Clients connect to Sentinel to discover the master

---

### Redis Cluster

Horizontal sharding — data is automatically partitioned across multiple master nodes.

```
          [Master A]          [Master B]          [Master C]
         slots 0–5460      slots 5461–10922    slots 10923–16383
            │                   │                    │
         [Replica A]         [Replica B]          [Replica C]
```

- **16384 hash slots** total — data distributed by `CRC16(key) % 16384`
- Minimum 3 masters + 3 replicas (6 nodes) for production
- Each master has at least one replica for failover

```bash
redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1
```

| Mode | Use Case | Auto-Failover | Sharding |
|---|---|---|---|
| Standalone | Dev, small apps | No | No |
| Replication | Read scaling | No (manual) | No |
| Sentinel | HA single shard | Yes | No |
| Cluster | HA + scale-out | Yes | Yes |

---

## Redis vs Memcached

| Feature | Redis | Memcached |
|---|---|---|
| Data structures | 7+ (String, List, Set, ZSet, Hash, Bitmap, HLL) | String only |
| Persistence | Yes (RDB + AOF) | No |
| Replication | Yes | No |
| Pub/Sub | Yes | No |
| Scripting | Yes (Lua) | No |
| Clustering | Yes (native) | Client-side only |
| Multi-threading | Single-threaded (commands) | Multi-threaded |
| Memory efficiency | Slightly less | Slightly more |

> **When to choose Memcached**: Pure simple caching of strings, need multi-threading for high concurrency, simplicity is priority.
> **When to choose Redis**: Any real-world use case — persistence, complex data types, pub/sub, clustering.

---

## Common Use Cases

### 1. Distributed Lock (Redlock)

```bash
# Acquire lock: SET key value NX EX timeout
SET lock:payment:order123 "server1" NX EX 30
# NX = only set if key does NOT exist
# Returns OK = lock acquired, nil = lock held by someone else

# Release lock (Lua for atomicity)
EVAL "
  if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
  else
    return 0
  end
" 1 lock:payment:order123 "server1"
```

---

### 2. Rate Limiting

```bash
# Sliding window rate limiter
# Allow 100 requests per minute per user

INCR rate_limit:user:123
EXPIRE rate_limit:user:123 60  # reset after 1 minute

# Or with atomic MULTI
MULTI
INCR rate_limit:user:123
EXPIRE rate_limit:user:123 60
EXEC
# Check if count > 100 → reject request
```

---

### 3. Session Storage

```bash
SET session:abc123 '{"userId":1,"role":"admin"}' EX 3600
GET session:abc123
DEL session:abc123   # logout
```

---

### 4. Leaderboard

```bash
ZADD game:leaderboard 9500 "player1"
ZADD game:leaderboard 8200 "player2"
ZADD game:leaderboard 11000 "player3"

# Top 10 players
ZREVRANGE game:leaderboard 0 9 WITHSCORES

# Player rank (1-indexed)
ZREVRANK game:leaderboard "player1"  # 1 (second from top, 0-indexed)
```

---

### 5. Job Queue

```bash
# Producer
LPUSH job:queue '{"id":1,"type":"email","to":"user@example.com"}'

# Consumer (blocking pop — waits for jobs)
BRPOP job:queue 0    # block indefinitely
BRPOP job:queue 30   # block up to 30 seconds
```

---

### 6. Caching Database Queries

```java
// Spring Boot with Spring Cache + Redis
@Cacheable(value = "users", key = "#id", unless = "#result == null")
public User getUserById(Long id) {
    return userRepository.findById(id).orElse(null);
}

@CacheEvict(value = "users", key = "#user.id")
public User updateUser(User user) {
    return userRepository.save(user);
}
```

---

## Common Interview Questions

### Q: How does Redis achieve high performance?

1. **In-memory storage** — data lives in RAM, no disk I/O for reads/writes
2. **Single-threaded command processing** — no lock contention, no context switching overhead
3. **Efficient data structures** — each type is optimized (e.g., ziplist for small hashes)
4. **I/O multiplexing** — single thread handles thousands of connections via epoll/kqueue
5. **Pipelining** — batch multiple commands in one network round trip

---

### Q: What is the difference between RDB and AOF persistence?

- **RDB**: Point-in-time snapshots at intervals. Fast startup, compact file, but potential for data loss between snapshots (minutes).
- **AOF**: Logs every write operation. Near-zero data loss (1 second with `everysec`), but larger files and slower restart.
- **Best practice**: Use both — RDB for backups and fast recovery, AOF for minimal data loss.

---

### Q: How do you implement a distributed lock in Redis?

```bash
SET lock:resource uuid NX EX 30
# NX = only succeeds if key doesn't exist (prevents double-locking)
# EX = auto-expire prevents deadlock if holder crashes
```

Release with a Lua script to atomically check-and-delete (prevents releasing another process's lock).

---

### Q: What is the difference between EXPIRE and TTL?

- `EXPIRE key 60` — sets the key to expire in 60 seconds
- `TTL key` — returns remaining time to live in seconds (-1 = no expiry, -2 = key doesn't exist)
- `PERSIST key` — removes the expiry (makes key permanent)

---

### Q: How would you cache database results with automatic invalidation?

```
Strategy: Cache-Aside with TTL
1. On read: check Redis → hit: return cache, miss: query DB → cache result with TTL
2. On write: update DB → delete or update cache key
3. TTL as safety net: stale cache auto-expires even if explicit invalidation is missed
```

---

### Q: What are the limitations of Redis Pub/Sub?

1. **No message persistence** — messages lost if no subscriber is connected
2. **No consumer groups** — all subscribers get all messages (broadcast)
3. **No acknowledgement** — fire-and-forget, no delivery guarantee
4. **Use Redis Streams instead** for durable, consumer-group messaging (like Kafka).

---

### Q: What is a Redis pipeline?

Pipelining sends multiple commands to Redis in one network round trip without waiting for each response individually. Dramatically improves throughput for bulk operations.

```java
// Without pipelining: 100 round trips
for (int i = 0; i < 100; i++) redis.set("key:" + i, "val");

// With pipelining: 1 round trip
Pipeline pipe = jedis.pipelined();
for (int i = 0; i < 100; i++) pipe.set("key:" + i, "val");
pipe.sync();
```

---

## Quick Reference Cheat Sheet

```
String:  GET, SET, MGET, MSET, INCR, INCRBY, SETEX, NX
List:    LPUSH, RPUSH, LPOP, RPOP, BLPOP, LRANGE, LLEN
Set:     SADD, SMEMBERS, SISMEMBER, SCARD, SINTER, SUNION, SDIFF
ZSet:    ZADD, ZRANGE, ZREVRANGE, ZRANK, ZSCORE, ZINCRBY
Hash:    HSET, HGET, HMGET, HGETALL, HDEL, HINCRBY
Bitmap:  SETBIT, GETBIT, BITCOUNT
HLL:     PFADD, PFCOUNT

Expiry:  EXPIRE, TTL, PTTL, PERSIST
Tx:      MULTI, EXEC, DISCARD, WATCH
Script:  EVAL, EVALSHA, SCRIPT LOAD

Persistence: RDB (snapshot) vs AOF (append log)
Eviction:    noeviction | allkeys-lru | allkeys-lfu | volatile-lru
HA:          Replication → Sentinel → Cluster
Cluster:     16384 hash slots, sharded across master nodes
Lock:        SET key uuid NX EX ttl

Redis vs Memcached: Redis wins on data structures, persistence, pub/sub, cluster
```

---

*Last Updated: 2026-06-04*
