# Redis Interview Questions & Study Guide

## Overview

Redis (Remote Dictionary Server) is an in-memory key-value store used as a cache, message broker, and database. It is a common interview topic for backend roles, especially around caching strategies and system design.

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
- **Fast**: sub-millisecond reads/writes (100k+ ops/sec)
- **Single-threaded**: commands execute atomically, no race conditions
- **Versatile**: supports 5+ complex data structures
- **Persistent**: can write to disk (RDB snapshots, AOF logs)
- **Distributed**: supports Replication, Sentinel, and Cluster modes

---

## Data Structures

### 1. String

Basic type. Stores text, numbers, or binary data (up to 512 MB).

```bash
SET key "value"
GET key
INCR counter        # atomic increment
INCRBY counter 5
SET session:user123 "data" EX 3600   # expires in 3600 seconds
SET lock:resource "1" NX EX 30       # NX = only set if NOT exists
```

**Use cases**: Session storage, counters, rate limiting, distributed locks.

---

### 2. List

Ordered collection of strings. O(1) push/pop from both ends.

```bash
LPUSH mylist "a"
RPUSH mylist "b" "c"
LPOP mylist
BLPOP queue 30       # blocking pop — waits up to 30s (great for job queues)
LRANGE mylist 0 -1   # get all elements
```

**Use cases**: Message queues, activity feeds, task queues.

---

### 3. Set

Unordered collection of **unique** strings.

```bash
SADD tags "java" "redis" "java"  # {"java", "redis"} — duplicates ignored
SISMEMBER tags "java"            # 1 (true)
SCARD tags                       # count
SINTER set1 set2                 # intersection
SUNION set1 set2                 # union
```

**Use cases**: Tags, unique visitors, friend lists.

---

### 4. Sorted Set (ZSet)

Like a Set, but each member has a **score**. Members are always sorted by score.

```bash
ZADD leaderboard 1000 "alice"
ZADD leaderboard 1200 "charlie"
ZREVRANGE leaderboard 0 2 WITHSCORES  # top 3, high to low
ZRANK leaderboard "alice"             # rank from lowest
ZINCRBY leaderboard 50 "alice"        # update score
```

**Use cases**: Leaderboards, priority queues, rate limiting.

---

### 5. Hash

Key-value pairs within a key — like a mini object/map.

```bash
HSET user:1 name "Alice" age 30
HGET user:1 name
HGETALL user:1
HINCRBY user:1 loginCount 1   # atomic field increment
```

**Use cases**: User profiles, product data, shopping carts.

---

### 6. Bitmap

Bit-level operations on a string. Very memory-efficient for boolean tracking.

```bash
SETBIT active_users:2024-01-15 1001 1   # user 1001 was active
GETBIT active_users:2024-01-15 1001     # 1
BITCOUNT active_users:2024-01-15        # count of active users
BITOP AND result key1 key2              # active on BOTH days
```

**Use cases**: Daily active users, login streaks, feature flags per user.

---

### 7. HyperLogLog

Probabilistic structure for **approximate cardinality counting** (~12KB memory, ~0.81% error rate).

```bash
PFADD unique_visitors "user:1" "user:2"
PFCOUNT unique_visitors    # approximately 2
PFMERGE total day1 day2    # merge counts
```

**Use cases**: Count unique visitors at massive scale.

---

### Data Structures Summary

| Type | Use Case | Key Operations |
|---|---|---|
| String | Cache, counters, sessions | GET, SET, INCR |
| List | Queues, feeds | LPUSH, RPUSH, LPOP, BLPOP |
| Set | Tags, unique users | SADD, SISMEMBER, SINTER |
| Sorted Set | Leaderboards, ranking | ZADD, ZRANGE, ZREVRANK |
| Hash | Objects, profiles | HSET, HGET, HGETALL |
| Bitmap | Daily flags, tracking | SETBIT, BITCOUNT |
| HyperLogLog | Approx unique counts | PFADD, PFCOUNT |

---

## Caching Patterns

### Cache-Aside (Lazy Loading) — Most Common

Application checks cache first; loads from DB on miss and populates cache.

```java
public String getUserProfile(String userId) {
    String cached = redis.get("user:" + userId);
    if (cached != null) return cached;  // cache hit

    String data = database.query(userId);
    redis.setex("user:" + userId, 3600, data);
    return data;
}
```

**Pros**: Cache only what's needed; app works if Redis is down.  
**Cons**: Cache miss penalty; potential stale data.

---

### Write-Through

Write to cache and DB simultaneously on every write.

**Pros**: Cache always up to date. **Cons**: Write latency; cache may hold unread data.

---

### Write-Behind (Write-Back)

Write to cache immediately; DB write is async.

**Pros**: Very fast writes. **Cons**: Risk of data loss if Redis crashes before DB write.

---

### Read-Through

Cache sits in front of DB and handles DB loading transparently.

**Pros**: Simpler app code. **Cons**: First request always slow (cold cache).

---

### Refresh-Ahead

Proactively refresh cache before it expires for hot keys.

**Pros**: No miss latency for hot data. **Cons**: May load data not needed after refresh.

---

## Expiry & Eviction

### Setting Expiry

```bash
SET key "value" EX 60        # expire in 60 seconds
EXPIRE key 60                # set expiry on existing key
TTL key                      # seconds remaining (-1 = no expiry, -2 = gone)
PERSIST key                  # remove expiry
```

### Eviction Policies

When Redis hits `maxmemory`, the configured policy decides what to evict:

| Policy | Description |
|---|---|
| `noeviction` | Return error when memory full (default) |
| `allkeys-lru` | Evict least recently used from all keys |
| `allkeys-lfu` | Evict least frequently used from all keys |
| `volatile-lru` | Evict LRU from keys with TTL set |
| `volatile-ttl` | Evict key with shortest TTL first |

```bash
CONFIG SET maxmemory 256mb
CONFIG SET maxmemory-policy allkeys-lru
```

> **Interview Tip**: For a pure cache, use `allkeys-lru`. For mixed cache+persistent data, use `volatile-lru` to only evict keys with TTL.

---

## Persistence

### RDB (Snapshots)

Saves a point-in-time snapshot at configurable intervals.

```bash
save 900 1     # if at least 1 key changed in 900s
BGSAVE         # async snapshot
```

**Pros**: Compact file, fast restart. **Cons**: Data loss up to minutes between snapshots.

---

### AOF (Append-Only File)

Logs every write operation; replays on restart.

```bash
appendonly yes
appendfsync everysec   # flush every second (recommended)
```

**Pros**: Near-zero data loss (1 second). **Cons**: Larger file, slower restart.

---

### RDB vs AOF

| | RDB | AOF |
|---|---|---|
| Data loss on crash | Up to minutes | Up to 1 second |
| Restart speed | Fast | Slow (replay writes) |
| File size | Compact | Large |

> **Best practice**: Use both — RDB for fast restart, AOF for durability.

---

## Pub/Sub & Streams

### Pub/Sub

Fire-and-forget messaging. Messages are **not persisted**.

```bash
SUBSCRIBE channel:notifications      # subscriber
PUBLISH channel:notifications '{"msg":"alert"}'  # publisher
```

**Limitations**: Messages lost if no subscriber connected; no consumer groups; no delivery guarantee.

---

### Redis Streams (Redis 5.0+)

Persistent append-only log with consumer groups — like a lightweight Kafka.

```bash
XADD orders "*" orderId 123 status "placed"   # produce
XREAD COUNT 10 STREAMS orders 0               # read from beginning
XGROUP CREATE orders mygroup $ MKSTREAM
XREADGROUP GROUP mygroup consumer1 COUNT 5 STREAMS orders >
XACK orders mygroup <message-id>              # acknowledge
```

Use Streams instead of Pub/Sub when you need durability or consumer groups.

---

## Transactions

`MULTI`/`EXEC` are atomic — all commands run or none do (no partial execution). No rollback on runtime errors.

```bash
MULTI
SET balance 100
INCR counter
EXEC    # execute all atomically

# Optimistic locking
WATCH balance           # if balance changes before EXEC, transaction is cancelled
MULTI
DECRBY balance 50
EXEC                    # returns nil if balance was modified
```

> **Interview Tip**: Redis transactions do NOT rollback on runtime errors. Use Lua scripts for true conditional atomicity.

---

## Lua Scripting

Lua scripts run atomically — no other command executes during the script.

```bash
EVAL "return redis.call('GET', KEYS[1])" 1 mykey

# Atomic check-and-set
EVAL "
  local val = redis.call('GET', KEYS[1])
  if val == ARGV[1] then
    return redis.call('SET', KEYS[1], ARGV[2])
  end
  return 0
" 1 mykey expectedValue newValue
```

**Use cases**: Distributed locks, rate limiters, conditional updates.

---

## Redis Cluster & High Availability

### Replication (Master–Replica)

Master handles writes; replicas handle reads. Manual failover if master fails.

```bash
replicaof 192.168.1.100 6379   # in replica's redis.conf
```

### Redis Sentinel

Provides **automatic failover** for a single-instance Redis. Requires at least 3 Sentinel instances for quorum.

### Redis Cluster

Horizontal sharding across multiple master nodes.

- **16384 hash slots** — distributed by `CRC16(key) % 16384`
- Minimum 3 masters + 3 replicas for production

| Mode | Auto-Failover | Sharding |
|---|---|---|
| Standalone | No | No |
| Replication | No (manual) | No |
| Sentinel | Yes | No |
| Cluster | Yes | Yes |

---

## Redis vs Memcached

| Feature | Redis | Memcached |
|---|---|---|
| Data structures | 7+ types | String only |
| Persistence | Yes (RDB + AOF) | No |
| Replication | Yes | No |
| Pub/Sub | Yes | No |
| Clustering | Yes (native) | Client-side only |

> Choose Memcached only for pure string caching where simplicity is paramount. Choose Redis for any real-world use case.

---

## Common Use Cases

### 1. Distributed Lock

```bash
SET lock:payment:order123 "server1" NX EX 30
# Release with Lua (atomic check-and-delete)
EVAL "
  if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
  else return 0 end
" 1 lock:payment:order123 "server1"
```

### 2. Rate Limiting

```bash
MULTI
INCR rate_limit:user:123
EXPIRE rate_limit:user:123 60
EXEC
# If count > 100 → reject request
```

### 3. Session Storage

```bash
SET session:abc123 '{"userId":1,"role":"admin"}' EX 3600
DEL session:abc123   # logout
```

### 4. Leaderboard

```bash
ZADD game:leaderboard 9500 "player1"
ZREVRANGE game:leaderboard 0 9 WITHSCORES   # top 10
ZREVRANK game:leaderboard "player1"          # player's rank
```

### 5. Job Queue

```bash
LPUSH job:queue '{"id":1,"type":"email"}'    # producer
BRPOP job:queue 30                            # consumer, block up to 30s
```

### 6. Spring Boot Caching

```java
@Cacheable(value = "users", key = "#id")
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

In-memory storage eliminates disk I/O. Single-threaded command processing avoids lock contention. I/O multiplexing (epoll) handles thousands of connections on one thread. Pipelining batches multiple commands in one network round trip.

---

### Q: What is the difference between RDB and AOF?

RDB saves snapshots at intervals — fast startup but up to minutes of data loss. AOF logs every write — near-zero data loss (1 second) but slower restart. Best practice is to use both.

---

### Q: How do you implement a distributed lock?

```bash
SET lock:resource uuid NX EX 30
```
`NX` prevents double-locking; `EX` auto-expires to prevent deadlock if the holder crashes. Release using a Lua script that atomically checks the UUID before deleting.

---

### Q: What is the difference between EXPIRE and TTL?

`EXPIRE key 60` sets the key to expire in 60 seconds. `TTL key` returns remaining time (-1 = no expiry, -2 = key gone). `PERSIST key` removes the expiry.

---

### Q: How do you cache DB results with automatic invalidation?

Use Cache-Aside with TTL: on read, check Redis first; on miss, query DB and cache with TTL. On write, update DB and delete/update the cache key. TTL is the safety net for missed invalidations.

---

### Q: What are the limitations of Redis Pub/Sub?

Messages are lost if no subscriber is connected. All subscribers get every message (broadcast, no consumer groups). No delivery acknowledgement. Use Redis Streams for durable, consumer-group messaging.

---

### Q: What is a Redis pipeline?

Pipelining sends multiple commands in one network round trip without waiting for individual responses, dramatically improving throughput for bulk operations.

```java
Pipeline pipe = jedis.pipelined();
for (int i = 0; i < 100; i++) pipe.set("key:" + i, "val");
pipe.sync();  // 1 round trip instead of 100
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

Persistence: RDB (snapshot) vs AOF (append log) — use both in production
Eviction:    noeviction | allkeys-lru | allkeys-lfu | volatile-lru
HA:          Replication → Sentinel → Cluster
Cluster:     16384 hash slots, sharded across master nodes
Lock:        SET key uuid NX EX ttl

Redis vs Memcached: Redis wins on data structures, persistence, pub/sub, cluster
```

---

*Last Updated: 2026-06-18*
