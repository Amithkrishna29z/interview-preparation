# Clean Architecture — Interview Preparation Study Guide

> A deep-dive into Clean Architecture covering theory, layers, SOLID principles, real-world patterns, and the most commonly asked interview questions.

---

## 1. What is Clean Architecture?

Clean Architecture is a **software design philosophy** proposed by **Robert C. Martin (Uncle Bob)** in 2012. It combines ideas from Hexagonal Architecture, Onion Architecture, and DCI to produce a system that is:

- **Independent of frameworks** — the framework is a tool, not the foundation
- **Independently testable** — business rules can be tested without UI, database, or any external element
- **Independent of the UI** — the UI can change without changing business rules
- **Independent of the database** — you can swap Oracle for MongoDB without touching business logic
- **Independent of any external agency** — business rules don't know anything about the outside world

**Real-world analogy:** Think of Clean Architecture like the human body. Your **brain** (domain/business logic) doesn't know if it's controlling a left hand or a right hand, doesn't care what language you speak, and doesn't change based on what clothes you wear. The core logic is completely isolated from the outer, changeable details.

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
WRONG (outer depends on inner is OK, inner depending on outer is FORBIDDEN):

Use Case ──────────────────→ Database
          (business logic knows about MySQL — VIOLATION!)

CORRECT:

Use Case ──→ Repository Interface (defined in use case layer)
                       ↑
               Repository Implementation (in infrastructure layer)
```

**What this means in practice:**

| Layer | Can depend on | Cannot depend on |
|---|---|---|
| Entities | Nothing | Everything outer |
| Use Cases | Entities only | Interface Adapters, Frameworks |
| Interface Adapters | Use Cases, Entities | Frameworks/DBs directly |
| Frameworks | Everything | (Outermost layer) |

**The crossing-layer rule:** When data crosses a layer boundary, it must be in the form of simple data structures or DTOs — never pass framework objects (like a JPA Entity or HttpRequest) into inner layers.

---

## 3. The Four Layers

```
┌──────────────────────────────────────────────┐
│  LAYER 4: FRAMEWORKS & DRIVERS               │
│  (Spring Boot, JPA, REST, React, Kafka...)   │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │  LAYER 3: INTERFACE ADAPTERS           │  │
│  │  (Controllers, Presenters, Repositories│  │
│  │   Gateways, Mappers)                   │  │
│  │                                        │  │
│  │  ┌──────────────────────────────────┐  │  │
│  │  │  LAYER 2: USE CASES              │  │  │
│  │  │  (Application Business Rules)    │  │  │
│  │  │  PlaceOrderUseCase               │  │  │
│  │  │  RegisterUserUseCase             │  │  │
│  │  │                                  │  │  │
│  │  │  ┌────────────────────────────┐  │  │  │
│  │  │  │  LAYER 1: ENTITIES         │  │  │  │
│  │  │  │  (Enterprise Rules)        │  │  │  │
│  │  │  │  Order, User, Product      │  │  │  │
│  │  │  └────────────────────────────┘  │  │  │
│  │  └──────────────────────────────────┘  │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

---

## 4. Layer 1 — Entities (Enterprise Business Rules)

Entities encapsulate the **most general, high-level business rules** of the enterprise. They are the least likely to change when something external changes.

**Characteristics:**
- Pure Java/Python/etc. objects — no framework annotations
- Contain core business logic and validation
- Can be used by many different applications in the enterprise
- Completely framework-independent

```java
// ENTITY — Pure domain object, no Spring, no JPA, no framework
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private Money totalAmount;

    // Business rules live here
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
        if (status != OrderStatus.PENDING) {
            throw new DomainException("Cannot add items to a " + status + " order");
        }
        if (quantity <= 0) {
            throw new DomainException("Quantity must be positive");
        }
        items.add(new OrderItem(product, quantity));
        recalculateTotal();
    }

    // Business rule: minimum order amount
    public void confirm() {
        if (items.isEmpty()) {
            throw new DomainException("Cannot confirm an empty order");
        }
        if (totalAmount.isLessThan(Money.of(10.00))) {
            throw new DomainException("Minimum order value is $10.00");
        }
        this.status = OrderStatus.CONFIRMED;
    }

    // Getters only — no setters (immutable from outside)
    public OrderId getId() { return id; }
    public OrderStatus getStatus() { return status; }
    public Money getTotalAmount() { return totalAmount; }
}

// Value Object — immutable, compared by value not identity (e.g. Money, Email).
// Validates in its constructor and has no setters. Kept short here; the idea is
// "a small wrapper that is always valid and compared by its value, not identity".
```

> **Interview tip:** Entities in Clean Architecture are NOT the same as JPA `@Entity` classes. JPA entities are infrastructure concerns. A clean architecture entity is a pure domain object.

---

## 5. Layer 2 — Use Cases (Application Business Rules)

Use Cases contain the **application-specific business rules**. They orchestrate the flow of data to and from entities, and direct those entities to use their business rules to achieve the goal of the use case.

**Characteristics:**
- One class per use case (Single Responsibility Principle)
- Depends only on entities and interfaces it defines itself
- Contains no framework code
- Defines the **input port** (interface the controller calls) and **output port** (interface for delivering results)

```java
// INPUT PORT — interface the controller calls (defined in use case layer)
public interface PlaceOrderUseCase {
    PlaceOrderResponse execute(PlaceOrderRequest request);
}

// REQUEST/RESPONSE — simple data structures crossing layer boundaries (no framework objects!)
public record PlaceOrderRequest(String customerId, List<OrderItemRequest> items) {}
public record OrderItemRequest(String productId, int quantity) {}
public record PlaceOrderResponse(String orderId, String status, double totalAmount) {}

// OUTPUT PORTS — interfaces defined in the use case layer, implemented in infrastructure
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}
public interface ProductRepository { Optional<Product> findById(ProductId id); }
public interface NotificationService { void sendOrderConfirmation(CustomerId id, Order order); }

// USE CASE IMPLEMENTATION — pure business orchestration (constructor injection, no framework code)
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
            order.addItem(product, item.quantity());  // entity validates
        }

        order.confirm();                                  // entity validates min amount, etc.
        orderRepository.save(order);                      // output port (don't know if MySQL/Mongo)
        notificationService.sendOrderConfirmation(customerId, order);  // output port (email/SMS)

        return new PlaceOrderResponse(
            order.getId().getValue(), order.getStatus().name(),
            order.getTotalAmount().getAmount().doubleValue());  // return DTO, not the entity
    }
}
```

**Key insight:** The use case doesn't know:
- Whether it's being called from a REST API, a CLI, a message queue
- Whether data is stored in MySQL, MongoDB, or in memory
- Whether notifications go via email, SMS, or push

---

## 6. Layer 3 — Interface Adapters

This layer **converts data** between the format convenient for use cases/entities and the format convenient for external agencies (DB, web, etc.).

Contains: **Controllers, Presenters, Gateways, Mappers, Repository Implementations**

```java
// CONTROLLER — converts HTTP request to use case request and back (no business logic)
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final PlaceOrderUseCase placeOrderUseCase;

    public OrderController(PlaceOrderUseCase placeOrderUseCase) {
        this.placeOrderUseCase = placeOrderUseCase;
    }

    @PostMapping
    public ResponseEntity<PlaceOrderResponse> placeOrder(@RequestBody @Valid PlaceOrderHttpRequest req) {
        // Map HTTP request → use case request (crossing layer boundary)
        PlaceOrderRequest useCaseRequest = new PlaceOrderRequest(
            req.getCustomerId(),
            req.getItems().stream().map(i -> new OrderItemRequest(i.getProductId(), i.getQuantity())).toList());

        PlaceOrderResponse response = placeOrderUseCase.execute(useCaseRequest);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}

// REPOSITORY IMPLEMENTATION — adapter between use case port and JPA.
// A mapper converts domain Order <-> OrderJpaEntity at this boundary.
@Repository
public class JpaOrderRepository implements OrderRepository {

    private final SpringDataOrderRepository springRepo;  // Spring Data JPA repo
    private final OrderMapper mapper;

    public JpaOrderRepository(SpringDataOrderRepository springRepo, OrderMapper mapper) {
        this.springRepo = springRepo;
        this.mapper = mapper;
    }

    @Override
    public void save(Order order) {
        springRepo.save(mapper.toJpaEntity(order));         // Domain → JPA
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return springRepo.findById(id.getValue()).map(mapper::toDomain);  // JPA → Domain
    }
}
```

The other adapters follow the same shape:
- `OrderJpaEntity` (a separate `@Entity @Table` class) and `OrderMapper` keep JPA out of the domain.
- `EmailNotificationService implements NotificationService` is a driven adapter that uses Spring Mail — an infrastructure detail the use case never sees.

---

## 7. Layer 4 — Frameworks & Drivers (Infrastructure)

The outermost layer. Contains all **framework-specific** code, configuration, and wiring.

```java
// Spring Boot Configuration / Wiring
@Configuration
public class ApplicationConfig {

    // Manually wire dependencies (alternative to @Component scanning)
    @Bean
    public PlaceOrderUseCase placeOrderUseCase(
        OrderRepository orderRepository,
        ProductRepository productRepository,
        NotificationService notificationService
    ) {
        return new PlaceOrderUseCaseImpl(orderRepository, productRepository, notificationService);
    }
}

// Spring Data JPA interface (infrastructure)
public interface SpringDataOrderRepository extends JpaRepository<OrderJpaEntity, String> {
    List<OrderJpaEntity> findByCustomerId(String customerId);
}

// Main application entry point
@SpringBootApplication
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}

// application.yml — infrastructure configuration
// spring:
//   datasource:
//     url: jdbc:mysql://localhost:3306/orders
//     username: root
//     password: secret
//   jpa:
//     hibernate:
//       ddl-auto: validate
```

---

## 8. SOLID Principles in Clean Architecture

Clean Architecture is built on SOLID. Understanding how they apply together is critical for interviews.

### S — Single Responsibility Principle

> "A class should have only one reason to change."

```java
// VIOLATION — OrderService does too much
class OrderService {
    void placeOrder(Order order) { /* business logic */ }
    void sendEmail(Order order) { /* email logic */ }
    void saveToDatabase(Order order) { /* DB logic */ }
    void generateInvoicePdf(Order order) { /* PDF logic */ }
}

// CORRECT — each class has ONE responsibility
class PlaceOrderUseCase { /* orchestrates the order placement */ }
class EmailNotificationService { /* only sends emails */ }
class JpaOrderRepository { /* only persists orders */ }
class InvoiceGenerator { /* only generates invoices */ }
```

### O — Open/Closed Principle

> "Open for extension, closed for modification."

```java
// VIOLATION — adding a new payment method requires modifying this class
class PaymentProcessor {
    void process(String method, double amount) {
        if (method.equals("CREDIT_CARD")) { /* ... */ }
        else if (method.equals("PAYPAL")) { /* ... */ }
        // Adding crypto requires modifying this file
    }
}

// CORRECT — new payment methods extend without modifying existing code
public interface PaymentGateway {
    void process(double amount);
}

public class CreditCardGateway implements PaymentGateway { /* ... */ }
public class PayPalGateway implements PaymentGateway { /* ... */ }
public class CryptoGateway implements PaymentGateway { /* NEW — no modification needed */ }
```

### L — Liskov Substitution Principle

> "Subtypes must be substitutable for their base types."

A subtype must honour the contract of its base type. The classic violation is `Square extends Rectangle` overriding `setWidth` to also change height — code that works with a `Rectangle` breaks when handed a `Square`. Fix: model both as `Shape` with a `getArea()` contract instead of inheriting behaviour that breaks expectations.

### I — Interface Segregation Principle

> "No client should be forced to depend on interfaces it does not use."

```java
// VIOLATION — fat interface forces implementations to stub methods they don't need
interface UserRepository {
    Optional<User> findById(UserId id);
    void save(User user);
    List<User> search(String query);
    void bulkImport(List<User> users);   // Not all repos need this!
}

// CORRECT — split into focused interfaces (read / write / search)
interface UserReadRepository  { Optional<User> findById(UserId id); }
interface UserWriteRepository { void save(User user); }
interface UserSearchRepository { List<User> search(String query); }
```

### D — Dependency Inversion Principle

> "Depend on abstractions, not concretions. High-level modules should not depend on low-level modules."

```java
// VIOLATION — use case depends on a concrete implementation
class PlaceOrderUseCase { private MySQLOrderRepository repository; }   // concrete — VIOLATION

// CORRECT — use case depends on an abstraction; the MySQL impl is injected from outside
class PlaceOrderUseCase { private OrderRepository repository; }        // interface — CORRECT
```

---

## 9. Dependency Inversion in Practice

This is the mechanism that makes Clean Architecture work. The **inner layer defines an interface**, the **outer layer implements it**, and a **DI container** (Spring) connects them.

```
┌─────────────────────────────────────────────────────┐
│                   USE CASE LAYER                     │
│                                                      │
│  PlaceOrderUseCase ──→ OrderRepository (interface)  │
│                              ↑                       │
│                      DEPENDS ON ABSTRACTION          │
└──────────────────────────────┼──────────────────────┘
                               │ implements
┌──────────────────────────────┼──────────────────────┐
│              INFRASTRUCTURE LAYER                    │
│                                                      │
│                    JpaOrderRepository                │
│                    (implements the interface)        │
└─────────────────────────────────────────────────────┘

Arrow of dependency: Use Case → Interface ← Implementation
Arrow of control:    Use Case → Interface → Implementation (at runtime via DI)
```

```java
// The interface is OWNED by the use case layer
// package com.example.application.port.out;
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

// The implementation is in the infrastructure layer
// package com.example.infrastructure.persistence;
@Repository
public class JpaOrderRepository implements OrderRepository {
    // Spring injects this at runtime
    // Use case doesn't know this class exists
}

// Spring wires it together:
// When PlaceOrderUseCase asks for OrderRepository,
// Spring injects JpaOrderRepository automatically
```

---

## 10. Ports & Adapters (Hexagonal Architecture)

Clean Architecture is closely related to Hexagonal (Ports & Adapters) Architecture. Understanding both helps in interviews.

```
                        ┌─────────────────────┐
REST API ──→ [Adapter] ─→│                     │─→ [Adapter] ──→ MySQL
                        │                     │
CLI      ──→ [Adapter] ─→│    HEXAGON          │─→ [Adapter] ──→ Email
                        │   (Domain +         │
Message  ──→ [Adapter] ─→│   Use Cases)        │─→ [Adapter] ──→ S3
Queue                   │                     │
                        └─────────────────────┘
                              ↑ PORTS ↑
```

**Two types of ports:**

| Port Type | Direction | Example |
|---|---|---|
| **Driving / Input Port** | External → System | REST API calls use case |
| **Driven / Output Port** | System → External | Use case calls repository |

```java
// DRIVING PORT (input) — how the outside world calls the application
public interface PlaceOrderUseCase { PlaceOrderResponse execute(PlaceOrderRequest req); }

// DRIVEN PORTS (output) — how the application calls the outside world
public interface OrderRepository { void save(Order order); }
public interface PaymentGateway  { PaymentResult charge(Money amount, String cardToken); }

// DRIVING ADAPTER — translates an external call into a driving-port call
@RestController
public class OrderController { private final PlaceOrderUseCase useCase; /* ... */ }

// DRIVEN ADAPTERS — implement output ports to reach external systems
@Repository public class JpaOrderRepository implements OrderRepository {}
@Component  public class StripePaymentGateway implements PaymentGateway {}
```

> **Awareness note (junior scope):** Hexagonal is just Clean Architecture viewed from the side — same dependency rule, same ports-and-adapters idea. You rarely need more than the "controller calls an input port, the use case calls output ports" mental model.

---

## 11. Clean Architecture vs Other Architectures

### Traditional Layered (N-Tier) Architecture

```
┌─────────────────────┐
│   Presentation      │  ← Knows about Application
├─────────────────────┤
│   Application       │  ← Knows about Domain
├─────────────────────┤
│   Domain            │  ← Knows about Persistence
├─────────────────────┤
│   Persistence       │  ← Database
└─────────────────────┘

Problem: Domain depends on Persistence!
         Business logic is tied to the database.
```

### Clean Architecture

```
┌──────────────────────────────┐
│  Infrastructure / Frameworks │  ← Knows about Use Cases
│  ┌────────────────────────┐  │
│  │   Interface Adapters   │  │  ← Knows about Use Cases
│  │  ┌──────────────────┐  │  │
│  │  │   Use Cases       │  │  │  ← Knows about Entities only
│  │  │  ┌────────────┐  │  │  │
│  │  │  │  Entities   │  │  │  │  ← Knows nothing
│  │  │  └────────────┘  │  │  │
│  │  └──────────────────┘  │  │
│  └────────────────────────┘  │
└──────────────────────────────┘

Business logic has ZERO database dependency!
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
| Learning curve | Low | High | Low |

---

## 12. Domain-Driven Design Alignment

Clean Architecture and DDD align very well. Key DDD concepts map to Clean Architecture layers:

| DDD Concept | Clean Architecture Layer | Description |
|---|---|---|
| **Entity** | Entities layer | Objects with identity and lifecycle |
| **Value Object** | Entities layer | Immutable, compared by value |
| **Aggregate** | Entities layer | Cluster of entities, one root |
| **Domain Service** | Entities/Use Case layer | Logic that doesn't fit one entity |
| **Repository** | Output Port (Use Case layer) | Interface for aggregate persistence |
| **Application Service** | Use Cases layer | Orchestrates domain objects |
| **Domain Event** | Use Cases/Entities layer | Something that happened in the domain |
| **Factory** | Entities layer | Creates complex objects |

> **Awareness note (junior scope):** You don't need DDD's full tactical toolkit to use Clean Architecture. The two ideas worth knowing:
> - **Aggregate Root** — one entity (e.g. `Order`) guards a cluster of related entities (its `OrderItem`s); outside code only calls methods on the root, never on the children directly.
> - **Domain Service** — business logic that doesn't naturally belong to a single entity (e.g. a `PricingService` that needs both an `Order` and a `Customer` to compute a discount) lives in its own class rather than being forced onto one entity.

---

## 13. Project Structure & Package Layout

### Package by Layer (Traditional)

```
com.example.
├── controller/
│   └── OrderController.java
├── service/
│   └── OrderService.java
├── repository/
│   └── OrderRepository.java
└── model/
    └── Order.java
```

**Problem:** All order-related code is scattered. Hard to see what the app does.

### Package by Feature / Component (Clean Architecture)

```
com.example.
├── order/
│   ├── domain/                         ← Layer 1: Entities
│   │   ├── Order.java
│   │   ├── OrderItem.java
│   │   ├── OrderStatus.java
│   │   ├── OrderId.java
│   │   └── Money.java
│   ├── application/                    ← Layer 2: Use Cases
│   │   ├── port/
│   │   │   ├── in/
│   │   │   │   ├── PlaceOrderUseCase.java       (Input Port)
│   │   │   │   ├── PlaceOrderRequest.java
│   │   │   │   └── PlaceOrderResponse.java
│   │   │   └── out/
│   │   │       ├── OrderRepository.java          (Output Port)
│   │   │       └── NotificationService.java      (Output Port)
│   │   └── service/
│   │       └── PlaceOrderService.java             (Use Case Impl)
│   └── adapter/                        ← Layer 3: Interface Adapters
│       ├── in/
│       │   └── web/
│       │       ├── OrderController.java
│       │       └── PlaceOrderHttpRequest.java
│       └── out/
│           ├── persistence/
│           │   ├── JpaOrderRepository.java
│           │   ├── OrderJpaEntity.java
│           │   └── OrderMapper.java
│           └── notification/
│               └── EmailNotificationService.java
├── product/
│   ├── domain/
│   ├── application/
│   └── adapter/
└── shared/
    ├── domain/
    │   └── DomainEvent.java
    └── infrastructure/
        └── config/
            └── ApplicationConfig.java
```

> **Interview tip:** "Package by feature" is strongly recommended because it makes the codebase navigable — you can understand what the system does just by looking at top-level package names, not technical layers.

---

## 14. Full Worked Example — Spring Boot

Sections 4–7 already walked the full Order flow end to end. Here is a second, shorter slice — a "Get User Profile" feature — to reinforce how a business rule lives in the **use case**, not the controller. The adapter layer (controller, JPA entity, `JpaUserRepository`, Spring Data interface) follows exactly the same shape shown in Section 6, so it is not repeated.

```java
// ─── DOMAIN LAYER — Entity + Value Object ────────────────────────────────────
public class User {
    private final UserId id;
    private String name;
    private UserRole role;
    private boolean active;
    // constructor validates name/email; behaviour: deactivate(), promoteToAdmin()
    public UserId getId() { return id; }
    public String getName() { return name; }
    public UserRole getRole() { return role; }
    public boolean isActive() { return active; }
}

public record Email(String value) {       // Value Object — validates on creation
    public Email {
        if (value == null || !value.matches("^[\\w.-]+@[\\w.-]+\\.[a-zA-Z]{2,}$"))
            throw new DomainException("Invalid email: " + value);
        value = value.toLowerCase();
    }
}

// ─── APPLICATION LAYER — input port + use case ───────────────────────────────
public interface GetUserProfileUseCase { UserProfileResponse execute(GetUserProfileRequest req); }
public record GetUserProfileRequest(String requesterId, String targetUserId) {}
public record UserProfileResponse(String id, String name, String role, boolean active) {}

@Component
public class GetUserProfileService implements GetUserProfileUseCase {

    private final UserRepository userRepository;   // output port (see Section 6 for the adapter)

    public GetUserProfileService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserProfileResponse execute(GetUserProfileRequest req) {
        User requester = userRepository.findById(new UserId(req.requesterId()))
            .orElseThrow(() -> new EntityNotFoundException("Requester not found"));
        User target = userRepository.findById(new UserId(req.targetUserId()))
            .orElseThrow(() -> new EntityNotFoundException("User not found"));

        // Business rule lives in the use case, not the controller:
        if (!target.isActive() && requester.getRole() != UserRole.ADMIN) {
            throw new AccessDeniedException("Cannot view inactive user");
        }

        return new UserProfileResponse(
            target.getId().getValue(), target.getName(),
            target.getRole().name(), target.isActive());
    }
}
```

---

## 15. Testing Strategy

Clean Architecture makes testing straightforward because each layer can be tested in isolation.

```
TEST PYRAMID in Clean Architecture:

         /\
        /  \       E2E Tests (few, slow)
       /    \      — Full stack, real DB, real HTTP
      /------\
     /        \    Integration Tests (some)
    /          \   — Test adapters with real DB or test containers
   /------------\
  /              \ Unit Tests (many, fast)
 /                \ — Domain entities, Use Cases with mocked ports
/------------------\
```

```java
// ─── UNIT TEST — Entity (no dependencies needed) ─────────────────────────────

class OrderTest {

    @Test
    void should_confirm_order_with_valid_items() {
        Order order = new Order(OrderId.generate(), new CustomerId("c1"));
        order.addItem(aProduct("p1", 25.00), 2);

        order.confirm();

        assertEquals(OrderStatus.CONFIRMED, order.getStatus());
        assertEquals(Money.of(50.00), order.getTotalAmount());
    }

    @Test
    void should_reject_confirmation_of_empty_order() {
        Order order = new Order(OrderId.generate(), new CustomerId("c1"));

        assertThrows(DomainException.class, order::confirm);
    }

    @Test
    void should_reject_confirmation_of_empty_order() {
        Order order = new Order(OrderId.generate(), new CustomerId("c1"));
        assertThrows(DomainException.class, order::confirm);
    }
}

// ─── UNIT TEST — Use Case (mock all ports, no Spring/DB/HTTP) ─────────────────

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

**Integration tests** verify the adapters: spin up a real (or test-container) database and confirm `JpaOrderRepository.save` / `findById` round-trip correctly (`@DataJpaTest`). **Architecture tests** (using the ArchUnit library in CI) can enforce the dependency rule automatically — e.g. "no class in `..domain..` may depend on `..application..` or on `org.springframework..`".

---

## 16. Common Mistakes & Anti-Patterns

### 1. Using JPA Entity as Domain Entity

```java
// WRONG — @Entity annotation leaks JPA into domain
@Entity                          // ← JPA concern in domain layer!
@Table(name = "orders")
public class Order {
    @Id @GeneratedValue
    private Long id;
    @Column
    private String status;
    // No business logic — just getters/setters
}

// CORRECT — separate JPA entity from domain entity
public class Order {             // Pure domain — no annotations
    private OrderId id;
    private OrderStatus status;
    public void confirm() { /* real business logic */ }
}

@Entity @Table(name = "orders")  // JPA concern in infrastructure layer
public class OrderJpaEntity {
    @Id private String id;
    private String status;
}
```

### 2. Business Logic in the Controller

```java
// WRONG — controller contains business rules
@PostMapping("/orders")
public ResponseEntity<?> placeOrder(@RequestBody OrderRequest req) {
    if (req.getItems().isEmpty()) return ResponseEntity.badRequest().body("No items");
    double total = req.getItems().stream().mapToDouble(i -> i.price * i.qty).sum();
    if (total < 10.0) return ResponseEntity.badRequest().body("Min $10");
    // ... more business logic
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
// WRONG — entity is just a data bag (no behavior)
public class Order {
    private String status;
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    // All logic is in OrderService — this is anemic!
}

// CORRECT — rich domain model with behavior
public class Order {
    private OrderStatus status;
    public void confirm() {
        if (items.isEmpty()) throw new DomainException("...");
        this.status = OrderStatus.CONFIRMED;
    }
    public OrderStatus getStatus() { return status; }
    // No setStatus() — state changes only through business methods
}
```

### 4. Use Case Directly Using Framework Classes

```java
// WRONG — use case parameter is a framework type
public User execute(HttpServletRequest request) { ... }   // framework leak!

// CORRECT — use case takes a plain request object
public UserResponse execute(GetUserRequest request) { ... }
```

### 5. Skipping the Ports

```java
// WRONG — use case imports the concrete repository (coupled to infrastructure)
private JpaOrderRepository repo;

// CORRECT — use case depends on the port (interface it owns in the application layer)
private OrderRepository repo;
```

---

## 17. When TO and When NOT TO Use Clean Architecture

### Use Clean Architecture when:

| Scenario | Reason |
|---|---|
| Complex domain with many business rules | Entities and use cases protect logic from framework churn |
| Long-lived application (5+ years) | Easy to swap frameworks, DBs, and UIs over time |
| Multiple interfaces (REST + CLI + messaging) | Core logic is untouched when adding new adapters |
| Large team | Clear layer boundaries reduce merge conflicts |
| High test coverage required | Each layer is independently testable |
| Microservices that need to be extracted later | Clean boundaries make extraction straightforward |

### Do NOT use Clean Architecture when:

| Scenario | Reason |
|---|---|
| Simple CRUD app | Over-engineering — mappers and ports add cost with zero benefit |
| Proof of concept / hackathon | Speed matters more than maintainability |
| Very small team (1-2 devs) | Overhead may slow you down more than it helps |
| Short-lived project | Won't live long enough to benefit from the investment |
| Tight deadline with no refactoring time | Start simpler, extract layers when complexity grows |

> **Interview tip:** Always acknowledge the **trade-offs**. Interviewers love candidates who say "Clean Architecture adds complexity and upfront cost, but pays off in maintainability for complex, long-lived systems." Blind advocacy raises red flags.

---

## 18. Common Interview Questions & Answers

### Q1: What is Clean Architecture and who created it?

**Answer:** Clean Architecture is a software design philosophy by Robert C. Martin (Uncle Bob) that organizes code into concentric layers where **source code dependencies only point inward** — toward higher-level policies. The goal is to produce systems that are independent of frameworks, databases, and UIs so that business logic can be tested and changed in isolation.

---

### Q2: What is the Dependency Rule?

**Answer:** The Dependency Rule states that source code dependencies must always point inward — toward the more abstract, higher-level layers. Nothing in an inner layer can know anything about an outer layer. Specifically, a use case cannot import from the infrastructure layer. This is enforced by defining interfaces (ports) in the inner layer and having the outer layer implement them (Dependency Inversion Principle).

---

### Q3: How does Clean Architecture differ from traditional N-Tier architecture?

**Answer:** In traditional N-Tier architecture, dependencies flow downward and the domain layer typically depends on the persistence layer. This means business logic is coupled to the database. In Clean Architecture, the domain and use case layers define interfaces that the infrastructure layer implements, inverting this dependency. Business logic never imports database code — it defines what it needs and the infrastructure provides it. This makes the business logic testable without a database.

---

### Q4: What is the difference between a domain Entity and a JPA Entity?

**Answer:** A domain Entity is a pure business object that encapsulates business rules and has an identity. It has no framework annotations — just Java. A JPA Entity is an infrastructure concern decorated with `@Entity`, `@Table`, etc. for ORM mapping. In Clean Architecture, these are separate classes: the domain entity lives in the domain layer; the JPA entity lives in the infrastructure adapter. A mapper converts between them at the boundary.

---

### Q5: What is a Port in Clean Architecture / Hexagonal Architecture?

**Answer:** A Port is an interface that defines a contract between the application core and the outside world. There are two types:
- **Input/Driving ports** — interfaces that the application exposes for external callers (e.g., `PlaceOrderUseCase`)
- **Output/Driven ports** — interfaces the application defines but expects the outside world to implement (e.g., `OrderRepository`, `NotificationService`)

The key insight: ports are **owned by the application**, not by the infrastructure.

---

### Q6: What is an Adapter in Hexagonal Architecture?

**Answer:** An Adapter is a concrete implementation that connects a port to a specific external technology. Driving adapters translate external calls (HTTP, CLI, MQ) into calls on input ports. Driven adapters implement output ports to talk to external systems (databases, email services, APIs). This means you can swap MySQL for MongoDB by writing a new driven adapter — the use case is completely unaffected.

---

### Q7: What is an Anemic Domain Model and why is it considered an anti-pattern?

**Answer:** An anemic domain model is when domain objects (entities) contain only data (getters/setters) and all business logic lives in service classes. It's an anti-pattern because it violates the object-oriented principle of encapsulation — data and behavior should be together. In Clean Architecture, entities should be rich with behavior: `order.confirm()`, `order.cancel()`, `order.addItem()`. Logic in services that operates on entity data is a sign of anemic design.

---

### Q8: How do you handle cross-cutting concerns like logging and transactions in Clean Architecture?

**Answer:** Cross-cutting concerns should NOT be inside use cases — that would violate the Single Responsibility Principle. Instead:
- **Transactions:** handled by the adapter layer or a declarative mechanism (`@Transactional` on the adapter, not the use case)
- **Logging:** an aspect (AOP) in the infrastructure layer, or a decorator pattern wrapping the use case
- **Security:** driven by the adapter layer — the controller validates authentication before calling the use case, or passes identity as part of the request object

---

### Q9: How would you test a Use Case in Clean Architecture?

**Answer:** Use case tests are pure unit tests. All output ports (repository, notification service) are mocked with Mockito. No Spring context, no database, no HTTP. This makes tests extremely fast and reliable. You test: does the use case call the right ports? Does it throw the right exceptions? Does it return the correct response? Because the use case only depends on interfaces, mocking is straightforward with `@Mock` + `@InjectMocks`.

---

### Q10: What are the trade-offs of Clean Architecture?

**Answer:**

*Advantages:*
- High testability — inner layers tested without external systems
- Technology independence — swap DB/framework without touching business logic
- Clear separation of concerns — easy to understand responsibilities
- Domain logic is the center — survives framework evolution

*Disadvantages:*
- Significant upfront complexity — many more classes than simple N-tier
- Mapper boilerplate — converting between domain objects and JPA/HTTP objects
- Overkill for CRUD-heavy, low-complexity applications
- Steeper learning curve — team must understand and follow the rules consistently

---

## 19. Quick Revision Cheat Sheet

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
Domain Event → Fact that something happened (OrderConfirmedEvent)

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
✗ Transaction annotation on use case (infrastructure concern)

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

> **Final Interview Tip:** The single most important thing to demonstrate in an interview is **why** each rule exists, not just what it is. "The domain layer doesn't know about JPA **because** if we change the database, the business logic shouldn't need to change" is far more impressive than just reciting layer names. Always connect the rule to the benefit it provides.
