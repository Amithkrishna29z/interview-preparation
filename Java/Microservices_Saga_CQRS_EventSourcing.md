# Microservices: Saga Pattern, CQRS, and Event Sourcing

## Table of Contents
1. [Distributed Transactions Problem](#1-distributed-transactions-problem)
2. [BASE vs ACID](#2-base-vs-acid)
3. [Saga Pattern](#3-saga-pattern)
   - [What is a Saga](#31-what-is-a-saga)
   - [Choreography-Based Saga](#32-choreography-based-saga)
   - [Orchestration-Based Saga](#33-orchestration-based-saga)
   - [Choreography vs Orchestration](#34-choreography-vs-orchestration)
   - [Compensating Transactions](#35-compensating-transactions)
   - [Saga Failure Modes](#36-saga-failure-modes)
   - [Outbox Pattern](#37-outbox-pattern)
   - [Axon Framework Saga](#38-implementing-saga-with-axon-framework)
4. [CQRS](#4-cqrs-command-query-responsibility-segregation)
   - [What is CQRS](#41-what-is-cqrs)
   - [Simple CQRS (Same Database)](#42-simple-cqrs-same-database)
   - [CQRS with Separate Read/Write Stores](#43-cqrs-with-separate-readwrite-stores)
   - [Benefits and Challenges](#44-benefits-and-challenges-of-cqrs)
5. [Event Sourcing](#5-event-sourcing)
   - [What is Event Sourcing](#51-what-is-event-sourcing)
   - [Core Concepts](#52-core-concepts)
   - [Event Store Design](#53-event-store-design)
   - [Aggregate Implementation](#54-java-aggregate-implementation)
   - [Projections](#55-projections-and-read-models)
   - [Snapshots](#56-snapshots)
   - [Event Versioning](#57-event-versioning)
   - [Benefits and Challenges](#58-benefits-and-challenges)
6. [Event Sourcing + CQRS Together](#6-event-sourcing--cqrs-together)
7. [When to Use What](#7-when-to-use-what)
8. [Spring + Axon Example](#8-spring--axon-framework-example)
9. [Kafka for Saga and Event Sourcing](#9-kafka-for-saga-and-event-sourcing)
10. [Interview Questions & Answers](#10-interview-questions--answers)
11. [Quick Reference Cheat Sheet](#11-quick-reference-cheat-sheet)

---

## 1. Distributed Transactions Problem

### 1.1 Monolith vs Microservices Transaction Model

**In a monolith**, a single ACID transaction spans all operations — one database, one rollback.

**In microservices**, each service owns its own database:
```
OrderService     → orders_db     (PostgreSQL)
InventoryService → inventory_db  (MySQL)
PaymentService   → payments_db   (MongoDB)
```
**The core problem**: How do you guarantee that either ALL operations succeed or ALL are rolled back when each lives in a different database?

### 1.2 Two-Phase Commit (2PC)

2PC uses a **coordinator** and **participants**. Phase 1: coordinator asks everyone "Can you commit?" and participants lock resources. Phase 2: if all say YES → commit; any says NO → rollback.

**Problems with 2PC:**
- **Blocking** — participants hold locks until Phase 2; coordinator crash leaves them stuck.
- **SPOF** — coordinator is a bottleneck; all wait for the slowest participant.
- **Not cloud-native** — DynamoDB, MongoDB, and third-party APIs don't support XA transactions.

> **Common junior mistake:** Reaching for 2PC across microservices. Use a Saga instead.

---

## 2. BASE vs ACID

### 2.1 ACID (Traditional Databases)

| Property | Meaning |
|---|---|
| **Atomicity** | All succeed or all fail |
| **Consistency** | DB moves between valid states |
| **Isolation** | Concurrent transactions behave as serial |
| **Durability** | Committed data survives crashes |

### 2.2 BASE (Distributed Systems)

| Property | Meaning |
|---|---|
| **Basically Available** | System stays available but may serve stale data |
| **Soft State** | State may change over time due to propagation |
| **Eventually Consistent** | All replicas converge given no new updates |

**The Trade-off:** Microservices trade ACID for availability. Each service is ACID locally; across services the system is only eventually consistent.

---

## 3. Saga Pattern

### 3.1 What is a Saga?

A **Saga** is a sequence of local transactions where each step updates its own database and publishes an event to trigger the next step. If any step fails, **compensating transactions** undo the completed steps.

**Key insight:** No distributed lock. Instead of preventing inconsistency, sagas *repair* it.

**Order Processing Saga Example:**
```
Step 1: OrderService       → Create order (PENDING)
Step 2: InventoryService   → Reserve items
Step 3: PaymentService     → Charge customer
Step 4: ShippingService    → Schedule delivery
Step 5: OrderService       → Mark order CONFIRMED

Compensation (if Step 3 fails):
  Compensate Step 2: InventoryService → Release reserved items
  Compensate Step 1: OrderService     → Mark order CANCELLED
```

---

### 3.2 Choreography-Based Saga

No central orchestrator. Each service listens for events and reacts autonomously.

**Architecture:**
```
OrderService       publishes: OrderCreatedEvent
InventoryService   listens: OrderCreatedEvent → publishes: StockReservedEvent OR StockReservationFailedEvent
PaymentService     listens: StockReservedEvent → publishes: PaymentProcessedEvent OR PaymentFailedEvent
ShippingService    listens: PaymentProcessedEvent → publishes: ShipmentScheduledEvent
```

**Java + Spring + Kafka Implementation:**

```java
// Events (use records for immutability)
public record OrderCreatedEvent(String orderId, String customerId,
    String productId, int quantity, BigDecimal totalAmount) {}
public record StockReservedEvent(String orderId, String productId, int quantity) {}
public record PaymentFailedEvent(String orderId, String reason) {}

// OrderService — creates order and listens for compensation events
@Service
public class OrderService {
    @Autowired private OrderRepository orderRepository;
    @Autowired private KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public String createOrder(CreateOrderCommand cmd) {
        Order order = new Order(UUID.randomUUID().toString(),
            cmd.customerId(), cmd.productId(), cmd.quantity(), OrderStatus.PENDING);
        orderRepository.save(order);
        kafkaTemplate.send("order-events", order.getId(),
            new OrderCreatedEvent(order.getId(), order.getCustomerId(),
                order.getProductId(), order.getQuantity(), order.getTotalAmount()));
        return order.getId();
    }

    @KafkaListener(topics = "payment-events", groupId = "order-service")
    public void onPaymentFailed(PaymentFailedEvent event) {
        orderRepository.findById(event.orderId()).ifPresent(order -> {
            order.setStatus(OrderStatus.CANCELLED);
            orderRepository.save(order);
        });
    }
}

// InventoryService — listens for order events, reserves stock or fires failure
@Service
public class InventoryService {
    @Autowired private InventoryRepository inventoryRepository;
    @Autowired private KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        inventoryRepository.findByProductId(event.productId()).ifPresentOrElse(
            inventory -> {
                if (inventory.getAvailableQuantity() >= event.quantity()) {
                    inventory.setReservedQuantity(
                        inventory.getReservedQuantity() + event.quantity());
                    inventoryRepository.save(inventory);
                    kafkaTemplate.send("inventory-events", event.orderId(),
                        new StockReservedEvent(event.orderId(), event.productId(), event.quantity()));
                } else {
                    kafkaTemplate.send("inventory-events", event.orderId(),
                        new StockReservationFailedEvent(event.orderId(), "Insufficient stock"));
                }
            },
            () -> kafkaTemplate.send("inventory-events", event.orderId(),
                new StockReservationFailedEvent(event.orderId(), "Product not found"))
        );
    }

    @KafkaListener(topics = "payment-events", groupId = "inventory-service")
    @Transactional
    public void onPaymentFailed(PaymentFailedEvent event) {
        // Release reserved stock (compensating transaction)
    }
}
```

**Pros and Cons:**

| Pros | Cons |
|---|---|
| Loose coupling — services don't know each other | Distributed logic — hard to see full saga flow |
| No single point of failure | Risk of cyclic event dependencies |
| Easy to add new participants | Hard to debug — must trace events across topics |
| Scales well | Harder to enforce step ordering |

---

### 3.3 Orchestration-Based Saga

A **central saga orchestrator** sends explicit commands to participants and waits for replies. The orchestrator is a state machine that knows the full flow.

**State Machine:**
```
PENDING → RESERVING_INVENTORY → PROCESSING_PAYMENT → SCHEDULING_SHIPMENT → COMPLETED

Compensation:
  PAYMENT_FAILED → COMPENSATING_INVENTORY → CANCELLED
```

**Java Implementation — Manual Orchestrator:**

```java
@Entity
public class OrderSaga {
    @Id private String sagaId;
    private String orderId;
    private SagaState state;
    private String failureReason;

    public enum SagaState {
        PENDING, RESERVING_INVENTORY, INVENTORY_RESERVED,
        PROCESSING_PAYMENT, PAYMENT_PROCESSED, SCHEDULING_SHIPMENT,
        COMPLETED, COMPENSATING_INVENTORY, COMPENSATING_PAYMENT, CANCELLED
    }
}

@Service
public class OrderSagaOrchestrator {
    @Autowired private OrderSagaRepository sagaRepository;
    @Autowired private KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public void startSaga(String orderId, OrderDetails details) {
        OrderSaga saga = new OrderSaga(UUID.randomUUID().toString(), orderId, SagaState.PENDING);
        sagaRepository.save(saga);
        saga.setState(SagaState.RESERVING_INVENTORY);
        sagaRepository.save(saga);
        kafkaTemplate.send("inventory-commands", saga.getOrderId(),
            new ReserveStockCommand(saga.getSagaId(), saga.getOrderId(),
                details.productId(), details.quantity()));
    }

    @KafkaListener(topics = "saga-replies", groupId = "saga-orchestrator")
    @Transactional
    public void handleReply(SagaReply reply) {
        OrderSaga saga = sagaRepository.findBySagaId(reply.sagaId()).orElseThrow();
        switch (saga.getState()) {
            case RESERVING_INVENTORY -> handleInventoryReply(saga, reply);
            case PROCESSING_PAYMENT  -> handlePaymentReply(saga, reply);
            case SCHEDULING_SHIPMENT -> handleShipmentReply(saga, reply);
            case COMPENSATING_INVENTORY -> handleInventoryCompensationReply(saga, reply);
            default -> log.warn("Unexpected reply in state {}", saga.getState());
        }
    }

    private void handleInventoryReply(OrderSaga saga, SagaReply reply) {
        if (reply.success()) {
            saga.setState(SagaState.PROCESSING_PAYMENT);
            sagaRepository.save(saga);
            kafkaTemplate.send("payment-commands", saga.getOrderId(),
                new ProcessPaymentCommand(saga.getSagaId(), saga.getOrderId(), reply.amount()));
        } else {
            saga.setState(SagaState.CANCELLED);
            saga.setFailureReason("Inventory failed: " + reply.reason());
            sagaRepository.save(saga);
        }
    }
    // Other handlers: advance state on success, begin compensation on failure
}
```

**Pros and Cons:**

| Pros | Cons |
|---|---|
| Saga logic in ONE place — easy to understand | Orchestrator coupled to all participants |
| Easier to debug — single state table | Orchestrator can be a bottleneck / SPOF |
| Easier to test with mocks | More boilerplate for commands/replies |
| Clear audit trail | Services become less autonomous |

---

### 3.4 Choreography vs Orchestration

| Aspect | Choreography | Orchestration |
|---|---|---|
| Logic location | Distributed across services | Central orchestrator |
| Coupling | Event-based (loose) | Command/reply (tighter) |
| Debugging | Hard (trace events across topics) | Easy (inspect saga state table) |
| Visibility | Low | High |
| Use case | Simple flows (2–3 steps) | Complex multi-step workflows |
| SPOF risk | None | Orchestrator (mitigate with HA) |

---

### 3.5 Compensating Transactions

A compensating transaction **semantically undoes** a completed local transaction. It is NOT a database rollback — the original transaction was committed. Compensation is a new business operation.

**Key properties:**
- **Idempotent** — executing multiple times = same result as once (messages may be delivered twice).
- **Semantically correct** — must undo the *business effect*.
- **May not be fully reversible** — e.g. sending an email; the compensation is a corrective action.

**Transaction Types in a Saga:**

| Type | Description | Example |
|---|---|---|
| **Compensable** | Can be undone | Reserve stock → Release stock |
| **Pivot** | The go/no-go point; if it commits, saga will complete | Charge credit card |
| **Retriable** | Guaranteed to succeed eventually; no compensation needed | Send confirmation email |

**Common Examples:**

| Original | Compensating |
|---|---|
| Reserve inventory | Release reservation |
| Charge credit card | Issue refund |
| Create order (PENDING) | Cancel order |
| Send confirmation email | Send cancellation email (cannot undo) |

---

### 3.6 Saga Failure Modes

**Semantic lock** — mark data as "in-progress" to prevent concurrent saga conflicts:
```java
@Transactional
public void startOrderProcessing(String orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    if (order.getStatus() != OrderStatus.PENDING)
        throw new OrderAlreadyBeingProcessedException(orderId);
    order.setStatus(OrderStatus.PROCESSING); // semantic lock
    orderRepository.save(order);
}
```

**Idempotency check** — guard against duplicate event delivery:
```java
@KafkaListener(topics = "inventory-events", groupId = "payment-service")
@Transactional
public void onStockReserved(StockReservedEvent event) {
    if (paymentRepository.existsByOrderId(event.orderId())) {
        return; // already processed — skip
    }
    // Process payment...
}
```

> **Common junior mistake:** Assuming each event arrives exactly once. Messaging is at-least-once — a retry would charge the customer twice without an idempotency check.

---

### 3.7 Outbox Pattern

**The problem:** Updating the DB and publishing to Kafka are two separate operations. If the DB commits but Kafka fails, the event is lost and the saga never starts.

```java
// WRONG — not atomic:
@Transactional
public void createOrder(CreateOrderCommand cmd) {
    orderRepository.save(order);          // DB commit
    kafkaTemplate.send("events", event);  // may fail AFTER DB commit
}
```

**The solution:** Write the event to an `outbox` table in the **same transaction** as business data. A relay (Debezium or polling publisher) reads unpublished outbox rows and pushes them to Kafka.

```sql
CREATE TABLE outbox (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id    VARCHAR(255) NOT NULL,
    event_type      VARCHAR(255) NOT NULL,
    payload         JSONB NOT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    published       BOOLEAN NOT NULL DEFAULT FALSE
);
```

```java
@Transactional  // One transaction covers BOTH saves
public String createOrder(CreateOrderCommand cmd) throws JsonProcessingException {
    Order order = new Order(...);
    orderRepository.save(order);

    OutboxEvent outbox = new OutboxEvent(order.getId(), "Order", "OrderCreatedEvent",
        objectMapper.writeValueAsString(new OrderCreatedEvent(...)));
    outboxRepository.save(outbox);
    // If anything fails, both order AND outbox row are rolled back
    return order.getId();
}

// Simple polling relay (Debezium CDC is the production alternative)
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishPendingEvents() {
    List<OutboxEvent> pending =
        outboxRepository.findTop100ByPublishedFalseOrderByCreatedAtAsc();
    for (OutboxEvent event : pending) {
        try {
            kafkaTemplate.send(topicFor(event.getEventType()),
                event.getAggregateId(), event.getPayload()).get();
            event.setPublished(true);
            outboxRepository.save(event);
        } catch (Exception ex) {
            log.error("Failed to publish {}: {}", event.getId(), ex.getMessage());
            // Will retry on next poll
        }
    }
}
```

**Debezium CDC** (production-grade): instead of polling, Debezium reads PostgreSQL's WAL and publishes outbox rows to Kafka via its `EventRouter` transform — zero polling overhead and sub-second latency.

---

### 3.8 Implementing Saga with Axon Framework

Axon provides first-class saga support via annotations — no need to hand-build a state machine.

```java
@Saga
public class OrderProcessingSaga {
    @Autowired private transient CommandGateway commandGateway;
    private String orderId;

    @StartSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void handle(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        SagaLifecycle.associateWith("orderId", orderId);
        commandGateway.send(new ReserveStockCommand(orderId,
            event.getProductId(), event.getQuantity()));
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void handle(StockReservedEvent event) {
        commandGateway.send(new ProcessPaymentCommand(orderId, event.getAmount()));
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void handle(PaymentFailedEvent event) {
        commandGateway.send(new ReleaseStockCommand(orderId));
    }

    @EndSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void handle(ShipmentScheduledEvent event) {
        commandGateway.send(new ConfirmOrderCommand(orderId));
    }
}
```

Axon automatically persists saga state between events and resumes after restarts.

---

## 4. CQRS (Command Query Responsibility Segregation)

### 4.1 What is CQRS?

CQRS separates the **write model** (commands that change state) from the **read model** (queries that return data), using different objects and often different databases.

| | CQS | CQRS |
|---|---|---|
| Level | Method level | Architecture level |
| Idea | A method either changes state OR returns data, never both | Separate entire command stack from query stack |

---

### 4.2 Simple CQRS (Same Database)

Separate command/query handlers, single database — the simplest form:

```java
// Command handler — writes
@Service @Transactional
public class ProductCommandHandler {
    @Autowired private ProductRepository productRepository;
    @Autowired private ApplicationEventPublisher eventPublisher;

    public String handle(CreateProductCommand cmd) {
        Product product = new Product(UUID.randomUUID().toString(),
            cmd.name(), cmd.description(), cmd.price(), cmd.initialStock());
        productRepository.save(product);
        eventPublisher.publishEvent(new ProductCreatedEvent(product.getId(), product.getName()));
        return product.getId();
    }
}

// Query handler — reads only
@Service @Transactional(readOnly = true)
public class ProductQueryHandler {
    @Autowired private ProductRepository productRepository;

    public ProductDto handle(GetProductQuery query) {
        return productRepository.findById(query.productId())
            .map(this::toDto)
            .orElseThrow(() -> new ProductNotFoundException(query.productId()));
    }
}
```

---

### 4.3 CQRS with Separate Read/Write Stores

The full pattern uses different databases optimized per side:

```
Write Side: CommandHandler → PostgreSQL (normalized, strong consistency)
                │ Domain Events
                ↓
           Projector → Elasticsearch (denormalized, optimized for search)
Read Side:  QueryHandler → Elasticsearch
```

```java
// Projector — keeps denormalized read model in sync
@Component
public class ProductReadModelProjector {
    @Autowired private ElasticsearchOperations elasticsearch;

    @EventListener
    public void on(ProductCreatedEvent event) {
        elasticsearch.save(new ProductDocument(event.productId(), event.name(),
            event.description(), event.price(), event.stock()));
    }

    @EventListener
    public void on(ProductPriceUpdatedEvent event) {
        ProductDocument doc = elasticsearch.get(event.productId(), ProductDocument.class);
        if (doc != null) { doc.setPrice(event.newPrice()); elasticsearch.save(doc); }
    }
}
```

---

### 4.4 Benefits and Challenges of CQRS

**Benefits:**

| Benefit | Explanation |
|---|---|
| Independent scaling | Read side scales independently (reads typically 10x writes) |
| Optimized read models | Denormalized, no joins needed |
| Multiple read models | Same events → Elasticsearch, Redis, reporting DB |
| Technology flexibility | Best tool per job |

**Challenges:**

| Challenge | Explanation |
|---|---|
| Eventual consistency | Read model lags behind writes by ms–seconds |
| Increased complexity | Two models + sync logic — overkill for simple CRUD |
| Read-after-write problem | User creates something, queries immediately — may see stale data |

**Read-after-write mitigation:** Return the written data directly from the command endpoint instead of querying the (potentially lagging) read model.

**When to use CQRS:** Read/write loads differ significantly; multiple read models needed; combined with Event Sourcing.  
**When NOT to use:** Simple CRUD; small team; strong read-after-write consistency required.

---

## 5. Event Sourcing

### 5.1 What is Event Sourcing?

**Traditional:** Store only current state. History is lost.
```
orders: id=123, status=SHIPPED, total=99.99
```

**Event Sourcing:** Store every event that occurred. Current state is derived by replay.
```
order_events:
  1: OrderPlaced   {address:"123 Main St", total:99.99}   2024-01-10
  2: PaymentTaken  {amount:99.99, card:"****1234"}         2024-01-10
  3: OrderPicked   {warehouse:"WH-01"}                     2024-01-12
  4: OrderShipped  {courier:"FedEx", tracking:"ABC123"}    2024-01-15
```

**Analogy:** A bank account balance isn't stored — it's computed from the ledger of all transactions. The ledger IS the truth.

---

### 5.2 Core Concepts

- **Event:** Immutable fact about something that happened. Named in past tense. Never modified.
- **Aggregate:** Cluster of domain objects as a single consistency unit. Rebuilds state by replaying events.
- **Event Store:** Append-only database. Never update, never delete.
- **Projection:** Read model built by processing events (also: view model, query model).
- **Snapshot:** Periodic aggregate state save to avoid replaying thousands of events.

```java
// Events are immutable — records are ideal
public record OrderPlacedEvent(String orderId, String customerId,
    List<OrderItem> items, String deliveryAddress,
    BigDecimal totalAmount, Instant occurredAt) implements DomainEvent {}

public record OrderShippedEvent(String orderId, String courierName,
    String trackingNumber, Instant occurredAt) implements DomainEvent {}
```

---

### 5.3 Event Store Design

```sql
CREATE TABLE events (
    id              BIGSERIAL PRIMARY KEY,
    aggregate_id    VARCHAR(255) NOT NULL,
    aggregate_type  VARCHAR(255) NOT NULL,
    event_type      VARCHAR(255) NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB,           -- correlation_id, user_id, etc.
    version         INTEGER NOT NULL,
    global_sequence BIGSERIAL,
    occurred_at     TIMESTAMP NOT NULL DEFAULT NOW(),
    -- Optimistic concurrency: prevents two writes at the same version
    CONSTRAINT uq_aggregate_version UNIQUE (aggregate_id, version)
);

CREATE TABLE snapshots (
    aggregate_id   VARCHAR(255) PRIMARY KEY,
    aggregate_type VARCHAR(255) NOT NULL,
    state          JSONB NOT NULL,
    version        INTEGER NOT NULL,
    created_at     TIMESTAMP NOT NULL DEFAULT NOW()
);
```

**Optimistic concurrency:** The `UNIQUE(aggregate_id, version)` constraint means a concurrent write at the same version throws `DuplicateKeyException`, which you translate to `OptimisticConcurrencyException` — the caller reloads and retries. This is the event-sourcing equivalent of JPA's `@Version`.

---

### 5.4 Java Aggregate Implementation

The key rule: **state changes happen only inside `apply()` methods**, not in business methods.

```java
public class BankAccount extends AggregateRoot {
    private String accountHolderId;
    private BigDecimal balance;
    private AccountStatus status;

    private BankAccount() {}

    public static BankAccount open(String accountId, String holderId,
                                   BigDecimal initialDeposit, String currency) {
        BankAccount account = new BankAccount();
        account.raiseEvent(new AccountOpenedEvent(accountId, holderId, initialDeposit, currency));
        return account;
    }

    // Business method — validates, then raises event. Does NOT touch state directly.
    public void deposit(BigDecimal amount) {
        if (status != AccountStatus.ACTIVE) throw new AccountNotActiveException(id);
        if (amount.compareTo(BigDecimal.ZERO) <= 0) throw new InvalidAmountException("Must be positive");
        raiseEvent(new MoneyDepositedEvent(id, amount, balance.add(amount)));
    }

    public void withdraw(BigDecimal amount) {
        if (amount.compareTo(balance) > 0) throw new InsufficientFundsException(id, balance, amount);
        raiseEvent(new MoneyWithdrawnEvent(id, amount, balance.subtract(amount)));
    }

    // apply() methods — state mutations ONLY happen here
    // Called on new events AND when replaying from the event store
    private void apply(AccountOpenedEvent event) {
        this.id = event.accountId();
        this.accountHolderId = event.holderId();
        this.balance = event.initialDeposit();
        this.status = AccountStatus.ACTIVE;
    }

    private void apply(MoneyDepositedEvent event)  { this.balance = event.newBalance(); }
    private void apply(MoneyWithdrawnEvent event)  { this.balance = event.newBalance(); }
    private void apply(AccountClosedEvent event)   { this.status = AccountStatus.CLOSED; }
}
```

> **Common junior mistake:** Writing `this.balance = newBalance` inside `withdraw()`. In event sourcing, state changes happen ONLY inside `apply()`. Bypassing this breaks event replay.

---

### 5.5 Projections and Read Models

```java
@Component
public class AccountBalanceProjection {
    @Autowired private AccountReadModelRepository readModelRepository;

    @EventListener @Transactional
    public void on(AccountOpenedEvent event) {
        readModelRepository.save(new AccountReadModel(
            event.accountId(), event.holderId(), event.initialDeposit(), "ACTIVE"));
    }

    @EventListener @Transactional
    public void on(MoneyDepositedEvent event) {
        readModelRepository.findById(event.accountId()).ifPresent(model -> {
            model.setBalance(event.newBalance());
            readModelRepository.save(model);
        });
    }
}
```

**Multiple projections from the same events:** A `TransactionHistoryProjection` can append a `TransactionRecord` per event for a statement view — independently of the balance projection.

**Rebuilding a projection:** Since all events are preserved, replay from the beginning to rebuild any projection at any time.

---

### 5.6 Snapshots

A snapshot saves the aggregate's serialized state at a given version. Loading becomes: restore latest snapshot → replay only events after it (e.g., replay 50 instead of 1000).

Trigger strategies: every N events (`version % 100 == 0`), on-demand, or time-based.

---

### 5.7 Event Versioning

Events are immutable and stored forever — never modify them. When the schema must change:

| Strategy | When to use |
|---|---|
| Additive fields only | Simple field additions — safest, no migration |
| Upcasters | Structural changes — transform old events to new format on load |
| Multiple event versions | Keep both `UserRegisteredV1` and `V2` with separate handlers |
| Event migration | Bulk transform of stored events (risky, rarely recommended) |

---

### 5.8 Benefits and Challenges

**Benefits:**

| Benefit | Explanation |
|---|---|
| Complete audit trail | Every change recorded — who changed what and when |
| Temporal queries | "What was the balance on March 1st?" — replay to that date |
| Multiple projections | Same events → different read models |
| Debugging | Reproduce any bug by replaying the exact event sequence |

**Challenges:**

| Challenge | Explanation |
|---|---|
| Query complexity | Must build projections — can't query current state directly |
| Schema evolution | Upcasting adds complexity over time |
| Eventual consistency | Read models lag behind writes |
| Not for simple CRUD | Massive overhead without audit requirements |

---

## 6. Event Sourcing + CQRS Together

They combine naturally: Event Sourcing generates events; CQRS uses those events to build optimized read models.

```
CLIENT
  │ Commands                  │ Queries
  ↓                           ↓
CommandHandler             QueryHandler
  │                           │
Aggregate               Read Model (AccountBalanceView)
  │                           ↑
Event Store ────Events────► Projector
(append-only)
```

**Full flow:**
```
1. Client sends: DepositMoneyCommand(accountId="ACC-1", amount=100)
2. CommandHandler: load BankAccount → account.deposit(100) → raises MoneyDepositedEvent
3. Event stored: {aggregate_id:"ACC-1", type:"MoneyDepositedEvent", version:5}
4. Projector updates: AccountBalanceView → balance: 1100.00
5. Client queries: GetAccountBalanceQuery("ACC-1")
6. QueryHandler reads AccountBalanceView → returns 1100.00 (fast, no replay)
```

In Axon: `@Aggregate` with `@CommandHandler` + `@EventSourcingHandler` (command side); `@Component` with `@EventHandler` (projector/query side).

---

## 7. When to Use What

**Use Saga when:** Business operation spans multiple services; 2PC is not feasible; eventual consistency is acceptable.

**Choreography:** 2–3 steps, truly independent services, simple event reactions.  
**Orchestration:** 4+ steps, complex conditional flows, need clear visibility/audit.

**Use CQRS when:** Read load >> write load; multiple read models needed; complex domain + complex queries; combined with Event Sourcing.

**Use Event Sourcing when:** Complete audit trail is a business requirement (banking, healthcare, legal); temporal queries needed; undo/redo or event replay is valuable.

**Do NOT use any of these for simple CRUD.** Plain Spring + JPA is the right choice for most apps.

### Combined Decision Matrix

| Scenario | Recommendation |
|---|---|
| Multi-service checkout flow | Saga (orchestration) + Outbox |
| E-commerce product catalog | CQRS (separate write/read stores) |
| Banking transaction history | Event Sourcing + CQRS |
| Simple user registration | Plain Spring + JPA |
| Order fulfillment with 6+ steps | Saga (orchestration) + Axon |
| Financial ledger with audit | Event Sourcing |

---

## 8. Spring + Axon Framework Example

### 8.1 Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.axonframework</groupId>
        <artifactId>axon-spring-boot-starter</artifactId>
        <version>4.9.3</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
</dependencies>
```

### 8.2 Axon Aggregate + Projector

```java
// Aggregate — command side
@Aggregate
public class OrderAggregate {
    @AggregateIdentifier private String orderId;
    private OrderStatus status;
    protected OrderAggregate() {}  // required by Axon

    @CommandHandler
    public OrderAggregate(CreateOrderCommand cmd) {
        if (cmd.getQuantity() <= 0) throw new IllegalArgumentException("Quantity must be positive");
        AggregateLifecycle.apply(new OrderCreatedEvent(
            cmd.getOrderId(), cmd.getProductId(), cmd.getQuantity()));
    }

    @EventSourcingHandler
    public void on(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        this.status = OrderStatus.PENDING;
    }
    // ConfirmOrderCommand / CancelOrderCommand + their @EventSourcingHandlers follow the same shape
}

// Projector — query side
@Component
public class OrderProjector {
    @Autowired private OrderSummaryRepository repository;

    @EventHandler
    public void on(OrderCreatedEvent event) {
        repository.save(new OrderSummary(event.getOrderId(), OrderStatus.PENDING));
    }
}
```

### 8.3 application.yml for Axon

```yaml
axon:
  axonserver:
    servers: localhost:8124
  eventhandling:
    processors:
      order-processor:
        mode: tracking
        thread-count: 2
  serializer:
    general: jackson
    events: jackson
```

---

## 9. Kafka for Saga and Event Sourcing

### 9.1 Kafka as Event Store — Key Points

**Advantages:** High throughput, built-in replication, consumer groups for multiple projections, log compaction for latest-state topics.

**Limitations vs EventStoreDB:** No built-in optimistic concurrency; no aggregate-level event retrieval; retention policy may expire old events (set `retention.ms=-1`); schema evolution harder without a schema registry.

```
Topic per aggregate type:  bank-account-events, order-events
Partition key = aggregateId → guarantees ordering within a partition
```

### 9.2 Topic Configuration

```java
@Bean
public NewTopic bankAccountEventsTopic() {
    return TopicBuilder.name("bank-account-events")
        .partitions(12).replicas(3)
        .config(TopicConfig.RETENTION_MS_CONFIG, "-1")        // keep forever
        .config(TopicConfig.CLEANUP_POLICY_CONFIG, "delete")  // keep all events (not compacted)
        .build();
}
```

### 9.3 Consumer Groups for Projections

Each projection uses its own consumer group so all receive every event independently:

```java
@KafkaListener(topics = "bank-account-events", groupId = "balance-projection")
public void updateBalanceProjection(ConsumerRecord<String, String> record) { ... }

@KafkaListener(topics = "bank-account-events", groupId = "fraud-detection")
public void detectFraud(ConsumerRecord<String, String> record) { ... }
```

---

## 10. Interview Questions & Answers

### Saga Pattern

**Q1: What is the Saga pattern? Why is it used in microservices?**

A: A Saga is a sequence of local transactions — each service performs its own ACID transaction and publishes an event to trigger the next step. If any step fails, compensating transactions undo the completed steps in reverse. Used because each microservice has its own database, making 2PC impractical across services.

---

**Q2: What is the difference between choreography and orchestration sagas?**

A: Choreography has no central controller — services react to events autonomously (A publishes → B listens and acts → publishes). Orchestration uses a central saga orchestrator that sends explicit commands and collects replies. Choreography suits simple/loosely-coupled flows; orchestration suits complex flows needing visibility and control.

---

**Q3: What are compensating transactions? What properties must they have?**

A: A business operation that semantically undoes a completed local transaction. They must be (1) **idempotent** — running twice produces the same result as once, because the triggering message may be delivered more than once; (2) **semantically correct** — must reverse the business effect (not a DB rollback). Some operations like sending email can't be undone — the compensation is a corrective action.

---

**Q4: How do you handle partial failures in a saga?**

A: Through compensating transactions in reverse order. When step N fails, execute compensations for N-1, N-2, etc. The orchestrator tracks completed steps and issues compensations. All compensations must be idempotent, and saga state must be persisted durably so the orchestrator can resume after a crash.

---

**Q5: What is the Outbox Pattern? Why is it needed?**

A: Solves the dual-write problem: updating the DB and publishing to Kafka can't be made atomic. If DB commits but Kafka fails, the event is lost. Solution: write the event to an `outbox` table in the same DB transaction. A relay (Debezium or polling publisher) reads unpublished rows and pushes to Kafka. Guarantees at-least-once delivery — consumers must be idempotent.

---

**Q6: What is a semantic lock and why is it used in sagas?**

A: A flag in the data record (e.g., order status `PROCESSING`) indicating the record is part of an in-progress saga. Other requests check the flag and reject or wait, preventing the "lost update" anomaly where two concurrent sagas overwrite each other's changes.

---

**Q7: How do you ensure idempotency in saga steps?**

A: Before processing, check whether the action already happened: check for an existing record by business key (`paymentRepository.existsByOrderId(orderId)`), use a unique DB constraint, or maintain a `processed_events` table keyed by event ID. The idempotency check and business operation must be in the same `@Transactional` boundary.

---

**Q8: What is the Axon Framework and how does it help with sagas?**

A: A Java framework with first-class CQRS, Event Sourcing, and Saga support. For sagas: `@Saga`, `@StartSaga`, `@SagaEventHandler`, `@EndSaga` annotations; automatic saga state persistence; resume after restarts. Eliminates the boilerplate of building a saga state machine from scratch.

---

### CQRS

**Q9: What is CQRS? What problem does it solve?**

A: Separates the write model (commands) from the read model (queries). Solves: (1) reads and writes often need different scaling; (2) write side has complex domain logic while read side needs denormalized views; (3) a single model serving both creates impedance mismatch.

---

**Q10: What is the difference between CQS and CQRS?**

A: CQS is method-level: a method either changes state (command) OR returns data (query), never both. CQRS is architectural: separates the entire write stack (command handlers, write DB) from the read stack (query handlers, read DB).

---

**Q11: When would you NOT use CQRS?**

A: Avoid it for simple CRUD, small teams that can't absorb the complexity, similar read/write loads, or when strong read-after-write consistency is required (CQRS adds eventual consistency).

---

**Q12: How do you handle the read-after-write problem in CQRS?**

A: The read model may lag after a write. Best solution: return the written data directly from the command endpoint — the API already knows what was just created. Alternatively, update the UI optimistically, or poll until the read model version matches the expected version from the command response.

---

**Q13: How do you synchronize write and read models?**

A: Via domain events. Command handler publishes domain events after a successful write. Projectors subscribe and update read models. Use `ApplicationEventPublisher` + `@EventListener` (synchronous, in-process) or Kafka (asynchronous, cross-service). Use the Outbox Pattern to prevent event loss.

---

### Event Sourcing

**Q14: What is Event Sourcing? How does it differ from traditional state storage?**

A: Traditional storage holds only current state. Event Sourcing holds an append-only log of all events — current state is derived by replay. Traditional: `UPDATE orders SET status='SHIPPED'`; Event Sourcing: `INSERT INTO events (type='OrderShipped', ...)`. Trade-off: Event Sourcing has full history and temporal queries, but needs projections for fast reads.

---

**Q15: What are projections in event sourcing?**

A: Read models built by replaying events — the ES equivalent of a DB table. A projector listens for events and updates the projection. Since events are immutable, projections can always be rebuilt from scratch. Multiple projections can be built from the same event stream.

---

**Q16: What are snapshots in event sourcing? When should you use them?**

A: A snapshot serializes aggregate state at a specific version. Loading then replays only events after the snapshot instead of all events. Use when aggregates accumulate hundreds of events and replay time becomes slow. Common strategy: snapshot every 50–100 events.

---

**Q17: How do you handle schema changes in event sourcing?**

A: Never modify stored events. Options: (1) additive changes — add optional fields with defaults (safest); (2) upcasters — transform old-format events to new format on deserialization; (3) keep both `EventV1` and `EventV2` as separate classes; (4) bulk event migration (risky, rarely recommended).

---

**Q18: What is the difference between Event Sourcing and CDC?**

A: CDC captures technical DB changes (INSERT/UPDATE/DELETE) at the infrastructure level. Event Sourcing captures domain events at the application level with rich business meaning (`OrderPlaced`, `PaymentProcessed`). CDC works on existing DBs without code changes; ES requires redesigning the data model.

---

**Q19: How do you combine CQRS with Event Sourcing?**

A: Naturally — Event Sourcing generates events stored in the event store (command side persistence); projectors build optimized read models from those events (CQRS query side). The event store is the bridge: it persists writes AND sources all read models.

---

**Q20: What guarantees does an append-only event store provide?**

A: Immutability (events never modified); optimistic concurrency (unique version constraint); causal ordering within an aggregate; complete history; replayability to rebuild any projection.

---

**Q21: How would you implement an order processing saga?**

A: Orchestration-based saga + Outbox Pattern: (1) OrderService creates order in PENDING state, writes `OrderCreatedEvent` to outbox in same transaction. (2) Debezium publishes to Kafka. (3) SagaOrchestrator receives event, sends `ReserveStockCommand`. (4) InventoryService processes idempotently, publishes `StockReservedEvent`. (5) Orchestrator sends `ProcessPaymentCommand`. (6) On failure, orchestrator sends compensating commands in reverse. All participants are idempotent; orchestrator persists state per step. Use Axon Framework to avoid hand-building the state machine.

---

**Q22: How does Axon Framework handle saga persistence?**

A: Axon serializes the entire saga object and stores it in a backing DB. When a matching event arrives, Axon loads the saga, calls the `@SagaEventHandler`, then saves updated state. On restart, sagas resume from their persisted state. An association table maps property values (e.g., `orderId="123"`) to saga instances.

---

**Q23: What is the difference between a command and an event?**

A: A **command** is an intent — a request to do something; it can be rejected. Named imperatively: `CreateOrderCommand`. An **event** is a fact — something that already happened; it cannot be rejected. Named in past tense: `OrderCreatedEvent`. Commands go to aggregates; events go to projectors, sagas, and other consumers.

---

**Q24: How do you prevent duplicate processing of Kafka messages?**

A: Multiple layers: (1) business-level check before acting (`existsByOrderId(orderId)`); (2) unique DB constraint as a safety net; (3) processed event log table. The `@Transactional` boundary must cover both the check and the action to prevent race conditions.

---

**Q25: What is a pivot transaction in a saga?**

A: The point of no return — once it commits, the saga will complete (no compensation needed). Steps before are compensable; steps after are retriable. For an order saga: charging the card is the pivot. Before: reserve stock (compensable). After: send confirmation email (retriable — can't undo, just retry until it sends).

---

**Q26: How would you test a choreography-based saga?**

A: (1) `@EmbeddedKafka` in Spring tests — publish events, verify correct Kafka messages produced/consumed. (2) Contract testing with Pact for event contracts. (3) End-to-end test with Docker Compose + TestContainers — trigger initiating event, poll for expected final state. The hardest part: inject failures at specific steps and verify compensations execute correctly.

---

**Q27: What are the trade-offs of Kafka vs EventStoreDB as an event store?**

A: Kafka: high throughput, familiar infrastructure, but lacks aggregate-level querying and built-in optimistic concurrency. EventStoreDB: purpose-built — aggregate streams, built-in optimistic concurrency, first-class projections, guaranteed infinite retention. Use EventStoreDB when Event Sourcing is primary; use Kafka when you already have it and need a good-enough event store.

---

**Q28: How does the `version` field prevent concurrent updates?**

A: The `UNIQUE(aggregateId, version)` DB constraint allows only one write per version. If two transactions both try to write version 5, one gets a `DuplicateKeyException` → translated to `OptimisticConcurrencyException` → caller reloads and retries. This is the event-sourcing equivalent of JPA's `@Version`.

---

**Q29: What is the global sequence in the event store?**

A: A monotonically increasing number across all aggregates. Lets a projector checkpoint its position and resume after a crash. Axon Server and EventStoreDB provide it natively.

---

**Q30: How do you rebuild a projection after adding a new read model?**

A: Catch-up subscription: start a new projector at position 0, replay all historical events to build initial state, then switch to live events once caught up. In Axon, tracking processors support this natively. Possible because Event Sourcing never deletes events.

---

**Q31: What is the difference between `@EventSourcingHandler` and `@EventHandler` in Axon?**

A: `@EventSourcingHandler` (inside `@Aggregate`) rebuilds aggregate state on replay — used for write-side state reconstruction. `@EventHandler` (inside `@Component`, typically a projector) reacts to published events to update read models — used for read-side reactions.

---

**Q32: How do you handle a saga stuck in an intermediate state after a system crash?**

A: Saga state is persisted. On restart: load in-progress sagas, determine current state, resume by re-triggering the pending command. Also: use timeouts (if no reply in N minutes, trigger retry or rollback); route failed messages to a dead-letter queue; monitor for sagas stuck in non-terminal states beyond a threshold.

---

**Q33: What is at-least-once delivery and why is idempotency critical?**

A: At-least-once means a message is guaranteed to be delivered but may arrive more than once (due to retries). All message handlers must be idempotent — processing the same message twice must produce the same result as once. Without this, a retry would charge the card twice.

---

**Q34: What is eventual consistency and how do you explain it to stakeholders?**

A: After a write, readers may briefly see stale data, but all copies converge "soon" (typically milliseconds). Analogy: posting on social media — your friend sees it 2 seconds later. For most business operations, this lag is imperceptible. For critical paths (payment confirmation), use synchronous consistency for that specific operation.

---

**Q35: What happens if a compensating transaction itself fails?**

A: Make compensations idempotent and retry until they succeed. After N retries, route to a dead-letter queue and escalate to humans. Some reversals are manual (credit notes, customer service). Accept that perfect automated compensation isn't always possible.

---

**Q36: What is the "tell, don't ask" principle in event sourcing?**

A: Instead of asking the aggregate for its state and modifying it externally, you tell the aggregate to do something: `account.withdraw(amount)`. The aggregate validates internally and raises an event. State changes happen ONLY inside `apply()` methods — never set state from outside. This maintains event log integrity.

---

**Q37: What are the ACID properties of an individual saga step?**

A: Each step is a local ACID transaction. The saga as a whole is ACD without I: Atomicity (local), Consistency (local), Durability (local), but no global Isolation — other sagas can see intermediate states (e.g., order in PENDING while saga runs). This is why semantic locks matter.

---

## 11. Quick Reference Cheat Sheet

```
WHICH PATTERN?
  Saga           → ONE business action spans MULTIPLE services (order → stock → payment)
  CQRS           → reads and writes have VERY different needs (complex queries / high read volume)
  Event Sourcing → you must keep the FULL HISTORY (audit log, time-travel queries)

WHEN NOT TO USE (these add real complexity)
  → Single service + one DB? Use a normal ACID transaction.
  → Simple CRUD? Plain Spring + JPA beats CQRS every time.
  → No audit needed? Skip Event Sourcing.

SAGA STYLES
  Choreography  → no central brain; services react to events; pick for ≤3 steps, loose coupling
  Orchestration → one orchestrator commands each service; pick for 4+ steps, complex logic

COMPENSATING TRANSACTION
  No global ROLLBACK across services.
  "Undo" with a new forward action that reverses the business effect:
    charged the card  → issue a refund
    reserved stock    → release the reservation

2PC  → lock everyone, then commit. Strong consistency but blocks/locks. Avoid across microservices.
Saga → local transactions + compensations. No global lock; eventually consistent.

ACID → strong: every transaction leaves DB fully consistent right now (single DB)
BASE → relaxed: Basically Available, Soft state, Eventually consistent (across services)

COMMON JUNIOR MISTAKES
  ✗ Using 2PC / distributed transaction across microservices → use a Saga instead
  ✗ Forgetting idempotency → duplicate event charges the card twice
  ✗ Modifying state directly in event sourcing → state may ONLY change via apply()
  ✗ Expecting reads to be instantly up-to-date in CQRS/ES → read model is eventually consistent
  ✗ No compensation path → failed saga leaves orphaned/inconsistent data

GLOSSARY
  Idempotency          → running the same operation twice = same result as once
  Eventual consistency → data not in sync immediately, but all copies catch up "soon"
  Aggregate            → cluster of objects as one consistency unit (Order + its OrderLines)
  Projection/read model→ query-optimized view BUILT from events, used only for reads
  Event store          → append-only log of all events; source of truth in event sourcing
  Outbox pattern       → write event to DB in same transaction; relay publishes to Kafka
  Semantic lock        → a status flag marking a record as "in-progress" to prevent concurrent modification
```

---

*End of Document — Microservices: Saga Pattern, CQRS, and Event Sourcing*

*Last Updated: 2026-06-18*
