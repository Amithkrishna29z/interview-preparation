# Distributed Systems Core Concepts — Study Guide

> Study notes covering the 7 most critical distributed systems topics for backend and system design interviews.

---

## 1. Load Balancing

### What Is It?

**Easy Explanation:** A load balancer is a traffic cop that sits in front of your servers and distributes incoming requests across multiple servers so no single server gets overwhelmed.

**Real-world analogy:** A supermarket manager directing customers to the shortest checkout lane so no single cashier has a massive queue while others are idle.

```
With Load Balancer:
  Client → [Load Balancer] → [Server 1]
                           → [Server 2]
                           → [Server 3]
```

### Why Is Load Balancing Needed?

| Problem | How Load Balancing Solves It |
|---------|------------------------------|
| Single server gets overloaded | Distributes requests across multiple servers |
| Server crashes → entire app down | Traffic rerouted to healthy servers |
| Traffic spikes (flash sales) | Scale horizontally — add more servers |
| Geographically distributed users | Route users to nearest server |

### Load Balancing Algorithms

The common ones a junior should know:

- **Round Robin** — each server in turn, cyclically. Best for identical servers; ignores actual load.
- **Weighted Round Robin** — servers with more capacity get proportionally more traffic. Best for mixed hardware.
- **Least Connections** — new request goes to the server with the fewest active connections. Best for long-lived connections (WebSockets).
- **Least Response Time** — routes to the server with lowest average response time. Best for latency-sensitive APIs.
- **IP Hash (Sticky Sessions)** — hashes client IP to always route to the same server. Best for stateful apps; if that server dies, its sessions are lost. Modern fix: shared session store (Redis).
- **Random** — simple, surprisingly effective at scale.

```
Round Robin:
  Request 1 → Server 1
  Request 2 → Server 2
  Request 3 → Server 3
  Request 4 → Server 1  (starts over)
```

### Layer 4 vs Layer 7 Load Balancing

| | Layer 4 (Transport) | Layer 7 (Application) |
|-|--------------------|-----------------------|
| **Operates at** | TCP/UDP level | HTTP/HTTPS level |
| **Sees** | IP addresses and ports | URLs, headers, cookies, body |
| **Routing based on** | IP + port only | URL path, header values, cookies |
| **Performance** | Faster (less inspection) | Slightly slower (deep inspection) |
| **Examples** | AWS NLB, HAProxy (TCP) | AWS ALB, Nginx, HAProxy (HTTP) |

**Example — Layer 7 routing:** route `/api/*` to API servers, `/static/*` to a CDN, `/admin/` to an internal service.

### Health Checks

Load balancers continuously probe backends and remove unhealthy ones from rotation:

```
Every 30 seconds:
  GET /health → 200 OK    ✓ Keep in rotation
  GET /health → 503       ✗ Remove from rotation
  GET /health → 200 OK    ✓ Add back when recovered
```

### Sticky Sessions (Session Affinity)

The problem: if Server 1 stores a user's cart in memory and the next request goes to Server 2, the cart is gone.

```
Option 1: Sticky Sessions → always send user X to Server 1
  → Downside: uneven load, server failure loses sessions

Option 2: Shared Session Store (RECOMMENDED)
  → All session data in Redis → any server handles any request
  → Stateless servers = true horizontal scaling
```

### Global Load Balancing (GeoDNS)

*Awareness:* DNS resolves users to the nearest data center's load balancer (EU users → EU servers). Reduces latency and provides disaster recovery — if a region fails, DNS reroutes traffic.

### Common Interview Questions — Load Balancing

**Q: What is a single point of failure (SPOF) and how does a load balancer help?**
A SPOF is a component whose failure brings down the whole system. A load balancer distributes traffic so one server failing doesn't take down the app. The load balancer itself can be a SPOF — solved by running redundant load balancers (active-passive or active-active HA pairs).

**Q: What is the difference between a load balancer and a reverse proxy?**
A reverse proxy forwards client requests to backend servers. A load balancer is a reverse proxy that also distributes requests across multiple servers. All load balancers are reverse proxies; not all reverse proxies are load balancers.

**Q: How does a load balancer handle a server that crashes mid-request?**
The client gets a connection error/timeout. Health checks mark the server unhealthy so future requests skip it. The failed request must be retried by the client. Mitigate with timeouts, circuit breakers, and retry logic.

---

## 2. CAP Theorem

### What Is It?

**CAP Theorem** states that a distributed system can guarantee **at most 2 of these 3 properties** at the same time:

| Property | Meaning |
|----------|---------|
| **C — Consistency** | Every read gets the most recent write (or an error). All nodes see the same data. |
| **A — Availability** | Every request gets a response (not necessarily the latest data). |
| **P — Partition Tolerance** | The system keeps working even if network messages between nodes are lost or delayed. |

**Real-world analogy (ATMs):**
- **Consistency** = every ATM shows your exact current balance.
- **Availability** = every ATM always responds, never "try again later."
- **Partition Tolerance** = ATMs keep working even if the network between branches goes down.

### Why You Can Only Have 2 of 3

In any real distributed system, **network partitions WILL happen** — so **P is not optional**. The real-world choice is CP vs AP:

```
CP — Consistency + Partition Tolerance
  → Sacrifice Availability: during a partition, return an error instead of stale data
  → Examples: HBase, MongoDB (strong mode), Zookeeper, etcd

AP — Availability + Partition Tolerance
  → Sacrifice Consistency: during a partition, return possibly stale data
  → Examples: Cassandra, CouchDB, DynamoDB, DNS

CA — only possible on a single node (not truly distributed)
```

### CP vs AP in Practice

```
Partition between Node A and Node B; A took a write that B hasn't seen.

CP choice: Node B refuses reads → "System unavailable" (no stale data served)
  Use case: banking, payments — stale data = disaster

AP choice: Node B answers with its last known (stale) value
  Use case: social feeds, DNS, product catalogs — stale data is acceptable
```

### Choosing CP vs AP — Decision Guide

```
Choose CP when: financial transactions, inventory (can't oversell),
                auth/token validation, leader election

Choose AP when: social feeds, ratings/reviews, DNS, shopping cart browsing,
                metrics/analytics, profile data (bio, avatar)
```

### PACELC Theorem (Awareness)

PACELC extends CAP: if there's a **P**artition, choose **A** vs **C**; **E**lse (normal operation), choose **L**atency vs **C**onsistency. The point: even without partitions, strong consistency costs latency because nodes must coordinate. (e.g., DynamoDB = PA/EL, MongoDB = PC/EC.)

### Common Interview Questions — CAP Theorem

**Q: Can a system be CA in a distributed setup?**
Technically no — partitions are inevitable, so you must tolerate them. A "CA" system is effectively single-node or runs in a tightly controlled network where partitions are near-impossible.

**Q: Is CAP a limitation or a design choice?**
Both. It's a proven limitation, but how a system handles the trade-off is a deliberate choice. Cassandra chooses AP; Zookeeper chooses CP — neither is wrong, it depends on the use case.

**Q: How does CAP relate to ACID and BASE?**
- **ACID** (traditional RDBMS guarantees) maps to CP.
- **BASE** (Basically Available, Soft state, Eventually consistent — the NoSQL approach) maps to AP.

---

## 3. Eventual Consistency

### What Is It?

**Easy Explanation:** Eventual consistency means that if no new updates are made, eventually all nodes will return the same value. There's no guarantee of *when*, but it will happen.

**Real-world analogy:** DNS propagation. Change a domain's IP and it doesn't update everywhere instantly — for some time, some users get the old server, others the new one. Eventually all DNS servers reflect the update.

### Strong vs Eventual Consistency

```
Strong Consistency:
  Write → all nodes updated → read always returns the new value
  Slower writes (must confirm all nodes); every read is current.

Eventual Consistency:
  Write → primary updated → replicas updated asynchronously
  [Write X=5] → Node A: X=5 | Node B: X=3 | Node C: X=3  (replication lag)
  ...time passes... → all nodes converge to X=5
  Faster writes; reads may be stale briefly.
```

### Consistency Models (Awareness)

A spectrum from strongest to weakest — know the names exist:

- **Linearizable** — strongest; behaves like a single node (etcd, Zookeeper).
- **Sequential / Causal** — all nodes see operations in the same / causally-correct order.
- **Read-your-own-writes** — you always see your own updates immediately (most web apps need this).
- **Monotonic reads** — once you've seen a value, you won't later see an older one.
- **Eventual** — weakest; converges with no time guarantee (Cassandra, DynamoDB default).

### How It Works — The Mechanism

```
Step 1: Client writes to primary → primary ACKs immediately
Step 2: Primary replicates async to replicas (10ms–500ms)
Step 3: During the lag, a read from a replica may be stale ("stale read")
Step 4: Replication completes → all replicas converge → reads are current
```

### Conflict Resolution (Awareness)

When two nodes accept writes for the same data during a partition, the system must resolve the conflict. Approaches you should be able to name:

- **Last Write Wins (LWW)** — highest timestamp wins. Simple, but vulnerable to clock skew between servers.
- **Vector Clocks** — version vectors track causality so the system can detect concurrent (conflicting) writes and surface them for resolution. Used by DynamoDB, Riak.
- **CRDTs (Conflict-free Replicated Data Types)** — data structures that mathematically always converge without conflicts (e.g., a grow-only counter). Used in collaborative editing (Google Docs) and distributed counters.

### Read Repair and Anti-Entropy (Awareness)

Background mechanisms that restore consistency over time:

- **Read Repair** — when a read sees replicas disagree, it serves the latest value and updates the stale replica.
- **Anti-Entropy** — a background process compares replicas (using Merkle trees to find diffs efficiently) and syncs them.

### Common Interview Questions — Eventual Consistency

**Q: Difference between eventual and strong consistency?**
Strong guarantees every read returns the most recent write (slower, needs coordination). Eventual only guarantees nodes converge *eventually* — reads may be stale during the convergence window (faster, allows async replication).

**Q: Is eventual consistency acceptable for banking?**
Generally no for balances/transactions — a stale balance could allow an invalid transfer. But parts like transaction-history display or notifications can tolerate slight delays.

**Q: What is "read-your-own-writes" consistency?**
A guarantee that after you write, your subsequent reads return that write — e.g., you update your profile picture and immediately see it. Achieved by routing your reads to the replica you wrote to, or using sticky sessions.

---

## 4. Distributed Locks

### What Is It?

**Easy Explanation:** A distributed lock ensures only one server/process can perform a critical operation at a time, across the whole system.

**Real-world analogy:** A single bathroom key at a restaurant. Only one person holds it at a time; everyone else waits.

```
Without distributed lock:
  Server 1 reads stock = 1, sells
  Server 2 reads stock = 1, sells  ← BOTH think 1 item available → oversold

With distributed lock:
  Server 1 acquires lock, sells, sets stock = 0, releases
  Server 2 blocked until release → reads stock = 0 → rejects sale
```

### Why Distributed Locks Are Hard

A single-machine mutex is easy; distributed locks face network failures (lock holder crashes before releasing), clock skew (unreliable TTLs), GC pauses (a process pauses past its lock TTL), and split-brain (two servers both think they hold the lock).

### Implementing Distributed Locks

#### Option 1: Redis-based Lock (SETNX)

```
Acquire:  SET lock_key "server1_uuid" NX PX 30000
  NX = only set if key doesn't exist
  PX 30000 = expire in 30s (TTL prevents deadlock if the holder crashes)

Release (Lua script for atomicity — only delete if you still own it):
  if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
  else
    return 0
  end
```

**Why the Lua script?** Without an atomic check-then-delete, an expired lock could be re-acquired by Server 2, and Server 1's late `DEL` would delete Server 2's lock.

#### Option 2: Redlock (Awareness)

Single Redis is a SPOF. Redlock acquires the lock on a **majority of N independent Redis nodes** (e.g., 3 of 5) for fault tolerance. **Controversial** — Martin Kleppmann argued it isn't safe under GC pauses and clock drift; use Zookeeper/etcd for safety-critical locks.

#### Option 3: Zookeeper Locks (Awareness)

A distributed coordination service. Clients create **ephemeral sequential znodes**; the lowest-numbered node holds the lock, others watch the next-lowest. Key advantage: ephemeral nodes auto-delete when a client disconnects, so crashes release locks automatically.

#### Option 4: Database-based Locks

```sql
-- Pessimistic: row locked until the transaction ends
SELECT * FROM inventory WHERE product_id = 1 FOR UPDATE;

-- Optimistic: version check, no actual lock
UPDATE inventory SET stock = stock - 1, version = version + 1
WHERE product_id = 1 AND version = 5;
-- 0 rows updated → someone else changed it → retry
```

### Fencing Tokens (Awareness)

If a lock holder GC-pauses past its TTL, two servers can run the critical section at once. **Fencing tokens** fix this: the lock service hands out a monotonically increasing token with each grant; storage rejects any write carrying a token older than the highest it has seen, so a stale holder can't write.

### Common Interview Questions — Distributed Locks

**Q: What is a deadlock and how do you prevent it?**
Two processes each hold a lock the other needs, so both wait forever. Prevent by acquiring locks in a consistent order, using TTLs, using tryLock-with-timeout instead of blocking forever, and deadlock detection via timeouts.

**Q: Why not just use a database row lock as a distributed lock?**
It works but has limits: higher latency than Redis, the DB becomes a bottleneck, it doesn't span multiple databases, and there's no built-in TTL to protect against crashed clients. Redis/Zookeeper are purpose-built and faster.

**Q: Difference between a distributed lock and a semaphore?**
A lock (mutex) allows 1 holder; a semaphore allows N holders. Rate-limiting to 10 concurrent operations uses a semaphore of 10; protecting a single critical section uses a mutex.

---

## 5. Database Sharding

### What Is It?

**Easy Explanation:** Sharding splits a large database into smaller independent pieces (shards), each holding a subset of the data on different servers.

**Real-world analogy:** An encyclopedia split into 26 volumes (A-B, C-D, ...) — each smaller and faster to search.

```
With sharding:
  [Users A-D → Shard 1] [Users E-K → Shard 2]
  [Users L-R → Shard 3] [Users S-Z → Shard 4]
  → Each shard holds a fraction of users → faster queries, distributed storage
```

### Sharding vs Partitioning

- **Partitioning** — splitting data within a single database (logical split).
- **Sharding** — splitting data across multiple databases/servers (physical split). It's horizontal partitioning across separate instances.

### Sharding Strategies

- **Range-Based** — split by value ranges of the shard key (IDs 1–1M → Shard 1). Pro: efficient range queries. Con: **hotspots** (new data all hits the latest shard).
- **Hash-Based** — `shard = hash(key) % N`. Pro: even distribution, no hotspots. Con: range queries impossible; adding a shard (`% 4` → `% 5`) remaps almost all data (the **resharding problem**).
- **Consistent Hashing** — places shards on a ring; a key goes to the next shard clockwise. Adding a shard only moves the keys in its neighborhood (~1/N), not everything. Used by DynamoDB, Cassandra, Memcached, Riak.
- **Directory-Based** — a lookup table maps data to shards. Most flexible (move data by updating the directory), but the directory is a bottleneck/SPOF and must be cached.

```
Hash-based:
  shard = hash(user_id) % 4
  user_id 12345 → hash → % 4 = 1 → Shard 1
```

### Choosing a Shard Key

The most critical decision — a bad key creates hotspots.

```
Good shard key:
  ✓ High cardinality      (many distinct values; user_id good, boolean bad)
  ✓ Even distribution     (UUIDs good; sequential IDs cause hotspots)
  ✓ Immutable             (user_id good; email can change)
  ✓ Matches query patterns + keeps related data co-located

Bad: created_at (all new data → one shard), country_code (US/UK hotspots), boolean status
```

### Challenges of Sharding (Awareness)

- **Cross-shard joins** — "all orders for premium users" may need a scatter-gather across every shard. Mitigate by denormalizing, co-locating related data on the same shard, or using a separate analytics DB.
- **Rebalancing** — moving data when shards become uneven, ideally live (double-write during migration, verify, switch).
- **Cross-shard transactions** — require 2-Phase Commit or the SAGA pattern; both add complexity, so many sharded systems avoid them by design.

### Common Interview Questions — Database Sharding

**Q: When should you shard?**
When a single node can't handle the write throughput, data exceeds one server, or latency is unacceptable. Sharding adds major complexity — exhaust vertical scaling, caching, and read replicas first.

**Q: What is the resharding problem and how does consistent hashing solve it?**
With `hash(key) % N`, adding a server changes N and remaps almost all keys. Consistent hashing places servers on a ring so adding/removing one only affects ~1/N of keys.

**Q: How do you handle transactions across shards?**
Best: avoid them by co-locating related data. Otherwise use the SAGA pattern (local transactions + compensations), 2PC (correct but slow/blocking), or accept eventual consistency.

---

## 6. Replication

### What Is It?

**Easy Explanation:** Replication keeps copies of your data on multiple servers. If one copy is lost, others survive, and multiple servers can serve reads at once.

**Real-world analogy:** A book in 10 libraries — if one burns down the book survives, and 10 people can read it simultaneously.

### Why Replicate?

| Goal | How Replication Helps |
|------|----------------------|
| **High Availability** | If primary fails, promote a replica |
| **Fault Tolerance** | Survive hardware/data-center failures |
| **Read Scalability** | Spread read load across replicas |
| **Disaster Recovery** | Geographic replicas survive regional outages |

### Replication Topologies

- **Primary-Replica (Master-Slave)** — primary takes all writes; replicas replay the write log and serve reads. Replicas may lag (**replication lag**). Used by MySQL, PostgreSQL, MongoDB.
- **Primary-Primary (Multi-Master)** — both nodes accept writes, replicated bidirectionally. Requires **conflict resolution** (LWW, app-level, CRDTs). Used by Galera, CockroachDB, active-active setups.
- **Chain Replication** *(awareness)* — writes enter at a head node, propagate through a chain, and only the tail serves reads, giving strong-consistency reads.

```
Primary-Replica:
        Writes → [Primary] → replicates to → [Replica 1] [Replica 2]
                                   Reads served from replicas
```

### Synchronous vs Asynchronous Replication

| | Synchronous | Asynchronous |
|-|-------------|--------------|
| **Write confirms when** | All replicas confirm | Primary writes locally; replicas catch up later |
| **Durability** | High — no data loss on primary failure | Risk of losing recent writes |
| **Write latency** | Higher (waits for slowest replica) | Lower |
| **Replication lag** | None | Possible (ms to seconds) |

**Semi-synchronous** (MySQL): confirm after at least 1 replica acks — a balance of safety and performance.

### Replication Lag and Its Consequences

Lag is the delay between a write on the primary and when it appears on replicas.

```
t=0ms:  Primary updated: email=new@email.com → user redirected to profile
t=50ms: Profile reads from Replica 1, still email=old@email.com  ← STALE READ
User: "Why is my old email still showing?"
```

**Solutions:** read-your-own-writes (route a user's reads to the primary briefly after a write), monotonic reads (sticky-route a user to the same replica), or read critical data (balance, inventory) from the primary.

### Failover (Awareness)

When the primary fails, a replica is promoted: detect failure → elect the most up-to-date replica → promote it → repoint other replicas and the app. Challenges: **split-brain** (both think they're primary), **data loss** (async replica missing latest writes), **false positives** (a network hiccup triggers needless failover). Mitigations: quorum-based election (Raft/Paxos), fencing tokens, semi-sync replication.

### Common Interview Questions — Replication

**Q: Difference between replication and backup?**
Replication is real-time copying to live servers (availability, read scaling). Backups are point-in-time snapshots for recovering from logical errors (accidental deletes, corruption). You need both — replication won't save you from "DELETE all rows," backups don't give instant failover.

**Q: How do you handle split-brain in a primary-replica setup?**
A partition makes both nodes think they're primary. Prevent with a quorum (majority must agree on the primary — Raft/Paxos) and fencing tokens so a fenced-off old primary can't write. Tools: Patroni (PostgreSQL), MHA (MySQL).

**Q: What is a read replica and when would you use it?**
A copy that serves SELECT queries, offloading reads from the primary. Use when read traffic far exceeds writes (common). Caveat: reads may be slightly stale due to lag.

---

## 7. Message Queues

### What Is It?

**Easy Explanation:** A message queue sits between services so they communicate asynchronously. Service A sends a message and keeps working; Service B reads it when ready.

**Real-world analogy:** Email. You send it and move on; the recipient processes it when they're ready.

```
Without queue (synchronous, tight coupling):
  Order Service → Payment / Inventory / Notification (waits for each)
  If any is slow or down → the whole flow fails or slows

With queue (asynchronous, loose coupling):
  Order Service → [Queue] → Payment / Inventory / Notification (process when ready)
  Order Service publishes and moves on — no waiting
```

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Producer** | Sends/publishes messages |
| **Consumer** | Reads/subscribes to messages |
| **Queue / Topic** | Buffer (queue) or named channel (topic, pub/sub) for messages |
| **Broker** | The queue server (RabbitMQ, Kafka) |
| **Acknowledgment (ACK)** | Consumer signals successful processing |
| **Dead Letter Queue (DLQ)** | Where failed/unprocessable messages go |

### Queue Models

- **Point-to-Point (Queue)** — one message consumed by exactly one consumer. Good for task distribution (10 workers each pick the next job). E.g., RabbitMQ queues, AWS SQS.
- **Publish-Subscribe (Topic)** — one message delivered to all subscribers. Good for event broadcasting (one event → many independent reactions). E.g., Kafka topics, AWS SNS.

### Why Use Message Queues?

- **Decoupling** — the producer doesn't know or care who consumes; consumers can be down and catch up later.
- **Load leveling** — the queue absorbs spikes (100k orders in 10s) while consumers process at a steady rate, smoothing the load.
- **Reliability** — a message stays until ACKed; a crashed consumer means the message is redelivered (at-least-once).
- **Async processing** — a video upload returns immediately while transcoding happens in the background.

### Message Delivery Guarantees

| Guarantee | Description | Risk |
|-----------|-------------|------|
| **At Most Once** | Delivered 0 or 1 times | Messages can be LOST |
| **At Least Once** | Delivered 1+ times (most common) | Messages can be DUPLICATED |
| **Exactly Once** | Delivered exactly once | Hardest/most expensive to implement |

```
At-least-once → consumer may see a message twice.
Fix: make consumers IDEMPOTENT.
  "Apply payment of $50 for order 12345"
  → Already paid? skip. Not paid? process.
```

### Idempotency

Idempotency means applying an operation multiple times has the same result as applying it once. It's essential with at-least-once delivery, where retries and redeliveries can deliver the same message twice. Implement by recording processed message IDs and skipping duplicates.

### Dead Letter Queue (DLQ)

```
[Queue] → Consumer → processing error → broker retries (e.g., 3x)
  → after max retries → message moved to [Dead Letter Queue]

DLQ lets you inspect failures, fix the bug, replay messages, and alert on DLQ size.
```

### Kafka vs RabbitMQ

| Feature | Apache Kafka | RabbitMQ |
|---------|-------------|----------|
| **Model** | Distributed log (pub/sub) | Traditional broker (queue + pub/sub) |
| **Retention** | Configurable (days/forever) | Deleted after consumption |
| **Throughput** | Extremely high (millions/sec) | High (tens of thousands/sec) |
| **Replay** | Yes — consumers track their offset | No — consumed messages are gone |
| **Use case** | Event streaming, audit logs, pipelines | Task queues, RPC, microservice messaging |
| **Complexity** | Higher (partitions, KRaft/Zookeeper) | Lower (easier setup) |

**Kafka basics (awareness):** a topic is split into partitions; each partition is consumed by one consumer per group; consumers track their **offset** (position) and can replay by resetting it. Ordering is guaranteed only **within a partition** — pick a partition key (e.g., `user_id`) so related events stay ordered.

### Common Patterns (Awareness)

- **Saga** — distributed transaction as a chain of local transactions across services, with compensating transactions on failure (e.g., refund payment if inventory fails).
- **Fan-Out** — one event drives many independent consumers (`user.registered` → email, CRM, analytics, fraud check).
- **Work Queue** — N workers compete for tasks; auto-scale workers based on queue depth.

### Common Interview Questions — Message Queues

**Q: Difference between a message queue and an event bus?**
A queue (RabbitMQ) is pull-based, focused on task distribution (one consumer per message). An event bus (Kafka) broadcasts events to many subscribers. The line is blurry — Kafka can do both, RabbitMQ supports pub/sub via exchanges.

**Q: How do you ensure messages aren't lost if a consumer crashes?**
Use ACKs — the broker keeps a message until the consumer ACKs; no ACK means redelivery. Combine with persistent (disk-backed) storage so messages survive broker restarts.

**Q: What is idempotency and why does it matter here?**
It lets a consumer handle the same message more than once safely — essential with at-least-once delivery. Use unique message IDs and track processed IDs; skip duplicates.

**Q: How does Kafka guarantee ordering?**
Only within a single partition. Same partition key → same partition → preserved order. Design the key around what needs ordering (e.g., `user_id`).

**Q: When choose RabbitMQ over Kafka?**
RabbitMQ for flexible routing, request-reply, per-message TTL, or simpler setup. Kafka for high throughput, replay, audit logging, or many independent consumers of the same events.

---

## How These Topics Connect

For system design interviews, see how the pieces fit together:

```
        [Clients] → [LOAD BALANCER] → [Server 1] [Server 2] [Server 3]
                                            │
              ┌─────────────────────────────┼─────────────────────────┐
              ↓                             ↓                          ↓
        DATABASE (sharded + replicated)  MESSAGE QUEUE (Kafka/Rabbit)  CACHE / Redis
                                                                       (distributed locks)
```

- **CAP** governs every storage decision: sharded DB + async replication = AP; Zookeeper (for locks) = CP.
- **Eventual Consistency** is the outcome of async replication and of message queues passing data between services.
- **Distributed Locks** protect shared resources when multiple servers (behind the load balancer) access them.
- **Message Queues** enable decoupling, load leveling, and fan-out.

---

## Quick Revision Cheat Sheet

```
LOAD BALANCING
  Algorithms: Round Robin | Weighted | Least Connections | IP Hash | Least Response Time
  Layers: L4 (TCP, fast) | L7 (HTTP, smart routing)
  Sticky sessions → prefer shared Redis session store; health checks auto-remove dead servers
  Run redundant LBs so the LB isn't a SPOF

CAP THEOREM
  C = latest read | A = always responds | P = survives partitions
  P is non-negotiable → real choice is CP vs AP
  CP: Zookeeper, etcd, HBase | AP: Cassandra, DynamoDB, DNS | ACID≈CP, BASE≈AP

EVENTUAL CONSISTENCY
  All nodes converge eventually; cause = async replication lag
  Conflict resolution: Last Write Wins | Vector Clocks | CRDTs
  Read repair fixes stale replicas on detected inconsistency

DISTRIBUTED LOCKS
  Redis: SET key val NX PX ttl (release via Lua check-then-delete)
  Redlock = majority of N nodes | Zookeeper = ephemeral sequential znodes
  Fencing token prevents stale holders writing; keep operations idempotent

DATABASE SHARDING
  Range (range queries, hotspots) | Hash (even, no range, resharding pain)
  Consistent Hashing (minimal resharding) | Directory (flexible)
  Shard key: high cardinality + immutable + even + matches queries
  Exhaust vertical scaling + caching + read replicas first

REPLICATION
  Primary-Replica (read scaling) | Primary-Primary (conflicts) | Chain (strong reads)
  Sync = no data loss/higher latency | Async = possible loss/low latency
  Lag → stale reads (use read-your-writes); avoid split-brain via quorum + fencing

MESSAGE QUEUES
  Point-to-Point (1 consumer) | Pub/Sub (all consumers)
  Guarantees: at-most-once | at-least-once (common) | exactly-once (hardest)
  At-least-once → make consumers idempotent | DLQ for failed messages
  Kafka: log, replay, high throughput, per-partition order | RabbitMQ: routing, simpler
```

---

*Last updated: 2026-06-17 | Focus: Backend and System Design Interviews*
