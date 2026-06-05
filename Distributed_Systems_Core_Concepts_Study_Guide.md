# Distributed Systems Core Concepts — Study Guide

> Deep-dive study notes covering the 7 most critical distributed systems topics for backend, cloud, and system design interviews.

---

## Table of Contents
1. [Load Balancing](#1-load-balancing)
2. [CAP Theorem](#2-cap-theorem)
3. [Eventual Consistency](#3-eventual-consistency)
4. [Distributed Locks](#4-distributed-locks)
5. [Database Sharding](#5-database-sharding)
6. [Replication](#6-replication)
7. [Message Queues](#7-message-queues)
8. [How These Topics Connect](#how-these-topics-connect)
9. [Quick Revision Cheat Sheet](#quick-revision-cheat-sheet)

---

## 1. Load Balancing

### What Is It?

**Easy Explanation:** A load balancer is a traffic cop that sits in front of your servers and distributes incoming requests across multiple servers so no single server gets overwhelmed.

**Real-world analogy:** A supermarket with 10 checkout lanes. A manager (load balancer) directs customers to the shortest/least-busy lane so no single cashier has a massive queue while others are idle.

```
Without Load Balancer:
  Client → [Single Server] ← overloaded, single point of failure

With Load Balancer:
  Client → [Load Balancer] → [Server 1]
                           → [Server 2]
                           → [Server 3]
```

---

### Why Is Load Balancing Needed?

| Problem | How Load Balancing Solves It |
|---------|------------------------------|
| Single server gets overloaded | Distributes requests across multiple servers |
| Server crashes → entire app down | Traffic rerouted to healthy servers |
| Traffic spikes (flash sales, viral posts) | Scale horizontally — add more servers |
| Geographically distributed users | Route users to nearest server |

---

### Load Balancing Algorithms

#### 1. Round Robin
Requests are sent to each server in turn, cyclically.

```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1  (starts over)
Request 5 → Server 2
...
```

- **Best for:** Servers with identical hardware and processing time
- **Problem:** Ignores server load — a slow request on Server 1 doesn't stop the next request from going to Server 1

#### 2. Weighted Round Robin
Servers with more capacity get proportionally more traffic.

```
Server 1 (weight 3): gets 3 out of every 5 requests
Server 2 (weight 2): gets 2 out of every 5 requests
```

- **Best for:** Heterogeneous server fleet (different specs)

#### 3. Least Connections
New requests go to the server with the fewest active connections.

```
Server 1: 100 active connections
Server 2:  20 active connections  ← next request goes here
Server 3:  75 active connections
```

- **Best for:** Long-lived connections (WebSockets, file uploads)
- **Problem:** Connection count alone doesn't reflect CPU/memory load

#### 4. Least Response Time
Routes to the server with the lowest average response time AND fewest active connections.

- **Best for:** Latency-sensitive applications (APIs, real-time systems)
- More sophisticated than least connections

#### 5. IP Hash (Sticky Sessions)
The client's IP address is hashed to always route to the same server.

```
Client IP 192.168.1.10 → always → Server 2
Client IP 10.0.0.5     → always → Server 1
```

- **Best for:** Stateful apps where session data is stored on the server
- **Problem:** If Server 2 goes down, all its clients lose their sessions
- **Modern alternative:** Use a shared session store (Redis) so any server can handle any client

#### 6. Random
Requests are sent to a randomly selected server.

- Simple, surprisingly effective at scale (large numbers)
- No consideration of server load

#### 7. Resource-Based (Adaptive)
Load balancer queries each server for current CPU/memory utilization and routes accordingly.

- Most intelligent algorithm
- Requires agents on each server to report health metrics

---

### Layer 4 vs Layer 7 Load Balancing

| | Layer 4 (Transport) | Layer 7 (Application) |
|-|--------------------|-----------------------|
| **Operates at** | TCP/UDP level | HTTP/HTTPS level |
| **Sees** | IP addresses and ports | URLs, headers, cookies, body |
| **Routing based on** | IP + port only | URL path, header values, cookies |
| **Performance** | Faster (less inspection) | Slightly slower (deep inspection) |
| **Example use** | Route all port 443 traffic | Route `/api/*` to API servers, `/static/*` to CDN |
| **Examples** | AWS NLB, HAProxy (TCP mode) | AWS ALB, Nginx, HAProxy (HTTP mode) |

**Example — Layer 7 routing:**
```
/api/users    → User Service (3 servers)
/api/orders   → Order Service (5 servers)
/static/      → CDN or Static File Server
/admin/       → Admin Service (internal network only)
```

---

### Health Checks

Load balancers continuously probe backend servers:

```
Every 30 seconds:
  GET http://server1/health → 200 OK    ✓ Keep in rotation
  GET http://server2/health → 503       ✗ Remove from rotation
  GET http://server3/health → timeout   ✗ Remove from rotation

When server2 recovers:
  GET http://server2/health → 200 OK    ✓ Add back to rotation
```

---

### Sticky Sessions (Session Affinity)

The problem: If Server 1 stores a user's shopping cart in memory, and the next request goes to Server 2, the cart is gone.

**Solutions:**

```
Option 1: Sticky Sessions (IP Hash or Cookie-based)
  → Load balancer always sends user X to Server 1
  → Problem: uneven load distribution, server failure loses sessions

Option 2: Shared Session Store (RECOMMENDED)
  → All session data stored in Redis
  → Any server can handle any request
  → Stateless servers = true horizontal scaling

  Client → Load Balancer → Server 1 or 2 or 3
                                    ↕
                                  Redis (shared session store)
```

---

### Global Load Balancing (GeoDNS)

For geographically distributed systems:

```
User in Europe    → DNS resolves to EU Load Balancer  → EU servers
User in USA       → DNS resolves to US Load Balancer  → US servers
User in Asia      → DNS resolves to AP Load Balancer  → AP servers
```

- Reduces latency by routing users to nearest data center
- Provides disaster recovery — if one region fails, DNS updated to reroute traffic

---

### Common Interview Questions — Load Balancing

**Q: What is a single point of failure and how does a load balancer help?**
A single point of failure (SPOF) is a component whose failure brings down the whole system. A load balancer helps by distributing traffic so if one server fails, others handle the load. However, the load balancer itself can become a SPOF — solved by running redundant load balancers (active-passive or active-active HA pairs).

**Q: What is the difference between a load balancer and a reverse proxy?**
A reverse proxy sits in front of servers and forwards client requests to them. A load balancer is a type of reverse proxy that also distributes requests across multiple servers. Nginx and HAProxy can act as both. All load balancers are reverse proxies but not all reverse proxies are load balancers.

**Q: How does a load balancer handle a server that crashes mid-request?**
The client receives a connection error or timeout. The load balancer marks the server unhealthy via health checks. Future requests are not sent to it. The failed request must be retried by the client (or an upstream service). To reduce impact: use timeouts, circuit breakers, and retry logic.

---

## 2. CAP Theorem

### What Is It?

**CAP Theorem** states that a distributed system can guarantee **at most 2 of the following 3 properties** simultaneously:

```
         C — Consistency
        / \
       /   \
      /     \
     A ——————P
     Availability   Partition Tolerance
```

| Property | Meaning |
|----------|---------|
| **C — Consistency** | Every read gets the most recent write (or an error). All nodes see the same data at the same time. |
| **A — Availability** | Every request gets a response (not necessarily the latest data). The system is always up and responsive. |
| **P — Partition Tolerance** | The system keeps working even if network messages between nodes are lost or delayed (network partition). |

**Real-world analogy:**
- **Consistency** = All ATMs show your exact current balance. If you deposit at Branch A, every ATM instantly reflects it.
- **Availability** = Every ATM always responds, never says "try again later."
- **Partition Tolerance** = ATMs keep working even if the network between branches goes down.

---

### Why You Can Only Have 2 of 3

**The catch:** In any real distributed system, **network partitions WILL happen** — cables fail, routers crash, data centers lose connectivity. So **P is not really optional**.

This means the real-world choice is:

```
CP — Consistency + Partition Tolerance
  → Sacrifice: Availability
  → Behavior: When partition occurs, return error instead of stale data
  → Examples: HBase, MongoDB (strong consistency mode), Zookeeper, etcd

AP — Availability + Partition Tolerance
  → Sacrifice: Consistency (allow stale reads)
  → Behavior: When partition occurs, return possibly outdated data
  → Examples: Cassandra, CouchDB, DynamoDB, DNS

CA — Consistency + Availability (NO partition tolerance)
  → Only possible on a single node or within a single data center
  → Not truly distributed — network partitions make this impossible at scale
  → Examples: Single-node PostgreSQL (not distributed)
```

---

### CP Systems in Practice

```
Scenario: Network partition between Node A and Node B
           Node A receives a write: user.balance = $500
           Node B cannot reach Node A

CP choice: Node B REFUSES to answer reads
  → Returns error: "System unavailable — partition detected"
  → Consistency preserved: no stale data served
  → Availability sacrificed: clients get errors

Use case: Banking, financial transactions, anything where stale data = disaster
```

---

### AP Systems in Practice

```
Scenario: Network partition between Node A and Node B
           Node A receives a write: user.bio = "Software Engineer"
           Node B cannot reach Node A

AP choice: Node B ANSWERS reads with its last known data
  → Returns: user.bio = "Developer"  (stale — old value)
  → Availability preserved: clients get a response
  → Consistency sacrificed: response may be outdated

Use case: Social media feeds, DNS, product catalogs — stale data is acceptable
```

---

### PACELC Theorem (Extension of CAP)

CAP only considers behavior during partitions. **PACELC** extends it:

```
If Partition (P):
  Choose between Availability (A) vs Consistency (C)

Else (E) — normal operation, no partition:
  Choose between Latency (L) vs Consistency (C)
```

Even without partitions, there's a trade-off: strong consistency requires coordination between nodes (adds latency). Eventual consistency is faster but may serve stale data.

| System | Partition behavior | Normal behavior | Classification |
|--------|--------------------|-----------------|----------------|
| DynamoDB | AP | EL (low latency) | PA/EL |
| Cassandra | AP | EL | PA/EL |
| MongoDB | CP | EC | PC/EC |
| PostgreSQL | CA (single node) | EC | — |

---

### Choosing CP vs AP — Decision Guide

```
Choose CP (Consistency over Availability) when:
  ✓ Financial transactions (bank transfers, payments)
  ✓ Inventory management (can't oversell)
  ✓ User authentication (token validation must be current)
  ✓ Leader election in distributed systems

Choose AP (Availability over Consistency) when:
  ✓ Social media posts/feeds (OK if slightly delayed)
  ✓ Product ratings and reviews
  ✓ DNS resolution
  ✓ Shopping cart (consistency applied at checkout)
  ✓ Metrics and analytics (approximate is fine)
  ✓ User profile data (bio, avatar — stale for seconds is OK)
```

---

### Common Interview Questions — CAP Theorem

**Q: Can a system be CA (consistent and available) in a distributed setup?**
Technically no — network partitions are inevitable in distributed systems, so you must tolerate them. A CA system (like a single-node database) is not truly distributed. In practice, "CA" databases are those that run on a single node or within a tightly controlled network where partitions are near-impossible.

**Q: Is CAP theorem a limitation or a design choice?**
Both. It's a mathematical theorem proving the limitation. But how a system handles the trade-off is a deliberate design choice. Cassandra chooses AP; Zookeeper chooses CP. Neither is wrong — it depends on the use case.

**Q: How does CAP relate to ACID and BASE?**
- **ACID** (Atomicity, Consistency, Isolation, Durability) = traditional RDBMS guarantees, maps to CP
- **BASE** (Basically Available, Soft state, Eventually consistent) = NoSQL approach, maps to AP
- CAP formalizes WHY you can't have both ACID guarantees AND high availability in a distributed system

---

## 3. Eventual Consistency

### What Is It?

**Easy Explanation:** Eventual consistency means that if no new updates are made to a piece of data, eventually (after some time), all nodes in the distributed system will return the same value. There's no guarantee of *when*, but it will happen.

**Real-world analogy:** DNS propagation. When you change a domain's IP address, it doesn't update everywhere instantly. For hours (or up to 48 hours), some users get the old server, others get the new one. But *eventually*, all DNS servers worldwide reflect the update.

---

### Strong Consistency vs Eventual Consistency

```
Strong Consistency:
  Write → All nodes updated → Read returns new value
  [Write: X=5] → [Node A: X=5] [Node B: X=5] [Node C: X=5]
  Time delay: write is slow (must confirm all nodes)
  Guarantee: Every read always returns the latest write

Eventual Consistency:
  Write → Primary updated → Replicas updated asynchronously
  [Write: X=5] → [Node A: X=5]
                  [Node B: X=3] ← still old value (replication lag)
                  [Node C: X=3] ← still old value
  ...time passes...
  [Node A: X=5] [Node B: X=5] [Node C: X=5]  ← all converged
  Time delay: write is fast (only confirm primary)
  Guarantee: Reads will *eventually* return the latest write
```

---

### Consistency Models (Spectrum)

From strongest to weakest:

```
STRONG CONSISTENCY
    │  Linearizability — reads/writes appear instantaneous and globally ordered
    │  Sequential Consistency — all nodes see operations in same order
    │  Causal Consistency — causally related writes are seen in order
    │  Read Your Own Writes — you always see your own updates
    │  Monotonic Reads — once you see a value, you won't see an older one
    │  Monotonic Writes — your writes are applied in order
EVENTUAL CONSISTENCY
```

| Model | Description | Example |
|-------|-------------|---------|
| **Linearizable** | Strongest — system behaves as single node | etcd, Zookeeper |
| **Sequential** | All nodes see same order, may lag | Some distributed databases |
| **Causal** | If A causes B, everyone sees A before B | MongoDB causal sessions |
| **Read-your-writes** | You see your own writes immediately | Most web apps need this |
| **Eventual** | Weakest — will converge, no time guarantee | Cassandra, DynamoDB default |

---

### How Eventual Consistency Works — The Mechanism

```
System: 3 replicas of a user profile database

Step 1: Client writes to primary node
  Client → [Primary Node] write: {name: "Alice", city: "London"}
  Primary: ACK → Client immediately (write succeeded)

Step 2: Primary replicates asynchronously to replicas
  [Primary] ---async--→ [Replica 1] (may take 10ms - 500ms)
             ---async--→ [Replica 2] (may take 10ms - 500ms)

Step 3: During replication lag, reads from replicas are stale
  Client reads from Replica 1 → gets old data {city: "Dublin"}
  This is called a "dirty read" or "stale read"

Step 4: Replication completes — system is consistent
  [Replica 1]: {city: "London"}
  [Replica 2]: {city: "London"}
  All future reads return "London"
```

---

### Conflict Resolution in Eventual Consistency

When two nodes accept writes for the same data simultaneously (during a partition):

#### Last Write Wins (LWW)
```
Node A at 10:00:01.100: user.name = "Alice"
Node B at 10:00:01.200: user.name = "Alicia"  ← higher timestamp wins

Result: user.name = "Alicia"
Problem: Clock skew — servers' clocks are not perfectly synchronized
```

#### Vector Clocks
Each write carries a version vector tracking causality:

```
Initial: {node_a: 0, node_b: 0}
Node A writes: {node_a: 1, node_b: 0} → value: "Alice"
Node B writes: {node_a: 0, node_b: 1} → value: "Alicia"

These are concurrent writes (neither caused the other)
System detects conflict → raises it for manual or application-level resolution
DynamoDB and Riak use this approach
```

#### Multi-Version Concurrency (MVCC)
Keep multiple versions; application or merge function resolves conflicts.

#### CRDT (Conflict-free Replicated Data Types)
Mathematical data structures that always converge without conflicts:

```
Example: G-Counter (grow-only counter)
  Node A increments: [A:3, B:0]
  Node B increments: [A:0, B:2]
  Merge: take max of each → [A:3, B:2] → total = 5

Used in: collaborative editing (Google Docs), distributed counters
```

---

### Read Repair and Anti-Entropy

Systems maintain consistency over time via background processes:

**Read Repair:** When a read detects inconsistency between replicas, it repairs the stale replica.
```
Client reads → asks all 3 replicas
  Replica 1: X=5 (latest)
  Replica 2: X=3 (stale)
  Replica 3: X=5 (latest)
System returns X=5 to client AND repairs Replica 2 → X=5
```

**Anti-Entropy:** Background process continuously compares replicas and syncs differences using a Merkle tree to efficiently find diverging data.

---

### Common Interview Questions — Eventual Consistency

**Q: What is the difference between eventual consistency and strong consistency?**
Strong consistency guarantees every read returns the most recent write — all nodes are always in sync. Eventual consistency only guarantees that nodes will converge to the same value *eventually* — reads may return stale data during the convergence window. Strong consistency is slower (requires coordination); eventual is faster (allows async replication).

**Q: Is eventual consistency acceptable for a banking application?**
Generally no for account balances and transactions — you cannot allow a user to see a stale balance and transfer money based on it. However, some parts of banking are eventually consistent — transaction history displays, statement generation, or notification delivery can tolerate slight delays.

**Q: What is "read-your-own-writes" consistency?**
A guarantee that after you write data, your subsequent reads will always return that write. This is weaker than full consistency but critical for user experience — you update your profile picture, and you see the new picture immediately. Achieved by routing your reads to the same replica you wrote to, or using sticky sessions.

---

## 4. Distributed Locks

### What Is It?

**Easy Explanation:** A distributed lock is a mechanism that ensures only one server (or process) can perform a critical operation at a time, across an entire distributed system.

**Real-world analogy:** A bathroom key at a restaurant. Only one person can have the key at a time. Everyone else must wait. The key is the "distributed lock."

```
Without distributed lock:
  Server 1: reads stock = 1, proceeds to sell
  Server 2: reads stock = 1, proceeds to sell  ← BOTH think 1 item available!
  Result: oversold by 1 item — data corruption

With distributed lock:
  Server 1: acquires lock, reads stock = 1, sells, updates stock = 0, releases lock
  Server 2: tries to acquire lock → BLOCKED until Server 1 releases
  Server 2: acquires lock, reads stock = 0, rejects sale, releases lock
  Result: no overselling
```

---

### Why Distributed Locks Are Hard

In a single machine, a mutex/synchronized block works. In distributed systems:

1. **Network failures** — the server holding the lock may crash before releasing it
2. **Clock skew** — servers have slightly different clocks, making TTL unreliable
3. **GC pauses** — a Java/Go process can pause for seconds (GC), outliving its lock TTL
4. **Split-brain** — two servers both think they hold the lock

---

### Implementing Distributed Locks

#### Option 1: Redis-based Lock (SETNX)

```
SETNX = SET if Not eXists

Acquire lock:
  SET lock_key "server1_uuid" NX PX 30000
  NX = only set if key doesn't exist
  PX 30000 = expire in 30 seconds (TTL — prevents deadlock if server crashes)

If SET succeeds → lock acquired
If SET fails → lock held by someone else → retry or fail

Release lock (Lua script for atomicity):
  if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
  else
    return 0
  end
```

**Why the Lua script?** You must check that YOU own the lock before deleting it. Without atomicity, this race condition exists:
```
Server 1: GET lock_key → "server1"  ← checks ownership
  [30s TTL expires here — lock auto-deleted]
  [Server 2 acquires the lock]
Server 1: DEL lock_key              ← deletes Server 2's lock!
```

#### Option 2: Redlock Algorithm (Multi-node Redis)

Single Redis node has a single point of failure. Redlock uses **N (usually 5) independent Redis nodes**:

```
To acquire lock:
  1. Record current time T1
  2. Try to SET lock on all 5 Redis nodes simultaneously
  3. Count how many succeeded (say 3 out of 5)
  4. Lock is acquired if:
     - Majority acquired (>= 3 out of 5)
     - Time taken < TTL (still valid)
  5. Actual TTL = original_TTL - time_elapsed

To release lock:
  Send DEL to all 5 Redis nodes
```

```
Node 1: ✓ acquired
Node 2: ✓ acquired
Node 3: ✓ acquired  ← quorum reached (3/5)
Node 4: ✗ failed (network issue)
Node 5: ✗ failed (crashed)

Lock is valid — majority obtained
```

**Controversy:** Martin Kleppmann argued Redlock is not safe due to GC pauses and clock drift. Use Zookeeper or etcd for safety-critical locks.

#### Option 3: Zookeeper (ZK) Locks

Zookeeper is a distributed coordination service designed specifically for this use case:

```
Lock acquisition using ephemeral sequential znodes:
  1. Create /locks/lock-0000000001 (ephemeral sequential node)
  2. List all children of /locks
  3. If your node is the lowest-numbered → you have the lock
  4. If not → watch the next-lowest node for deletion
  5. When that node is deleted → re-check if you're now lowest

Lock release:
  Delete your ephemeral node
  Zookeeper automatically deletes ephemeral nodes if session ends (crash protection)
```

**Advantage:** Handles server crashes automatically — ephemeral nodes disappear when the client disconnects.

#### Option 4: Database-based Locks

Using a relational database:

```sql
-- Pessimistic locking
SELECT * FROM inventory WHERE product_id = 1 FOR UPDATE;
-- Row is locked until transaction commits/rolls back

-- Optimistic locking (no actual lock, uses version check)
UPDATE inventory
SET stock = stock - 1, version = version + 1
WHERE product_id = 1 AND version = 5;
-- If 0 rows updated → someone else changed it → retry
```

---

### Lock Timeout and Fencing Tokens

**The GC pause problem:**

```
Server 1 acquires lock with 30s TTL
Server 1 enters GC pause for 35 seconds
  → Lock TTL expires
  → Server 2 acquires the lock
Server 1 wakes up from GC
  → Thinks it still holds the lock
  → BOTH Server 1 and Server 2 are executing the critical section!
```

**Fencing tokens** solve this:

```
Lock server issues monotonically increasing token with each lock grant:
  Server 1 acquires lock → receives token 33
  Lock expires
  Server 2 acquires lock → receives token 34

Server 1 sends write request to storage with token 33
Server 2 sends write request to storage with token 34
Storage rejects token 33 (already seen token 34 — Server 1's token is stale)
```

---

### Common Interview Questions — Distributed Locks

**Q: What is a deadlock in distributed systems and how do you prevent it?**
A deadlock occurs when two processes each hold a lock the other needs, causing both to wait forever. Prevention strategies: always acquire locks in a consistent order, use TTLs (automatic expiry), use tryLock with timeout instead of blocking forever, and use deadlock detection with timeouts.

**Q: Why not just use a database row lock as a distributed lock?**
Database row locks (`SELECT FOR UPDATE`) work but have limitations: higher latency than Redis, the database becomes a bottleneck, they don't work across multiple databases, and there's no built-in TTL protection against crashed clients holding locks. Redis and Zookeeper are purpose-built for coordination and are faster.

**Q: What is the difference between a distributed lock and a semaphore?**
A lock (mutex) allows only 1 holder at a time. A distributed semaphore allows N holders simultaneously. Example: rate limiting to 10 concurrent operations uses a semaphore with count 10; protecting a single critical section uses a mutex.

---

## 5. Database Sharding

### What Is It?

**Easy Explanation:** Sharding is splitting a large database into smaller, independent pieces called shards. Each shard holds a subset of the total data, distributed across multiple servers.

**Real-world analogy:** An encyclopedia set. Instead of one massive book, it's split into 26 volumes (A-B, C-D, ... Y-Z). Each volume is smaller and faster to search. Sharding does the same for database tables.

```
Without sharding (single database):
  [All 1 billion users in one DB] → slow queries, storage limit hit

With sharding:
  [Users A-D → Shard 1 DB]
  [Users E-K → Shard 2 DB]
  [Users L-R → Shard 3 DB]
  [Users S-Z → Shard 4 DB]
  → Each shard has ~250M users → faster queries, distributed storage
```

---

### Sharding vs Partitioning

| Concept | Definition |
|---------|-----------|
| **Partitioning** | Splitting data within a single database (logical split) |
| **Sharding** | Splitting data across multiple databases/servers (physical split) |

Sharding is a form of horizontal partitioning across separate database instances.

---

### Sharding Strategies

#### 1. Range-Based Sharding

Data is split by value ranges of the shard key:

```
User ID 1–1,000,000       → Shard 1
User ID 1,000,001–2,000,000 → Shard 2
User ID 2,000,001–3,000,000 → Shard 3

Orders by date:
  January 2024   → Shard 1
  February 2024  → Shard 2
  March 2024     → Shard 3
```

**Pros:** Range queries are efficient (find all orders in January → goes to one shard). Easy to add new ranges.

**Cons:** **Hotspots** — new users always go to the latest shard, creating uneven load. Recently created content always hits one shard.

#### 2. Hash-Based Sharding

Apply a hash function to the shard key to determine which shard:

```
shard_number = hash(user_id) % total_shards

user_id = 12345 → hash(12345) = 89273 → 89273 % 4 = 1 → Shard 1
user_id = 12346 → hash(12346) = 23891 → 23891 % 4 = 3 → Shard 3
user_id = 12347 → hash(12347) = 47201 → 47201 % 4 = 0 → Shard 0
```

**Pros:** Even data distribution, eliminates hotspots.

**Cons:** Range queries are impossible (users 1–1000 scattered across all shards). **Resharding problem** — if you add a new shard, almost all data needs to move (change from `% 4` to `% 5`).

#### 3. Consistent Hashing

Solves the resharding problem of hash-based sharding:

```
Imagine a ring (0 to 2^32):
  Shard A placed at position 100
  Shard B placed at position 200
  Shard C placed at position 300
  Shard D placed at position 400

Data key hashed to a position on the ring
Data goes to the NEXT shard clockwise from its position:
  key at 150 → Shard B (next clockwise from 150 is 200)
  key at 250 → Shard C
  key at 350 → Shard D

Adding Shard E at position 250:
  Only keys between 200-250 (previously going to C) now go to E
  All other keys unaffected
  Minimizes data movement
```

**Used by:** Amazon DynamoDB, Apache Cassandra, Memcached, Riak

#### 4. Directory-Based Sharding

A lookup table (directory service) maps each data item (or range) to a specific shard:

```
Directory service:
  user_id 1–500    → Shard 1
  user_id 501–900  → Shard 3
  user_id 901–1500 → Shard 2

Look up directory → route to correct shard
```

**Pros:** Most flexible — can move data between shards by updating the directory without rehashing.

**Cons:** Directory service becomes a bottleneck and single point of failure. Must be cached aggressively.

---

### Choosing a Shard Key

The shard key is the most critical decision in sharding. A bad shard key creates hotspots.

**Good shard key qualities:**
```
✓ High cardinality       — many distinct values (user_id is good; boolean is bad)
✓ Even distribution      — values spread uniformly (UUIDs are good; sequential IDs cause hotspots)
✓ Immutable              — once set, never changes (user_id is good; email can change)
✓ Low cross-shard joins  — related data on same shard (user + their orders → user_id as key)
✓ Query patterns match   — most queries filter by this key
```

**Bad shard key examples:**
```
✗ created_at (timestamp): all new data hits one shard — hotspot
✗ country_code: popular countries (US, UK) become hotspots
✗ boolean status: only 2 values → only 2 shards can be used effectively
```

---

### Challenges of Sharding

#### Cross-Shard Joins
```
"Find all orders for premium users"
  Users table: sharded by user_id across 4 shards
  Orders table: sharded by order_id across 4 shards

  → Must query ALL 4 user shards to find premium users
  → Then query ALL 4 order shards per user
  → Massive scatter-gather operation

Solutions:
  1. Denormalize — store user tier in the orders table
  2. Co-locate data — shard both users and orders by user_id
  3. Use a separate analytics DB (e.g., data warehouse) for complex queries
```

#### Rebalancing (Resharding)
When shards become uneven, data must be moved:

```
Shard 1: 80% full  ← hotspot
Shard 2: 20% full
Shard 3: 50% full

Rebalance: move some of Shard 1's data to Shard 2
  Challenge: must do this live without downtime
  Solution: double-write during migration, verify, then switch
```

#### Distributed Transactions (Cross-Shard)
```
Transfer $100 from User A (Shard 1) to User B (Shard 3)
  Must atomically debit Shard 1 AND credit Shard 3
  Cross-shard transactions require 2-Phase Commit (2PC) or SAGA pattern
  2PC is slow and complex → many sharded DBs avoid cross-shard transactions by design
```

---

### Common Interview Questions — Database Sharding

**Q: When should you shard a database?**
Shard when: single-node database can't handle the write throughput, data size exceeds what fits on one server, or query latency is unacceptable. Sharding adds significant complexity — exhaust vertical scaling, caching, and read replicas first.

**Q: What is the resharding problem and how does consistent hashing solve it?**
With `hash(key) % N` sharding, adding a server changes N, which remaps almost all keys. Consistent hashing places servers on a ring so adding/removing a server only affects keys in its neighborhood — typically 1/N of the data moves instead of nearly all of it.

**Q: How do you handle transactions across shards?**
Options: (1) **Avoid them** — design data model so related data is co-located on the same shard. (2) **SAGA pattern** — break into local transactions with compensating transactions on failure. (3) **Two-phase commit (2PC)** — distributed transaction protocol, but slow and blocks. (4) **Eventual consistency** — accept that cross-shard operations are eventually consistent.

---

## 6. Replication

### What Is It?

**Easy Explanation:** Replication means keeping copies of your data on multiple servers. If one copy is lost or unavailable, others are intact. It also allows multiple servers to serve read requests simultaneously.

**Real-world analogy:** A book that exists in 10 libraries. Even if one library burns down, the book survives. And 10 people can read the book simultaneously (one in each library).

---

### Why Replicate?

| Goal | How Replication Helps |
|------|----------------------|
| **High Availability** | If primary fails, promote a replica — no data loss |
| **Fault Tolerance** | Survive hardware failures, data center outages |
| **Read Scalability** | Spread read load across multiple replicas |
| **Disaster Recovery** | Geographic replicas survive regional outages |
| **Low Latency** | Serve users from geographically closest replica |

---

### Replication Topologies

#### 1. Primary-Replica (Master-Slave) Replication

```
         Writes
           ↓
      [Primary Node]
           │
    ┌──────┼──────┐
    ↓      ↓      ↓
[Replica1][Replica2][Replica3]
    ↑      ↑      ↑
         Reads
```

- **Primary** handles all writes
- **Replicas** handle reads (read scaling)
- Replicas receive write logs from primary and replay them
- **Replication lag** — replicas may be slightly behind the primary

**Use cases:** MySQL, PostgreSQL, MongoDB

#### 2. Primary-Primary (Multi-Master) Replication

```
Writes from App 1       Writes from App 2
       ↓                       ↓
  [Primary A] ←──sync──→ [Primary B]
```

- Both nodes accept writes simultaneously
- Changes are replicated bidirectionally
- **Conflict resolution required** — what if both nodes update the same row simultaneously?

**Conflict strategies:** Last-write-wins, application-level resolution, CRDTs

**Use cases:** MySQL Group Replication, Galera Cluster, CockroachDB, active-active geodistribution

#### 3. Chain Replication

```
[Head Node] → [Middle 1] → [Middle 2] → [Tail Node]
    ↑                                        ↑
  Writes go here                     Reads served here
```

- Writes enter at Head, propagate through the chain
- Only the Tail node serves reads (guaranteed fully replicated data)
- Tail acks client only after write propagates entire chain
- **Strong consistency guarantee** — reads are always up-to-date
- **Used by:** Microsoft Azure Storage, some HBase configurations

---

### Synchronous vs Asynchronous Replication

| | Synchronous | Asynchronous |
|-|-------------|--------------|
| **Write confirms when** | ALL replicas confirm receipt | Primary writes locally, replicas catch up later |
| **Data durability** | High — no data loss if primary fails | Risk of losing recent writes if primary crashes before replicating |
| **Write latency** | Higher (waits for slowest replica) | Lower (doesn't wait) |
| **Availability** | Lower (blocked if replica is slow/down) | Higher |
| **Replication lag** | None | Possible (milliseconds to seconds) |

**Semi-synchronous replication** (MySQL): Write confirmed after at least 1 replica acknowledges — balance between safety and performance.

```
Synchronous:
  Client → Primary → [wait] → Replica 1 confirmed
                            → Replica 2 confirmed
  → ACK to client
  Latency: ~5-50ms extra per write

Asynchronous:
  Client → Primary → ACK to client immediately
                   ↓ (async, later)
             Replica 1 updated
             Replica 2 updated
  Latency: No extra latency
  Risk: If primary crashes before async replication → data loss
```

---

### Replication Lag and Its Consequences

Replication lag is the delay between a write on the primary and when it appears on replicas.

```
Scenario: User changes their email on primary
  t=0ms:  Primary updated: email=new@email.com
  t=0ms:  Application redirects user to profile page
  t=50ms: Profile page reads from Replica 1
  t=50ms: Replica 1 still has: email=old@email.com  ← STALE READ

User sees: "Why is my old email still showing?"
```

**Solutions to replication lag issues:**

```
1. Read-your-own-writes consistency:
   → Route the user's reads to the primary (for a short period after a write)
   → Or track "last write timestamp" per user and wait for replica to catch up

2. Monotonic reads:
   → Ensure a user always reads from the same replica
   → Avoids the phenomenon of seeing newer data and then older data (time goes backward)
   → Implementation: sticky routing by user ID to a specific replica

3. Consistent reads on important data:
   → For reads that MUST be current (balance, inventory), always read from primary
   → Accept the replica lag only for non-critical reads (feeds, history)
```

---

### Failover

When the primary fails, a replica must be promoted to become the new primary:

```
Normal state:
  [Primary] → [Replica A] [Replica B]

Primary fails:
  [FAILED Primary]   [Replica A] [Replica B]

Failover:
  1. Detect failure (health check timeout)
  2. Elect new primary (usually most up-to-date replica)
  3. Promote Replica A → new Primary
  4. Update Replica B to follow new Primary
  5. Update application connection strings / DNS

  [New Primary (was Replica A)] → [Replica B]
```

**Failover challenges:**

| Challenge | Description |
|-----------|-------------|
| **Split-brain** | Both old and new primary think they're primary — data diverges |
| **Data loss** | Async replicas may not have the latest writes from the failed primary |
| **False positives** | Network hiccup detected as failure — unnecessary failover |
| **Application reconnection** | Apps must discover new primary address |

**Solutions:** Quorum-based election (Raft, Paxos), fencing tokens to prevent split-brain, semi-sync replication to reduce data loss risk.

---

### Replication in Practice — Examples

**PostgreSQL Streaming Replication:**
```
Primary → streams WAL (Write-Ahead Log) → Replica(s)
Replica replays the WAL → stays in sync
Synchronous_standby_names setting controls sync vs async per replica
```

**MySQL GTID Replication:**
```
Each transaction gets a Global Transaction ID (GTID)
Replica tracks which GTIDs it has executed
If replica falls behind → it knows exactly what to replay
GTID makes failover simpler — new replica can sync from any point
```

**MongoDB Replica Sets:**
```
One primary, multiple secondaries
Automatic election using Raft protocol
readPreference: allows routing reads to secondaries
writeConcern: w:majority ensures write is on majority before acknowledging
```

---

### Common Interview Questions — Replication

**Q: What is the difference between replication and backup?**
Replication is real-time copying to live servers — used for high availability and read scaling. Backups are point-in-time snapshots stored offline — used for data recovery from logical errors (accidental deletes, corruption). You need BOTH: replication doesn't protect against "DELETE all rows" mistakes; backups don't give you instant failover.

**Q: How do you handle split-brain in a primary-replica setup?**
Split-brain occurs when a network partition makes both primary and replica think they're the primary. Prevention: use a quorum — require majority of nodes to agree on who is primary (Raft/Paxos). Use fencing tokens to ensure the old primary can't write after being fenced off. Tools like Patroni (PostgreSQL) and MHA (MySQL) handle this.

**Q: What is a read replica and when would you use it?**
A read replica is a copy of the database that handles SELECT queries, offloading reads from the primary. Use when read traffic is significantly higher than write traffic (common in most apps). Caveat: reads may be slightly stale due to replication lag.

---

## 7. Message Queues

### What Is It?

**Easy Explanation:** A message queue is a component that sits between services, letting them communicate asynchronously. Service A sends a message to the queue and continues working. Service B reads from the queue when it's ready.

**Real-world analogy:** Email. You send an email and continue your work. The recipient doesn't need to be available at that exact moment. They process it when they're ready. The email server is the "queue."

```
Without message queue (synchronous, tight coupling):
  Order Service → Payment Service (waits for response)
                → Inventory Service (waits for response)
                → Notification Service (waits for response)
  Problem: If any service is slow or down → entire flow fails or slows

With message queue (asynchronous, loose coupling):
  Order Service → [Message Queue] → Payment Service (processes when ready)
                                  → Inventory Service (processes when ready)
                                  → Notification Service (processes when ready)
  Order Service: no waiting, publishes and moves on
```

---

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Producer** | Service that sends/publishes messages to the queue |
| **Consumer** | Service that reads/subscribes to messages from the queue |
| **Queue** | Ordered buffer that stores messages until consumed |
| **Topic** | Named channel (in pub/sub systems like Kafka) |
| **Message** | The data payload (JSON, bytes, Protobuf, Avro) |
| **Broker** | The message queue server (RabbitMQ, Kafka broker) |
| **Acknowledgment (ACK)** | Consumer signals it processed the message successfully |
| **Dead Letter Queue (DLQ)** | Where failed/unprocessable messages are sent |

---

### Queue Models

#### Point-to-Point (Queue)
One message is consumed by exactly one consumer:

```
Producer → [Queue] → Consumer A
                   ← (message removed from queue after consumption)

Use case: Task distribution — 10 workers, each picks the next available job
Example: RabbitMQ queues, AWS SQS
```

#### Publish-Subscribe (Topic)
One message is delivered to ALL subscribers:

```
Publisher → [Topic: order.created] → Consumer Group A (Payment Service)
                                    → Consumer Group B (Inventory Service)
                                    → Consumer Group C (Notification Service)

All three receive the same message
Use case: Event broadcasting — one event triggers multiple independent reactions
Example: Kafka topics, AWS SNS
```

---

### Why Use Message Queues?

#### 1. Decoupling
```
Without queue: Order Service directly calls Payment Service API
  → If Payment Service is down → Order Service fails
  → If Payment Service is slow → Order Service is slow

With queue: Order Service publishes to queue, doesn't know/care who processes
  → Payment Service can be down; messages accumulate, processed on restart
  → Services are independently deployable and scalable
```

#### 2. Load Leveling (Buffer)
```
Flash sale: 100,000 orders in 10 seconds
  Without queue: 100,000 simultaneous requests hit Payment Service → crashes

  With queue:
    Orders flood in → [Queue] ← absorbs the spike
    Payment Service: processes at its steady rate of 1,000/second
    Queue acts as a buffer, smoothing out the spike
    All orders eventually processed, no crash
```

#### 3. Reliability / At-Least-Once Delivery
```
Without queue (direct HTTP call):
  Order Service → POST /payment → network failure → payment may or may not have happened

With queue:
  Message stays in queue until consumer ACKs
  If consumer crashes mid-processing → message returned to queue → redelivered
  Guarantee: message is processed at least once
```

#### 4. Asynchronous Processing
```
User uploads a video → immediately gets "Upload successful"
  Queue message: {video_id: 123, action: "transcode"}
  Transcoding Service: picks up job, processes over next 5 minutes
  User is not waiting — they were immediately freed
```

---

### Message Delivery Guarantees

| Guarantee | Description | Risk |
|-----------|-------------|------|
| **At Most Once** | Message delivered 0 or 1 times — never duplicated | Messages can be LOST |
| **At Least Once** | Message delivered 1 or more times — never lost | Messages can be DUPLICATED |
| **Exactly Once** | Message delivered exactly 1 time — no loss, no duplicates | Hardest to implement — expensive |

```
At Least Once (most common):
  Producer sends message
  Consumer receives, processes, crashes before ACKing
  Broker redelivers message
  Consumer processes it again → DUPLICATE

To handle duplicates → make consumers IDEMPOTENT:
  "Apply payment of $50 for order_id 12345"
  → Check: has order 12345 already been paid?
  → If yes, skip (safe to process twice)
  → If no, process
```

---

### Dead Letter Queue (DLQ)

```
Normal flow:
  [Queue] → Consumer → ACK → message deleted

When processing fails:
  [Queue] → Consumer → processing error → NACK
  Broker: retries (e.g., 3 times)
  After max retries: message moved to [Dead Letter Queue]

DLQ allows:
  → Inspect why messages are failing
  → Fix the bug
  → Replay DLQ messages
  → Alert on DLQ size (indicates systemic issues)
```

---

### Kafka vs RabbitMQ

| Feature | Apache Kafka | RabbitMQ |
|---------|-------------|----------|
| **Model** | Distributed log (pub/sub) | Traditional message broker (queue + pub/sub) |
| **Message retention** | Configurable (days/weeks/forever) | Deleted after consumption (by default) |
| **Ordering** | Per-partition ordering guaranteed | Per-queue ordering guaranteed |
| **Throughput** | Extremely high (millions/sec) | High (tens of thousands/sec) |
| **Consumer groups** | Multiple groups each consume all messages | Competing consumers share one queue |
| **Replay** | Yes — consumers track their own offset | No — consumed messages are gone |
| **Use case** | Event streaming, audit logs, data pipelines | Task queues, RPC, microservice messaging |
| **Complexity** | Higher (Zookeeper/KRaft, partitions) | Lower (easier setup) |
| **Protocol** | Custom binary (Kafka protocol) | AMQP, STOMP, MQTT |
| **Message size** | Optimized for small-medium messages | Handles any size |

---

### Kafka Deep Dive

```
Topics and Partitions:
  Topic "orders" → 4 partitions
    Partition 0: [msg1, msg2, msg3, ...]  → consumed by Consumer A
    Partition 1: [msg4, msg5, msg6, ...]  → consumed by Consumer B
    Partition 2: [msg7, msg8, msg9, ...]  → consumed by Consumer C
    Partition 3: [msg10, ...]             → consumed by Consumer D

Consumer Groups:
  Group "payment-service": each partition consumed by one consumer
  Group "inventory-service": same messages, independent consumption

Offsets:
  Each consumer tracks its own offset (position in the partition)
  Consumer reads partition 0, offset 150 → next read from offset 151
  Consumer can replay by resetting offset to 0

Retention:
  Messages kept for 7 days (configurable) regardless of consumption
  This enables: replay, audit trail, multiple consumers at different speeds
```

---

### Common Patterns with Message Queues

#### Saga Pattern (Distributed Transactions)
```
Choreography-based Saga:
  Order Service: publishes "order.created"
  Payment Service: consumes → processes payment → publishes "payment.completed" or "payment.failed"
  Inventory Service: consumes "payment.completed" → reserves stock → publishes "stock.reserved"
  Shipping Service: consumes "stock.reserved" → creates shipment

Compensating transactions on failure:
  Inventory fails → publishes "stock.failed"
  Payment Service: consumes → refunds payment → publishes "payment.reversed"
  Order Service: consumes → marks order as failed
```

#### Fan-Out Pattern
```
One event → multiple independent consumers (all processing in parallel)
  "user.registered" event → Welcome Email Service
                          → CRM Service
                          → Analytics Service
                          → Fraud Detection Service
```

#### Work Queue (Task Distribution)
```
10 worker instances competing for tasks:
  [Queue: image-resize-jobs] ← producers add jobs
  Worker 1: picks job, processes, ACKs
  Worker 2: picks next available job, processes, ACKs
  ...
  Auto-scaling: queue depth grows → scale up workers
                queue depth shrinks → scale down workers
```

---

### Common Interview Questions — Message Queues

**Q: What is the difference between a message queue and an event bus?**
A message queue (like RabbitMQ) is pull-based — consumers pull messages when ready. An event bus (like Kafka) is publish-subscribe — events are broadcast to all interested subscribers. Message queues focus on task distribution (one consumer per message); event buses focus on event propagation (many consumers per event). The line is blurry — Kafka can do both, RabbitMQ supports pub/sub via exchanges.

**Q: How do you ensure messages are not lost if a consumer crashes?**
Use acknowledgments (ACKs). The broker keeps the message until the consumer sends a positive ACK. If the consumer crashes (no ACK), the broker redelivers the message to another consumer. Combined with persistent message storage (disk-backed queues), messages survive broker restarts too.

**Q: What is idempotency and why is it important with message queues?**
Idempotency means an operation can be applied multiple times with the same result as applying it once. Critical with "at-least-once" delivery — consumers may receive the same message more than once (retries, redeliveries). An idempotent consumer handles duplicates safely. Implementation: use unique message IDs and record "processed message IDs" in a store; skip if already processed.

**Q: How does Kafka guarantee ordering?**
Kafka guarantees ordering within a single partition. Messages with the same partition key always go to the same partition, maintaining relative order. Across partitions, there's no ordering guarantee. Design your partition key around what needs to be ordered (e.g., use `user_id` as key so all events for a user go to the same partition in order).

**Q: When would you choose RabbitMQ over Kafka?**
RabbitMQ: when you need traditional message routing (topic exchanges, fanout, direct routing with rules), request-reply patterns, per-message TTL, or simpler setup. Kafka: when you need high throughput, message replay, audit logging, stream processing, or when multiple independent services need to consume the same events.

---

## How These Topics Connect

Understanding how these 7 topics relate is critical for system design interviews:

```
                        ┌─────────────────────────────┐
                        │        Your Application      │
                        └──────────────┬──────────────┘
                                       │
                                       ↓
                           ┌───────────────────────┐
                           │     LOAD BALANCER      │ ← distributes traffic
                           └──────────┬────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ↓                 ↓                  ↓
               [Server 1]        [Server 2]         [Server 3]
                    │                 │                  │
                    └─────────────────┼──────────────────┘
                                      │
              ┌───────────────────────┼──────────────────────┐
              ↓                       ↓                      ↓
    ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
    │  DATABASE        │   │  MESSAGE QUEUE   │   │  CACHE (Redis)   │
    │  (sharded +      │   │  (Kafka/Rabbit)  │   │  (distributed    │
    │   replicated)    │   │                  │   │   locks too)     │
    └──────────────────┘   └──────────────────┘   └──────────────────┘
             │
    ┌────────┴────────┐
    │                 │
  Shard 1           Shard 2        ← DATABASE SHARDING
    │                 │
  Primary          Primary
    │                 │
  Replica          Replica         ← REPLICATION
```

**CAP Theorem governs every storage decision:**
- Sharded DB + async replication = AP (available but eventually consistent)
- Zookeeper (for distributed locks) = CP (consistent but may be unavailable during partition)

**Eventual Consistency is the outcome of async replication:**
- Replicas lag behind primary = eventual consistency window
- Message queues create eventual consistency between services

**Distributed Locks protect against race conditions when:**
- Multiple servers (behind load balancer) access the same shared resource
- Cross-shard operations need serialization

**Message Queues enable decoupling when:**
- Services can't afford synchronous coupling
- Load needs to be buffered (load leveling)
- Multiple consumers need the same event (fan-out)

---

## Quick Revision Cheat Sheet

### Load Balancing
```
Algorithms:  Round Robin | Weighted | Least Connections | IP Hash | Least Response Time
Layers:      L4 (TCP/IP — fast) | L7 (HTTP — smart routing)
Sticky:      IP Hash or cookie → same server (avoid: use Redis sessions instead)
Health:      Continuous health checks → auto-remove failed servers
HA LB:       Active-passive pair to avoid LB being a SPOF
```

### CAP Theorem
```
C = every read returns latest write
A = system always responds
P = works despite network partitions

P is non-negotiable → real choice is CP vs AP
CP: strong consistency, may reject requests during partition (Zookeeper, etcd, HBase)
AP: always responds, may return stale data (Cassandra, DynamoDB, DNS)
ACID ≈ CP | BASE ≈ AP
```

### Eventual Consistency
```
Guarantee: all nodes converge to same value — eventually
Cause: async replication introduces lag
Conflict resolution: Last Write Wins | Vector Clocks | CRDTs
Models: Linearizable > Sequential > Causal > Read-your-writes > Eventual
Read repair: fix stale replicas on detected inconsistency
```

### Distributed Locks
```
Redis SETNX:  SET key value NX PX ttl_ms (atomic, with TTL)
              Release with Lua script (check-then-delete atomically)
Redlock:      Acquire on majority of N Redis nodes (fault tolerant)
Zookeeper:    Ephemeral sequential znodes (auto-released on crash)
Fencing token: Monotonic counter prevents stale lock holders from writing
Idempotency:  Critical for lock-protected operations (handle duplicate execution)
```

### Database Sharding
```
Strategies:   Range (easy range queries, hotspots) | Hash (even dist, no range) 
              | Consistent Hashing (minimal resharding) | Directory (most flexible)
Shard key:    High cardinality + immutable + even distribution + matches query pattern
Challenges:   Cross-shard joins | Rebalancing | Distributed transactions
When to shard: Exhaust vertical scaling + caching + read replicas first
```

### Replication
```
Topologies:   Primary-Replica (read scaling) | Primary-Primary (write scaling, conflicts)
              | Chain (strong consistency reads)
Sync vs Async: Sync = no data loss, higher latency | Async = possible data loss, low latency
Replication lag: stale reads problem → read-your-writes, monotonic reads
Failover:     Detect → elect new primary → promote → update connections
Avoid:        Split-brain (quorum-based election + fencing tokens)
```

### Message Queues
```
Models:       Point-to-Point (1 consumer) | Pub/Sub (all consumers)
Guarantees:   At-most-once | At-least-once (common) | Exactly-once (hardest)
Idempotency:  Required with at-least-once (handle duplicate messages safely)
DLQ:          Store failed messages for inspection + replay
Kafka:        Distributed log, message replay, high throughput, partitioned ordering
RabbitMQ:     Routing rules, task queues, simpler setup, AMQP protocol
Use for:      Decoupling | Load leveling | Async processing | Fan-out | Saga pattern
```

---

*Last updated: 2026-06-05 | Focus: Backend, Cloud, and System Design Interviews*
