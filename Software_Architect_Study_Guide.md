# Software Architect Study Guide

## Overview

A Software Architect is responsible for making high-level design decisions, defining technical standards, and ensuring a system's structural integrity. This guide covers the knowledge, skills, and progression path needed to become a competent software architect.

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
11. [Soft Skills & Leadership](#soft-skills--leadership)
12. [Architecture Decision Records (ADRs)](#architecture-decision-records-adrs)
13. [Interview Preparation](#interview-preparation)
14. [Learning Roadmap](#learning-roadmap)
15. [Recommended Resources](#recommended-resources)

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
| Stakeholder Communication | Bridge the gap between technical teams and business stakeholders |
| Risk Management | Identify technical risks and define mitigation strategies |
| Documentation | Maintain architectural documentation and decision records |

### Types of Software Architects

```
Enterprise Architect
  └── Sets org-wide technology strategy and standards

Solution Architect
  └── Designs architecture for a specific project or solution

Application Architect
  └── Focuses on a single application's design and patterns

Infrastructure / Cloud Architect
  └── Designs cloud, network, and deployment infrastructure

Data Architect
  └── Designs data models, pipelines, and storage strategies

Security Architect
  └── Defines security protocols, controls, and threat models
```

---

## Core Foundations

### 1. Data Structures & Algorithms

You won't design efficient systems without understanding the underlying building blocks.

- Arrays, Linked Lists, Trees, Graphs, Hash Maps
- Sorting, Searching, Dynamic Programming
- Big-O analysis — time and space complexity
- When to choose which structure (e.g., B-Trees in databases, Graphs in social networks)

### 2. Object-Oriented Design Principles

#### SOLID Principles

```
S — Single Responsibility Principle
    A class should have only one reason to change.

O — Open/Closed Principle
    Open for extension, closed for modification.

L — Liskov Substitution Principle
    Subtypes must be substitutable for their base types.

I — Interface Segregation Principle
    No client should depend on methods it does not use.

D — Dependency Inversion Principle
    Depend on abstractions, not concretions.
```

#### Other Key Principles

- **DRY** — Don't Repeat Yourself
- **KISS** — Keep It Simple, Stupid
- **YAGNI** — You Aren't Gonna Need It
- **Law of Demeter** — Talk only to your immediate friends

### 3. Design Patterns (GoF)

#### Creational Patterns
| Pattern | Use Case |
|---|---|
| Singleton | Single shared instance (config, logger) |
| Factory Method | Delegate object creation to subclasses |
| Abstract Factory | Families of related objects |
| Builder | Complex object construction step-by-step |
| Prototype | Clone existing objects |

#### Structural Patterns
| Pattern | Use Case |
|---|---|
| Adapter | Bridge incompatible interfaces |
| Facade | Simplified interface to a complex subsystem |
| Decorator | Add behavior dynamically |
| Proxy | Controlled access to an object |
| Composite | Tree structures of objects |

#### Behavioral Patterns
| Pattern | Use Case |
|---|---|
| Observer | Event/notification systems |
| Strategy | Swappable algorithms at runtime |
| Command | Encapsulate requests as objects |
| Chain of Responsibility | Pass requests along a handler chain |
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

**Pros**: Simple to develop, deploy, test in early stages  
**Cons**: Scaling bottlenecks, deployment risk, technology lock-in

---

### Microservices Architecture

```
[Client]
   │
[API Gateway]
   ├── [User Service]   ──► [User DB]
   ├── [Order Service]  ──► [Order DB]
   ├── [Payment Service]──► [Payment DB]
   └── [Notification Service]
```

**Key Principles:**
- Each service owns its data (Database per Service)
- Services communicate via APIs or messaging
- Independent deployability
- Bounded contexts (from Domain-Driven Design)

**Challenges:**
- Distributed system complexity
- Network latency and failures
- Data consistency across services
- Service discovery and load balancing

---

### Event-Driven Architecture (EDA)

```
[Producer] ──► [Event Broker (Kafka/RabbitMQ)] ──► [Consumer A]
                                                 ──► [Consumer B]
                                                 ──► [Consumer C]
```

**Patterns:**
- **Event Notification** — Inform others something happened
- **Event-Carried State Transfer** — Carry the full state in the event
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

```
         [REST Adapter]   [CLI Adapter]
               │               │
               └───────┬───────┘
                  [Application Core]
               ┌───────┴───────┐
               │               │
       [DB Adapter]    [Message Adapter]
```

**Goal**: Decouple the core business logic from external concerns (databases, UIs, messaging).

---

### Clean Architecture

```
┌──────────────────────────────────┐
│  Frameworks & Drivers (outermost)│
│  ┌────────────────────────────┐  │
│  │  Interface Adapters        │  │
│  │  ┌──────────────────────┐  │  │
│  │  │  Application/Use Case│  │  │
│  │  │  ┌────────────────┐  │  │  │
│  │  │  │  Entities       │  │  │  │
│  │  │  │  (innermost)    │  │  │  │
│  │  │  └────────────────┘  │  │  │
│  │  └──────────────────────┘  │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
```

**Dependency Rule**: Dependencies only point inward. Inner layers know nothing about outer layers.

---

### CQRS (Command Query Responsibility Segregation)

```
[Client]
   ├── Command (Write) ──► [Write Model] ──► [Write DB]
   │                           │
   │                     [Event Bus]
   │                           │
   └── Query (Read)  ◄── [Read Model]  ◄── [Read DB (Projection)]
```

**When to use**: Read/write asymmetry, complex query needs, event sourcing.

---

### Saga Pattern (Distributed Transactions)

```
Choreography Saga:
[Service A] ──event──► [Service B] ──event──► [Service C]
                                               (on failure: compensating events flow back)

Orchestration Saga:
[Saga Orchestrator]
   ├──► [Service A] (step 1)
   ├──► [Service B] (step 2)
   └──► [Service C] (step 3 — on fail: call compensating transactions)
```

---

## Distributed Systems

### CAP Theorem

```
           Consistency
               /\
              /  \
             /    \
            /  CA  \
           /--------\
          / CP |  AP \
         /     |      \
        /------|-------\
    Partition Tolerance
```

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

```
Strong Consistency
  └── Every read reflects the latest write (expensive)

Eventual Consistency
  └── Reads may be stale but will converge (cheap, highly available)

Read-Your-Writes Consistency
  └── You always see your own writes

Monotonic Read Consistency
  └── Never see older data after seeing newer data
```

### Consensus Algorithms

- **Paxos** — Classic distributed consensus (complex)
- **Raft** — Simpler consensus used in etcd, CockroachDB
- **Gossip Protocol** — Peer-to-peer state propagation (Cassandra, Consul)

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
- **Serverless-first** — Prefer managed services over self-managed

### AWS Architecture Essentials

```
Region
  └── Availability Zones (AZs)
        └── VPC
              ├── Public Subnet  ──► Internet Gateway ──► Internet
              └── Private Subnet ──► NAT Gateway ──► Internet
```

#### Core AWS Services for Architects

| Category | Services |
|---|---|
| Compute | EC2, ECS, EKS, Lambda, Fargate |
| Storage | S3, EBS, EFS, Glacier |
| Database | RDS, DynamoDB, Aurora, ElastiCache, Redshift |
| Networking | VPC, Route 53, CloudFront, API Gateway, ALB/NLB |
| Messaging | SQS, SNS, EventBridge, Kinesis |
| Security | IAM, KMS, Secrets Manager, WAF, Shield |
| Observability | CloudWatch, X-Ray, CloudTrail |

### Well-Architected Framework (5 Pillars)

```
1. Operational Excellence
   └── Automate operations, refine procedures, learn from failures

2. Security
   └── Protect data, manage identities, detect threats

3. Reliability
   └── Recover from failures, scale dynamically, manage change

4. Performance Efficiency
   └── Use resources efficiently, monitor performance

5. Cost Optimization
   └── Avoid unnecessary costs, right-size resources
```

### Multi-Region Architecture

```
[Route 53 with Latency/Failover Routing]
          │
    ┌─────┴─────┐
[Region A]   [Region B]
  (Primary)   (DR/Active)
    │               │
  [RDS]         [RDS Replica]
    └──── Cross-region replication ────┘
```

---

## Data Architecture

### Database Selection Guide

| Use Case | Database Type | Examples |
|---|---|---|
| Structured relational data | Relational (SQL) | PostgreSQL, MySQL, Oracle |
| High-volume key-value lookups | Key-Value Store | Redis, DynamoDB |
| Document storage | Document Store | MongoDB, Couchbase |
| Time-series data | Time-Series DB | InfluxDB, TimescaleDB |
| Graph relationships | Graph DB | Neo4j, Amazon Neptune |
| Full-text search | Search Engine | Elasticsearch, OpenSearch |
| Analytics / OLAP | Columnar / Data Warehouse | Redshift, BigQuery, Snowflake |
| Wide-column large scale | Wide-Column | Cassandra, HBase |

### Data Warehouse vs Data Lake vs Data Lakehouse

```
Data Warehouse
  ├── Structured, processed data
  ├── SQL queries, BI tools
  └── Examples: Redshift, Snowflake

Data Lake
  ├── Raw, unstructured/semi-structured data
  ├── Schema on read
  └── Examples: S3, HDFS

Data Lakehouse
  ├── Combines warehouse + lake benefits
  ├── ACID transactions on raw data
  └── Examples: Delta Lake, Apache Iceberg, Databricks
```

### Data Pipeline Patterns

```
Batch Processing
  └── ETL jobs run periodically (Spark, Glue, dbt)

Stream Processing
  └── Real-time event processing (Kafka Streams, Flink, Kinesis)

Lambda Architecture
  └── Batch layer (accuracy) + Speed layer (low latency) + Serving layer

Kappa Architecture
  └── Single stream processing path (simpler than Lambda)
```

### Sharding Strategies

| Strategy | How It Works | Pros/Cons |
|---|---|---|
| Hash Sharding | Hash(key) % N shards | Even distribution; hard to range query |
| Range Sharding | By key range (A-M, N-Z) | Easy range queries; potential hotspots |
| Directory Sharding | Lookup table maps keys to shards | Flexible; lookup table is single point of failure |
| Geo Sharding | By geographic region | Low latency; uneven data distribution |

---

## Security Architecture

### Security Design Principles

```
Principle of Least Privilege
  └── Grant only the minimum permissions needed

Defense in Depth
  └── Multiple layers of security controls

Zero Trust Architecture
  └── Never trust, always verify — even inside the network

Shift Left Security
  └── Integrate security early in the development lifecycle (DevSecOps)
```

### OWASP Top 10 (Architecture Perspective)

| Risk | Architectural Control |
|---|---|
| Injection | Parameterized queries, input validation at boundaries |
| Broken Authentication | OAuth2/OIDC, MFA, token rotation |
| Sensitive Data Exposure | Encryption at rest/transit, key management (KMS) |
| Broken Access Control | RBAC/ABAC, policy enforcement at API gateway |
| Security Misconfiguration | IaC templates, config auditing, CIS benchmarks |
| SSRF | Network egress controls, allowlists |
| Vulnerable Dependencies | SCA tools (Snyk, Dependabot), SBOM |

### Authentication & Authorization Patterns

```
OAuth 2.0 + OIDC Flow:
[User] ──► [App] ──► [Authorization Server]
                           │
                     [Access Token + ID Token]
                           │
              [App calls Resource Server with token]

RBAC (Role-Based Access Control):
User ──► Role ──► Permissions

ABAC (Attribute-Based Access Control):
User attributes + Resource attributes + Environment ──► Policy Decision
```

### Encryption Strategies

| Type | Method | Examples |
|---|---|---|
| Data at Rest | AES-256, Transparent DB Encryption | AWS KMS, HashiCorp Vault |
| Data in Transit | TLS 1.2+ | HTTPS, mTLS between services |
| End-to-End Encryption | Client-side encryption | Signal Protocol, PGP |

---

## DevOps & Infrastructure

### CI/CD Pipeline Architecture

```
[Code Commit]
      │
[Source Control] ──► [CI Pipeline]
                          ├── Build
                          ├── Unit Tests
                          ├── SAST / Code Quality
                          ├── Container Build
                          └── Push to Registry
                                │
                         [CD Pipeline]
                          ├── Deploy to Staging
                          ├── Integration Tests
                          ├── Performance Tests
                          └── Deploy to Production (Blue/Green or Canary)
```

### Deployment Strategies

| Strategy | Description | Risk |
|---|---|---|
| **Rolling** | Gradually replace old instances with new | Medium |
| **Blue/Green** | Switch traffic from Blue (old) to Green (new) | Low (instant rollback) |
| **Canary** | Route small % of traffic to new version | Low (gradual exposure) |
| **Feature Flags** | Toggle features without redeployment | Very Low |
| **A/B Testing** | Route different user groups to different versions | Low |

### Container & Kubernetes Architecture

```
Kubernetes Cluster
  ├── Control Plane
  │     ├── API Server
  │     ├── Scheduler
  │     ├── Controller Manager
  │     └── etcd (cluster state)
  └── Worker Nodes
        ├── kubelet
        ├── kube-proxy
        └── Pods
              └── Containers
```

**Key Kubernetes Concepts for Architects:**
- **Deployments** — Manage replica sets and rolling updates
- **Services** — Stable network endpoint for pods
- **Ingress** — HTTP routing into the cluster
- **ConfigMaps / Secrets** — Externalize configuration
- **Horizontal Pod Autoscaler** — Scale pods based on CPU/memory
- **Namespaces** — Logical isolation within a cluster

### Infrastructure as Code (IaC)

| Tool | Scope | Approach |
|---|---|---|
| **Terraform** | Multi-cloud, infrastructure | Declarative HCL |
| **Pulumi** | Multi-cloud, infrastructure | Imperative (real code) |
| **AWS CDK** | AWS only | Declarative using Python/TS/Java |
| **Ansible** | Configuration management | Procedural YAML |
| **Helm** | Kubernetes packaging | Templated Kubernetes manifests |

---

## API Design

### REST API Design Principles

```
Resource-based URIs:
  GET    /users           → list users
  GET    /users/{id}      → get user
  POST   /users           → create user
  PUT    /users/{id}      → replace user
  PATCH  /users/{id}      → partial update
  DELETE /users/{id}      → delete user

HTTP Status Codes:
  2xx → Success (200 OK, 201 Created, 204 No Content)
  3xx → Redirection (301 Moved, 304 Not Modified)
  4xx → Client Error (400 Bad Request, 401 Unauthorized, 404 Not Found)
  5xx → Server Error (500 Internal Server Error, 503 Service Unavailable)
```

### GraphQL vs REST vs gRPC

| Dimension | REST | GraphQL | gRPC |
|---|---|---|---|
| Protocol | HTTP/1.1 | HTTP/1.1 or 2 | HTTP/2 |
| Data Format | JSON/XML | JSON | Protocol Buffers |
| Schema | OpenAPI/Swagger | SDL | .proto files |
| Over/Under Fetching | Common issue | Solved by design | N/A |
| Best For | Public APIs, CRUD | Complex data graphs, frontend-driven | Internal microservices, performance |
| Streaming | SSE/WebSocket | Subscriptions | Native bidirectional streaming |

### API Gateway Pattern

```
[Client]
   │
[API Gateway]
   ├── Authentication & Authorization
   ├── Rate Limiting & Throttling
   ├── Request/Response Transformation
   ├── SSL Termination
   ├── Load Balancing
   └── Routing to Backend Services
```

---

## Performance & Scalability

### Scalability Patterns

```
Vertical Scaling (Scale Up)
  └── Add CPU/RAM to existing server (limited ceiling)

Horizontal Scaling (Scale Out)
  └── Add more server instances (preferred for cloud)

Database Read Replicas
  └── Route read traffic to replicas; writes go to primary

Sharding
  └── Partition data across multiple database instances

CQRS
  └── Separate optimized read models from write models
```

### Caching Strategies

| Strategy | Description | Example |
|---|---|---|
| **Cache-Aside** | App checks cache first, loads from DB on miss | Redis + App |
| **Write-Through** | Write to cache and DB simultaneously | Consistent but slower writes |
| **Write-Behind** | Write to cache, async flush to DB | Fast writes, risk of loss |
| **Read-Through** | Cache handles DB loading transparently | Simplified app code |
| **Refresh-Ahead** | Proactively refresh cache before expiry | Low latency for hot data |

### Load Balancing Algorithms

| Algorithm | Description | Use Case |
|---|---|---|
| Round Robin | Requests distributed evenly in sequence | Uniform servers |
| Least Connections | Routes to server with fewest active connections | Long-lived connections |
| IP Hash | Client IP determines server | Session affinity |
| Weighted | Servers get traffic proportional to weight | Heterogeneous servers |

### Performance Anti-Patterns

- **N+1 Query Problem** — Execute 1 query + N sub-queries; fix with JOIN or batch loading
- **Thundering Herd** — Many requests hit DB simultaneously on cache expiry; fix with jitter or mutex
- **Chatty APIs** — Too many small requests; fix with batching or GraphQL
- **Synchronous Chains** — Long chains of synchronous calls; fix with async/event-driven patterns

---

## Soft Skills & Leadership

### Communication Skills for Architects

```
Stakeholder Communication:
  ├── Business stakeholders → ROI, risk, timelines (no jargon)
  ├── Dev team → Technical depth, patterns, trade-offs
  └── Operations → Runbooks, SLOs, incident response

Diagramming Levels (C4 Model):
  Level 1 (Context) ──► Who uses the system and what external systems interact
  Level 2 (Container)──► High-level tech decisions (apps, DBs, queues)
  Level 3 (Component)──► Internal components within a container
  Level 4 (Code) ──────► Class/function level (rarely needed for architects)
```

### C4 Model (Architecture Diagrams)

```
C1 - System Context Diagram
  └── Your system + users + external systems

C2 - Container Diagram
  └── Web App, API, Database, Mobile App (technology labels)

C3 - Component Diagram
  └── Internal modules/services within a container

C4 - Code Diagram (optional)
  └── UML class diagrams for critical components
```

### Trade-off Analysis Framework

When evaluating architectural options, always assess:

```
1. Functional Fit        → Does it meet the requirements?
2. Non-Functional Fit    → Performance, scalability, security, reliability
3. Cost                  → Licensing, infrastructure, operational cost
4. Team Expertise        → Learning curve and existing skills
5. Ecosystem & Support   → Community, vendor support, maturity
6. Reversibility         → How hard is it to change later?
```

---

## Architecture Decision Records (ADRs)

ADRs document significant architectural decisions. A well-maintained ADR log is a sign of a mature engineering culture.

### ADR Template

```markdown
# ADR-[number]: [Short title]

**Date**: YYYY-MM-DD
**Status**: Proposed | Accepted | Deprecated | Superseded

## Context
What is the problem or opportunity? What forces are at play?

## Decision
What is the change we're proposing or have made?

## Consequences
What becomes easier or harder as a result of this decision?
What are the trade-offs?

## Alternatives Considered
What other options were evaluated? Why were they rejected?
```

---

## Interview Preparation

### Common System Design Interview Topics

| Topic | Key Concepts to Know |
|---|---|
| Design URL Shortener | Hashing, KV store, redirection, analytics |
| Design Twitter/Feed | Fan-out on write vs read, timeline aggregation |
| Design Uber/Ride-sharing | Geo-indexing, real-time location, matching algorithms |
| Design Netflix | CDN, video encoding, recommendation engine |
| Design a Chat System | WebSockets, message queues, presence detection |
| Design a Search Engine | Inverted index, crawling, ranking |
| Design a Rate Limiter | Token bucket, leaky bucket, Redis sliding window |
| Design a Notification System | Push/pull, fan-out, at-least-once delivery |
| Design a Payment System | Idempotency, distributed transactions, ACID guarantees |

### System Design Interview Framework

```
Step 1: Clarify Requirements (5 min)
  ├── Functional requirements (what it does)
  ├── Non-functional requirements (scale, latency, availability)
  └── Constraints and assumptions

Step 2: Estimate Scale (3 min)
  ├── DAU / MAU
  ├── QPS (read/write ratio)
  └── Storage needs

Step 3: High-Level Design (10 min)
  ├── Identify core components
  ├── Draw the basic flow
  └── Choose data stores

Step 4: Deep Dive (15 min)
  ├── Pick 2-3 critical components to detail
  ├── Discuss trade-offs
  └── Address bottlenecks

Step 5: Summarize & Trade-offs (5 min)
  ├── Summarize design
  ├── Discuss what you'd improve
  └── Mention unresolved constraints
```

### Architecture Interview Questions

**Conceptual:**
- What is the difference between horizontal and vertical scaling?
- When would you choose microservices over a monolith?
- How does eventual consistency differ from strong consistency?
- What is the CAP theorem, and how does it affect system design?
- What is the difference between synchronous and asynchronous communication?

**Design:**
- How would you design a system that handles 1 million requests per second?
- How would you ensure zero-downtime deployments?
- How would you handle distributed transactions across microservices?
- How would you design for high availability with RTO of < 1 minute?

**Trade-off:**
- SQL vs NoSQL — when to choose which?
- REST vs gRPC — when to choose which?
- Event-driven vs request-driven — what drives the choice?
- Cache invalidation — how do you handle stale data?

---

## Learning Roadmap

### Phase 1: Foundation (Months 1–3)
- [ ] Solidify OOP, SOLID, Design Patterns
- [ ] Study Clean Architecture and Hexagonal Architecture
- [ ] Learn SQL deeply (joins, indexes, query optimization)
- [ ] Read: *Clean Architecture* — Robert C. Martin
- [ ] Read: *Design Patterns* — GoF

### Phase 2: Distributed Systems (Months 4–6)
- [ ] Study CAP theorem, BASE, eventual consistency
- [ ] Learn Kafka fundamentals (topics, partitions, consumer groups)
- [ ] Understand REST, gRPC, GraphQL
- [ ] Practice system design (Grokking System Design or Educative.io)
- [ ] Read: *Designing Data-Intensive Applications* — Martin Kleppmann

### Phase 3: Cloud & DevOps (Months 7–9)
- [ ] Get AWS Solutions Architect Associate certification
- [ ] Learn Kubernetes fundamentals (kubectl, Deployments, Services, Ingress)
- [ ] Learn Terraform (provision cloud resources with IaC)
- [ ] Understand CI/CD pipelines (GitHub Actions, Jenkins, ArgoCD)
- [ ] Read: *The Phoenix Project* — Gene Kim

### Phase 4: Advanced Architecture (Months 10–12)
- [ ] Study Domain-Driven Design (DDD)
- [ ] Implement Event Sourcing + CQRS in a side project
- [ ] Study security architecture (OAuth2, Zero Trust, mTLS)
- [ ] Practice writing ADRs for past decisions
- [ ] Read: *Software Architecture: The Hard Parts* — Neal Ford, Mark Richards

### Phase 5: Leadership & Influence (Ongoing)
- [ ] Lead a technical design review at work
- [ ] Mentor junior developers
- [ ] Present architecture proposals to stakeholders
- [ ] Contribute to open-source or write technical blogs
- [ ] Read: *The Staff Engineer's Path* — Tanya Reilly

---

## Recommended Resources

### Books

| Book | Author | Why Read It |
|---|---|---|
| Designing Data-Intensive Applications | Martin Kleppmann | Distributed systems bible |
| Clean Architecture | Robert C. Martin | Architecture principles and patterns |
| Software Architecture: The Hard Parts | Ford & Richards | Microservices trade-offs |
| Design Patterns (GoF) | Gang of Four | Classic design patterns |
| Domain-Driven Design | Eric Evans | Modeling complex business domains |
| The Phoenix Project | Gene Kim | DevOps culture and thinking |
| Building Microservices | Sam Newman | Microservices in depth |
| The Staff Engineer's Path | Tanya Reilly | Leadership beyond coding |
| Site Reliability Engineering | Google SRE Team | Reliability engineering at scale |

### Online Courses & Platforms

| Platform | What to Study |
|---|---|
| AWS Training (aws.amazon.com/training) | AWS Solutions Architect certification |
| Educative.io — Grokking System Design | System design interview patterns |
| Martin Fowler's blog (martinfowler.com) | Patterns, microservices, architecture |
| Coursera — Cloud Architecture | Multi-cloud architecture principles |
| YouTube: Gaurav Sen, ByteByteGo | System design visual explanations |

### Certifications

| Certification | Value | Difficulty |
|---|---|---|
| AWS Solutions Architect Associate | High — industry standard | Medium |
| AWS Solutions Architect Professional | Very High | Hard |
| Google Professional Cloud Architect | High | Hard |
| CNCF Certified Kubernetes Administrator (CKA) | High for cloud-native | Medium |
| TOGAF (Enterprise Architecture) | High for enterprise roles | Medium |

---

## Quick Reference: Architecture Trade-off Cheat Sheet

| Decision | Option A | Option B | Choose A When | Choose B When |
|---|---|---|---|---|
| Architecture style | Monolith | Microservices | Small team, early stage | Scale, team autonomy needed |
| Communication | Synchronous (REST) | Asynchronous (Events) | Simple request-response | Decoupling, resilience needed |
| Database | SQL (RDBMS) | NoSQL | Complex relations, ACID | High throughput, flexible schema |
| Consistency | Strong | Eventual | Financial, critical correctness | High availability, performance |
| Deployment | Rolling | Blue/Green | Low-risk, gradual | Instant rollback needed |
| Caching | Cache-Aside | Write-Through | Read-heavy workloads | Write consistency critical |
| Scaling | Vertical | Horizontal | Quick fix, stateful app | Long-term, stateless app |

---

*Last Updated: 2026-06-04*
