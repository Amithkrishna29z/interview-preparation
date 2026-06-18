# Clean Architecture — Interview Preparation Study Guide

> A junior-focused guide covering Clean Architecture theory, layers, SOLID principles, and common interview questions.

---

## Table of Contents

1. [What is Clean Architecture?](#1-what-is-clean-architecture)
2. [The Dependency Rule](#2-the-dependency-rule)
3. [The Four Layers](#3-the-four-layers)
4. [Layer 1 — Entities](#4-layer-1--entities-enterprise-business-rules)
5. [Layer 2 — Use Cases](#5-layer-2--use-cases-application-business-rules)
6. [Layer 3 — Interface Adapters](#6-layer-3--interface-adapters)
7. [Layer 4 — Frameworks & Drivers](#7-layer-4--frameworks--drivers-infrastructure)
8. [SOLID Principles in Clean Architecture](#8-solid-principles-in-clean-architecture)
9. [Ports & Adapters (Hexagonal Architecture)](#9-ports--adapters-hexagonal-architecture)
10. [Clean Architecture vs Other Architectures](#10-clean-architecture-vs-other-architectures)
11. [DDD Alignment](#11-domain-driven-design-alignment)
12. [Project Structure](#12-project-structure--package-layout)
13. [Testing Strategy](#13-testing-strategy)
14. [Common Mistakes & Anti-Patterns](#14-common-mistakes--anti-patterns)
15. [When to Use (and When Not To)](#15-when-to-use-and-when-not-to)
16. [Interview Q&A](#16-interview-qa)
17. [Quick Revision Cheat Sheet](#17-quick-revision-cheat-sheet)

---

## 1. What is Clean Architecture?

Clean Architecture is a **software design philosophy** by **Robert C. Martin (Uncle Bob)** that organizes code into concentric layers with one rule: **dependencies only point inward**.

- **Independent of frameworks** — the framework is a tool, not the foundation
- **Independently testable** — business rules tested without UI, DB, or external elements
- **Independent of the UI** — UI can change without touching business rules
- **Independent of the database** — swap Oracle for MongoDB without touching business logic

```
          ┌─────────────────────────────────────────┐
          │           FRAMEWORKS & DRIVERS           │  ← Spring, React, MySQL, REST
          │   ┌─────────────────────────────────┐   │
          │   │       INTERFACE ADAPTERS         │   │  ← Controllers, Presenters, Gateways
          │   │   ┌─────────────────────────┐   │   │
          │   │   │       USE CASES          │   │   │  ← Application Business Rules
          │   │   │   ┌─────────────────┐   │   │   │
          │   │   │   │    ENTITIES      │   │   │   │  ← Enterprise Business Rules
          │   │   │   └─────────────────┘   │   │   │
          │   │   └─────────────────────────┘   │   │
          │   └─────────────────────────────────┘   │
          └─────────────────────────────────────────┘

          ← dependencies only point INWARD →
```

---

## 2. The Dependency Rule

> **"Source code dependencies must point only inward, toward higher-level policies."**
> — Robert C. Martin

This is the **single most important rule** of Clean Architecture.

```
WRONG:
Use Case ──────────────────→ Database
          (business logic knows about MySQL — VIOLATION!)

CORRECT:
Use Case ──→ Repository Interface (defined in use case layer)
                       ↑
               Repository Implementation (in infrastructure layer)
```

| Layer | Can depend on | Cannot depend on |
|---|---|---|
| Entities | Nothing | Everything outer |
| Use Cases | Entities only | Interface Adapters, Frameworks |
| Interface Adapters | Use Cases, Entities | Frameworks/DBs directly |
| Frameworks | Everything | (Outermost layer) |

**Crossing-layer rule:** When data crosses a layer boundary, use simple data structures or DTOs — never pass framework objects (JPA Entity, HttpRequest) into inner layers.

---

## 3. The Four Layers

```
┌──────────────────────────────────────────────┐
│  LAYER 4: FRAMEWORKS & DRIVERS               │
│  (Spring Boot, JPA, REST, React, Kafka...)   │
│  ┌────────────────────────────────────────┐  │
│  │  LAYER 3: INTERFACE ADAPTERS           │  │
│  │  (Controllers, Repositories, Mappers)  │  │
│  │  ┌──────────────────────────────────┐  │  │
│  │  │  LAYER 2: USE CASES              │  │  │
│  │  │  (Application Business Rules)    │  │  │
│  │  │  ┌────────────────────────────┐  │  │  │
│  │  │  │  LAYER 1: ENTITIES         │  │  │  │
│  │  │  │  Order, User, Product      │  │  │  │
│  │  │  └────────────────────────────┘  │  │  │
│  │  └──────────────────────────────────┘  │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

---

## 4. Layer 1 — Entities (Enterprise Business Rules)

Entities are pure Java objects that encode your most stable business rules. No framework annotations. No Spring. No JPA.

```java
// Pure domain object — no framework dependencies
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private Money totalAmount;

    public Order(OrderId id, CustomerId customerId) {
        if (id == null) throw new IllegalArgumentException("Order ID required");
        if (customerId == null) throw new IllegalArgumentException("Customer required");
        this.id = id;
        this.customerId = customerId;
        this.items = new ArrayList<>();
        this.status = OrderStatus.PENDING;
        this.totalAmount = Money.ZERO;
    }

    // Business rule: can only add items to a PENDING order
    public void addItem(Product product, int quantity) {
        if (status != OrderStatus.PENDING)
            throw new DomainException("Cannot add items to a " + status + " order");
        if (quantity <= 0)
            throw new DomainException("Quantity must be positive");
        items.add(new OrderItem(product, quantity));
        recalculateTotal();
    }

    // Business rule: minimum order amount
    public void confirm() {
        if (items.isEmpty()) throw new DomainException("Cannot confirm an empty order");
        if (totalAmount.isLessThan(Money.of(10.00)))
            throw new DomainException("Minimum order value is $10.00");
        this.status = OrderStatus.CONFIRMED;
    }

    public OrderId getId() { return id; }
    public OrderStatus getStatus() { return status; }
    public Money getTotalAmount() { return totalAmount; }
}
```

> **Interview tip:** Entities in Clean Architecture are NOT JPA `@Entity` classes. JPA entities are infrastructure concerns. A clean architecture entity is a pure domain object.

---

## 5. Layer 2 — Use Cases (Application Business Rules)

Use Cases orchestrate entities to achieve a single business goal. One class per use case. No framework code.

```java
// INPUT PORT — interface the controller calls (defined in use case layer)
public interface PlaceOrderUseCase {
    PlaceOrderResponse execute(PlaceOrderRequest request);
}

// Simple data structures crossing layer boundaries (no framework objects!)
public record PlaceOrderRequest(String customerId, List<OrderItemRequest> items) {}
public record OrderItemRequest(String productId, int quantity) {}
public record PlaceOrderResponse(String orderId, String status, double totalAmount) {}

// OUTPUT PORTS — interfaces defined here, implemented in infrastructure
public interface OrderRepository { void save(Order order); Optional<Order> findById(OrderId id); }
public interface ProductRepository { Optional<Product> findById(ProductId id); }
public interface NotificationService { void sendOrderConfirmation(CustomerId id, Order order); }

// USE CASE IMPLEMENTATION
@Component
public class PlaceOrderUseCaseImpl implements PlaceOrderUseCase {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final NotificationService notificationService;

    public PlaceOrderUseCaseImpl(OrderRepository orderRepository,
                                 ProductRepository productRepository,
                                 NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.productRepository = productRepository;
        this.notificationService = notificationService;
    }

    @Override
    public PlaceOrderResponse execute(PlaceOrderRequest request) {
        CustomerId customerId = new CustomerId(request.customerId());
        Order order = new Order(OrderId.generate(), customerId);

        for (OrderItemRequest item : request.items()) {
            Product product = productRepository.findById(new ProductId(item.productId()))
                .orElseThrow(() -> new EntityNotFoundException("Product not found"));
            order.addItem(product, item.quantity());
        }

        order.confirm();
        orderRepository.save(order);
        notificationService.sendOrderConfirmation(customerId, order);

        return new PlaceOrderResponse(
            order.getId().getValue(), order.getStatus().name(),
            order.getTotalAmount().getAmount().doubleValue());
    }
}
```

**Key insight:** The use case doesn't know whether it's called from REST or CLI, whether data is stored in MySQL or MongoDB, or whether notifications go via email or SMS.

---

## 6. Layer 3 — Interface Adapters

Converts data between the format convenient for use cases/entities and the format convenient for external agencies (DB, web, etc.).

Contains: **Controllers, Presenters, Gateways, Mappers, Repository Implementations**

```java
// CONTROLLER — maps HTTP ↔ use case (no business logic)
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final PlaceOrderUseCase placeOrderUseCase;

    public OrderController(PlaceOrderUseCase placeOrderUseCase) {
        this.placeOrderUseCase = placeOrderUseCase;
    }

    @PostMapping
    public ResponseEntity<PlaceOrderResponse> placeOrder(@RequestBody @Valid PlaceOrderHttpRequest req) {
        PlaceOrderRequest useCaseRequest = new PlaceOrderRequest(
            req.getCustomerId(),
            req.getItems().stream().map(i -> new OrderItemRequest(i.getProductId(), i.getQuantity())).toList());

        PlaceOrderResponse response = placeOrderUseCase.execute(useCaseRequest);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}

// REPOSITORY IMPLEMENTATION — adapter between the output port and JPA
@Repository
public class JpaOrderRepository implements OrderRepository {

    private final SpringDataOrderRepository springRepo;
    private final OrderMapper mapper;

    public JpaOrderRepository(SpringDataOrderRepository springRepo, OrderMapper mapper) {
        this.springRepo = springRepo;
        this.mapper = mapper;
    }

    @Override
    public void save(Order order) {
        springRepo.save(mapper.toJpaEntity(order));
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return springRepo.findById(id.getValue()).map(mapper::toDomain);
    }
}
```

A separate `OrderJpaEntity` (`@Entity @Table`) and `OrderMapper` keep JPA annotations out of the domain. `EmailNotificationService implements NotificationService` is another driven adapter the use case never sees.

---

## 7. Layer 4 — Frameworks & Drivers (Infrastructure)

The outermost layer. Contains all framework-specific configuration and wiring.

```java
@Configuration
public class ApplicationConfig {
    @Bean
    public PlaceOrderUseCase placeOrderUseCase(
        OrderRepository orderRepository,
        ProductRepository productRepository,
        NotificationService notificationService
    ) {
        return new PlaceOrderUseCaseImpl(orderRepository, productRepository, notificationService);
    }
}

public interface SpringDataOrderRepository extends JpaRepository<OrderJpaEntity, String> {
    List<OrderJpaEntity> findByCustomerId(String customerId);
}

@SpringBootApplication
public class OrderApplication {
    public static void main(String[] args) { SpringApplication.run(OrderApplication.class, args); }
}
```

---

## 8. SOLID Principles in Clean Architecture

### S — Single Responsibility Principle

> "A class should have only one reason to change."

```java
// VIOLATION — one class doing too much
class OrderService {
    void placeOrder(Order order) { /* business logic */ }
    void sendEmail(Order order) { /* email logic */ }
    void saveToDatabase(Order order) { /* DB logic */ }
}

// CORRECT — each class has ONE responsibility
class PlaceOrderUseCase { /* orchestrates order placement */ }
class EmailNotificationService { /* only sends emails */ }
class JpaOrderRepository { /* only persists orders */ }
```

### O — Open/Closed Principle

> "Open for extension, closed for modification."

```java
// VIOLATION — adding a new payment method requires modifying this class
class PaymentProcessor {
    void process(String method, double amount) {
        if (method.equals("CREDIT_CARD")) { /* ... */ }
        else if (method.equals("PAYPAL")) { /* ... */ }
    }
}

// CORRECT — new payment methods extend without modifying existing code
public interface PaymentGateway { void process(double amount); }
public class CreditCardGateway implements PaymentGateway { /* ... */ }
public class PayPalGateway implements PaymentGateway { /* ... */ }
public class CryptoGateway implements PaymentGateway { /* NEW — no modification needed */ }
```

### L — Liskov Substitution Principle

> "Subtypes must be substitutable for their base types."

The classic violation: `Square extends Rectangle` overrides `setWidth` to also change height — code that works with a `Rectangle` breaks when handed a `Square`. Fix: model both as `Shape` with a `getArea()` contract.

### I — Interface Segregation Principle

> "No client should be forced to depend on interfaces it does not use."

```java
// VIOLATION — fat interface forces implementations to stub unused methods
interface UserRepository {
    Optional<User> findById(UserId id);
    void save(User user);
    void bulkImport(List<User> users);  // Not all repos need this!
}

// CORRECT — split into focused interfaces
interface UserReadRepository  { Optional<User> findById(UserId id); }
interface UserWriteRepository { void save(User user); }
```

### D — Dependency Inversion Principle

> "Depend on abstractions, not concretions."

```java
// VIOLATION — use case depends on a concrete class
class PlaceOrderUseCase { private MySQLOrderRepository repository; }

// CORRECT — use case depends on an interface; DI container injects the impl
class PlaceOrderUseCase { private OrderRepository repository; }
```

---

## 9. Ports & Adapters (Hexagonal Architecture)

Clean Architecture is closely related to Hexagonal (Ports & Adapters) Architecture — same dependency rule, viewed from the side.

```
                    ┌─────────────────────┐
REST API → [Adapter]→│                     │→[Adapter]→ MySQL
CLI      → [Adapter]→│   HEXAGON           │→[Adapter]→ Email
Message  → [Adapter]→│  (Domain +          │→[Adapter]→ S3
Queue               │   Use Cases)        │
                    └─────────────────────┘
                          ↑ PORTS ↑
```

| Port Type | Direction | Example |
|---|---|---|
| **Driving / Input Port** | External → System | REST API calls use case |
| **Driven / Output Port** | System → External | Use case calls repository |

```java
// DRIVING PORT (input)
public interface PlaceOrderUseCase { PlaceOrderResponse execute(PlaceOrderRequest req); }

// DRIVEN PORTS (output)
public interface OrderRepository { void save(Order order); }
public interface PaymentGateway  { PaymentResult charge(Money amount, String cardToken); }

// DRIVING ADAPTER
@RestController
public class OrderController { private final PlaceOrderUseCase useCase; /* ... */ }

// DRIVEN ADAPTERS
@Repository public class JpaOrderRepository implements OrderRepository {}
@Component  public class StripePaymentGateway implements PaymentGateway {}
```

---

## 10. Clean Architecture vs Other Architectures

### Traditional N-Tier problem

```
Presentation → Application → Domain → Persistence → Database
Problem: Domain depends on Persistence! Business logic is tied to the DB.
```

### Comparison Table

| Aspect | Traditional N-Tier | Clean Architecture | MVC |
|---|---|---|---|
| Business logic location | Service layer | Use Cases + Entities | Controller/Model (varies) |
| DB dependency in logic | Yes (often) | Never | Often |
| Testability | Hard (needs DB) | Easy (mock ports) | Moderate |
| Framework coupling | High | Low | High |
| Complexity | Low | High | Low-Medium |
| Best for | Small/CRUD apps | Complex domains | Simple web apps |

---

## 11. Domain-Driven Design Alignment

| DDD Concept | Clean Architecture Layer | Description |
|---|---|---|
| **Entity** | Entities layer | Objects with identity and lifecycle |
| **Value Object** | Entities layer | Immutable, compared by value |
| **Aggregate** | Entities layer | Cluster of entities, one root |
| **Domain Service** | Entities/Use Case layer | Logic that doesn't fit one entity |
| **Repository** | Output Port (Use Case layer) | Interface for aggregate persistence |
| **Application Service** | Use Cases layer | Orchestrates domain objects |

> **Two DDD ideas worth knowing:**
> - **Aggregate Root** — one entity (e.g. `Order`) guards its cluster of related entities (`OrderItem`s); outside code only calls methods on the root.
> - **Domain Service** — business logic that doesn't naturally belong to a single entity (e.g. `PricingService` needs both `Order` and `Customer`) lives in its own class.

---

## 12. Project Structure & Package Layout

### Package by Feature (Clean Architecture — recommended)

```
com.example.
├── order/
│   ├── domain/                         ← Layer 1: Entities
│   │   ├── Order.java
│   │   ├── OrderItem.java
│   │   ├── OrderStatus.java
│   │   └── Money.java
│   ├── application/                    ← Layer 2: Use Cases
│   │   ├── port/
│   │   │   ├── in/
│   │   │   │   ├── PlaceOrderUseCase.java
│   │   │   │   ├── PlaceOrderRequest.java
│   │   │   │   └── PlaceOrderResponse.java
│   │   │   └── out/
│   │   │       ├── OrderRepository.java
│   │   │       └── NotificationService.java
│   │   └── service/
│   │       └── PlaceOrderService.java
│   └── adapter/                        ← Layer 3: Interface Adapters
│       ├── in/web/
│       │   ├── OrderController.java
│       │   └── PlaceOrderHttpRequest.java
│       └── out/persistence/
│           ├── JpaOrderRepository.java
│           ├── OrderJpaEntity.java
│           └── OrderMapper.java
└── shared/
    └── infrastructure/config/
        └── ApplicationConfig.java
```

> **Interview tip:** "Package by feature" makes the codebase navigable — you understand what the system does just by looking at top-level package names.

---

## 13. Testing Strategy

```
TEST PYRAMID:
         /\
        /  \       E2E Tests (few) — Full stack, real DB, real HTTP
       /----\
      /      \     Integration Tests — Adapters with real DB / TestContainers
     /--------\
    /          \   Unit Tests (many, fast) — Entities + Use Cases with mocked ports
   /____________\
```

```java
// UNIT TEST — Entity (no dependencies needed)
class OrderTest {
    @Test
    void should_confirm_order_with_valid_items() {
        Order order = new Order(OrderId.generate(), new CustomerId("c1"));
        order.addItem(aProduct("p1", 25.00), 2);
        order.confirm();
        assertEquals(OrderStatus.CONFIRMED, order.getStatus());
    }

    @Test
    void should_reject_confirmation_of_empty_order() {
        Order order = new Order(OrderId.generate(), new CustomerId("c1"));
        assertThrows(DomainException.class, order::confirm);
    }
}

// UNIT TEST — Use Case (mock all ports, no Spring/DB/HTTP)
class PlaceOrderServiceTest {
    @Mock OrderRepository orderRepository;
    @Mock ProductRepository productRepository;
    @Mock NotificationService notificationService;
    PlaceOrderUseCase useCase;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        useCase = new PlaceOrderUseCaseImpl(orderRepository, productRepository, notificationService);
    }

    @Test
    void should_place_order_successfully() {
        when(productRepository.findById(new ProductId("p1")))
            .thenReturn(Optional.of(new Product(new ProductId("p1"), "Widget", Money.of(25.00))));

        PlaceOrderResponse response = useCase.execute(
            new PlaceOrderRequest("customer-1", List.of(new OrderItemRequest("p1", 2))));

        assertEquals("CONFIRMED", response.status());
        verify(orderRepository).save(any(Order.class));
    }
}
```

**Integration tests** verify adapters with a real (or test-container) DB using `@DataJpaTest`. **Architecture tests** (ArchUnit) can enforce the dependency rule in CI automatically.

---

## 14. Common Mistakes & Anti-Patterns

### 1. Using JPA Entity as Domain Entity

```java
// WRONG — @Entity leaks JPA into domain layer
@Entity @Table(name = "orders")
public class Order { @Id @GeneratedValue private Long id; private String status; }

// CORRECT — separate domain entity from JPA entity
public class Order { private OrderId id; private OrderStatus status; public void confirm() { /* real logic */ } }

@Entity @Table(name = "orders")  // lives in infrastructure
public class OrderJpaEntity { @Id private String id; private String status; }
```

### 2. Business Logic in the Controller

```java
// WRONG — controller contains business rules
@PostMapping("/orders")
public ResponseEntity<?> placeOrder(@RequestBody OrderRequest req) {
    if (req.getItems().isEmpty()) return ResponseEntity.badRequest().body("No items");
    double total = req.getItems().stream().mapToDouble(i -> i.price * i.qty).sum();
    if (total < 10.0) return ResponseEntity.badRequest().body("Min $10");
}

// CORRECT — controller only translates and delegates
@PostMapping("/orders")
public ResponseEntity<?> placeOrder(@RequestBody OrderRequest req) {
    PlaceOrderResponse response = placeOrderUseCase.execute(map(req));
    return ResponseEntity.created(URI.create("/orders/" + response.orderId())).body(response);
}
```

### 3. Anemic Domain Model

```java
// WRONG — entity is just a data bag
public class Order {
    private String status;
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }  // all logic lives in a Service
}

// CORRECT — rich domain model with behavior
public class Order {
    private OrderStatus status;
    public void confirm() {
        if (items.isEmpty()) throw new DomainException("...");
        this.status = OrderStatus.CONFIRMED;
    }
    // No setStatus() — state changes only through business methods
}
```

### 4. Skipping the Ports

```java
// WRONG — use case imports the concrete repository
private JpaOrderRepository repo;

// CORRECT — use case depends on the port interface
private OrderRepository repo;
```

### 5. Framework Objects Crossing Layer Boundaries

```java
// WRONG
public User execute(HttpServletRequest request) { ... }

// CORRECT
public UserResponse execute(GetUserRequest request) { ... }
```

---

## 15. When to Use (and When Not To)

| Use Clean Architecture | Reason |
|---|---|
| Complex domain with many business rules | Entities and use cases protect logic from framework churn |
| Long-lived application (5+ years) | Easy to swap frameworks, DBs, and UIs over time |
| Multiple interfaces (REST + CLI + messaging) | Core logic untouched when adding new adapters |
| High test coverage required | Each layer is independently testable |

| Avoid Clean Architecture | Reason |
|---|---|
| Simple CRUD app | Over-engineering — ports and mappers add cost with zero benefit |
| Proof of concept / hackathon | Speed matters more than maintainability |
| Very short-lived project | Won't live long enough to justify the investment |

> **Interview tip:** Always acknowledge trade-offs. "Clean Architecture adds upfront complexity but pays off in maintainability for complex, long-lived systems." Blind advocacy raises red flags.

---

## 16. Interview Q&A

### Q1: What is Clean Architecture and who created it?

Clean Architecture is a design philosophy by Robert C. Martin (Uncle Bob) that organizes code into concentric layers where **dependencies only point inward**. The goal is systems independent of frameworks, databases, and UIs so business logic can be tested and changed in isolation.

---

### Q2: What is the Dependency Rule?

Source code dependencies must always point inward — toward higher-level, more abstract layers. Nothing in an inner layer can know about an outer layer. This is enforced by defining interfaces (ports) in the inner layer and having the outer layer implement them (Dependency Inversion Principle).

---

### Q3: How does Clean Architecture differ from traditional N-Tier?

In N-Tier, dependencies flow downward and the domain layer typically depends on the persistence layer — business logic is coupled to the database. In Clean Architecture, the domain/use case layers define interfaces that infrastructure implements, inverting the dependency. Business logic never imports database code, making it testable without a database.

---

### Q4: What is the difference between a domain Entity and a JPA Entity?

A domain Entity is a pure business object with identity and business rules — no framework annotations. A JPA Entity is an infrastructure concern with `@Entity`, `@Table`, etc. for ORM mapping. They are separate classes: domain entity in the domain layer, JPA entity in the infrastructure adapter, with a mapper converting between them at the boundary.

---

### Q5: What is a Port?

A Port is an interface defining a contract between the application core and the outside world. **Input ports** are interfaces the application exposes (e.g., `PlaceOrderUseCase`). **Output ports** are interfaces the application defines but expects the infrastructure to implement (e.g., `OrderRepository`). Ports are owned by the application layer, not infrastructure.

---

### Q6: What is an Adapter?

An Adapter is a concrete implementation connecting a port to a specific technology. Driving adapters translate external calls (HTTP, CLI) into calls on input ports. Driven adapters implement output ports to talk to external systems (databases, email). You can swap MySQL for MongoDB by writing a new driven adapter — the use case is unaffected.

---

### Q7: What is an Anemic Domain Model?

An anemic domain model is when entities contain only data (getters/setters) and all business logic lives in service classes. It violates encapsulation — data and behavior should be together. Entities should be rich with behavior: `order.confirm()`, `order.cancel()`, `order.addItem()`.

---

### Q8: How do you handle cross-cutting concerns like logging and transactions?

Cross-cutting concerns should not be inside use cases (Single Responsibility Principle). **Transactions:** use `@Transactional` on the adapter, not the use case. **Logging:** AOP in the infrastructure layer, or a decorator wrapping the use case. **Security:** the controller validates authentication before calling the use case, or passes identity as part of the request object.

---

### Q9: How do you test a Use Case?

Use case tests are pure unit tests — all output ports are mocked with Mockito. No Spring context, no database, no HTTP. You verify: does the use case call the right ports? Does it throw the right exceptions? Because the use case only depends on interfaces, mocking is straightforward with `@Mock` + `@InjectMocks`.

---

### Q10: What are the trade-offs of Clean Architecture?

**Advantages:** High testability, technology independence, clear separation of concerns, business logic survives framework evolution.

**Disadvantages:** Significant upfront complexity, mapper boilerplate between domain/JPA/HTTP objects, overkill for CRUD-heavy apps, steeper learning curve requiring consistent team discipline.

---

## 17. Quick Revision Cheat Sheet

```
THE LAYERS (outer → inner)
──────────────────────────
4. Frameworks & Drivers  → Spring, JPA, REST, Kafka, React
3. Interface Adapters    → Controllers, Repositories (impl), Mappers, Gateways
2. Use Cases             → Application Business Rules, Ports (interfaces)
1. Entities              → Enterprise Business Rules, Domain Objects

THE DEPENDENCY RULE
───────────────────
Dependencies always point INWARD (outer → inner)
NEVER: inner layer imports outer layer
ALWAYS: inner layer defines interface, outer layer implements it

SOLID IN CLEAN ARCHITECTURE
────────────────────────────
S — One class per use case; controller ≠ business logic
O — New payment method = new Adapter, no existing code changes
L — Repository implementations are substitutable (MySQL ↔ MongoDB)
I — Separate input/output ports; don't force one fat interface
D — Use Case → Interface ← Implementation (depend on abstractions)

KEY VOCABULARY
──────────────
Port         → Interface defined by inner layer (application owns it)
Adapter      → Concrete implementation connecting port to technology
Driving Port → Input: external world calls the application
Driven Port  → Output: application calls the external world
Entity       → Domain object with identity + behavior (NOT JPA @Entity)
Value Object → Immutable, compared by value (e.g., Money, Email)
Use Case     → Orchestrates entities to achieve one business goal
Aggregate    → Cluster of entities with one root (Order + OrderItems)

CROSSING LAYER BOUNDARIES
──────────────────────────
Pass only: simple data classes (records/POJOs), primitives
Never pass: JPA entities, HttpRequest, Spring objects

TESTING BY LAYER
────────────────
Entities     → Plain unit tests (no mocks needed)
Use Cases    → Unit tests with mocked ports (Mockito)
Adapters     → Integration tests with real DB (TestContainers/@DataJpaTest)
Full system  → E2E tests with @SpringBootTest

COMMON ANTI-PATTERNS
────────────────────
✗ JPA @Entity used as domain entity (layer violation)
✗ Business logic in Controller (wrong layer)
✗ Use case imports concrete Repository (skips port)
✗ Anemic domain model (no behavior in entities)
✗ Use case parameter is HttpServletRequest (framework leak)
✗ @Transactional on use case (infrastructure concern)

WHEN TO USE
───────────
✓ Complex domain with many business rules
✓ Long-lived application (5+ years)
✓ Multiple input channels (REST + messaging + CLI)
✓ High test coverage required
✗ Simple CRUD app
✗ Proof of concept / prototype
✗ Very short project lifespan
```

---

> **Final Interview Tip:** Demonstrate *why* each rule exists, not just what it is. "The domain layer doesn't know about JPA *because* if we change the database, the business logic shouldn't need to change" is far more impressive than reciting layer names. Always connect the rule to the benefit it provides.

---

*Last Updated: 2026-06-18*
