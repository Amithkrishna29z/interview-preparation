# Software Architect Study Guide

## Overview

A Software Architect makes high-level design decisions, defines technical standards, and ensures a system's structural integrity. For juniors, this guide focuses on the foundational concepts you'll need to understand architecture discussions, system design interviews, and day-to-day work on larger teams.

---

## Table of Contents

1. [Roles & Responsibilities](#roles--responsibilities)
2. [Core Foundations](#core-foundations)
3. [System Design & Architecture Patterns](#system-design--architecture-patterns)
4. [Distributed Systems](#distributed-systems)
5. [Cloud Architecture](#cloud-architecture)
6. [Data Architecture](#data-architecture)
7. [Security Architecture](#security-architecture)
8. [DevOps & Infrastructure](#devops--infrastructure)
9. [API Design](#api-design)
10. [Performance & Scalability](#performance--scalability)
11. [Architecture Decision Records (ADRs)](#architecture-decision-records-adrs)
12. [Interview Preparation](#interview-preparation)
13. [Learning Roadmap](#learning-roadmap)
14. [Recommended Resources](#recommended-resources)

---

## Roles & Responsibilities

### What a Software Architect Does

| Responsibility | Description |
|---|---|
| System Design | Define the overall structure, components, and interactions of a system |
| Technology Selection | Evaluate and choose technologies, frameworks, and tools |
| Non-Functional Requirements | Ensure scalability, reliability, maintainability, security |
| Code Quality Standards | Define coding standards, patterns, and review processes |
| Technical Leadership | Guide and mentor engineering teams |
| Stakeholder Communication | Bridge technical teams and business stakeholders |
| Risk Management | Identify technical risks and define mitigation strategies |

### Types of Software Architects

```
Enterprise Architect   → Sets org-wide technology strategy
Solution Architect     → Designs architecture for a specific project
Application Architect  → Focuses on a single application's design
Infrastructure/Cloud   → Designs cloud, network, deployment infrastructure
Data Architect         → Designs data models, pipelines, storage
Security Architect     → Defines security protocols and threat models
```

---

## Core Foundations

### 1. Data Structures & Algorithms

- Arrays, Linked Lists, Trees, Graphs, Hash Maps
- Big-O analysis — time and space complexity
- Know when to pick which structure (e.g., B-Trees in DBs, Graphs in social networks)

### 2. Object-Oriented Design Principles

#### SOLID Principles

```
S — Single Responsibility: A class should have only one reason to change.
O — Open/Closed: Open for extension, closed for modification.
L — Liskov Substitution: Subtypes must be substitutable for their base types.
I — Interface Segregation: No client should depend on methods it doesn't use.
D — Dependency Inversion: Depend on abstractions, not concretions.
```

#### Other Key Principles

- **DRY** — Don't Repeat Yourself
- **KISS** — Keep It Simple, Stupid
- **YAGNI** — You Aren't Gonna Need It

### 3. Design Patterns (GoF)

#### Creational
| Pattern | Use Case |
|---|---|
| Singleton | Single shared instance (config, logger) |
| Factory Method | Delegate object creation to subclasses |
| Builder | Complex object construction step-by-step |

#### Structural
| Pattern | Use Case |
|---|---|
| Adapter | Bridge incompatible interfaces |
| Facade | Simplified interface to a complex subsystem |
| Decorator | Add behavior dynamically |
| Proxy | Controlled access to an object |

#### Behavioral
| Pattern | Use Case |
|---|---|
| Observer | Event/notification systems |
| Strategy | Swappable algorithms at runtime |
| Command | Encapsulate requests as objects |
| Template Method | Define skeleton, let subclasses fill in steps |
| State | Object behavior changes with internal state |

---

## System Design & Architecture Patterns

### Monolithic Architecture

```
[Client] ──► [Monolith App]
                 ├── UI Layer
                 ├── Business Logic
                 └── Data Access Layer
                        │
                   [Database]
```

**Pros**: Simple to develop, deploy, and test in early stages  
**Cons**: Scaling bottlenecks, deployment risk, technology lock-in

---

### Microservices Architecture

```
[Client]
   │
[API Gateway]
   ├── [User Service]    ──► [User DB]
   ├── [Order Service]   ──► [Order DB]
   └── [Payment Service] ──► [Payment DB]
```

**Key Principles:**
- Each service owns its data (Database per Service)
- Services communicate via APIs or messaging
- Independent deployability
- Bounded contexts (from Domain-Driven Design)

**Challenges:** Distributed complexity, network latency, data consistency across services

---

### Event-Driven Architecture (EDA)

```
[Producer] ──► [Event Broker (Kafka/RabbitMQ)] ──► [Consumer A]
                                                 ──► [Consumer B]
```

**Key Patterns:**
- **Event Notification** — Inform others something happened
- **Event Sourcing** — Store state as a sequence of events
- **CQRS** — Separate read and write models

---

### Layered Architecture (N-Tier)

```
┌─────────────────────────┐
│   Presentation Layer    │  ← UI, API Controllers
├─────────────────────────┤
│   Application Layer     │  ← Use Cases, Orchestration
├─────────────────────────┤
│   Domain/Business Layer │  ← Business Rules, Entities
├─────────────────────────┤
│   Infrastructure Layer  │  ← DB, External APIs, Messaging
└─────────────────────────┘
```

---

### Hexagonal Architecture (Ports & Adapters)

**Goal**: Decouple core business logic from external concerns (databases, UIs, messaging).

```
         [REST Adapter]   [CLI Adapter]
               └───────┬───────┘
                  [Application Core]
               ┌───────┴───────┐
       [DB Adapter]    [Message Adapter]
```

---

### Clean Architecture

**Dependency Rule**: Dependencies only point inward. Inner layers know nothing about outer layers.

```
Frameworks & Drivers (outermost)
  └── Interface Adapters
        └── Application / Use Cases
              └── Entities (innermost)
```

---

### CQRS & Saga Pattern

**CQRS** — Separate read and write models; use when read/write workloads differ significantly.

```
[Client]
  ├── Command (Write) ──► [Write Model] ──► [Write DB]
  └── Query  (Read)  ◄── [Read Model]  ◄── [Read DB]
```

**Saga Pattern** — Manages distributed transactions across microservices using compensating events on failure.

---

## Distributed Systems

### CAP Theorem

You can only guarantee **two** of the three:

| Trade-off | Examples | Behavior |
|---|---|---|
| **CP** (Consistent + Partition Tolerant) | HBase, Zookeeper | Returns error on network partition |
| **AP** (Available + Partition Tolerant) | Cassandra, DynamoDB | Returns stale data on partition |
| **CA** (Consistent + Available) | Traditional RDBMS | Not partition tolerant (single node) |

### BASE vs ACID

| Property | ACID (SQL) | BASE (NoSQL) |
|---|---|---|
| **A** | Atomicity | Basically Available |
| **C** | Consistency | Soft state |
| **I/D** | Isolation / Durability | Eventually consistent |

### Consistency Models

- **Strong Consistency** — Every read reflects the latest write (expensive)
- **Eventual Consistency** — Reads may be stale but will converge (cheap, highly available)
- **Read-Your-Writes** — You always see your own writes

### Distributed System Challenges

| Challenge | Solution Strategies |
|---|---|
| Network partitions | Circuit Breakers, Retries with backoff |
| Data replication lag | Eventual consistency, versioning |
| Service discovery | Consul, Eureka, Kubernetes DNS |
| Distributed tracing | Jaeger, Zipkin, OpenTelemetry |
| Idempotency | Idempotency keys, deduplication logic |

---

## Cloud Architecture

### Cloud Design Principles

- **Elasticity** — Scale in/out based on demand
- **Fault Tolerance** — Design for failure; assume components will fail
- **Loose Coupling** — Reduce dependencies between components

### AWS Core Services (by Category)

| Category | Services |
|---|---|
| Compute | EC2, ECS, EKS, Lambda, Fargate |
| Storage | S3, EBS, EFS |
| Database | RDS, DynamoDB, Aurora, ElastiCache |
| Networking | VPC, Route 53, CloudFront, API Gateway, ALB |
| Messaging | SQS, SNS, EventBridge, Kinesis |
| Security | IAM, KMS, Secrets Manager, WAF |
| Observability | CloudWatch, X-Ray, CloudTrail |

### Well-Architected Framework (5 Pillars)

```
1. Operational Excellence  → Automate operations, learn from failures
2. Security                → Protect data, manage identities, detect threats
3. Reliability             → Recover from failures, scale dynamically
4. Performance Efficiency  → Use resources efficiently, monitor performance
5. Cost Optimization       → Avoid unnecessary costs, right-size resources
```

---

## Data Architecture

### Database Selection Guide

| Use Case | Type | Examples |
|---|---|---|
| Structured relational data | SQL | PostgreSQL, MySQL |
| High-volume key-value lookups | Key-Value | Redis, DynamoDB |
| Document storage | Document | MongoDB |
| Time-series data | Time-Series | InfluxDB |
| Graph relationships | Graph | Neo4j |
| Full-text search | Search | Elasticsearch |
| Analytics / OLAP | Data Warehouse | Redshift, BigQuery |

### Data Pipeline Patterns

- **Batch Processing** — ETL jobs run periodically (Spark, dbt)
- **Stream Processing** — Real-time event processing (Kafka Streams, Flink)

### Sharding Strategies

| Strategy | How It Works | Trade-off |
|---|---|---|
| Hash Sharding | Hash(key) % N shards | Even distribution; hard to range query |
| Range Sharding | By key range (A-M, N-Z) | Easy range queries; potential hotspots |
| Geo Sharding | By geographic region | Low latency; uneven data distribution |

---

## Security Architecture

### Security Design Principles

- **Least Privilege** — Grant only the minimum permissions needed
- **Defense in Depth** — Multiple layers of security controls
- **Zero Trust** — Never trust, always verify — even inside the network
- **Shift Left** — Integrate security early in the development lifecycle

### OWASP Top 10 (Key Controls)

| Risk | Architectural Control |
|---|---|
| Injection | Parameterized queries, input validation at boundaries |
| Broken Authentication | OAuth2/OIDC, MFA, token rotation |
| Sensitive Data Exposure | Encryption at rest/transit, key management (KMS) |
| Broken Access Control | RBAC/ABAC, policy enforcement at API gateway |
| Security Misconfiguration | IaC templates, config auditing |

### Auth Patterns

- **OAuth 2.0 + OIDC** — Industry standard for delegated auth; produces Access Token + ID Token
- **RBAC** — User → Role → Permissions
- **ABAC** — Policy decision based on user + resource + environment attributes

### Encryption

| Type | Method |
|---|---|
| Data at Rest | AES-256, AWS KMS |
| Data in Transit | TLS 1.2+, mTLS between services |

---

## DevOps & Infrastructure

### CI/CD Pipeline

```
[Code Commit] ──► [CI: Build → Tests → SAST → Container Build → Push to Registry]
                        ──► [CD: Deploy Staging → Integration Tests → Deploy Prod]
```

### Deployment Strategies

| Strategy | Description | Risk |
|---|---|---|
| **Rolling** | Gradually replace old instances | Medium |
| **Blue/Green** | Switch traffic from old to new env | Low (instant rollback) |
| **Canary** | Route small % of traffic to new version | Low (gradual exposure) |
| **Feature Flags** | Toggle features without redeployment | Very Low |

### Kubernetes Key Concepts

```
Cluster
  ├── Control Plane (API Server, Scheduler, etcd)
  └── Worker Nodes (kubelet, kube-proxy, Pods → Containers)
```

- **Deployments** — Manage replica sets and rolling updates
- **Services** — Stable network endpoint for pods
- **Ingress** — HTTP routing into the cluster
- **ConfigMaps / Secrets** — Externalize configuration
- **HPA** — Scale pods based on CPU/memory

### Infrastructure as Code (IaC)

| Tool | Scope |
|---|---|
| Terraform | Multi-cloud infrastructure (declarative HCL) |
| AWS CDK | AWS only (Python/TS/Java) |
| Helm | Kubernetes packaging |

---

## API Design

### REST Principles

```
GET    /users          → list
GET    /users/{id}     → get one
POST   /users          → create
PUT    /users/{id}     → replace
PATCH  /users/{id}     → partial update
DELETE /users/{id}     → delete

2xx → Success | 4xx → Client Error | 5xx → Server Error
```

### REST vs GraphQL vs gRPC

| Dimension | REST | GraphQL | gRPC |
|---|---|---|---|
| Protocol | HTTP/1.1 | HTTP/1.1 or 2 | HTTP/2 |
| Data Format | JSON | JSON | Protocol Buffers |
| Best For | Public APIs, CRUD | Complex data graphs, frontend-driven | Internal microservices, performance |

### API Gateway Pattern

Sits in front of all backend services and handles: Authentication, Rate Limiting, Request Transformation, SSL Termination, Load Balancing, and Routing.

---

## Performance & Scalability

### Scalability Patterns

- **Vertical Scaling** — Add CPU/RAM to existing server (limited ceiling)
- **Horizontal Scaling** — Add more instances (preferred for cloud)
- **Read Replicas** — Route reads to replicas, writes to primary
- **Sharding** — Partition data across multiple DB instances

### Caching Strategies

| Strategy | Description |
|---|---|
| **Cache-Aside** | App checks cache first, loads from DB on miss |
| **Write-Through** | Write to cache and DB simultaneously |
| **Write-Behind** | Write to cache, async flush to DB |

### Load Balancing Algorithms

| Algorithm | Use Case |
|---|---|
| Round Robin | Uniform servers |
| Least Connections | Long-lived connections |
| IP Hash | Session affinity |

### Performance Anti-Patterns

- **N+1 Query** — Fix with JOIN or batch loading
- **Thundering Herd** — Many requests hit DB on cache expiry; fix with jitter
- **Chatty APIs** — Too many small requests; fix with batching or GraphQL
- **Synchronous Chains** — Long sync call chains; fix with async/event-driven patterns

---

## Architecture Decision Records (ADRs)

ADRs document significant architectural decisions. A well-maintained ADR log signals a mature engineering culture.

### ADR Template

```markdown
# ADR-[number]: [Short title]

**Date**: YYYY-MM-DD
**Status**: Proposed | Accepted | Deprecated | Superseded

## Context
What is the problem or opportunity?

## Decision
What change are we making?

## Consequences
What becomes easier or harder? What are the trade-offs?

## Alternatives Considered
What other options were evaluated? Why rejected?
```

---

## Interview Preparation

### Common System Design Topics

| Topic | Key Concepts |
|---|---|
| URL Shortener | Hashing, KV store, redirection |
| Twitter/Feed | Fan-out on write vs read, timeline aggregation |
| Chat System | WebSockets, message queues, presence |
| Rate Limiter | Token bucket, Redis sliding window |
| Notification System | Push/pull, fan-out, at-least-once delivery |
| Payment System | Idempotency, distributed transactions, ACID |

### System Design Interview Framework

```
Step 1: Clarify Requirements (5 min)  — functional, non-functional, constraints
Step 2: Estimate Scale (3 min)        — DAU, QPS, storage needs
Step 3: High-Level Design (10 min)    — core components, data stores, basic flow
Step 4: Deep Dive (15 min)            — 2-3 critical components, trade-offs, bottlenecks
Step 5: Summarize (5 min)             — recap, improvements, unresolved constraints
```

### Architecture Interview Q&A

**Q: Horizontal vs vertical scaling?**  
Vertical adds resources to one machine; horizontal adds more machines. Horizontal is preferred in cloud because it has no ceiling and allows high availability.

**Q: When to choose microservices over a monolith?**  
Start with a monolith. Move to microservices when teams need independent deployability, services have very different scaling needs, or the codebase has grown too large for one team.

**Q: Eventual consistency vs strong consistency?**  
Strong consistency guarantees every read sees the latest write (higher latency). Eventual consistency allows stale reads but converges over time (higher availability). Use strong for financial data; eventual for social feeds.

**Q: What is the CAP theorem?**  
In a distributed system you can guarantee at most two of: Consistency, Availability, Partition Tolerance. Since network partitions always happen, you choose between CP (consistent, may be unavailable) or AP (always available, may be stale).

**Q: SQL vs NoSQL — when to choose?**  
SQL for complex relationships, ACID guarantees, and structured schemas. NoSQL for high throughput, flexible/evolving schemas, or massive scale with simpler access patterns.

**Q: REST vs gRPC?**  
REST is standard for public-facing APIs due to wide tooling and human readability. gRPC is better for internal microservice communication — faster (binary protocol), strongly typed, and supports streaming.

**Q: How would you design for zero-downtime deployments?**  
Use Blue/Green or Canary deployments. Blue/Green switches traffic instantly with an easy rollback path. Canary routes a small percentage first to validate before full rollout.

**Q: How do you handle distributed transactions?**  
Avoid 2PC when possible. Use the Saga pattern — break the transaction into local steps with compensating actions on failure. Choose choreography (event-driven) or orchestration (central coordinator).

**Q: Trade-offs of event-driven vs request-driven?**  
Event-driven decouples services and improves resilience but adds complexity (ordering, idempotency, debugging). Request-driven is simpler and easier to trace but creates tight coupling and cascading failures.

**Q: How do you handle cache invalidation / stale data?**  
Use TTL for time-insensitive data. For consistency, write-through caching keeps cache and DB in sync. For high-write loads, cache-aside with short TTL and event-based invalidation on updates.

---

## Learning Roadmap

### Phase 1: Foundation (Months 1–3)
- [ ] Solidify OOP, SOLID, Design Patterns
- [ ] Study Clean Architecture and Hexagonal Architecture
- [ ] Learn SQL deeply (joins, indexes, query optimization)
- [ ] Read: *Clean Architecture* — Robert C. Martin

### Phase 2: Distributed Systems (Months 4–6)
- [ ] Study CAP theorem, BASE, eventual consistency
- [ ] Learn Kafka fundamentals (topics, partitions, consumer groups)
- [ ] Understand REST, gRPC, GraphQL
- [ ] Practice system design (Grokking System Design)
- [ ] Read: *Designing Data-Intensive Applications* — Martin Kleppmann

### Phase 3: Cloud & DevOps (Months 7–9)
- [ ] Get AWS Solutions Architect Associate certification
- [ ] Learn Kubernetes fundamentals (Deployments, Services, Ingress)
- [ ] Learn Terraform basics
- [ ] Understand CI/CD pipelines (GitHub Actions, ArgoCD)

### Phase 4: Deeper Architecture (Months 10–12)
- [ ] Study Domain-Driven Design (DDD) basics
- [ ] Implement CQRS in a side project
- [ ] Study security architecture (OAuth2, Zero Trust)
- [ ] Practice writing ADRs

---

## Recommended Resources

### Books

| Book | Author | Why Read It |
|---|---|---|
| Designing Data-Intensive Applications | Martin Kleppmann | Distributed systems bible |
| Clean Architecture | Robert C. Martin | Architecture principles and patterns |
| Design Patterns (GoF) | Gang of Four | Classic design patterns |
| Building Microservices | Sam Newman | Microservices in depth |
| The Phoenix Project | Gene Kim | DevOps culture and thinking |

### Online

| Platform | What to Study |
|---|---|
| Educative.io — Grokking System Design | System design interview patterns |
| Martin Fowler's blog (martinfowler.com) | Patterns, microservices, architecture |
| YouTube: Gaurav Sen, ByteByteGo | System design visual explanations |

---

## Quick Reference: Architecture Trade-off Cheat Sheet

| Decision | Option A | Option B | Choose A When | Choose B When |
|---|---|---|---|---|
| Architecture style | Monolith | Microservices | Small team, early stage | Scale, team autonomy needed |
| Communication | Synchronous (REST) | Asynchronous (Events) | Simple request-response | Decoupling, resilience needed |
| Database | SQL | NoSQL | Complex relations, ACID | High throughput, flexible schema |
| Consistency | Strong | Eventual | Financial, critical correctness | High availability, performance |
| Deployment | Rolling | Blue/Green | Low-risk gradual rollout | Instant rollback needed |
| Caching | Cache-Aside | Write-Through | Read-heavy workloads | Write consistency critical |
| Scaling | Vertical | Horizontal | Quick fix, stateful app | Long-term, stateless app |

---

*Last Updated: 2026-06-18*
