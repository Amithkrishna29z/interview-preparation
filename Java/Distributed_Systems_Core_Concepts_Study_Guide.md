# Distributed Systems Core Concepts — Study Guide

> Study notes covering the 7 most critical distributed systems topics for backend and system design interviews.

---

## 1. Load Balancing

### What Is It?

A load balancer is a traffic cop that sits in front of your servers and distributes incoming requests so no single server gets overwhelmed.

**Analogy:** A supermarket manager directing customers to the shortest checkout lane.

```
Client → [Load Balancer] → [Server 1]
                         → [Server 2]
                         → [Server 3]
```

### Why It's Needed

| Problem | How Load Balancing Solves It |
|---------|------------------------------|
| Single server overloaded | Distributes requests across multiple servers |
| Server crashes | Traffic rerouted to healthy servers |
| Traffic spikes | Scale horizontally — add more servers |
| Geographically distributed users | Route to nearest server |

### Algorithms

- **Round Robin** — each server in turn. Best for identical servers.
- **Weighted Round Robin** — servers with more capacity get more traffic.
- **Least Connections** — routes to server with fewest active connections. Best for long-lived connections (WebSockets).
- **IP Hash (Sticky Sessions)** — hashes client IP to always route to the same server. If that server dies, sessions are lost. Fix: shared session store (Redis).
- **Least Response Time** — routes to server with lowest average response time.

### Layer 4 vs Layer 7

| | Layer 4 (Transport) | Layer 7 (Application) |
|-|--------------------|-----------------------|
| **Operates at** | TCP/UDP | HTTP/HTTPS |
| **Routing based on** | IP + port | URL path, headers, cookies |
| **Performance** | Faster | Slightly slower (deep inspection) |
| **Examples** | AWS NLB, HAProxy (TCP) | AWS ALB, Nginx, HAProxy (HTTP) |

### Health Checks

Load balancers probe backends and remove unhealthy ones from rotation:
`GET /health → 200 OK ✓ keep | 503 ✗ remove`

### Sticky Sessions

**Problem:** Server 1 stores a user's cart in memory; the next request hits Server 2 — cart is gone.

**Solution:** Use a shared session store (Redis) — all session data is external, any server handles any request (stateless servers = true horizontal scaling).

### Common Interview Questions — Load Balancing

**Q: What is a single point of failure (SPOF) and how does a load balancer help?**
A SPOF is a component whose failure brings down the whole system. A load balancer distributes traffic so one server failing doesn't take down the app. The load balancer itself can be a SPOF — solved by running redundant load balancers (active-passive or active-active HA pairs).

**Q: Difference between a load balancer and a reverse proxy?**
A reverse proxy forwards client requests to backend servers. A load balancer is a reverse proxy that also distributes requests across multiple servers. All load balancers are reverse proxies; not all reverse proxies are load balancers.

**Q: How does a load balancer handle a server that crashes mid-request?**
The client gets a connection error/timeout. Health checks mark the server unhealthy so future requests skip it. Mitigate with timeouts, circuit breakers, and retry logic.

---

## 2. CAP Theorem

### What Is It?

A distributed system can guarantee **at most 2 of these 3 properties** simultaneously:

| Property | Meaning |
|----------|---------|
| **C — Consistency** | Every read gets the most recent write (or an error). |
| **A — Availability** | Every request gets a response (not necessarily latest data). |
| **P — Partition Tolerance** | System keeps working even if network messages between nodes are lost. |

**Analogy (ATMs):** Consistency = every ATM shows your exact balance. Availability = every ATM always responds. Partition Tolerance = ATMs work even if the branch network goes down.

### The Real Choice: CP vs AP

In any real distributed system, **network partitions will happen** — so **P is not optional**. The choice is CP vs AP:

```
CP — Consistency + Partition Tolerance
  → During a partition, return an error instead of stale data
  → Examples: HBase, MongoDB (strong mode), Zookeeper, etcd

AP — Availability + Partition Tolerance
  → During a partition, return possibly stale data
  → Examples: Cassandra, CouchDB, DynamoDB, DNS
```

### When to Choose Which

```
Choose CP: financial transactions, inventory (can't oversell), auth/token validation
Choose AP: social feeds, DNS, product catalogs, metrics/analytics
```

### PACELC (Awareness)

Extends CAP: if there's a Partition → A vs C; Else (normal) → Latency vs Consistency. Even without partitions, strong consistency costs latency because nodes must coordinate.

### Common Interview Questions — CAP Theorem

**Q: Can a system be CA in a distributed setup?**
Technically no — partitions are inevitable so you must tolerate them. A "CA" system is effectively single-node.

**Q: How does CAP relate to ACID and BASE?**
ACID (traditional RDBMS guarantees) maps to CP. BASE (Basically Available, Soft state, Eventually consistent — the NoSQL approach) maps to AP.

**Q: Is CAP a limitation or a design choice?**
Both. It's a proven limitation, but how a system handles the trade-off is deliberate. Cassandra chooses AP; Zookeeper chooses CP — neither is wrong, it depends on the use case.

---

## 3. Eventual Consistency

### What Is It?

If no new updates are made, eventually all nodes will return the same value. There's no guarantee of *when*, but it will happen.

**Analogy:** DNS propagation — change a domain's IP and for a while some users hit the old server, others the new one; eventually all DNS servers converge.

### Strong vs Eventual Consistency

```
Strong Consistency:
  Write → all nodes updated → every read returns the new value
  Slower writes (must confirm all nodes)

Eventual Consistency:
  Write → primary updated → replicas updated asynchronously
  [Write X=5] → Node A: X=5 | Node B: X=3 (stale, replication lag)
  ...time passes... → all nodes converge to X=5
  Faster writes; reads may be stale briefly.
```

### Consistency Models (Awareness)

A spectrum from strongest to weakest:
- **Linearizable** — behaves like a single node (etcd, Zookeeper).
- **Read-your-own-writes** — you always see your own updates immediately (most web apps need this).
- **Monotonic reads** — once you've seen a value, you won't later see an older one.
- **Eventual** — weakest; converges with no time guarantee (Cassandra, DynamoDB default).

### Conflict Resolution (Awareness)

When two nodes accept writes for the same data during a partition:
- **Last Write Wins (LWW)** — highest timestamp wins. Simple, but vulnerable to clock skew.
- **Vector Clocks** — track causality to detect concurrent conflicting writes. Used by DynamoDB, Riak.
- **CRDTs** — data structures that always converge without conflicts (e.g., grow-only counter). Used in collaborative editing, distributed counters.

### Common Interview Questions — Eventual Consistency

**Q: Difference between eventual and strong consistency?**
Strong guarantees every read returns the most recent write (slower, needs coordination). Eventual only guarantees nodes converge eventually — reads may be stale during the convergence window (faster, async replication).

**Q: Is eventual consistency acceptable for banking?**
Generally no for balances/transactions — stale data could allow an invalid transfer. Parts like transaction history display can tolerate slight delays.

**Q: What is "read-your-own-writes" consistency?**
After you write, your subsequent reads return that write — e.g., you update your profile picture and immediately see it. Achieved by routing your reads to the replica you wrote to, or using sticky sessions.

---

## 4. Distributed Locks

### What Is It?

A distributed lock ensures only one server/process can perform a critical operation at a time, across the whole system.

**Analogy:** A single bathroom key at a restaurant — only one person holds it at a time.

```
Without distributed lock:
  Server 1 reads stock = 1, sells
  Server 2 reads stock = 1, sells  ← BOTH think 1 item available → oversold

With distributed lock:
  Server 1 acquires lock, sells, sets stock = 0, releases
  Server 2 blocked until release → reads stock = 0 → rejects sale
```

### Implementing Distributed Locks

#### Option 1: Redis-based Lock (SETNX)

```
Acquire:  SET lock_key "server1_uuid" NX PX 30000
  NX = only set if key doesn't exist
  PX 30000 = expire in 30s (TTL prevents deadlock if the holder crashes)

Release (Lua script — atomic check-then-delete):
  if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
  else
    return 0
  end
```

The Lua script ensures you only delete a lock you still own — without it, an expired lock could be re-acquired by Server 2 and then deleted by Server 1's late `DEL`.

#### Option 2: Redlock (Awareness)

Single Redis is a SPOF. Redlock acquires the lock on a **majority of N independent Redis nodes** for fault tolerance. Controversial for safety-critical locks — prefer Zookeeper/etcd in those cases.

#### Option 3: Database-based Locks

```sql
-- Pessimistic: row locked until transaction ends
SELECT * FROM inventory WHERE product_id = 1 FOR UPDATE;

-- Optimistic: version check, no actual lock
UPDATE inventory SET stock = stock - 1, version = version + 1
WHERE product_id = 1 AND version = 5;
-- 0 rows updated → someone else changed it → retry
```

### Common Interview Questions — Distributed Locks

**Q: What is a deadlock and how do you prevent it?**
Two processes each hold a lock the other needs, so both wait forever. Prevent by acquiring locks in a consistent order, using TTLs, and using tryLock-with-timeout instead of blocking forever.

**Q: Why not just use a database row lock as a distributed lock?**
It works but has limits: higher latency than Redis, the DB becomes a bottleneck, no built-in TTL to protect against crashed clients. Redis/Zookeeper are purpose-built and faster.

**Q: Difference between a distributed lock and a semaphore?**
A lock (mutex) allows 1 holder; a semaphore allows N holders. Rate-limiting to 10 concurrent operations uses a semaphore; protecting a single critical section uses a mutex.

---

## 5. Database Sharding

### What Is It?

Sharding splits a large database into smaller independent pieces (shards), each holding a subset of the data on different servers.

**Analogy:** An encyclopedia split into 26 volumes (A-B, C-D, ...) — each smaller and faster to search.

### Sharding vs Partitioning

- **Partitioning** — splitting data within a single database (logical split).
- **Sharding** — splitting data across multiple databases/servers (physical split).

### Sharding Strategies

- **Range-Based** — split by value ranges (IDs 1–1M → Shard 1). Pro: efficient range queries. Con: **hotspots** (new data all hits the latest shard).
- **Hash-Based** — `shard = hash(key) % N`. Pro: even distribution. Con: range queries impossible; adding a shard remaps almost all data (**resharding problem**).
- **Consistent Hashing** — places shards on a ring; a key goes to the next shard clockwise. Adding a shard only moves ~1/N of keys. Used by DynamoDB, Cassandra, Memcached.
- **Directory-Based** — a lookup table maps data to shards. Most flexible, but the directory is a bottleneck/SPOF and must be cached.

```
Hash-based:
  shard = hash(user_id) % 4
  user_id 12345 → hash → % 4 = 1 → Shard 1
```

### Choosing a Shard Key

```
Good shard key:
  ✓ High cardinality  (user_id good, boolean bad)
  ✓ Even distribution (UUIDs good; sequential IDs cause hotspots)
  ✓ Immutable         (user_id good; email can change)
  ✓ Matches query patterns

Bad: created_at (all new data → one shard), country_code (US/UK hotspots)
```

### Challenges (Awareness)

- **Cross-shard joins** — scatter-gather across shards is expensive. Mitigate by co-locating related data.
- **Cross-shard transactions** — require 2-Phase Commit or SAGA pattern; many sharded systems avoid them by design.
- **Rebalancing** — moving data when shards become uneven, ideally live (double-write, verify, switch).

### Common Interview Questions — Database Sharding

**Q: When should you shard?**
When a single node can't handle write throughput or data exceeds one server. Sharding adds major complexity — exhaust vertical scaling, caching, and read replicas first.

**Q: What is the resharding problem and how does consistent hashing solve it?**
With `hash(key) % N`, adding a server changes N and remaps almost all keys. Consistent hashing places servers on a ring so adding/removing one only affects ~1/N of keys.

**Q: How do you handle transactions across shards?**
Best: avoid them by co-locating related data. Otherwise use the SAGA pattern (local transactions + compensations) or accept eventual consistency.

---

## 6. Replication

### What Is It?

Replication keeps copies of your data on multiple servers. If one copy is lost, others survive, and multiple servers can serve reads at once.

### Why Replicate?

| Goal | How Replication Helps |
|------|----------------------|
| **High Availability** | If primary fails, promote a replica |
| **Read Scalability** | Spread read load across replicas |
| **Disaster Recovery** | Geographic replicas survive regional outages |

### Replication Topologies

- **Primary-Replica (Master-Slave)** — primary takes all writes; replicas serve reads. Replicas may lag. Used by MySQL, PostgreSQL, MongoDB.
- **Primary-Primary (Multi-Master)** — both nodes accept writes, replicated bidirectionally. Requires conflict resolution (LWW, CRDTs). Used by Galera, CockroachDB.

```
Primary-Replica:
  Writes → [Primary] → replicates to → [Replica 1] [Replica 2]
                              Reads served from replicas
```

### Synchronous vs Asynchronous Replication

| | Synchronous | Asynchronous |
|-|-------------|--------------|
| **Write confirms when** | All replicas confirm | Primary writes locally; replicas catch up |
| **Durability** | High — no data loss on primary failure | Risk of losing recent writes |
| **Write latency** | Higher | Lower |

**Semi-synchronous** (MySQL): confirm after at least 1 replica acks — balances safety and performance.

### Replication Lag

Lag = the delay between a write on the primary and when it appears on replicas.

```
t=0ms:  Primary updated: email=new@email.com
t=50ms: Profile reads from Replica 1 → still email=old@email.com  ← STALE READ
```

**Solutions:** route a user's reads to the primary briefly after a write (read-your-writes), or sticky-route a user to the same replica (monotonic reads).

### Common Interview Questions — Replication

**Q: Difference between replication and backup?**
Replication is real-time copying to live servers (availability, read scaling). Backups are point-in-time snapshots for recovering from logical errors. You need both — replication won't save you from "DELETE all rows."

**Q: How do you handle split-brain in a primary-replica setup?**
A partition makes both nodes think they're primary. Prevent with a quorum (majority must agree — Raft/Paxos) and fencing tokens so an old primary can't write. Tools: Patroni (PostgreSQL), MHA (MySQL).

**Q: What is a read replica and when would you use it?**
A copy that serves SELECT queries, offloading reads from the primary. Use when read traffic far exceeds writes. Caveat: reads may be slightly stale due to lag.

---

## 7. Message Queues

### What Is It?

A message queue sits between services so they communicate asynchronously. Service A sends a message and keeps working; Service B reads it when ready.

**Analogy:** Email — you send it and move on; the recipient processes it when they're ready.

```
Without queue (synchronous, tight coupling):
  Order Service → Payment / Inventory / Notification (waits for each)
  If any is slow or down → the whole flow fails

With queue (asynchronous, loose coupling):
  Order Service → [Queue] → Payment / Inventory / Notification (process when ready)
```

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Producer** | Sends/publishes messages |
| **Consumer** | Reads/subscribes to messages |
| **Broker** | The queue server (RabbitMQ, Kafka) |
| **Acknowledgment (ACK)** | Consumer signals successful processing |
| **Dead Letter Queue (DLQ)** | Where failed/unprocessable messages go |

### Queue Models

- **Point-to-Point (Queue)** — one message consumed by exactly one consumer. Good for task distribution. E.g., RabbitMQ, AWS SQS.
- **Publish-Subscribe (Topic)** — one message delivered to all subscribers. Good for event broadcasting. E.g., Kafka, AWS SNS.

### Message Delivery Guarantees

| Guarantee | Description | Risk |
|-----------|-------------|------|
| **At Most Once** | Delivered 0 or 1 times | Messages can be LOST |
| **At Least Once** | Delivered 1+ times (most common) | Messages can be DUPLICATED |
| **Exactly Once** | Delivered exactly once | Hardest/most expensive |

At-least-once → consumers may see a message twice. Fix: make consumers **idempotent** — record processed message IDs and skip duplicates.

### Dead Letter Queue (DLQ)

```
[Queue] → Consumer → error → broker retries (e.g., 3x)
  → after max retries → message moved to [DLQ]
DLQ lets you inspect failures, fix the bug, and replay messages.
```

### Kafka vs RabbitMQ

| Feature | Apache Kafka | RabbitMQ |
|---------|-------------|----------|
| **Model** | Distributed log (pub/sub) | Traditional broker (queue + pub/sub) |
| **Retention** | Configurable (days/forever) | Deleted after consumption |
| **Throughput** | Extremely high (millions/sec) | High (tens of thousands/sec) |
| **Replay** | Yes — consumers track their offset | No |
| **Use case** | Event streaming, audit logs, pipelines | Task queues, RPC, microservice messaging |

**Kafka basics:** a topic is split into partitions; each partition is consumed by one consumer per group; consumers track their **offset** and can replay by resetting it. Ordering is guaranteed **within a partition** — pick a partition key (e.g., `user_id`) so related events stay ordered.

### Common Interview Questions — Message Queues

**Q: How do you ensure messages aren't lost if a consumer crashes?**
Use ACKs — the broker keeps a message until the consumer ACKs; no ACK means redelivery. Combine with persistent (disk-backed) storage so messages survive broker restarts.

**Q: What is idempotency and why does it matter here?**
It lets a consumer handle the same message more than once safely — essential with at-least-once delivery. Use unique message IDs and skip duplicates.

**Q: How does Kafka guarantee ordering?**
Only within a single partition. Same partition key → same partition → preserved order.

**Q: When choose RabbitMQ over Kafka?**
RabbitMQ for flexible routing, request-reply, simpler setup. Kafka for high throughput, replay, audit logging, or many independent consumers of the same events.

---

## How These Topics Connect

```
[Clients] → [LOAD BALANCER] → [Server 1] [Server 2] [Server 3]
                                      │
        ┌─────────────────────────────┼──────────────────────────┐
        ↓                             ↓                           ↓
  DATABASE (sharded + replicated)  MESSAGE QUEUE           CACHE / Redis
                                                        (distributed locks)
```

- **CAP** governs every storage decision: sharded DB + async replication = AP; Zookeeper (for locks) = CP.
- **Eventual Consistency** is the outcome of async replication and message queues passing data between services.
- **Distributed Locks** protect shared resources when multiple servers access them.
- **Message Queues** enable decoupling, load leveling, and fan-out.

---

## Quick Revision Cheat Sheet

```
LOAD BALANCING
  Algorithms: Round Robin | Weighted | Least Connections | IP Hash | Least Response Time
  Layers: L4 (TCP, fast) | L7 (HTTP, smart routing)
  Sticky sessions → prefer shared Redis session store; health checks auto-remove dead servers
  Run redundant LBs to avoid SPOF

CAP THEOREM
  C = latest read | A = always responds | P = survives partitions
  P is non-negotiable → real choice is CP vs AP
  CP: Zookeeper, etcd, HBase | AP: Cassandra, DynamoDB, DNS | ACID≈CP, BASE≈AP

EVENTUAL CONSISTENCY
  All nodes converge eventually; cause = async replication lag
  Conflict resolution: Last Write Wins | Vector Clocks | CRDTs

DISTRIBUTED LOCKS
  Redis: SET key val NX PX ttl (release via Lua check-then-delete)
  Redlock = majority of N nodes | Zookeeper = ephemeral sequential znodes
  Keep critical operations idempotent

DATABASE SHARDING
  Range (range queries, hotspots) | Hash (even, no range, resharding pain)
  Consistent Hashing (minimal resharding) | Directory (flexible, SPOF risk)
  Shard key: high cardinality + immutable + even + matches queries
  Exhaust vertical scaling + caching + read replicas first

REPLICATION
  Primary-Replica (read scaling) | Primary-Primary (conflicts)
  Sync = no data loss/higher latency | Async = possible loss/low latency
  Lag → stale reads; avoid split-brain via quorum + fencing

MESSAGE QUEUES
  Point-to-Point (1 consumer) | Pub/Sub (all consumers)
  Guarantees: at-most-once | at-least-once (common) | exactly-once (hardest)
  At-least-once → make consumers idempotent | DLQ for failed messages
  Kafka: log, replay, high throughput, per-partition order | RabbitMQ: routing, simpler
```

---

*Last updated: 2026-06-18 | Focus: Backend and System Design Interviews*
