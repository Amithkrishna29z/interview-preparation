# Microservices: Saga Pattern, CQRS, and Event Sourcing

## 1. Distributed Transactions Problem

### 1.1 Monolith vs Microservices Transaction Model

**In a monolith**, a single ACID transaction can span all operations:

```
BEGIN TRANSACTION
  UPDATE orders SET status='CONFIRMED' WHERE id=123;
  UPDATE inventory SET quantity=quantity-1 WHERE product_id=456;
  INSERT INTO payments (order_id, amount) VALUES (123, 99.99);
COMMIT;
-- If anything fails → ROLLBACK undoes everything atomically
```

Everything lives in one database. The database engine guarantees atomicity.

**In microservices**, each service owns its own database (Database-per-Service pattern):

```
OrderService        → orders_db        (PostgreSQL)
InventoryService    → inventory_db     (MySQL)
PaymentService      → payments_db      (MongoDB)
```

Real-world analogy: Imagine you need to simultaneously update records in three different banks in three different countries. There is no single authority that can lock all three at once and roll back all three atomically. This is the distributed transaction problem.

**The core problem**: How do you guarantee that either ALL operations succeed or ALL operations are rolled back, when each operation lives in a different database managed by a different service?

---

### 1.2 Two-Phase Commit (2PC)

2PC is the classical solution to distributed transactions. It introduces a **coordinator** (transaction manager) and **participants** (the databases/services).

**Phase 1 — Prepare Phase:**
```
Coordinator → OrderService:    "Can you commit?"  → OrderService:    "YES (PREPARED)"
Coordinator → InventoryService:"Can you commit?"  → InventoryService:"YES (PREPARED)"
Coordinator → PaymentService:  "Can you commit?"  → PaymentService:  "YES (PREPARED)"
```
Each participant locks its resources and writes a prepare log. It can now commit or rollback on command.

**Phase 2 — Commit Phase:**
```
If ALL said YES:
  Coordinator → All: "COMMIT"  → Each participant commits and releases locks

If ANY said NO:
  Coordinator → All: "ROLLBACK" → Each participant rolls back and releases locks
```

**Problems with 2PC:**
- **Blocking protocol** — participants hold locks from Phase 1 until Phase 2. If the coordinator crashes in between, they are stuck holding locks in an uncertain state.
- **Coordinator is a SPOF** and a performance bottleneck — all participants wait for the slowest, and lock contention kills throughput.
- **Not cloud-native** — many modern databases (DynamoDB, MongoDB Atlas) and third-party APIs do not support the XA transactions 2PC requires.

2PC is only acceptable for very short transactions, few participants (2–3), all on XA-capable databases you fully control, where consistency is non-negotiable (e.g., banking ledgers). Avoid it across independently deployed microservices, third-party APIs, or different DB technologies.

> **Common junior mistake:** Reaching for 2PC (or a "distributed transaction") to keep microservices in sync. Across independently deployed services this blocks resources and stalls on any slow/dead participant. Use a Saga (local transactions + compensations) instead.

---

## 2. BASE vs ACID

### 2.1 ACID (Traditional Databases)

| Property | Meaning |
|---|---|
| **Atomicity** | All operations succeed or all fail. No partial success. |
| **Consistency** | Database moves from one valid state to another. Constraints are always satisfied. |
| **Isolation** | Concurrent transactions behave as if they ran serially. No dirty reads. |
| **Durability** | Once committed, data survives crashes. Written to disk. |

### 2.2 BASE (Distributed Systems)

| Property | Meaning |
|---|---|
| **Basically Available** | The system remains available even during failures — but may serve stale data. |
| **Soft State** | The state of the system may change over time even without input (due to eventual consistency propagation). |
| **Eventually Consistent** | Given no new updates, all replicas will eventually converge to the same value. |

**The Trade-off:** Microservices deliberately trade ACID guarantees for availability and partition tolerance (CAP theorem). Each service maintains local ACID transactions within its own database, but across services, the system is only eventually consistent.

**Real-world analogy:** When you book a flight online, the airline website may show you a seat as available. By the time you complete checkout (5 seconds later), another user may have grabbed it. The system is eventually consistent — the seat availability eventually converges to "taken" — but there is a brief window where two users see conflicting information.

```
ACID world:  Seat A available? → Lock it → You get it → Others see "taken" instantly
BASE world:  Seat A available? → You reserve it → System propagates "taken" → 
             (1-2 seconds later) Others see "taken"
             Risk: another user sees "available" in that 1-2 second window
```

---

## 3. Saga Pattern — Complete Deep Dive

### 3.1 What is a Saga?

A **Saga** is a sequence of local transactions where each local transaction:
1. Updates its own service's database (local ACID transaction)
2. Publishes an event or sends a message to trigger the next step

If any step fails, the saga executes **compensating transactions** to undo the previously completed steps, rolling back the overall business operation.

**Key insight:** A saga does NOT use 2PC. There is no distributed lock. Instead of preventing inconsistency, sagas *repair* inconsistency when it occurs.

**Real-world analogy:** Booking a vacation package (flight + hotel + car rental). You book each separately. If the car rental fails after flight and hotel are booked, you don't hold locks on the flight and hotel — instead, you *cancel* (compensate) the flight and hotel bookings you already made.

**Order Processing Saga Example:**
```
Step 1: OrderService       → Create order (status: PENDING)
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

In choreography, there is **no central orchestrator**. Each service listens for events and decides what to do next. Services are autonomous — they react to domain events published by other services.

**Architecture:**
```
OrderService
  ├── Creates order
  ├── Publishes: OrderCreatedEvent
  └── Listens for: OrderCancelledEvent (to cancel)

InventoryService
  ├── Listens for: OrderCreatedEvent
  ├── Reserves stock
  ├── Publishes: StockReservedEvent OR StockReservationFailedEvent
  └── Listens for: PaymentFailedEvent (to release stock)

PaymentService
  ├── Listens for: StockReservedEvent
  ├── Processes payment
  └── Publishes: PaymentProcessedEvent OR PaymentFailedEvent

ShippingService
  ├── Listens for: PaymentProcessedEvent
  ├── Schedules delivery
  └── Publishes: ShipmentScheduledEvent
```

**Event Flow (Happy Path):**
```
OrderService       --[OrderCreatedEvent]--------→ Kafka Topic: order-events
InventoryService   --[StockReservedEvent]--------→ Kafka Topic: inventory-events
PaymentService     --[PaymentProcessedEvent]-----→ Kafka Topic: payment-events
ShippingService    --[ShipmentScheduledEvent]----→ Kafka Topic: shipping-events
OrderService       ← listens → marks order CONFIRMED
```

**Event Flow (Compensation — Payment Fails):**
```
OrderService       --[OrderCreatedEvent]----------→ inventory-events
InventoryService   --[StockReservedEvent]---------→ payment-events
PaymentService     --[PaymentFailedEvent]---------→ inventory-events, order-events
InventoryService   ← listens → releases stock     --[StockReleasedEvent]→
OrderService       ← listens → marks order CANCELLED
```

**Java + Spring + Kafka Implementation:**

```java
// ============ EVENTS ============

public record OrderCreatedEvent(
    String orderId,
    String customerId,
    String productId,
    int quantity,
    BigDecimal totalAmount
) {}

public record StockReservedEvent(
    String orderId,
    String productId,
    int quantity
) {}

public record StockReservationFailedEvent(
    String orderId,
    String reason
) {}

public record PaymentProcessedEvent(
    String orderId,
    String transactionId,
    BigDecimal amount
) {}

public record PaymentFailedEvent(
    String orderId,
    String reason
) {}

// ============ ORDER SERVICE ============

@Service
@Slf4j
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public String createOrder(CreateOrderCommand cmd) {
        Order order = new Order(
            UUID.randomUUID().toString(),
            cmd.customerId(),
            cmd.productId(),
            cmd.quantity(),
            cmd.totalAmount(),
            OrderStatus.PENDING
        );
        orderRepository.save(order);

        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(), order.getCustomerId(),
            order.getProductId(), order.getQuantity(), order.getTotalAmount()
        );
        kafkaTemplate.send("order-events", order.getId(), event);
        log.info("Order created: {}", order.getId());
        return order.getId();
    }

    // Compensation listener — triggered when payment fails
    @KafkaListener(topics = "payment-events", groupId = "order-service")
    public void onPaymentFailed(PaymentFailedEvent event) {
        orderRepository.findById(event.orderId()).ifPresent(order -> {
            order.setStatus(OrderStatus.CANCELLED);
            orderRepository.save(order);
            log.info("Order cancelled due to payment failure: {}", event.orderId());
        });
    }

    @KafkaListener(topics = "shipping-events", groupId = "order-service")
    public void onShipmentScheduled(ShipmentScheduledEvent event) {
        orderRepository.findById(event.orderId()).ifPresent(order -> {
            order.setStatus(OrderStatus.CONFIRMED);
            orderRepository.save(order);
        });
    }
}

// ============ INVENTORY SERVICE ============

@Service
@Slf4j
public class InventoryService {

    @Autowired
    private InventoryRepository inventoryRepository;

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        inventoryRepository.findByProductId(event.productId()).ifPresentOrElse(
            inventory -> {
                if (inventory.getAvailableQuantity() >= event.quantity()) {
                    inventory.setReservedQuantity(
                        inventory.getReservedQuantity() + event.quantity()
                    );
                    inventoryRepository.save(inventory);

                    kafkaTemplate.send("inventory-events", event.orderId(),
                        new StockReservedEvent(event.orderId(), event.productId(), event.quantity())
                    );
                    log.info("Stock reserved for order: {}", event.orderId());
                } else {
                    kafkaTemplate.send("inventory-events", event.orderId(),
                        new StockReservationFailedEvent(event.orderId(), "Insufficient stock")
                    );
                }
            },
            () -> kafkaTemplate.send("inventory-events", event.orderId(),
                new StockReservationFailedEvent(event.orderId(), "Product not found")
            )
        );
    }

    // Compensation: release reserved stock when payment fails
    @KafkaListener(topics = "payment-events", groupId = "inventory-service")
    @Transactional
    public void onPaymentFailed(PaymentFailedEvent event) {
        // Reverse the reservation (compensating transaction)
        log.info("Releasing reserved stock for order: {}", event.orderId());
        // ... find reservation and release it
    }
}

// ============ PAYMENT SERVICE ============
// Listens for StockReservedEvent, charges the customer, and publishes
// PaymentProcessedEvent on success or PaymentFailedEvent on failure.
// (Same shape as the services above: @KafkaListener in, kafkaTemplate.send out.)
```

(Standard Spring Kafka `ProducerFactory`/`KafkaTemplate` config is assumed — see section 9.2 for a producer config example.)

**Choreography Pros and Cons:**

| Pros | Cons |
|---|---|
| Loose coupling — services don't know about each other | Distributed logic — hard to get a full picture of the saga |
| No single point of failure | Risk of cyclic dependencies (Service A triggers B, B triggers A) |
| Services are autonomous and independently deployable | Difficult to debug — must trace events across multiple services/topics |
| Easy to add new participants (just subscribe to events) | Harder to test end-to-end (must simulate all event flows) |
| Scales well horizontally | Harder to enforce ordering/sequencing of steps |

---

### 3.3 Orchestration-Based Saga

In orchestration, a **central saga orchestrator** directs all participants by sending explicit commands and waiting for replies. The orchestrator is a state machine that knows the entire saga flow.

**Architecture:**
```
SagaOrchestrator (central brain)
  │
  ├── send ReserveStockCommand    → InventoryService
  │   ← receive StockReservedEvent or StockReservationFailedEvent
  │
  ├── send ProcessPaymentCommand  → PaymentService
  │   ← receive PaymentProcessedEvent or PaymentFailedEvent
  │
  ├── send ScheduleShipmentCommand → ShippingService
  │   ← receive ShipmentScheduledEvent
  │
  └── send ConfirmOrderCommand    → OrderService
```

**State Machine:**
```
PENDING
  └─[ReserveStock]──→ INVENTORY_RESERVED
                          └─[ProcessPayment]──→ PAYMENT_PROCESSED
                                                    └─[ScheduleShipment]──→ COMPLETED

Compensation flows:
  PAYMENT_FAILED
    └─[ReleaseStock]──→ COMPENSATING_INVENTORY
                            └─[CancelOrder]──→ CANCELLED

  SHIPMENT_FAILED
    └─[RefundPayment]──→ COMPENSATING_PAYMENT
                            └─[ReleaseStock]──→ COMPENSATING_INVENTORY
                                                   └─[CancelOrder]──→ CANCELLED
```

**Java Implementation — Manual Orchestrator:**

```java
// ============ SAGA STATE ============

@Entity
@Table(name = "order_sagas")
public class OrderSaga {
    @Id
    private String sagaId;
    private String orderId;
    private SagaState state;
    private String failureReason;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    public enum SagaState {
        PENDING,
        RESERVING_INVENTORY,
        INVENTORY_RESERVED,
        PROCESSING_PAYMENT,
        PAYMENT_PROCESSED,
        SCHEDULING_SHIPMENT,
        COMPLETED,
        COMPENSATING_INVENTORY,
        COMPENSATING_PAYMENT,
        CANCELLED
    }
}

// ============ SAGA ORCHESTRATOR ============

@Service
@Slf4j
public class OrderSagaOrchestrator {

    @Autowired private OrderSagaRepository sagaRepository;
    @Autowired private KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public void startSaga(String orderId, OrderDetails details) {
        OrderSaga saga = new OrderSaga(
            UUID.randomUUID().toString(), orderId, SagaState.PENDING
        );
        sagaRepository.save(saga);
        sendReserveStockCommand(saga, details);
    }

    private void sendReserveStockCommand(OrderSaga saga, OrderDetails details) {
        saga.setState(SagaState.RESERVING_INVENTORY);
        sagaRepository.save(saga);

        kafkaTemplate.send("inventory-commands", saga.getOrderId(),
            new ReserveStockCommand(saga.getSagaId(), saga.getOrderId(),
                details.productId(), details.quantity())
        );
    }

    // ---- Step 1 reply: Inventory ----

    @KafkaListener(topics = "saga-replies", groupId = "saga-orchestrator",
                   containerFactory = "sagaKafkaListenerContainerFactory")
    @Transactional
    public void handleReply(SagaReply reply) {
        OrderSaga saga = sagaRepository.findBySagaId(reply.sagaId())
            .orElseThrow(() -> new IllegalStateException("Saga not found: " + reply.sagaId()));

        switch (saga.getState()) {
            case RESERVING_INVENTORY -> handleInventoryReply(saga, reply);
            case PROCESSING_PAYMENT  -> handlePaymentReply(saga, reply);
            case SCHEDULING_SHIPMENT -> handleShipmentReply(saga, reply);
            case COMPENSATING_INVENTORY -> handleInventoryCompensationReply(saga, reply);
            case COMPENSATING_PAYMENT   -> handlePaymentCompensationReply(saga, reply);
            default -> log.warn("Unexpected reply in state {}", saga.getState());
        }
    }

    private void handleInventoryReply(OrderSaga saga, SagaReply reply) {
        if (reply.success()) {
            // Move forward: persist new state, then send the next command
            saga.setState(SagaState.PROCESSING_PAYMENT);
            sagaRepository.save(saga);
            kafkaTemplate.send("payment-commands", saga.getOrderId(),
                new ProcessPaymentCommand(saga.getSagaId(), saga.getOrderId(), reply.amount()));
        } else {
            // Nothing done yet — no compensation needed, just cancel
            saga.setState(SagaState.CANCELLED);
            saga.setFailureReason("Inventory reservation failed: " + reply.reason());
            sagaRepository.save(saga);
            notifyOrderCancelled(saga);
        }
    }

    // Remaining handlers follow the same shape:
    //  - On success: advance state + send the next forward command.
    //  - On failure: switch to a COMPENSATING_* state and send a compensating
    //    command (ReleaseStockCommand, RefundPaymentCommand) in reverse order,
    //    ending at CANCELLED once all completed steps are undone.
    // handlePaymentReply, handleShipmentReply, handleInventoryCompensationReply,
    // handlePaymentCompensationReply are omitted here for brevity.
}
```

**Orchestration Pros and Cons:**

| Pros | Cons |
|---|---|
| Saga logic in ONE place — easy to understand the full flow | Orchestrator is coupled to all participant services |
| Easier to debug — single point to check state | Orchestrator can become a bottleneck under high load |
| Easier to test — test orchestrator in isolation with mocked services | Orchestrator is a SPOF (mitigate with clustering/persistence) |
| Clear audit trail — saga state table shows exactly where things are | More boilerplate code for commands and replies |
| Handles complex compensation flows cleanly | Services become less autonomous (they respond to commands) |

---

### 3.4 Choreography vs Orchestration Comparison Table

| Aspect | Choreography | Orchestration |
|---|---|---|
| Logic location | Distributed across services | Central saga orchestrator |
| Coupling | Event-based (loose) | Command/reply-based (tighter) |
| Testability | Harder (must simulate all events) | Easier (test orchestrator with mocks) |
| Debugging | Harder (trace events across topics) | Easier (inspect saga state table) |
| Visibility | Low (no single place to see saga state) | High (state machine is explicit) |
| Scalability | Better (no central bottleneck) | Orchestrator can be bottleneck |
| Complexity | Low for simple flows | Low for complex flows |
| Use case | Simple, short sagas (2–3 steps) | Complex, multi-step business workflows |
| SPOF risk | None | Orchestrator (mitigate with HA deployment) |
| New participants | Easy (just subscribe to events) | Must update orchestrator |

---

### 3.5 Compensating Transactions

A compensating transaction is an operation that **semantically undoes** a previously completed local transaction. It is NOT a database rollback — the original transaction is committed. Compensation is a new transaction that reverses the business effect.

**Properties of compensating transactions:**
- **Idempotent**: Executing compensation multiple times has the same effect as executing once. Critical because the message triggering compensation may be delivered more than once.
- **Semantically correct**: Must undo the *business effect*, not just reverse the data change.
- **May not be technically possible**: Some operations cannot be truly undone (e.g., sending an email). In such cases, the compensation is a corrective action (send a "Sorry, your order was cancelled" email).

**Transaction Types in a Saga:**

| Type | Description | Example |
|---|---|---|
| **Compensable transaction** | Can be undone by a compensating transaction | Reserve stock → can release stock |
| **Pivot transaction** | The go/no-go point of the saga. If it commits, saga will complete. If it fails, compensate everything before it. | Payment processing (charging the card) |
| **Retriable transaction** | Guaranteed to succeed eventually (retry until success). No compensation needed. | Send confirmation email, update analytics |

**Examples of Compensating Transactions:**

| Original Operation | Compensating Transaction |
|---|---|
| Reserve inventory | Release inventory reservation |
| Charge customer credit card | Refund to customer credit card |
| Create order (status: PENDING) | Update order status to CANCELLED |
| Allocate seats on flight | Release seats |
| Debit bank account | Credit bank account |
| Send confirmation email | Cannot undo — send cancellation email instead |
| Create user account | Deactivate/delete user account |

---

### 3.6 Saga Failure Modes and Countermeasures

**Problem: Lost updates** — One saga step reads data modified by another concurrent saga.

**Countermeasure — Semantic lock:**
```java
// Mark data as "in-progress" so other sagas/requests know it's being processed
@Transactional
public void startOrderProcessing(String orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    if (order.getStatus() != OrderStatus.PENDING) {
        throw new OrderAlreadyBeingProcessedException(orderId);
    }
    order.setStatus(OrderStatus.PROCESSING); // semantic lock
    orderRepository.save(order);
    // Now publish event to start saga
}
```

**Problem: Dirty reads** — A transaction reads data written by a saga that has not yet committed (or may be compensated).

**Countermeasure — Commutative updates:**
Design updates so that they can be applied in any order and still produce the correct result. For example, instead of `SET balance = 500`, use `SET balance = balance - 100` (commutative with other debit/credit operations).

**Problem: Non-repeatable reads** — A saga reads a value, another saga modifies it, and the first saga reads a different value on a second read.

**Countermeasure — Pessimistic view:**
Reorder saga steps so that compensable transactions happen before retriable transactions. Or add a version field (optimistic locking) to detect concurrent modifications.

**Idempotency requirements:**
```java
@KafkaListener(topics = "inventory-events", groupId = "payment-service")
@Transactional
public void onStockReserved(StockReservedEvent event) {
    // Idempotency check: skip if already processed this event
    if (paymentRepository.existsByOrderId(event.orderId())) {
        log.info("Payment already processed for order {}. Skipping.", event.orderId());
        return;
    }
    // Process payment...
}
```

> **Common junior mistake:** Assuming each event is delivered exactly once. Messaging is at-least-once, so the same event can arrive twice. Without an idempotency check like the one above, a retry would charge the customer twice.

---

### 3.7 Implementing Saga with Axon Framework

Axon Framework is a Java framework with first-class support for CQRS, Event Sourcing, and Sagas. Instead of hand-building an orchestrator + state table, you annotate a saga class and Axon handles state persistence and routing.

**Key Axon saga annotations:** `@Saga` (marks the class), `@StartSaga` (the initiating event handler), `@SagaEventHandler(associationProperty = "orderId")` (handles each event, matched by association key), and `@EndSaga` (completes the saga).

```java
@Saga
public class OrderProcessingSaga {

    @Autowired
    private transient CommandGateway commandGateway;
    private String orderId, productId;
    private int quantity;

    @StartSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void handle(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        SagaLifecycle.associateWith("orderId", orderId);  // route future events here
        commandGateway.send(new ReserveStockCommand(orderId, event.getProductId(), event.getQuantity()));
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void handle(StockReservedEvent event) {           // next forward step
        commandGateway.send(new ProcessPaymentCommand(orderId, event.getAmount()));
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void handle(PaymentFailedEvent event) {           // compensation step
        commandGateway.send(new ReleaseStockCommand(orderId, productId, quantity));
    }

    @EndSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void handle(ShipmentScheduledEvent event) {       // happy path completion
        commandGateway.send(new ConfirmOrderCommand(orderId));
    }
    // (Other event handlers — payment success, stock-released compensation, etc. — follow the same shape.)
}
```

Axon automatically persists saga state between events, so the saga resumes from where it left off after a restart.

---

### 3.8 Outbox Pattern

**The problem:** When a service updates its database AND publishes an event, these are two separate operations. If the database commits but Kafka publish fails, events are lost. If Kafka publishes but the database rolls back, phantom events are sent.

```java
// WRONG — not atomic:
@Transactional
public void createOrder(CreateOrderCommand cmd) {
    Order order = new Order(...);
    orderRepository.save(order);          // DB commit
    kafkaTemplate.send("events", event);  // May fail AFTER DB commit!
    // If Kafka fails here, order is saved but no event is published
    // → saga never starts → order stuck in PENDING forever
}
```

**The solution — Outbox Pattern:**
Store events in an **outbox table** in the **same database** as the business data, within the **same transaction**. A separate process (message relay) reads from the outbox and publishes to Kafka.

```
┌────────────────────────────────────┐
│         PostgreSQL Database        │
│                                    │
│  ┌─────────────┐  ┌─────────────┐  │
│  │  orders     │  │  outbox     │  │
│  │ (business)  │  │ (events)    │  │
│  └─────────────┘  └─────────────┘  │
│        ↑ same transaction  ↑       │
└────────────────────────────────────┘
                │
         Message Relay
         (Debezium CDC)
                │
                ↓
         ┌───────────┐
         │   Kafka   │
         └───────────┘
```

**Outbox Table Schema:**
```sql
CREATE TABLE outbox (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id    VARCHAR(255) NOT NULL,
    aggregate_type  VARCHAR(255) NOT NULL,
    event_type      VARCHAR(255) NOT NULL,
    payload         JSONB NOT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    published       BOOLEAN NOT NULL DEFAULT FALSE,
    published_at    TIMESTAMP
);

CREATE INDEX idx_outbox_unpublished ON outbox(created_at) WHERE published = FALSE;
```

**Java Implementation:**
```java
@Entity
@Table(name = "outbox")
public class OutboxEvent {
    @Id
    @GeneratedValue
    private UUID id;
    private String aggregateId;
    private String aggregateType;
    private String eventType;
    @Column(columnDefinition = "jsonb")
    private String payload;
    private LocalDateTime createdAt;
    private boolean published;
}

@Service
@Slf4j
public class OrderCommandService {

    @Autowired private OrderRepository orderRepository;
    @Autowired private OutboxRepository outboxRepository;
    @Autowired private ObjectMapper objectMapper;

    @Transactional  // Single transaction covers BOTH saves
    public String createOrder(CreateOrderCommand cmd) throws JsonProcessingException {
        // 1. Save business data
        Order order = new Order(UUID.randomUUID().toString(), cmd.customerId(),
            cmd.productId(), cmd.quantity(), OrderStatus.PENDING);
        orderRepository.save(order);

        // 2. Save event to outbox IN THE SAME TRANSACTION
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(), order.getCustomerId(),
            order.getProductId(), order.getQuantity(), order.getTotalAmount()
        );
        OutboxEvent outbox = new OutboxEvent(
            order.getId(), "Order", "OrderCreatedEvent",
            objectMapper.writeValueAsString(event)
        );
        outboxRepository.save(outbox);
        // If anything fails, BOTH order AND outbox are rolled back atomically

        return order.getId();
    }
}

// ============ POLLING PUBLISHER (simple alternative to Debezium) ============

@Component
@Slf4j
public class OutboxPollingPublisher {

    @Autowired private OutboxRepository outboxRepository;
    @Autowired private KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000) // Poll every second
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pending = outboxRepository.findTop100ByPublishedFalseOrderByCreatedAtAsc();
        for (OutboxEvent event : pending) {
            try {
                kafkaTemplate.send(topicFor(event.getEventType()), 
                    event.getAggregateId(), event.getPayload()).get(); // sync send
                event.setPublished(true);
                event.setPublishedAt(LocalDateTime.now());
                outboxRepository.save(event);
            } catch (Exception ex) {
                log.error("Failed to publish outbox event {}: {}", event.getId(), ex.getMessage());
                // Will retry on next poll
            }
        }
    }
}
```

**Debezium CDC approach** (production-grade, awareness): Instead of polling, a Debezium connector reads PostgreSQL's WAL (Write-Ahead Log) and publishes outbox rows to Kafka via its built-in `EventRouter` transform — zero polling overhead and sub-second latency. You register it as a Kafka Connect connector (JSON config pointing at the `outbox` table and mapping its columns to event key/type/payload). The polling publisher above is the simpler equivalent when you don't have Kafka Connect.

---

## 4. CQRS (Command Query Responsibility Segregation)

### 4.1 What is CQRS?

CQRS separates the **write model** (commands that change state) from the **read model** (queries that return data). They use different objects, different services, and often different databases.

**CQS Principle vs CQRS Pattern:**

| | CQS (Command Query Separation) | CQRS |
|---|---|---|
| Level | Method level | Architectural pattern |
| Scope | Single object/class | Entire application layer |
| Idea | A method either changes state (command) OR returns data (query), never both | Separate entire command stack from query stack |
| Example | `void save()` vs `User getUser()` | `CommandService` + `CommandHandler` vs `QueryService` + `QueryHandler` |

**Real-world analogy:** A library has a **librarian** who processes checkouts/returns (commands — modifies state) and a **catalog system** (queries — optimized for fast lookup). The checkout desk doesn't need to be fast at searching; the catalog doesn't need to process transactions. They are optimized separately.

---

### 4.2 Simple CQRS (Same Database)

The simplest form of CQRS just separates command handlers from query handlers, using the same database. No event sourcing required.

```
┌──────────────────────────────────────────────────────────┐
│                      Client Request                      │
└──────────────────────────────────────────────────────────┘
          │                              │
    Commands (writes)              Queries (reads)
          │                              │
          ↓                              ↓
┌──────────────────┐          ┌──────────────────┐
│  CommandHandler  │          │   QueryHandler   │
│  (business logic)│          │ (data retrieval) │
└──────────────────┘          └──────────────────┘
          │                              │
          └──────────── DB ─────────────┘
                   (same database)
```

```java
// ============ COMMANDS ============

public record CreateProductCommand(
    String name,
    String description,
    BigDecimal price,
    int initialStock
) {}

public record UpdateProductPriceCommand(String productId, BigDecimal newPrice) {}

// ============ COMMAND HANDLER ============

@Service
@Transactional
public class ProductCommandHandler {

    @Autowired private ProductRepository productRepository;
    @Autowired private ApplicationEventPublisher eventPublisher;

    public String handle(CreateProductCommand cmd) {
        Product product = new Product(
            UUID.randomUUID().toString(),
            cmd.name(), cmd.description(), cmd.price(), cmd.initialStock()
        );
        productRepository.save(product);
        eventPublisher.publishEvent(new ProductCreatedEvent(product.getId(), product.getName()));
        return product.getId();
    }

    public void handle(UpdateProductPriceCommand cmd) {
        Product product = productRepository.findById(cmd.productId()).orElseThrow();
        product.setPrice(cmd.newPrice());
        productRepository.save(product);
        eventPublisher.publishEvent(new ProductPriceUpdatedEvent(cmd.productId(), cmd.newPrice()));
    }
}

// ============ QUERY HANDLER ============

@Service
@Transactional(readOnly = true) // Read-only transaction for optimization
public class ProductQueryHandler {

    @Autowired private ProductRepository productRepository;

    public ProductDto handle(GetProductQuery query) {
        return productRepository.findById(query.productId())
            .map(this::toDto)
            .orElseThrow(() -> new ProductNotFoundException(query.productId()));
    }

    public List<ProductDto> handle(SearchProductsQuery query) {
        return productRepository.findByNameContainingIgnoreCase(query.searchTerm())
            .stream()
            .map(this::toDto)
            .collect(Collectors.toList());
    }

    private ProductDto toDto(Product product) {
        return new ProductDto(product.getId(), product.getName(),
            product.getDescription(), product.getPrice(), product.getStock());
    }
}

// ============ CONTROLLER ============

@RestController
@RequestMapping("/api/products")
public class ProductController {

    @Autowired private ProductCommandHandler commandHandler;
    @Autowired private ProductQueryHandler queryHandler;

    @PostMapping
    public ResponseEntity<String> createProduct(@RequestBody CreateProductCommand cmd) {
        String productId = commandHandler.handle(cmd);
        return ResponseEntity.status(HttpStatus.CREATED).body(productId);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductDto> getProduct(@PathVariable String id) {
        return ResponseEntity.ok(queryHandler.handle(new GetProductQuery(id)));
    }
}
```

---

### 4.3 CQRS with Separate Read/Write Stores

The full CQRS pattern uses different databases optimized for each side.

```
Write Side (Command)                Read Side (Query)
─────────────────────               ─────────────────────
CommandHandler                      QueryHandler
    │                                   │
    ↓                                   ↓
PostgreSQL (normalized)            Elasticsearch (denormalized)
(strong consistency)               (optimized for search)
    │
    │ Domain Events
    ↓
Event Handler (projector)
    │
    └──────────────────→ Elasticsearch
                         (updates read model)
```

The pieces are the same as Simple CQRS, plus a **projector** that syncs the read store. The write-side command handler saves to PostgreSQL and publishes a domain event; a projector listens and updates the read store; the query side reads from there.

```java
// PROJECTOR — keeps the denormalized read model (Elasticsearch) in sync
@Component
public class ProductReadModelProjector {

    @Autowired private ElasticsearchOperations elasticsearch;

    @EventListener
    public void on(ProductCreatedEvent event) {
        // ProductDocument is denormalized: it also stores categoryName, brandName,
        // averageRating, reviewCount, etc. so queries need no joins.
        elasticsearch.save(new ProductDocument(
            event.productId(), event.name(), event.description(),
            event.price(), event.stock()));
    }

    @EventListener
    public void on(ProductPriceUpdatedEvent event) {
        ProductDocument doc = elasticsearch.get(event.productId(), ProductDocument.class);
        if (doc != null) { doc.setPrice(event.newPrice()); elasticsearch.save(doc); }
    }
}
```

The read side then queries Elasticsearch directly (`ProductSearchRepository`) — full-text search with no joins, since everything is denormalized into the document.

---

### 4.4 Benefits and Challenges of CQRS

**Benefits:**

| Benefit | Explanation |
|---|---|
| Independent scaling | Read side can be scaled independently (reads usually 10x more than writes) |
| Optimized read models | No joins needed, denormalized for each specific query pattern |
| Multiple read models | Same events can build different projections (Elasticsearch for search, Redis for cache, reporting DB) |
| Audit trail | Domain events provide implicit audit log |
| Better separation of concerns | Complex write logic separated from complex read logic |
| Technology flexibility | Write: PostgreSQL. Read: Elasticsearch + Redis + MongoDB — pick best tool for each job |

**Challenges:**

| Challenge | Explanation |
|---|---|
| Eventual consistency | After a write, the read model may lag by milliseconds to seconds |
| Increased complexity | Two models, two services, synchronization logic — not suitable for simple CRUD |
| Read-after-write problem | User creates something, immediately queries it — may not see it yet in read model |
| Synchronization failures | If event publishing fails, read models become stale |
| Learning curve | Developers must understand the pattern and its implications |

**Read-after-write mitigation:**
```java
@PostMapping("/products")
public ResponseEntity<ProductDto> createProduct(@RequestBody CreateProductCommand cmd) {
    String productId = commandHandler.handle(cmd);
    // Return the write model's data directly (don't query read model immediately)
    // The client already knows what they just created
    ProductDto result = new ProductDto(productId, cmd.name(), cmd.price());
    return ResponseEntity.status(HttpStatus.CREATED).body(result);
}
```

**When to use CQRS:**
- Read and write loads are significantly different in scale
- Multiple different read models needed from the same data
- Complex domain logic on write side, complex reporting on read side
- Combined with Event Sourcing

**When NOT to use CQRS:**
- Simple CRUD applications (basic to-do list, settings management)
- Small teams — overhead not worth the benefit
- Strong consistency required (CQRS adds eventual consistency complexity)
- Read and write loads are similar and simple

---

## 5. Event Sourcing — Complete Deep Dive

### 5.1 What is Event Sourcing?

**Traditional approach** (state-based):
```
orders table:
id=123, status=SHIPPED, address="123 Main St", total=99.99, updated_at=2024-01-15
```
You only know the *current* state. You don't know how you got there.

**Event Sourcing approach**:
```
order_events table:
id=1, order_id=123, type=OrderPlaced,   payload={address:"123 Main St", total:99.99}, ts=2024-01-10
id=2, order_id=123, type=PaymentTaken,  payload={amount:99.99, card:"****1234"},      ts=2024-01-10
id=3, order_id=123, type=OrderPicked,   payload={warehouse:"WH-01"},                 ts=2024-01-12
id=4, order_id=123, type=OrderShipped,  payload={courier:"FedEx", tracking:"ABC123"},ts=2024-01-15
```
You know *everything* that happened. Current state is derived by replaying these events.

**Real-world analogy:** A bank account. The balance is not stored directly — it is computed from the ledger of all transactions (deposits, withdrawals, transfers). The ledger IS the truth. The balance is derived. If you need the balance at any past date, replay transactions up to that date.

**"The log IS the data"** — The event log is the source of truth, not a derived artifact.

---

### 5.2 Core Concepts

**Event:** An immutable fact about something that happened in the past. Named in past tense.
```java
// Events are immutable value objects — records are perfect for this
public record OrderPlacedEvent(
    String orderId,
    String customerId,
    List<OrderItem> items,
    String deliveryAddress,
    BigDecimal totalAmount,
    Instant occurredAt
) implements DomainEvent {}

public record PaymentProcessedEvent(
    String orderId,
    String transactionId,
    BigDecimal amount,
    String cardLast4,
    Instant occurredAt
) implements DomainEvent {}

public record OrderShippedEvent(
    String orderId,
    String courierName,
    String trackingNumber,
    Instant occurredAt
) implements DomainEvent {}
```

**Aggregate:** A cluster of domain objects treated as a single unit of consistency. In event sourcing, the aggregate rebuilds its state by replaying events through `apply()` methods.

**Event Store:** An append-only database where events are persisted. Never update, never delete.

**Projection:** A read model built by processing events. Also called a "view model" or "query model."

**Snapshot:** A periodic snapshot of the aggregate's state to avoid replaying thousands of events on every load.

---

### 5.3 Event Store Design

**Schema:**
```sql
CREATE TABLE events (
    id              BIGSERIAL PRIMARY KEY,
    aggregate_id    VARCHAR(255) NOT NULL,
    aggregate_type  VARCHAR(255) NOT NULL,
    event_type      VARCHAR(255) NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB,                    -- correlation_id, causation_id, user_id
    version         INTEGER NOT NULL,         -- version within the aggregate stream
    global_sequence BIGSERIAL,                -- global ordering across all aggregates
    occurred_at     TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Optimistic concurrency: prevent two transactions from both writing version 3
    CONSTRAINT uq_aggregate_version UNIQUE (aggregate_id, version)
);

CREATE INDEX idx_events_aggregate ON events(aggregate_id, version);
CREATE INDEX idx_events_global_seq ON events(global_sequence);

CREATE TABLE snapshots (
    aggregate_id    VARCHAR(255) PRIMARY KEY,
    aggregate_type  VARCHAR(255) NOT NULL,
    state           JSONB NOT NULL,
    version         INTEGER NOT NULL,         -- version at time of snapshot
    created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);
```

**Optimistic concurrency control:** A `PostgresEventStore` (JdbcTemplate) implements two operations:
- `saveEvents(aggregateId, events, expectedVersion)` — inserts each event with an incrementing `version`. The `UNIQUE(aggregate_id, version)` constraint means a concurrent write at the same version throws `DuplicateKeyException`, which you translate to an `OptimisticConcurrencyException` so the caller can reload and retry.
- `loadEvents(aggregateId[, fromVersion])` — selects events ordered by `version`, deserializing each `payload` back into its `DomainEvent` class.

This is the event-sourcing equivalent of JPA's `@Version` optimistic locking.

---

### 5.4 Java Aggregate Implementation

A base `AggregateRoot` provides the machinery: `raiseEvent(event)` applies the event to update state *and* stages it in an `uncommittedEvents` list for persistence; `replayEvents(events)` rebuilds state from the stored log by dispatching each event to the matching `apply(...)` method (Axon does this dispatch for you; a hand-rolled version uses reflection). The aggregate below shows the important part — business methods raise events, and **state changes happen only inside `apply()` methods**.

```java
// ============ BANK ACCOUNT AGGREGATE ============

public class BankAccount extends AggregateRoot {

    private String accountHolderId;
    private BigDecimal balance;
    private AccountStatus status;
    private String currency;

    // Private constructor — use static factory method
    private BankAccount() {}

    // Factory method — creates new account (raises AccountOpenedEvent)
    public static BankAccount open(String accountId, String holderId,
                                   BigDecimal initialDeposit, String currency) {
        BankAccount account = new BankAccount();
        account.raiseEvent(new AccountOpenedEvent(accountId, holderId, initialDeposit, currency));
        return account;
    }

    // Business method — raises events, does NOT directly modify state
    public void deposit(BigDecimal amount) {
        if (status != AccountStatus.ACTIVE) {
            throw new AccountNotActiveException(id);
        }
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidAmountException("Deposit amount must be positive");
        }
        raiseEvent(new MoneyDepositedEvent(id, amount, balance.add(amount)));
    }

    public void withdraw(BigDecimal amount) {
        if (status != AccountStatus.ACTIVE) {
            throw new AccountNotActiveException(id);
        }
        if (amount.compareTo(balance) > 0) {
            throw new InsufficientFundsException(id, balance, amount);
        }
        raiseEvent(new MoneyWithdrawnEvent(id, amount, balance.subtract(amount)));
    }

    public void close(String reason) {
        if (status == AccountStatus.CLOSED) {
            throw new AccountAlreadyClosedException(id);
        }
        raiseEvent(new AccountClosedEvent(id, reason));
    }

    // ============ APPLY METHODS — state mutations ONLY happen here ============
    // These are called both when raising new events AND when replaying stored events

    private void apply(AccountOpenedEvent event) {
        this.id = event.accountId();
        this.accountHolderId = event.holderId();
        this.balance = event.initialDeposit();
        this.currency = event.currency();
        this.status = AccountStatus.ACTIVE;
    }

    private void apply(MoneyDepositedEvent event) {
        this.balance = event.newBalance();
    }

    private void apply(MoneyWithdrawnEvent event) {
        this.balance = event.newBalance();
    }

    private void apply(AccountClosedEvent event) {
        this.status = AccountStatus.CLOSED;
    }
}
```

The aggregate **repository** ties it together:
- `load(id)` — optionally restore the latest snapshot, then `replayEvents()` for events after the snapshot version to rebuild current state.
- `save(account)` — call `eventStore.saveEvents(...)` with the uncommitted events (optimistic concurrency check built in), publish them so projectors/sagas react, clear the uncommitted list, and optionally take a snapshot every N events.

> **Common junior mistake:** Mutating state directly (e.g., `this.balance = newBalance` inside `withdraw()`). In event sourcing, business methods only raise events; state changes happen ONLY inside `apply()`. Skip this and replaying the event log will no longer reproduce the correct state.

---

### 5.5 Projections and Read Models

```java
// ============ ACCOUNT BALANCE PROJECTION ============

@Component
@Slf4j
public class AccountBalanceProjection {

    @Autowired private AccountReadModelRepository readModelRepository;

    @EventListener
    @Transactional
    public void on(AccountOpenedEvent event) {
        AccountReadModel model = new AccountReadModel(
            event.accountId(), event.holderId(),
            event.initialDeposit(), event.currency(), "ACTIVE"
        );
        readModelRepository.save(model);
        log.info("Read model created for account: {}", event.accountId());
    }

    @EventListener
    @Transactional
    public void on(MoneyDepositedEvent event) {
        readModelRepository.findById(event.accountId()).ifPresent(model -> {
            model.setBalance(event.newBalance());
            model.setLastTransactionAt(Instant.now());
            readModelRepository.save(model);
        });
    }

    // on(MoneyWithdrawnEvent) is identical: update balance + lastTransactionAt.
}
```

The same events can feed **multiple projections** — e.g. a separate `TransactionHistoryProjection` that appends a `TransactionRecord` (DEPOSIT/WITHDRAWAL) per event to power a statement view.

**Catch-up subscription (rebuild a projection from scratch):** because every event is preserved, you can always rebuild a read model by clearing it and replaying the full history — e.g. `eventStore.streamAllEvents("BankAccount", projection::apply)`. This is also how you add a brand-new projection after the fact.

---

### 5.6 Snapshots

A snapshot stores an aggregate's serialized state at a given version (a `snapshots` table: `aggregateId`, `aggregateType`, `state` JSON, `version`). Loading then becomes: restore the latest snapshot (say version 950), load only events after it (951–1000), and replay just those 50 instead of all 1000 — a big speedup for long-lived aggregates.

Snapshot trigger strategies: every N events (`version % 100 == 0`), on-demand via an admin endpoint, time-based (every 24h), or size-based once the event count crosses a threshold.

---

### 5.7 Event Versioning

Over time, events evolve. Since old events are immutable and stored forever, you need strategies for schema changes. The key rule: **never modify events already in the store** — add new fields or transform on read.

- **Additive changes** (safest, no migration): add an optional field with a default. Old events deserialize fine with the field null/empty.
- **Upcasters**: when structure changes (e.g. `fullName` split into `firstName`/`lastName`), an upcaster transforms an old `...V1` event into the new `...V2` shape during deserialization.
- **Schema version in metadata**: store a `schema_version` column on each event so the loader knows which upcaster (if any) to apply.

**Event versioning strategies summary:**

| Strategy | When to use |
|---|---|
| Additive fields only | Simple field additions — safest, no migration needed |
| Weak schema (permissive deserialization) | Tolerate missing fields with defaults |
| Upcasters | Structural changes — transform old events to new format on load |
| Multiple event versions | Keep both `UserRegisteredV1` and `UserRegisteredV2` classes |
| Event migration | One-time bulk migration of stored events (risky, rarely recommended) |

---

### 5.8 Benefits and Challenges

**Benefits:**

| Benefit | Explanation |
|---|---|
| Complete audit trail | Every change is recorded as an immutable event — who changed what and when |
| Temporal queries | "What was the account balance on March 1st?" — replay events up to that date |
| Event replay for debugging | Reproduce any bug by replaying the exact sequence of events that caused it |
| Multiple projections | Same events power different read models (dashboard, reports, search index) |
| Natural CQRS enabler | Events are the bridge between write and read sides |
| Undo/Redo | Theoretically possible by applying/unapplying events |
| Decoupled consumers | New services can subscribe to historical event streams and catch up |

**Challenges:**

| Challenge | Explanation |
|---|---|
| Query complexity | Cannot query current state directly — must build projections |
| Learning curve | Different mental model from traditional CRUD |
| Schema evolution | Event upcasting adds complexity as schemas change |
| Event store technology | Specialized tool (EventStoreDB) or custom implementation required |
| Eventual consistency | Read models may lag behind writes |
| Not for simple CRUD | Massive overhead for simple create/read/update/delete with no audit requirements |

---

## 6. Event Sourcing + CQRS Together

Event Sourcing and CQRS are naturally complementary. Event Sourcing generates a stream of events — CQRS uses those events to build optimized read models.

**Architecture (text diagram):**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CLIENT                                             │
│            │ Commands                         │ Queries                     │
└────────────│─────────────────────────────────│─────────────────────────────┘
             │                                 │
             ↓                                 ↓
┌────────────────────────┐         ┌────────────────────────┐
│    COMMAND SIDE        │         │     QUERY SIDE         │
│                        │         │                        │
│  CommandHandler        │         │  QueryHandler          │
│      │                 │         │      │                 │
│      ↓                 │         │      ↓                 │
│  Aggregate             │         │  Read Model            │
│  (BankAccount)         │         │  (AccountBalanceView)  │
│      │                 │         │                        │
│      ↓                 │         └────────────────────────┘
│  Event Store           │                  ↑
│  (append-only)─────────┼──Events──────────┤
│      │                 │                  │
│      ↓                 │         Projector/Event Handler
│  Events published      │         (updates read model)
└────────────────────────┘
```

**Full flow, step by step:**
```
1. Client sends Command: DepositMoneyCommand(accountId="ACC-1", amount=100)

2. CommandHandler:
   a. Load BankAccount from EventStore (replay events or use snapshot)
   b. Call account.deposit(100)
   c. BankAccount raises MoneyDepositedEvent internally
   d. Save uncommitted events to EventStore
   e. Publish MoneyDepositedEvent

3. EventStore:
   Appends: {aggregate_id:"ACC-1", type:"MoneyDepositedEvent", version:5, payload:{...}}

4. Projector receives MoneyDepositedEvent:
   Updates AccountBalanceView: {accountId:"ACC-1", balance:1100.00}

5. Client queries: GetAccountBalanceQuery(accountId="ACC-1")

6. QueryHandler:
   Reads from AccountBalanceView (fast, no event replay needed for queries)
   Returns: AccountBalanceDto(accountId="ACC-1", balance=1100.00)
```

**In Axon**, this maps cleanly onto annotations: an `@Aggregate` class with `@CommandHandler` methods that call `AggregateLifecycle.apply(event)` (command side) and `@EventSourcingHandler` methods that mutate aggregate state on replay; plus a separate `@Component` projector with `@EventHandler` methods that update the read-model views (query side). See section 8 for the full Axon example.

---

## 7. When to Use What

### 7.1 Decision Framework

**Use Saga when:**
- Business operation spans multiple microservices/databases
- 2PC is not feasible (different DB technologies, third-party services)
- Eventual consistency is acceptable for the business operation
- The operation can be described as a sequence of steps with compensations

**Use Choreography Saga when:**
- Saga has 3 or fewer steps
- Services are truly independent and should not know about each other
- You want maximum autonomy and decoupling
- Simple event-based reactions are sufficient

**Use Orchestration Saga when:**
- Saga has 4+ steps
- Complex conditional flows (if payment method A fails, try method B)
- You need clear visibility into the saga state
- The team needs to understand and debug the flow easily
- Regulatory compliance requires audit of business processes

**Use CQRS when:**
- Read load significantly exceeds write load (10x or more)
- Multiple different representations of the same data are needed
- Write side has complex domain logic, read side has complex query patterns
- Combining with Event Sourcing

**Do NOT use CQRS when:**
- The application is simple CRUD
- The team is small and cannot absorb the complexity
- Strong read-after-write consistency is required
- There are no meaningfully different read and write patterns

**Use Event Sourcing when:**
- Complete audit trail is a business requirement (banking, healthcare, legal)
- Temporal queries needed ("state at time T")
- Domain events are the primary integration mechanism
- Multiple projections needed from the same data
- Undo/redo or event replay for debugging is valuable

**Do NOT use Event Sourcing when:**
- Simple CRUD with no audit requirements
- The team is not familiar with the pattern
- Low event volume — overhead not worth the complexity
- Reporting queries need complex aggregations across many aggregates (projections can handle this but adds work)

### 7.2 Combined Decision Matrix

| Scenario | Recommendation |
|---|---|
| Multi-service checkout flow | Saga (orchestration) + Outbox pattern |
| E-commerce product catalog | CQRS (separate write/read stores) |
| Banking transaction history | Event Sourcing + CQRS |
| Simple user registration | Neither — plain Spring + JPA is fine |
| Order fulfillment with 6+ steps | Saga (orchestration) + Axon |
| Real-time search with complex filters | CQRS (Elasticsearch read model) |
| Financial ledger with audit | Event Sourcing |
| IoT data ingestion + reporting | CQRS (write to TimescaleDB, read from aggregated views) |

---

## 8. Spring + Axon Framework Complete Example

### 8.1 Maven Dependencies

```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- Axon Framework -->
    <dependency>
        <groupId>org.axonframework</groupId>
        <artifactId>axon-spring-boot-starter</artifactId>
        <version>4.9.3</version>
    </dependency>
    <!-- Axon Kafka Extension -->
    <dependency>
        <groupId>org.axonframework.extensions.kafka</groupId>
        <artifactId>axon-kafka-spring-boot-starter</artifactId>
        <version>4.0.0</version>
    </dependency>

    <!-- Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>

    <!-- Database -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>

    <!-- Utilities -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

### 8.2 Complete Order Processing with Axon

A full Axon order flow wires together five kinds of components (the saga is already shown in 3.7, so here is just the gist of the rest):

```java
// AGGREGATE — command side: @CommandHandler validates + applies events,
// @EventSourcingHandler rebuilds state. State changes ONLY via events.
@Aggregate
public class OrderAggregate {

    @AggregateIdentifier private String orderId;
    private OrderStatus status;
    protected OrderAggregate() {}                 // required by Axon

    @CommandHandler
    public OrderAggregate(CreateOrderCommand cmd) {
        if (cmd.getQuantity() <= 0) throw new IllegalArgumentException("Quantity must be positive");
        AggregateLifecycle.apply(new OrderCreatedEvent(cmd.getOrderId(), cmd.getProductId(), cmd.getQuantity()));
    }

    @EventSourcingHandler
    public void on(OrderCreatedEvent event) {     // state mutation lives here
        this.orderId = event.getOrderId();
        this.status = OrderStatus.PENDING;
    }
    // handle(ConfirmOrderCommand)/handle(CancelOrderCommand) + their @EventSourcingHandlers follow the same shape.
}
```

The remaining pieces, omitted for brevity, all follow standard Axon patterns:
- **Commands/Events** — plain classes; commands carry `@TargetAggregateIdentifier`, events are past-tense facts.
- **Saga** (`OrderManagementSaga`) — orchestrates the cross-service steps; see 3.7.
- **Projector** (`@Component` with `@EventHandler` methods) — builds an `OrderSummary` read model.
- **REST controller** — injects `CommandGateway` (send commands) and `QueryGateway` (run queries), both returning `CompletableFuture`.

### 8.3 application.yml for Axon

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orders_db
    username: postgres
    password: password
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect

axon:
  axonserver:
    servers: localhost:8124   # Axon Server for event store and message routing
  eventhandling:
    processors:
      order-processor:
        mode: tracking          # Tracking processor (persistent position)
        thread-count: 2
  serializer:
    general: jackson
    events: jackson
    messages: jackson

kafka:
  bootstrap-servers: localhost:9092
```

---

## 9. Kafka for Saga and Event Sourcing

### 9.1 Using Kafka as an Event Store

Kafka can serve as an event store with important caveats:

**Advantages:**
- Built-in replication and durability (configured with `min.insync.replicas`)
- High throughput (millions of events/second)
- Built-in consumer groups for multiple independent projections
- Log compaction for keeping latest state per key
- Native integration with stream processing (Kafka Streams, ksqlDB)

**Limitations compared to EventStoreDB/PostgreSQL event store:**
- No optimistic concurrency control out of the box
- No aggregate-level event retrieval (must scan the whole topic or use consumer offsets)
- Retention policy may expire old events (configure `retention.ms=-1` for infinite retention)
- Schema evolution is harder without a schema registry

```
Topic per aggregate type:
  bank-account-events  (all BankAccount aggregate events)
  order-events         (all Order aggregate events)

Partition key = aggregateId → guarantees ordering within a partition
```

### 9.2 Kafka Topics as Event Streams

Create an event-store topic with enough partitions (partition key = `aggregateId`, which guarantees ordering per aggregate), replicas for durability, and infinite retention so events are never expired:

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

The producer should be configured for durability/idempotence: `enable.idempotence=true`, `acks=all`, `retries=MAX_VALUE`, with `aggregateId` as the message key.

### 9.3 Consumer Groups for Projections

Each independent projection uses its **own consumer group**, so all of them receive every event and track their own offset:

```java
@KafkaListener(topics = "bank-account-events", groupId = "balance-projection")
public void updateBalanceProjection(ConsumerRecord<String, String> record) {
    // deserialize and apply event to the balance read model
}

@KafkaListener(topics = "bank-account-events", groupId = "fraud-detection")
public void detectFraud(ConsumerRecord<String, String> record) {
    // same events, different consumer group → independent processing
}
// e.g. add a "reporting-projection" group for a reporting DB, etc.
```

### 9.4 Exactly-Once with Kafka Transactions

For exactly-once publishing, set a `transactional.id` on the producer and wrap the sends in `@Transactional("kafkaTransactionManager")`. If an exception is thrown mid-batch, the Kafka transaction is aborted and none of the events become visible to consumers — avoiding partially published event batches. (Even so, downstream consumers should remain idempotent.)

---

## 10. Interview Questions & Answers

### Saga Pattern

**Q1: What is the Saga pattern? Why is it used in microservices?**

A: The Saga pattern is a way to manage distributed transactions across multiple microservices without using 2PC. A saga is a sequence of local transactions — each service performs its own ACID transaction and publishes an event or message to trigger the next step. If any step fails, compensating transactions are executed in reverse order to undo the completed steps. It is used because each microservice has its own database, making traditional distributed transactions (2PC) impractical, brittle, and a performance bottleneck.

---

**Q2: What is the difference between choreography-based and orchestration-based sagas?**

A: In choreography, there is no central controller — each service listens for events from other services and reacts autonomously. Service A publishes an event → Service B listens and acts → publishes its own event → Service C listens, etc. In orchestration, a central saga orchestrator sends explicit commands to each participant and collects replies. The orchestrator contains the full business logic and acts as a state machine. Choreography is better for simple, loosely coupled flows; orchestration is better for complex flows where visibility and control matter.

---

**Q3: What are compensating transactions? What properties must they have?**

A: A compensating transaction is a business operation that semantically undoes a previously completed local transaction. They must be: (1) **idempotent** — executing them multiple times has the same effect as once, because the message triggering compensation may be delivered more than once; (2) **semantically correct** — they must reverse the business effect, not just the data change. For example, the compensation for "charge credit card" is "refund to credit card," not a database rollback. Some operations like "send email" cannot be undone — in these cases, the compensation is a corrective action (send a "sorry, cancelled" email).

---

**Q4: How do you handle partial failures in a saga?**

A: Through compensating transactions executed in reverse order. When step N fails: step N's compensation runs (if needed), then step N-1's compensation, N-2, and so on back to step 1. The orchestrator tracks which steps completed and issues compensations for each. Critically: (a) all compensations must be idempotent (retry-safe); (b) use semantic locks on affected data to prevent concurrent modifications during compensation; (c) persist saga state in a durable store so that if the orchestrator crashes, it can resume from the correct compensation step after restart.

---

**Q5: What is the Outbox Pattern? Why is it needed?**

A: The Outbox Pattern solves the dual-write problem: when a service must both update its database and publish an event/message to Kafka, these are two separate operations that cannot be made atomic. If the DB commits but Kafka publish fails, the event is lost and the saga never progresses. The solution: write the event to an `outbox` table in the same database transaction as the business data. A separate message relay process (Debezium CDC or a polling publisher) reads unpublished events from the outbox and publishes them to Kafka. This guarantees at-least-once delivery because if the relay crashes after publishing but before marking the event as published, it will retry — so consumers must be idempotent.

---

**Q6: What is a semantic lock and why is it used in sagas?**

A: A semantic lock is a flag in the data record (e.g., an order status of `PROCESSING`) that indicates the record is part of an in-progress saga. Other requests that try to modify this record check the flag and either reject or wait. This prevents the "lost update" anomaly where two concurrent sagas both read the same record, both decide to modify it, and one's changes overwrite the other's.

---

**Q7: How do you ensure idempotency in saga steps?**

A: Each step checks whether it has already been executed before performing the action. Common approaches: (1) Store a `processed_events` table keyed by event ID — before processing, check if the event ID was already handled; (2) Use the business key as a unique constraint — attempting to create a duplicate payment for the same order fails gracefully; (3) Use optimistic locking — the version field prevents applying the same change twice. The kafka consumer should also commit offsets only after the business operation succeeds.

---

**Q8: What is the Axon Framework and how does it help with sagas?**

A: Axon Framework is a Java framework that provides first-class support for CQRS, Event Sourcing, and Sagas. For sagas, it provides: `@Saga` to mark saga classes, `@StartSaga` for the initiating event, `@SagaEventHandler` for events the saga handles, `@EndSaga` for completion, and automatic saga state persistence between events. Axon automatically associates sagas with aggregate IDs, persists saga state in a database, and resumes sagas after application restarts. It eliminates the boilerplate of building a saga state machine from scratch.

---

### CQRS

**Q9: What is CQRS? What problem does it solve?**

A: CQRS (Command Query Responsibility Segregation) separates the write model (commands that change state) from the read model (queries that return data). It solves several problems: (1) Read and write sides often have very different performance characteristics and scaling needs — reads may be 10–100x more frequent than writes; (2) Write side has complex domain logic, read side needs denormalized views for efficient querying; (3) A single model that serves both reads and writes creates impedance mismatch — the normalized write model is inefficient for reads, and a denormalized read model is wrong for writes.

---

**Q10: What is the difference between CQS and CQRS?**

A: CQS (Command Query Separation) is a method-level principle: a method either modifies state (command — returns void) OR returns data (query — has no side effects), never both. CQRS applies this concept at the architectural level, separating the entire write stack (command handlers, domain model, write DB) from the read stack (query handlers, read models, read DB). CQS is about object design; CQRS is about application architecture.

---

**Q11: When would you NOT use CQRS?**

A: Avoid CQRS when: (1) The application is simple CRUD — the overhead of maintaining two models is not justified; (2) The team is small and cannot absorb the complexity cost; (3) Read and write loads are similar in complexity and scale; (4) Strong read-after-write consistency is required — CQRS introduces eventual consistency between write and read stores; (5) You have no clear use case for multiple distinct read models. A simple Spring + JPA application with `@Service` and `@Repository` is the right choice for most CRUD apps.

---

**Q12: How do you handle eventual consistency in CQRS? What is the read-after-write problem?**

A: The read-after-write problem: a user creates or updates a resource and immediately queries it, but the read model hasn't been updated yet (event handler lag), so they see stale data. Solutions: (1) Return the written data directly from the command endpoint instead of redirecting to a query — the API already knows what was just written; (2) Optimistic UI — update the UI immediately with the data just submitted, without waiting for a read-model query; (3) Wait for read model update — poll until the read model version matches the write; (4) Add a version number to the read model and include the expected version in the command response — the client can retry the query until it sees the expected version.

---

**Q13: How do you synchronize write and read models in CQRS?**

A: Via domain events. The command handler publishes domain events after a successful write. Event handlers (projectors) subscribe to these events and update the read models. In Spring, this can be done with `ApplicationEventPublisher` + `@EventListener` (synchronous, in-process) or Kafka (asynchronous, cross-service). The Outbox Pattern ensures events are not lost during the write → publish handoff.

---

### Event Sourcing

**Q14: What is Event Sourcing? How does it differ from traditional state storage?**

A: In traditional state storage, the database holds only the *current state* — the latest values of all fields. History is lost. In Event Sourcing, the database holds an append-only log of all *events* that occurred. Current state is derived by replaying these events. The event log is the source of truth, not the current state. Key differences: (1) Traditional: `UPDATE orders SET status='SHIPPED'`; Event Sourcing: `INSERT INTO events (type='OrderShipped', orderId=123, ...)`; (2) Traditional queries are fast (direct table read); Event Sourcing needs projections for fast queries; (3) Traditional has no built-in history; Event Sourcing has complete history.

---

**Q15: What are projections in event sourcing?**

A: Projections (also called read models or view models) are current-state representations built by replaying events. They are the Event Sourcing equivalent of a database table in traditional systems. A projector listens for events and updates the projection accordingly. Since events are immutable, projections can always be rebuilt from scratch by replaying all events. Multiple different projections can be built from the same event stream — for example: account balance view, transaction history view, monthly statement view, fraud detection model.

---

**Q16: What are snapshots in event sourcing? When should you use them?**

A: A snapshot is a serialized copy of an aggregate's state at a specific event version. When loading an aggregate, instead of replaying all events from the beginning, you load the latest snapshot and replay only the events that occurred after it. Use snapshots when: (1) Aggregates accumulate hundreds or thousands of events — replaying all of them on every command becomes slow; (2) The aggregate is accessed frequently in read-heavy patterns. A common strategy: take a snapshot every N events (e.g., 50 or 100). Trade-off: snapshots add storage overhead and snapshot loading logic, but dramatically reduce replay time for long-lived aggregates.

---

**Q17: How do you handle schema changes in event sourcing? (Event versioning)**

A: Events are immutable and stored forever, so when the event schema must change: (1) **Additive changes** — add optional fields with defaults (safest, no migration); (2) **Upcasters** — transform old-format events to new format during deserialization; an `OrderPlacedV1` event is automatically upcasted to `OrderPlacedV2` when loaded from the store; (3) **Multiple version classes** — keep both `OrderPlacedEventV1` and `OrderPlacedEventV2` as separate classes with separate handlers; (4) **Event migration** — bulk-transform stored events to new format (risky, rarely recommended). The key principle: never modify existing events in the store. Always write new events or transform on read.

---

**Q18: What is the difference between Event Sourcing and CDC (Change Data Capture)?**

A: CDC captures changes to a database (inserts, updates, deletes) at the infrastructure level — it reads the database transaction log (WAL) and publishes change events. Event Sourcing captures domain events at the application level — developers explicitly define and publish business events with rich domain meaning. Key differences: (1) CDC events are technical (`ROW_INSERTED`, `ROW_UPDATED`); ES events are business-meaningful (`OrderPlaced`, `PaymentProcessed`); (2) CDC works on any existing database without code changes; ES requires redesigning the data model; (3) CDC cannot distinguish the *reason* for a change; ES events carry full business context; (4) CDC is used for data replication/integration; ES is a primary architectural pattern for the domain.

---

**Q19: How do you combine CQRS with Event Sourcing?**

A: They combine naturally: Event Sourcing stores all domain events in the event store. These events are published to projectors. Projectors build optimized read models (CQRS query side). Commands go to the aggregate (CQRS command side), which stores state as events. Queries go to the read models, which are fast and denormalized. The event store serves as the bridge: it is the write model's persistence mechanism AND the source of data for all read models. Every projection is eventually consistent with the write side.

---

**Q20: What guarantees does an append-only event store provide?**

A: (1) **Immutability** — events are never modified or deleted, ensuring auditability and correctness; (2) **Optimistic concurrency** — the `(aggregateId, version)` unique constraint prevents two concurrent transactions from both writing at the same version, catching lost updates; (3) **Causality ordering** — events within an aggregate are always in causal order (version 1 before 2 before 3); (4) **Complete history** — no information is ever lost; (5) **Replayability** — any projection can be rebuilt by replaying the full event history.

---

**Q21: How would you implement the order processing saga in your system?**

A: I would use orchestration-based saga with the Outbox Pattern:
1. `OrderService` creates the order in `PENDING` state and writes an `OrderCreatedEvent` to the outbox table in the same DB transaction.
2. Debezium CDC reads the outbox and publishes to Kafka topic `order-events`.
3. `SagaOrchestrator` (separate service with durable saga state table) receives the event and sends `ReserveStockCommand` to `InventoryService`.
4. `InventoryService` processes the command idempotently (checks if already processed), reserves stock, publishes `StockReservedEvent`.
5. Orchestrator receives the reply, updates saga state, sends `ProcessPaymentCommand` to `PaymentService`.
6. On payment success, orchestrator sends `ConfirmOrderCommand`. On failure, sends `ReleaseStockCommand` (compensation) then `CancelOrderCommand`.
7. All participants implement idempotent event handlers. Orchestrator persists state after each step so it can resume after crashes.
I would use Axon Framework for the orchestrator to avoid building the state machine infrastructure from scratch.

---

**Q22: How does Axon Framework handle saga persistence?**

A: Axon serializes the entire saga object (including all instance fields) and stores it in a saga store (backed by a relational database by default). When an event is published that matches a saga's association property (e.g., `orderId`), Axon loads the saga from the store, calls the appropriate `@SagaEventHandler` method, then saves the updated saga state back. If the application crashes between events, the saga is automatically recovered from the persistent store on restart. Axon also maintains an association table that maps property values (like `orderId="123"`) to saga instances.

---

**Q23: What is the difference between a command and an event in CQRS/ES?**

A: A **command** is an intent — a request to do something. It can be rejected (business rule validation fails). It is named in imperative: `CreateOrderCommand`, `DepositMoneyCommand`. A **event** is a fact — something that has already happened. It cannot be rejected. It is named in past tense: `OrderCreatedEvent`, `MoneyDepositedEvent`. Commands are sent to aggregates; aggregates validate them and produce events. Events are stored in the event store and published to projectors/sagas. Commands are typically sent to a single handler; events can be subscribed to by multiple handlers.

---

**Q24: How do you prevent duplicate processing of Kafka messages in saga steps?**

A: Multiple layers: (1) **Kafka consumer idempotence** — track processed message offsets; (2) **Business-level idempotence** — check if the business action has already been performed using a unique key (e.g., `paymentRepository.existsByOrderId(orderId)`); (3) **Database unique constraints** — attempting to insert a duplicate record fails gracefully; (4) **Processed event log** — store event IDs in a `processed_events` table; before handling, check if the event ID is already there. The `@Transactional` boundary should cover both the idempotency check and the business operation to prevent TOCTOU races.

---

**Q25: What is a pivot transaction in a saga?**

A: The pivot transaction is the point of no return in a saga — the transaction that, once committed, means the saga will run to completion (no compensation needed). Steps before the pivot are compensable; steps after are retriable. For an order saga: creating the order (compensable) → reserving stock (compensable) → charging the credit card (pivot — once charged, you don't "undo" it, you issue a refund as a separate transaction) → sending confirmation email (retriable — retry until it sends, but if all else fails, the money was taken, so the order must complete). Identifying the pivot transaction helps design compensation flows.

---

**Q26: How would you test a choreography-based saga?**

A: Unit testing individual services is straightforward. The challenge is integration testing. Approaches: (1) **Embedded Kafka** — use `@EmbeddedKafka` in Spring tests; publish events and verify that the right Kafka messages are produced and consumed; (2) **Contract testing** — use Pact to define event contracts between services; (3) **End-to-end saga test** — spin up all services with Docker Compose + TestContainers, trigger the initiating event, poll for the expected final state; (4) **Saga choreography mock** — use a test framework that simulates the full event flow by injecting events at each step and verifying compensations. The hardest part is testing failure scenarios — inject failures at specific steps and verify that compensations execute correctly.

---

**Q27: What are the trade-offs of using Kafka as an event store vs EventStoreDB?**

A: Kafka as event store: high throughput, built-in consumer groups, battle-tested infrastructure, but lacks aggregate-level querying, has no built-in optimistic concurrency, and retention policies may lose events. EventStoreDB is purpose-built: provides aggregate streams, server-side subscriptions with positions, built-in optimistic concurrency, projections as first-class features, and guaranteed infinite retention. EventStoreDB is the right choice when Event Sourcing is the primary pattern. Kafka is better when you already have Kafka for messaging and need a good-enough event store without additional infrastructure.

---

**Q28: How does the `version` field in the event store prevent concurrent updates?**

A: Each event has a `version` number within its aggregate stream (version 1, 2, 3...). The `UNIQUE(aggregateId, version)` constraint in the DB means only one transaction can write version 5 for a given aggregate. When loading an aggregate, you note its current version (e.g., 4). When saving new events, you start from version 5. If another transaction already wrote version 5 while you were processing, your insert throws a `DuplicateKeyException`, which you translate to an `OptimisticConcurrencyException`. The caller retries by reloading the aggregate (now at version 5) and reapplying the command. This is the event sourcing equivalent of JPA's `@Version` optimistic locking.

---

**Q29: What is the global sequence in the event store? Why is it important?**

A: (Senior-level awareness.) A monotonically increasing number assigned to every event across all aggregates. It lets a projector checkpoint its position and resume after a crash without scanning every aggregate stream. Axon Server and EventStoreDB provide it natively.

---

**Q30: How do you rebuild a projection after adding a new read model?**

A: Catch-up subscription: (1) Start a new projector that reads from the beginning of the event store (position 0); (2) Apply each historical event to build the initial state of the new read model; (3) Once the projector catches up to the live position, switch to processing new events in real-time; (4) The service can serve queries from the new read model once it is fully caught up. In Axon Framework, tracking processors support this natively — configure them with `initialTrackingToken = GapAwareTrackingToken.of(0)`. The key advantage of Event Sourcing: rebuilding any projection is always possible because all historical events are preserved.

---

**Q31: What is the difference between an event handler and an event sourcing handler in Axon?**

A: `@EventSourcingHandler` (inside `@Aggregate`) rebuilds the aggregate's state when replaying events — used only within the aggregate class to update its fields. It is called both when raising new events AND when loading the aggregate from the event store. `@EventHandler` (inside `@Component`, typically a projector) reacts to published events to update read models, send notifications, start sagas, etc. The distinction: `@EventSourcingHandler` is for write-side state reconstruction; `@EventHandler` is for read-side reactions.

---

**Q32: What is log compaction in Kafka and how does it relate to event sourcing?**

A: Log compaction keeps only the latest message per key. For an event store you do NOT want this — you need all events, so use `cleanup.policy=delete` with `retention.ms=-1`. Compaction suits a "latest-state" topic (a read-model cache) where only the current state per key matters.

---

**Q33: How do you handle a saga that gets stuck in an intermediate state after a system crash?**

A: Saga state is persisted to a durable store (database or Axon Server). On system restart: (1) Load all in-progress sagas; (2) For each saga, determine its current state from the persisted state machine; (3) Resume by re-triggering the pending command or re-checking for the expected reply event. Additionally: (a) Use timeouts — if a saga step doesn't receive a reply within N minutes, trigger a timeout event that either retries the command or rolls back; (b) Dead letter queues — failed message processing goes to a DLQ for manual investigation; (c) Monitoring alerts for sagas stuck in non-terminal states beyond a threshold duration.

---

**Q34: What is the "at-least-once delivery" guarantee and why is idempotency critical?**

A: At-least-once delivery means a message is guaranteed to be delivered but may be delivered multiple times (due to retries after timeouts or failures). Kafka guarantees at-least-once delivery by default (offset is committed only after successful processing; if the consumer crashes before committing, the message is re-delivered). Since messages can be duplicated, ALL message handlers must be idempotent — processing the same message twice must produce the same result as processing it once. For saga steps: check `IF NOT EXISTS` before inserting records, use unique constraints as safety nets, and store processed event IDs.

---

**Q35: How does Axon Server differ from using Kafka directly for event distribution?**

A: (Senior-level awareness.) Axon Server is a purpose-built event store + message router (event store with global sequence, command/query routing, saga persistence). Kafka is a general-purpose distributed log. Use Axon Server for a full Axon CQRS/ES stack; use Kafka for cross-technology/external event distribution. They are often used together.

---

**Q36: What happens if a compensating transaction itself fails?**

A: (Senior-level awareness.) One of the hardest saga problems. Mitigations: make compensations idempotent and retry until they succeed; after N retries, route to a dead-letter queue and escalate to humans (some reversals are manual); keep a compensation log for audit. Accept that perfect automated compensation isn't always possible — the business process must allow for credit notes or manual resolution.

---

**Q37: What is eventual consistency and how do you communicate it to business stakeholders?**

A: Eventual consistency means that after a write operation, readers may temporarily see stale data, but given enough time (typically milliseconds to seconds), all readers will see the latest data. To stakeholders: frame it as a known and acceptable trade-off for scalability and resilience. Analogy: "When you post on social media, your friend in another country may see the post 2 seconds later than your friend next door — but they will all see it." In practice, for most business operations (e-commerce orders, user profiles), millisecond lags are imperceptible. For critical operations (payment confirmation), use synchronous consistency for that specific path and eventual consistency elsewhere.

---

**Q38: What is a process manager and how is it different from a saga?**

A: (Senior-level awareness.) Roughly: saga = distributed transaction coordination with compensation; process manager = a more general long-running workflow (days/weeks, human tasks, branching). The terms are often used interchangeably — Axon calls both "sagas."

---

**Q39: How would you monitor sagas in production?**

A: (Senior-level awareness.) Key signals: a saga-state dashboard (count by state, flag STUCK), saga duration metrics with alerts, compensation/failure rate, dead-letter-queue accumulation, distributed tracing using the saga ID as correlation ID (Zipkin/Jaeger), and Kafka consumer lag per group (projectors falling behind).

---

**Q40: What is the "tell, don't ask" principle and how does it relate to event sourcing?**

A: "Tell, don't ask" means instead of asking an object for its state and then deciding what to do, you tell the object to do something. In event sourcing: instead of `if (account.getBalance() > amount) { account.setBalance(account.getBalance() - amount); }` (asking), you do `account.withdraw(amount)` (telling). The aggregate internally validates and applies the event. State changes happen ONLY inside the aggregate's `apply()` methods, triggered by events — you never set state directly from outside. This encapsulates business rules inside the aggregate and ensures state only changes via domain events, maintaining the integrity of the event log.

---

**Q41: What are the ACID properties of an individual saga step?**

A: Each individual step in a saga is a local ACID transaction within one service's database. The saga as a whole is ACD without I (Isolation): (A) Atomicity is provided locally for each step; (C) Consistency is maintained locally; (D) Durability is provided locally; but (I) Isolation is NOT guaranteed globally — other sagas can see intermediate states of this saga (e.g., the order in PENDING state while the saga is running). This is why semantic locks and careful saga design are critical — to prevent anomalies caused by the lack of global isolation.

---

**Q42: How does the Outbox Pattern guarantee at-least-once delivery without losing events?**

A: The guarantee comes from three properties: (1) **Atomic write** — the outbox event is written in the same database transaction as the business data; either both succeed or both fail — the event is never lost; (2) **Persistent relay** — the message relay (Debezium or polling publisher) reads from the outbox and marks events as published only after Kafka confirms delivery; (3) **Retry on failure** — if Kafka publish fails, the relay retries until success; if the relay crashes mid-publish, it retries from the last successfully published event. The downside: events may be published more than once (at-least-once), so consumers must be idempotent.

---

## 11. Quick Reference Cheat Sheet

```
WHICH PATTERN? (pick the smallest thing that solves your problem)
  Saga            → you need ONE business action to span MULTIPLE services
                    and stay consistent (place order → reserve stock → charge card)
  CQRS            → reads and writes have VERY different needs
                    (complex queries / high read volume vs simple writes)
  Event Sourcing  → you must keep the FULL HISTORY of changes
                    (audit log, "how did we get to this state?", time-travel)

WHEN **NOT** TO USE THEM (these are advanced patterns — they add real complexity)
  → A single service with one database? Use a normal ACID transaction. Done.
  → Simple CRUD app? Plain reads/writes beat CQRS every time.
  → Don't need history/audit? Skip Event Sourcing — current-state storage is simpler.
  Rule of thumb: don't reach for these unless the simple approach has actually failed.

SAGA STYLES
  Choreography  → no central brain; each service reacts to events and emits its own
                  Pick when: few steps, you want loose coupling, no single owner needed
  Orchestration → one orchestrator tells each service what to do, in order
                  Pick when: many steps / complex logic — you want one place to see the flow

COMPENSATING TRANSACTION (the heart of Saga)
  You CANNOT roll back across services (no global ROLLBACK).
  So you "undo" with a NEW forward action that reverses the effect:
    charged the card  → issue a refund
    reserved stock    → release the reservation
  It's a business-level undo, not a database rollback.

2PC  → Two-Phase Commit: lock everyone, then commit together. Strong consistency,
       but blocks/locks resources and one slow/dead service stalls all → avoid across microservices.
Saga → Sequence of LOCAL transactions + compensations. No global lock; eventually consistent.

ACID → strong: every transaction leaves the DB fully consistent right now (single DB).
BASE → relaxed: Basically Available, Soft state, Eventually consistent (across services).

COMMON JUNIOR MISTAKES
  ✗ Trying to use 2PC / a distributed transaction across microservices
        → use a Saga instead.
  ✗ Forgetting idempotency → a retried/duplicate event charges the card twice.
        → make handlers safe to run more than once (dedupe by event/message id).
  ✗ Modifying state directly (setBalance(...)) in event sourcing
        → state may ONLY change by applying an event ("tell, don't ask").
  ✗ Expecting reads to be instantly up to date in CQRS / event sourcing
        → the read model is eventually consistent; design the UX for that.
  ✗ No compensation path → a saga that fails halfway leaves orphaned/inconsistent data.

GLOSSARY (terms juniors trip on)
  Idempotency           → running the same operation twice = same result as once.
                          (deleting an already-deleted row changes nothing extra)
  Eventual consistency  → data is NOT in sync immediately, but all copies catch up "soon".
  Aggregate             → a cluster of objects treated as one unit for changes;
                          the consistency boundary (e.g., Order + its OrderLines).
  Projection / read model → a query-optimized view BUILT from events, used only for reads.
  Event store           → append-only log of all events; the source of truth in event sourcing
                          (you rebuild current state by replaying events).
```

---

*End of Document — Microservices: Saga Pattern, CQRS, and Event Sourcing*
