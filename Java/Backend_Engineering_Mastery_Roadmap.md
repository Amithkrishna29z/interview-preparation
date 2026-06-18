# Backend Engineering Mastery Roadmap – Study Guide

> **Target Audience:** Java/Spring Boot developers preparing for junior backend roles.  
> **How to Use:** Study each phase in order. For each topic, understand the *why* before the *how*.

---

## Table of Contents

- [Phase 1 – SQL + PostgreSQL Internals + Redis](#phase-1)
- [Phase 2 – Kafka + Event-Driven Systems + Messaging Patterns](#phase-2)
- [Phase 3 – System Design Fundamentals + HLD + LLD](#phase-3)
- [Phase 4 – Docker + Kubernetes](#phase-4)
- [Phase 5 – AWS](#phase-5)
- [Phase 6 – Distributed Systems + Consistency + Consensus](#phase-6)
- [Phase 7 – AI Agents + Spring AI + Modern Architectures](#phase-7)
- [Master Revision Cheat Sheet](#master-revision-cheat-sheet)

---

# Phase 1 – SQL Deep Dive + PostgreSQL Internals + Redis {#phase-1}

## 1.1 SQL Deep Dive

### ACID Properties

| Property | Meaning | Example |
|----------|---------|---------|
| Atomicity | All or nothing | Bank transfer: debit + credit both succeed or both fail |
| Consistency | Data always valid | Account balance never goes negative if a constraint exists |
| Isolation | Transactions don't interfere | Two users booking the last seat see a clean state |
| Durability | Committed data survives crash | Order confirmed = exists even after server restart |

### Indexing

- **B-Tree** (default) – equality + range queries. `CREATE INDEX idx ON users(email);`
- **Hash** – equality only, faster for `=`. `CREATE INDEX ... USING HASH (token);`
- **Composite** – leftmost prefix rule: index on `(last_name, first_name)` works for `last_name` alone but NOT `first_name` alone.
- **Partial** – index a subset of rows: `CREATE INDEX ... WHERE status = 'ACTIVE';`
- **Covering** – index contains all columns the query needs, avoids touching the table.

### Window Functions

Compute a value across related rows without collapsing them like `GROUP BY`. Key functions: `ROW_NUMBER()/RANK()`, `LAG()/LEAD()`, `SUM() OVER (...)`. Use `OVER (PARTITION BY ... ORDER BY ...)`.

```sql
SELECT employee_id, department, salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_in_dept
FROM employees;
```

### CTEs vs Subqueries

A CTE (`WITH ...`) names a subquery to make it readable and reusable. A **recursive CTE** (`WITH RECURSIVE`) walks hierarchical data (org charts, category trees) via `UNION ALL`.

### Transaction Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Default? |
|-------|-----------|--------------------|-----------|----|
| READ COMMITTED | Prevented | Possible | Possible | PostgreSQL default |
| REPEATABLE READ | Prevented | Prevented | Possible | Financial reports |
| SERIALIZABLE | Prevented | Prevented | Prevented | High-integrity banking |

### Query Optimization with EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE SELECT u.name, COUNT(o.id) FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.name;
-- Seq Scan = full table scan → add an index
-- Index Scan = good
-- cost=X..Y / actual time=X..Y rows=Z: planner vs actual
```

---

## 1.2 PostgreSQL Internals

### MVCC

PostgreSQL keeps old row versions visible to concurrent readers (no read-write locking). Each row has hidden `xmin`/`xmax` columns. On `UPDATE`, the old row is logically deleted and a new row is inserted. Dead rows accumulate until VACUUM cleans them up.

### WAL (Write-Ahead Log)

Changes are written to the sequential WAL before touching actual data pages — ensures crash recovery. WAL is also used for **streaming replication** (primary → replicas).

### VACUUM

```sql
VACUUM orders;           -- reclaim dead tuple space
VACUUM ANALYZE orders;   -- reclaim + update planner statistics
VACUUM FULL orders;      -- compact table (exclusive lock — avoid in production)
```

### Connection Pooling (PgBouncer)

PostgreSQL spawns a full OS process per connection. 500 connections = 500 processes = memory exhaustion. PgBouncer sits between app and Postgres, multiplexing many app connections through a small real pool. **Transaction pooling** is the recommended mode.

### Partitioning

Splits one large table into smaller physical partitions so queries scan only relevant ones. Common strategies: **range** (by date), **list** (by region), **hash**.

```sql
CREATE TABLE orders (...) PARTITION BY RANGE (created_at);
CREATE TABLE orders_2024 PARTITION OF orders FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

### PostgreSQL-Specific Features

- **JSONB** – binary JSON with GIN indexing and `@>` containment queries.
- **Full-Text Search** – `to_tsvector`/`to_tsquery`/`ts_rank` for ranked keyword search.
- **UPSERT** – `INSERT ... ON CONFLICT (col) DO UPDATE SET ...`

---

## 1.3 Redis

### Core Data Structures

| Structure | Use Case |
|-----------|----------|
| String | Cache, counters, sessions |
| List | Message queues, activity feeds |
| Set | Unique collections, tags |
| Sorted Set | Leaderboards, rate limiting |
| Hash | Object storage (user profile fields) |
| Stream | Durable event log |

### Caching Patterns

- **Cache-Aside (Lazy Loading)** – check cache → on miss, load from DB and populate cache. Most common.
- **Write-Through** – write to DB and cache simultaneously. Keeps cache fresh.
- **Write-Behind** – write to cache, async flush to DB. Risky (data loss on crash).

### Redis Persistence

| Mode | Durability | Notes |
|------|-----------|-------|
| RDB | May lose last few minutes | Fast restart, snapshots |
| AOF | Near zero data loss | Slower restart |
| No persistence | Full loss on crash | Fastest (pure cache use) |

### Distributed Locking (Redisson)

Use `RLock.tryLock(waitTime, leaseTime, unit)` for coordinating access across app instances. Always release in a `finally` block guarded by `isHeldByCurrentThread()`.

### Pub/Sub vs Redis Streams

**Pub/Sub** is fire-and-forget — messages delivered only to currently-connected subscribers, no persistence. **Redis Streams** are a durable append-only log with consumer groups and acknowledgements. Prefer Streams when you need reliability.

---

## Phase 1 – Interview Questions

**Q: What is the difference between a clustered and non-clustered index?**  
A: In PostgreSQL all tables are heap-organized (no true clustered index; `CLUSTER` physically reorders once). In SQL Server/MySQL InnoDB the clustered index IS the table — rows sorted by primary key. A non-clustered index is a separate B-Tree with a pointer back to the row.

**Q: Explain MVCC.**  
A: PostgreSQL keeps multiple row versions so readers never block writers. Each transaction sees a snapshot. Dead versions accumulate and must be cleaned by VACUUM.

**Q: When would you use a Sorted Set over a regular Set?**  
A: When you need ordering. Sorted Sets store members with a float score — use for leaderboards, sliding-window rate limiting (timestamp as score), priority queues. Tradeoff: O(log N) vs O(1) for Set membership checks.

**Q: What is the N+1 query problem?**  
A: Fetching N entities then making 1 extra query per entity for a related collection. Fix with: SQL JOIN, `JOIN FETCH` in JPA, `@BatchSize`, or DTO projections.

**Q: What is connection pooling and why is it critical?**  
A: PostgreSQL spawns a full OS process per connection (~5-10MB RAM each). HikariCP pools at the JVM level; PgBouncer pools at infrastructure level (shared across multiple app instances).

---

# Phase 2 – Kafka + Event-Driven Systems + Messaging Patterns {#phase-2}

## 2.1 Kafka Architecture

```
Producers → Kafka Cluster (Brokers) → Consumers
             Topic: "orders"
             Partition0 | Partition1 | Partition2
             [msg0,1,2]  [msg3,4,5]   [msg6,7,8]
```

Key concepts: **topic** (logical stream), **partition** (ordered, append-only log), **consumer group** (partitions distributed across consumers for parallelism), **offset** (consumer's position in a partition).

### Key Producer Config

```java
config.put(ProducerConfig.ACKS_CONFIG, "all");              // all ISR must confirm
config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // exactly-once per producer
config.put(ProducerConfig.LINGER_MS_CONFIG, 5);             // batch for 5ms
config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
```

### Key Consumer Config

```java
config.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false); // manual commit for at-least-once
```

### Delivery Guarantees

| Guarantee | How | Risk |
|-----------|-----|------|
| At-most-once | Commit offset before processing | Data loss |
| At-least-once | Commit offset after processing | Duplicate processing |
| Exactly-once | Idempotent producer + transactional API | Highest overhead |

Manual offset commit pattern: `acknowledgment.acknowledge()` after successful processing; send to DLQ for non-retryable errors.

---

## 2.2 Event-Driven Systems

### Event Sourcing

Store an append-only sequence of events instead of mutable current state. Rebuild state by replaying events. Benefits: full audit history, reconstruct any past state. Cost: complexity, need snapshots for long streams.

### CQRS

Separate the **write model** (commands → write DB) from the **read model** (queries → read-optimized views/cache). A projector listens for events and builds denormalized read models. Use when read/write workloads have very different shapes — adds complexity, don't use by default.

---

## 2.3 Messaging Patterns

### Saga Pattern

Manages distributed transactions as a sequence of local transactions with **compensating actions** on failure.

- **Choreography** – services react to events, emit new events. Decoupled, harder to trace.
- **Orchestration** – a central orchestrator calls each service and handles failures. Easier to reason about.

### Outbox Pattern

**Problem:** DB write succeeds but Kafka publish fails → inconsistent state.  
**Solution:** Save the event in an `outbox` table in the **same DB transaction** as the business entity. A separate relay process polls the outbox and publishes to Kafka.

### Dead Letter Queue (DLQ)

Messages go to a DLQ after repeated processing failures. In Spring Kafka: use `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` — retries N times with backoff, then sends to `<topic>.DLT`.

---

## Phase 2 – Interview Questions

**Q: How does Kafka guarantee message ordering?**  
A: Within a single partition only. Use entity ID (orderId, userId) as the key — same key always routes to the same partition.

**Q: Kafka vs RabbitMQ?**  
A: Kafka is a distributed log — messages are retained and replayable, multiple consumer groups each read independently. RabbitMQ is a traditional broker — messages deleted after consumption, flexible routing. Kafka for high-throughput event streaming; RabbitMQ for complex routing and task queues.

**Q: Explain the Outbox Pattern.**  
A: Avoids dual-write inconsistency by writing event and business data in the same DB transaction. A relay (or CDC tool like Debezium) handles eventual Kafka delivery.

**Q: What is a consumer group?**  
A: A set of consumers that jointly consume a topic — each partition assigned to exactly one consumer. More consumers than partitions = excess consumers idle. Multiple separate groups each receive all messages independently.

---

# Phase 3 – System Design Fundamentals + HLD + LLD {#phase-3}

## 3.1 System Design Fundamentals

### Scalability Patterns

| Problem | Solution | Trade-off |
|---------|---------|----------|
| Single server overloaded | Horizontal scaling + Load Balancer | Session management complexity |
| DB read bottleneck | Read replicas | Replication lag |
| DB write bottleneck | Sharding | Cross-shard queries become hard |
| Expensive repeated computation | Caching (Redis) | Cache invalidation |
| Large files / static assets | CDN | Cost, cache invalidation |
| Tight coupling | Message queue (async) | Eventual consistency |

### CAP Theorem

A distributed system can guarantee at most 2 of: Consistency, Availability, Partition Tolerance. Since network partitions will happen, P is non-negotiable — the real choice is **CP vs AP**.

- **CP** (ZooKeeper, etcd): rejects requests on minority partition.
- **AP** (Cassandra, DynamoDB): serves potentially stale data, eventually converges.

### Load Balancing Algorithms

| Algorithm | Best For |
|-----------|----------|
| Round Robin | Equal-capacity stateless services |
| Least Connections | Long-lived connections (WebSocket) |
| IP Hash | Session stickiness |
| Weighted Round Robin | Heterogeneous server capacities |

### Caching

Levels (fastest → slowest): in-process (Caffeine) → distributed (Redis) → DB query cache → CDN.

Invalidation strategies: TTL-based, event-based, write-through, cache-aside.

**Cache stampede** (many requests rebuild the same expired key at once): prevent with a mutex lock, probabilistic early expiration, or background refresh.

### Database Sharding

- **Range-based** – `user_id 1-1M → Shard A`. Range queries easy; hotspot risk.
- **Hash-based** – `shard = hash(id) % N`. Even distribution; range queries need scatter-gather.
- **Directory-based** – lookup service maps entity → shard. Flexible; lookup is a SPOF.

Cross-shard pain: joins move to the app layer, aggregates need scatter-gather, no ACID across shards (use Saga).

---

## 3.2 High-Level Design (HLD)

### URL Shortener

Flow: Client → Load Balancer → API Service → PostgreSQL, with Redis caching hot URLs.

Key decisions:
- **ID generation:** Base62-encode an auto-increment ID (simple, common pick).
- **Redirect type:** `301` (permanent, client caches) vs `302` (temporary, every request hits server — needed for click tracking).

### Notification System

Producer → Kafka (`notifications` topic) → Notification Service → per-channel workers (Email/SMS/Push) that scale independently. Key: check user opt-outs/quiet hours, enforce idempotency by `notification_id`, rate-limit per user/channel, DLQ with retry for failures.

---

## 3.3 Low-Level Design (LLD)

### Rate Limiter (Token Bucket)

Maintain a `tokens` counter; refill at a fixed rate; decrement on each request; reject if `tokens < 1`. For multi-instance use, run refill-and-consume atomically in a **Redis Lua script**.

### Parking Lot (OOP Design)

Key classes: `Vehicle` hierarchy (Motorcycle/Car/Truck) with `requiredSpotType()`, `ParkingSpot` with `canFit(vehicle)`, `ParkingLevel`, `ParkingLot`. Use enums + polymorphism instead of `if`/`switch` on type.

---

## Phase 3 – Interview Questions

**Q: How would you handle zero-downtime schema migrations?**  
A: Expand-contract pattern: (1) add new column (nullable, non-breaking); (2) backfill data; (3) switch reads to new column; (4) remove old column in a separate deployment. Use Flyway or Liquibase.

**Q: Monolith vs microservices?**  
A: Start with a monolith. Adopt microservices when specific parts need independent scaling, teams need independent deployments, or domains are truly autonomous. The operational overhead (networking, observability, distributed transactions) is real cost.

**Q: What is consistent hashing?**  
A: Places nodes on a virtual ring; each key maps to the nearest clockwise node. Adding/removing a node reshuffles only `1/N` keys instead of nearly all. Used in Redis Cluster, Cassandra, Memcached.

---

# Phase 4 – Docker + Kubernetes {#phase-4}

## 4.1 Docker

### How Docker Works

Linux primitives: **namespaces** (isolate PID, network, mount, etc.), **cgroups** (limit CPU/memory/I/O), **OverlayFS** (layered images), **seccomp** (restrict syscalls).

### Dockerfile Best Practices

```dockerfile
# Multi-stage: build in one stage, ship only runtime artifacts
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -q   # cache deps separately
COPY src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
USER app                              # run as non-root
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-jar", "app.jar"]
```

Key rules: order `COPY` statements with stable files first (pom.xml before src) for layer caching; use multi-stage builds; run as non-root; use exec form for `ENTRYPOINT`.

### docker-compose (Local Dev)

```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16-alpine
    environment: { POSTGRES_DB: mydb, POSTGRES_PASSWORD: secret }
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
  redis:
    image: redis:7-alpine
```

---

## 4.2 Kubernetes

### Architecture

- **Control Plane:** API Server, etcd (cluster state), Scheduler, Controller Manager.
- **Worker Nodes:** kubelet (manages pods), kube-proxy (networking), Pods (containers).

### Core Objects

- **Deployment** – manages a ReplicaSet, handles rolling updates.
- **Service** – stable network endpoint for Pods (`ClusterIP`, `LoadBalancer`, `NodePort`).
- **HorizontalPodAutoscaler (HPA)** – scales replicas based on CPU/memory/custom metrics.
- **Ingress** – HTTP routing rules (path/host → service).
- **ConfigMap** – non-sensitive config; **Secret** – sensitive data (base64-encoded; use Vault in production).

### Key Deployment Config Points

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # create 1 extra pod before killing old
    maxUnavailable: 0  # zero-downtime: never kill before new one is ready
resources:
  requests: { memory: "256Mi", cpu: "250m" }  # used for scheduling
  limits:   { memory: "512Mi", cpu: "500m" }  # OOM kill if exceeded
readinessProbe:   # remove pod from LB if fails (no restart)
  httpGet: { path: /actuator/health/readiness, port: 8080 }
livenessProbe:    # restart container if fails
  httpGet: { path: /actuator/health/liveness, port: 8080 }
```

---

## Phase 4 – Interview Questions

**Q: Docker image vs container?**  
A: Image = read-only layered blueprint. Container = running instance of an image. Image is like a class; container is like an object.

**Q: What happens when a Pod is OOMKilled?**  
A: Container exceeded `limits.memory`; kernel OOM killer terminates it; Kubernetes restarts per `restartPolicy`. Fix: increase limits, fix memory leaks, or set `-XX:MaxRAMPercentage=75.0` so JVM respects container limits.

**Q: readinessProbe vs livenessProbe?**  
A: **readiness** — "ready to receive traffic?" Failure removes pod from LB, no restart. Use for warm-up or temporary unavailability. **liveness** — "still alive?" Failure triggers restart. Use for deadlocks/unrecoverable states.

**Q: Deployment vs StatefulSet?**  
A: Deployment = stateless, pods interchangeable, random names. StatefulSet = stateful apps (DBs, Kafka), stable ordinal names (pod-0, pod-1), stable network identity, ordered startup/shutdown, per-pod PersistentVolumeClaim.

---

# Phase 5 – AWS {#phase-5}

## 5.1 Core AWS Services

### Compute

| Service | Use Case |
|---------|---------|
| EC2 | VMs, full control |
| Lambda | Serverless, event-driven, short-lived |
| ECS (Fargate) | Containers without managing nodes |
| EKS | Managed Kubernetes |

### Storage & Databases

| Service | Type | Use Case |
|---------|------|---------|
| S3 | Object storage | Files, backups, static assets |
| EBS | Block storage | EC2 volumes |
| RDS | Managed SQL | PostgreSQL, MySQL |
| Aurora | AWS-native SQL | High performance, auto-scaling storage |
| DynamoDB | NoSQL | Massive scale, single-digit ms latency |
| ElastiCache | Managed Redis/Memcached | Caching, sessions |

### Networking

- **VPC** – logically isolated network. Public subnets (with Internet Gateway) for load balancers; private subnets for app servers and databases.
- **ALB** (Layer 7) – path/header-based routing, SSL termination. Use for web apps.
- **NLB** (Layer 4) – TCP/UDP, ultra-low latency, static IP. Use for gaming/IoT/financial.
- **Security Groups** – stateful, instance-level, allow-only rules.
- **NACLs** – stateless, subnet-level, allow + deny rules. Evaluated before SGs.

### IAM Best Practices

Least privilege: grant only the specific actions on specific resources needed. Use **IAM Roles** for EC2/ECS/Lambda (auto-rotated, never hardcode credentials). **MFA** for human users.

### SQS vs SNS

- **SQS** – point-to-point queue. Pull-based, messages stored until consumed, DLQ for failures. FIFO variant for ordered/exactly-once delivery.
- **SNS** – pub/sub fan-out. One publisher → multiple subscribers (SQS queues, Lambda, HTTP). Fan-out pattern: SNS → multiple SQS queues for independent per-service processing.

### CloudWatch

Metrics, logs, alarms, dashboards. Publish custom business metrics via SDK. Set alarms on error rate or latency to notify or trigger auto-scaling.

---

## Phase 5 – Interview Questions

**Q: Aurora vs RDS PostgreSQL?**  
A: Aurora has <10ms replica lag (vs RDS async replication), auto-scaling storage to 128TB, Aurora Serverless for bursty workloads, and Global Database for multi-region. RDS is cheaper and simpler when Aurora's features aren't needed.

**Q: ALB vs NLB?**  
A: ALB = Layer 7 (HTTP), path/host/header routing, SSL termination. NLB = Layer 4 (TCP/UDP), millions of req/sec, ultra-low latency, static IP per AZ.

**Q: Security Groups vs NACLs?**  
A: SGs = stateful (response auto-allowed), instance-level, allow-only. NACLs = stateless (must explicitly allow both directions), subnet-level, allow + deny. NACLs evaluated first.

**Q: Serverless image processing pipeline?**  
A: S3 upload → S3 event → Lambda (validate + resize → save thumbnail) → SNS → SQS → Lambda (send notification). Key considerations: Lambda 15-min timeout, 256MB async payload limit, use S3 presigned URLs for direct browser uploads.

---

# Phase 6 – Distributed Systems + Consistency Models + Consensus {#phase-6}

## 6.1 Consistency Models

```
Strongest ────────────────────────────────────────▶ Weakest
Linearizability → Sequential → Causal → Eventual
```

- **Linearizability** – every read sees the most recent write. Highest latency. (etcd, ZooKeeper, Spanner)
- **Causal** – causally related ops seen in same order everywhere; concurrent ops may differ.
- **Eventual** – replicas eventually converge with no new writes. Lowest latency. (DNS, DynamoDB/Cassandra defaults)

### Conflict Resolution in Eventual Consistency

- **Last-Write-Wins (LWW)** – highest timestamp wins. Simple, but clock skew can lose writes.
- **Vector Clocks** – track causal history to detect concurrent writes; app resolves conflicts.
- **CRDTs** – data structures that merge automatically (grow-only set, counters).

---

## 6.2 Consensus Algorithms

### Raft

Three states: **Follower** (default), **Candidate** (on timeout), **Leader** (accepts all writes).

Leader election: follower timeout → candidate → requests votes → majority → leader.  
Log replication: leader appends → replicates to followers → majority acknowledge → committed.

Safety: at most one leader per term; all nodes apply entries in the same order.

### Distributed Clocks

No global clock in distributed systems — wall-clock time drifts.
- **Lamport timestamps** – logical counter, establishes causal ordering.
- **Vector clocks** – detects concurrent (conflicting) events.
- HLC and TrueTime are advanced/architect-level — know what problem they solve.

### Distributed Transactions

**2PC (Two-Phase Commit):** Coordinator asks all participants to prepare → all agree → commit. Problems: blocking if coordinator crashes after prepare; single point of failure.

**Prefer Saga pattern** (see Phase 2) – no locking, compensating transactions, better availability.

---

## 6.3 Common Distributed System Problems

### Leader Election

Use a coordination service (ZooKeeper + Curator's `LeaderSelector`, or Redis/DB lock) rather than implementing consensus yourself. One node holds leadership until it crashes or relinquishes it.

### Idempotency

Every write should be idempotent. Use client-generated **idempotency keys** mapped to results in a store. Return the same result for duplicate requests.

```java
@PostMapping("/payments")
public ResponseEntity<Payment> processPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {
    Optional<Payment> existing = idempotencyStore.get(idempotencyKey);
    if (existing.isPresent()) return ResponseEntity.ok(existing.get());
    Payment payment = paymentService.process(request);
    idempotencyStore.store(idempotencyKey, payment, Duration.ofDays(7));
    return ResponseEntity.status(HttpStatus.CREATED).body(payment);
}
```

---

## Phase 6 – Interview Questions

**Q: CAP theorem in real systems.**  
A: P is non-negotiable (networks fail). CP systems (ZooKeeper, etcd) reject requests on minority partition. AP systems (Cassandra, DynamoDB) serve potentially stale data and converge after the partition heals. Most real systems offer tunable consistency (Cassandra's QUORUM vs ONE).

**Q: Split-brain and how to prevent it.**  
A: Network partition → two groups both think they're leader → divergent writes. Prevention: quorum writes (only N/2 + 1 nodes partition accepts writes), fencing tokens (monotonically increasing; writes with lower tokens rejected).

**Q: Paxos vs Raft.**  
A: Both achieve consensus. Paxos is provably correct but hard to understand/implement. Raft was designed for understandability — decomposes into leader election, log replication, safety. etcd uses Raft; ZooKeeper uses ZAB (similar to Paxos).

**Q: What is idempotency and why does it matter?**  
A: An operation is idempotent if calling it multiple times produces the same result as once. In distributed systems, failures are partial — a request may be processed but the response lost, causing retries. Without idempotency, retries cause duplicate charges or duplicate orders.

---

# Phase 7 – AI Agents + Spring AI + Modern Architectures {#phase-7}

## 7.1 AI Agents

An AI agent combines an LLM (planner), tools (web search, DB query, API calls), and memory (context window + optional vector DB). The **ReAct pattern** loops: Thought → Action (tool call) → Observation → repeat until done.

---

## 7.2 Spring AI

### Setup

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

```yaml
spring.ai.openai.api-key: ${OPENAI_API_KEY}
spring.ai.openai.chat.options.model: gpt-4o
```

### Basic Chat

```java
this.chatClient = builder.defaultSystem("You are a helpful support agent.").build();
return chatClient.prompt().user(question).call().content();
```

### Tool Calling

Annotate methods with `@Tool(description = "...")`. Spring AI generates the JSON schema, the model decides which tool to call, and Spring AI invokes the method and feeds the result back automatically.

### RAG (Retrieval-Augmented Generation)

1. **Ingest:** chunk source docs → embed → store in vector DB (pgvector).
2. **Query:** embed the user question → `vectorStore.similaritySearch(...)` → inject top-K chunks as context → call LLM.

Use RAG when data changes frequently, you need citations, or data is proprietary. Fine-tuning is better when you need to change model behavior/style.

### Conversation Memory

LLM calls are stateless — replay prior messages each call. Use `MessageChatMemoryAdvisor` backed by a store (in-memory, JDBC, etc.), keyed by conversation ID, capped to last N messages.

### Structured Output

```java
return chatClient.prompt().user("Analyze: " + reviewText).call().entity(ProductReview.class);
```

Spring AI handles JSON schema generation from the Java record/class.

---

## 7.3 Modern Architectures

### Vector Databases

| Database | Strengths |
|---------|----------|
| pgvector | Familiar SQL, ACID, no extra infra |
| Pinecone | Fully managed, massive scale |
| Chroma | Simple, great for prototyping |
| Qdrant | High performance, Rust-based |

In pgvector: **HNSW** = better recall (production preferred); **IVFFLAT** = faster to build.

### Typical Spring Boot + AI Architecture

API layer (REST/GraphQL/WebSocket) → Spring Boot + Spring AI services (RAG, agents, CRUD) → data layer (PostgreSQL + pgvector, Redis, Kafka, S3) → AI provider (OpenAI, Anthropic, or local Ollama).

---

## Phase 7 – Interview Questions

**Q: RAG vs fine-tuning?**  
A: RAG retrieves relevant docs at query time and injects them into the prompt. Fine-tuning bakes knowledge into model weights. Use RAG when data changes frequently, you need citations, or data is proprietary. Fine-tune when you need to change model behavior/style or domain-specific terminology.

**Q: Semantic search vs keyword search?**  
A: Keyword search (BM25) finds exact terms. Semantic search converts text to embedding vectors and finds semantically similar documents — matches synonyms and paraphrasing. Hybrid (keyword + semantic) often outperforms either alone.

**Q: Risks of AI in production?**  
A: Hallucination (mitigate with RAG + citations), prompt injection (sanitize inputs, use structured tools), data privacy (self-hosted models for sensitive data), cost (cache common queries, smaller models for simple tasks), latency (stream responses).

**Q: Tool call vs function calling?**  
A: Same concept, different vendor terminology. The LLM generates structured JSON describing which function to call and with what args. Your code executes the function and returns the result to the LLM. The LLM decides WHAT to call; your code handles execution.

---

# Master Revision Cheat Sheet

### Phase 1 – SQL & Redis
- **B-Tree**: equality + range; **Composite**: leftmost prefix rule
- **MVCC**: readers don't block writers; dead tuples need VACUUM
- **WAL**: crash recovery + replication mechanism
- **Redis LRU**: `volatile-lru` for session stores
- **Redis Streams** > Pub/Sub for durable event delivery

### Phase 2 – Kafka & Events
- **Ordering**: guaranteed within a partition, not across
- **Consumer group**: 1 partition → 1 consumer (parallelism = partition count)
- **Delivery**: `acks=all` + idempotent producer = exactly-once per-producer
- **Outbox pattern**: solve dual-write with same-transaction outbox + relay
- **Saga**: choreography (event-driven) vs orchestration (central coordinator)

### Phase 3 – System Design
- **CAP**: CP or AP (P always required)
- **Consistent hashing**: minimizes reshuffling on node add/remove
- **Sharding**: hash (even distribution) vs range (range-query friendly)
- **Cache stampede**: mutex lock, probabilistic early expiration, background refresh
- **Zero-downtime migration**: expand-contract pattern

### Phase 4 – Docker & Kubernetes
- **Layers**: stable files first in Dockerfile (pom.xml before src)
- **Multi-stage**: ship only runtime artifacts
- **readiness vs liveness**: readiness = traffic gating; liveness = restart trigger
- **HPA**: requires resource requests set; scales on CPU/memory/custom metrics
- **StatefulSet**: stable identity + persistent volumes for stateful apps

### Phase 5 – AWS
- **SG**: stateful, instance level; **NACL**: stateless, subnet level
- **ALB**: Layer 7, path routing; **NLB**: Layer 4, ultra-low latency
- **IAM Roles**: always over hardcoded credentials
- **S3 presigned URL**: direct browser → S3 upload (bypass Lambda payload limit)
- **SQS FIFO**: exactly-once, ordered per message group ID

### Phase 6 – Distributed Systems
- **Linearizability**: strongest — every read sees latest write
- **Eventual consistency**: replicas converge after writes stop
- **Raft**: leader election + log replication (understandable consensus)
- **Split-brain**: prevent with quorum writes (N/2 + 1)
- **Idempotency**: idempotency-key → stored result mapping

### Phase 7 – AI Agents & Spring AI
- **RAG vs Fine-tuning**: RAG for changing data + citations; fine-tuning for behavior/style
- **Tool calling**: LLM generates JSON → your code executes → result returned to LLM
- **Vector similarity**: cosine (normalized), dot product (raw magnitude matters)
- **Hallucination**: mitigate with RAG, citations, structured output validation
- **Guardrails**: input moderation → system prompt hardening → output validation → logging

---

## Common Interview Patterns

### "How would you design X?" Framework
1. Clarify requirements (functional + non-functional: scale, latency, availability)
2. Estimate scale (users, req/sec, data size, read/write ratio)
3. High-level design (major components, data flow)
4. Data model (schema, indexing strategy)
5. Deep dive (bottlenecks, caching, scaling, failure modes)
6. Trade-offs

### "Explain X" Framework
1. What it is (one sentence)
2. Real-world analogy
3. How it works (key steps)
4. When to use it (trade-offs vs alternatives)
5. Gotchas

### System Design Numbers to Remember

| Metric | Value |
|--------|-------|
| L1 cache | ~1 ns |
| RAM access | ~100 ns |
| SSD random read | ~100 µs |
| Network round-trip (same DC) | ~0.5 ms |
| Network round-trip (cross-region) | ~150 ms |
| PostgreSQL query (indexed) | ~1 ms |
| Redis GET | ~0.1 ms |
| Kafka producer (async) | ~5 ms |
| LLM API call (GPT-4o) | ~500ms–3s |
