# Software Engineering Principles Interview Questions

> These questions appear in almost every junior developer interview.

---

## Table of Contents
1. [SDLC (Software Development Life Cycle)](#sdlc)
2. [Agile Methodology](#agile-methodology)
3. [Scrum Framework](#scrum-framework)
4. [Kanban](#kanban)
5. [SOLID Principles](#solid-principles)
6. [Design Patterns](#design-patterns)
7. [Clean Code Principles](#clean-code-principles)
8. [Testing Principles](#testing-principles)
9. [Quick Revision Summary](#quick-revision-summary)

---

## SDLC

### Q1: What is SDLC and what are its phases?

SDLC is the step-by-step process used to plan, create, test, and deliver software.

```
Phase 1: Planning       → Scope, timeline, budget, team
Phase 2: Requirements   → What should the software DO? (BRS)
Phase 3: System Design  → HOW to build it? (architecture, DB, tech stack)
Phase 4: Implementation → Developers write code
Phase 5: Testing        → Unit, integration, UAT
Phase 6: Deployment     → Release to production via CI/CD
Phase 7: Maintenance    → Bug fixes, new features, support
```

**SDLC Models:**

| Model | Description | Best For |
|-------|-------------|---------|
| **Waterfall** | Sequential phases, no going back | Fixed requirements, simple projects |
| **Agile** | Iterative, delivery in sprints | Changing requirements, modern apps |
| **Spiral** | Risk-driven, iterative | Large, complex, risky projects |
| **V-Model** | Testing paired with each dev phase | Safety-critical systems |

---

### Q2: Waterfall vs Agile?

- **Waterfall** = Finish all planning first, then build. No mid-way changes.
- **Agile** = Deliver in small increments, adjust based on feedback.

| Feature | Waterfall | Agile |
|---------|-----------|-------|
| **Delivery** | One big delivery at end | Small deliveries every 2-4 weeks |
| **Requirements** | Fixed upfront | Can change anytime |
| **Testing** | Only after development | Continuous throughout |
| **Customer** | Involved at start and end | Involved throughout |
| **Risk** | High (discovered late) | Low (discovered early) |
| **Documentation** | Heavy | Light |

---

## Agile Methodology

### Q3: What is Agile and what are its core values?

Agile is a **mindset** that values flexibility, collaboration, and frequent delivery of working software.

**The 4 Agile Values:**
```
1. Individuals and interactions  OVER  processes and tools
2. Working software              OVER  comprehensive documentation
3. Customer collaboration        OVER  contract negotiation
4. Responding to change          OVER  following a plan
```

**Key Agile Principles (top ones for interviews):**
- Deliver working software frequently (weeks, not months)
- Welcome changing requirements, even late in development
- Business and developers work together daily
- Working software is the primary measure of progress
- Self-organizing teams produce the best results

---

### Q4: Popular Agile frameworks?

| Framework | Best For | Key Feature |
|-----------|---------|------------|
| **Scrum** | Product development | Fixed sprints, defined roles |
| **Kanban** | Operations, support | Visual board, continuous flow |
| **XP** | Tech-focused teams | TDD, pair programming |
| **SAFe** | Large enterprises | Multiple teams, synchronized |

---

## Scrum Framework

### Q5: What is Scrum?

Scrum is the most popular Agile framework. It organizes work into fixed-length **sprints**, with defined roles, artifacts, and ceremonies.

---

### Q6: What are the 3 Scrum Roles?

```
1. PRODUCT OWNER (PO)
   → Represents the customer; owns and prioritizes the Product Backlog
   → Decides WHAT to build and in what order; writes User Stories

2. SCRUM MASTER (SM)
   → Facilitates Scrum; removes blockers; coaches the team
   → NOT a project manager — doesn't give orders ("servant-leader")

3. DEVELOPMENT TEAM
   → Cross-functional (devs, testers, designers); self-organizing
   → 3-9 people; collectively responsible for delivery
```

---

### Q7: What are the Scrum Artifacts?

```
1. PRODUCT BACKLOG   → Master list of all features (PO owns); always evolving
2. SPRINT BACKLOG    → Items selected for this sprint (team owns)
3. INCREMENT         → Working software delivered at end of sprint; must meet DoD
```

---

### Q8: What are the Scrum Events?

```
1. SPRINT              → 2-4 week time-box; no scope changes mid-sprint
2. SPRINT PLANNING     → Select backlog items + plan how to build them; outputs Sprint Goal
3. DAILY STANDUP       → 15 min daily: Yesterday? Today? Blockers?
4. SPRINT REVIEW       → Demo completed work to stakeholders; PO accepts/rejects
5. SPRINT RETROSPECTIVE → What went well? What to improve? Commit to one change.
```

---

### Q9: What is a User Story?

A user story describes a feature from the user's perspective in non-technical language.

**Format:**
```
As a [type of user], I want [to do something], so that [I get some benefit].
```

**Example:**
```
As a customer, I want to add items to my cart, so that I can purchase multiple items at once.
```

**INVEST Criteria** (good user story checklist):
```
I — Independent   V — Valuable    S — Small
N — Negotiable    E — Estimable   T — Testable
```

**Acceptance Criteria example:**
```
User Story: As a user, I want to reset my password.
✅ System sends reset email within 2 minutes
✅ Reset link expires after 24 hours
✅ New password must be min 8 characters
✅ Old password no longer works after reset
```

---

### Q10: What are Story Points and Velocity?

- **Story Points** = Estimate of effort/complexity (not hours). Uses Fibonacci scale: 1, 2, 3, 5, 8, 13, 21…
- **Velocity** = Story points completed per sprint — used to forecast how many sprints remain.

**Planning Poker:** Team estimates simultaneously to avoid anchoring bias; discuss differences and re-estimate.

---

### Q11: What is Definition of Done (DoD)?

A shared checklist that defines when a task is truly "finished." If any item is incomplete, the task is NOT done.

```
Example DoD:
✅ Code reviewed and merged
✅ Unit tests written (80%+ coverage)
✅ Integration tests pass
✅ Deployed to staging
✅ Product Owner accepted
✅ Documentation updated
```

---

## Kanban

### Q12: What is Kanban and how does it differ from Scrum?

Kanban is a visual board where tasks flow through stages continuously — no fixed sprints.

```
┌──────────┬──────────────┬─────────────┬──────────┐
│ BACKLOG  │  IN PROGRESS │   TESTING   │   DONE   │
│          │   (WIP: 3)   │   (WIP: 2)  │          │
└──────────┴──────────────┴─────────────┴──────────┘
```

**WIP Limit:** Max tasks allowed in each column — prevents multitasking overload.

| | Scrum | Kanban |
|--|-------|--------|
| **Cadence** | Fixed sprints | Continuous flow |
| **Roles** | PO, SM, Dev Team | No prescribed roles |
| **Changes** | No changes mid-sprint | Anytime |
| **Best for** | Product development | Operations, support |

---

## SOLID Principles

### Q13: What are SOLID principles?

SOLID = 5 rules for writing clean, maintainable code.

---

**S — Single Responsibility Principle (SRP)**
> A class should have only ONE reason to change.

```java
// BAD: One class doing too many things
class UserManager {
    public void createUser(User user) { ... }
    public void sendWelcomeEmail(User user) { ... }
    public void saveToDatabase(User user) { ... }
}

// GOOD: Each class has ONE job
class UserService    { public void createUser(User user) { ... } }
class EmailService   { public void sendWelcomeEmail(User user) { ... } }
class UserRepository { public void save(User user) { ... } }
```

---

**O — Open/Closed Principle (OCP)**
> Open for extension, closed for modification.

```java
// BAD: Must edit existing code to add new payment type
class PaymentProcessor {
    public void process(String type, double amount) {
        if (type.equals("CREDIT_CARD")) { ... }
        else if (type.equals("PAYPAL")) { ... }
    }
}

// GOOD: Add new payment by creating a new class
interface PaymentMethod { void process(double amount); }
class CreditCardPayment implements PaymentMethod { ... }
class PayPalPayment     implements PaymentMethod { ... }
// Add crypto? Just: class CryptoPayment implements PaymentMethod {}
```

---

**L — Liskov Substitution Principle (LSP)**
> A subclass must be substitutable for its parent without breaking behavior.

```java
// BAD: Square overrides Rectangle's setWidth and corrupts area()
Rectangle r = new Square();
r.setWidth(5); r.setHeight(10);
System.out.println(r.area()); // Expected 50, gets 100!

// GOOD: Use a common interface instead of forced inheritance
interface Shape { int area(); }
class Rectangle implements Shape { ... }
class Square    implements Shape { ... }
```

---

**I — Interface Segregation Principle (ISP)**
> Don't force classes to implement methods they don't need.

```java
// BAD: Fat interface forces Dog to implement fly()
interface Animal { void eat(); void fly(); void swim(); void run(); }
class Dog implements Animal {
    public void fly() { throw new UnsupportedOperationException(); }
}

// GOOD: Small, specific interfaces
interface Eatable  { void eat(); }
interface Flyable  { void fly(); }
interface Swimmable { void swim(); }

class Dog  implements Eatable, Swimmable { ... }
class Bird implements Eatable, Flyable   { ... }
```

---

**D — Dependency Inversion Principle (DIP)**
> Depend on abstractions (interfaces), not concrete implementations.

```java
// BAD: Tightly coupled to MySQL
class UserService {
    private MySQLUserRepository repository = new MySQLUserRepository();
}

// GOOD: Depend on interface; inject via constructor (Spring @Autowired does this)
class UserService {
    private UserRepository repository;
    public UserService(UserRepository repository) { this.repository = repository; }
}
// Swap MySQL for MongoDB without touching UserService!
```

---

## Design Patterns

### Q14: What are Design Patterns?

Proven solutions to common programming problems.

| Category | Purpose | Examples |
|----------|---------|---------|
| **Creational** | How objects are created | Singleton, Factory, Builder |
| **Structural** | How objects are composed | Adapter, Facade, Decorator |
| **Behavioral** | How objects communicate | Observer, Strategy |

---

**Singleton Pattern**
```java
// Only ONE instance — use for DB connection, config, logger
public class DatabaseConnection {
    private static DatabaseConnection instance;
    private DatabaseConnection() {}

    public static synchronized DatabaseConnection getInstance() {
        if (instance == null) instance = new DatabaseConnection();
        return instance;
    }
}
// conn1 == conn2 → true
```

---

**Factory Pattern**
```java
// Create objects without specifying the exact class
interface Notification { void send(String message); }
class EmailNotification implements Notification { ... }
class SMSNotification   implements Notification { ... }

class NotificationFactory {
    public static Notification create(String type) {
        return switch (type) {
            case "EMAIL" -> new EmailNotification();
            case "SMS"   -> new SMSNotification();
            default -> throw new IllegalArgumentException("Unknown: " + type);
        };
    }
}
Notification n = NotificationFactory.create("EMAIL");
```

---

**Builder Pattern**
```java
// Build complex objects step by step — useful for many optional fields
User user = new User.Builder("Amith", "amith@gmail.com")
    .phone("9876543210")
    .age(25)
    .build();
```

---

**Observer Pattern**
```java
// When one object changes, all subscribers are notified
// Spring Boot example:
@Component
public class OrderService {
    @Autowired ApplicationEventPublisher publisher;

    public void placeOrder(Order order) {
        orderRepository.save(order);
        publisher.publishEvent(new OrderPlacedEvent(order));
    }
}

@Component
public class EmailListener {
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        emailService.sendConfirmation(event.getOrder());
    }
}
```

---

## Clean Code Principles

### Q15: What is Clean Code?

```
1. MEANINGFUL NAMES
   ❌ int d;  void calc();
   ✅ int daysRemaining;  void calculateTax();

2. SMALL FUNCTIONS      → One function does ONE thing

3. DRY (Don't Repeat Yourself)
   ❌ Same logic copied in 5 places
   ✅ Extract to a shared method

4. COMMENTS             → Explain WHY, not WHAT
   ✅ // Batch size 1000 to avoid OOM on large datasets

5. NO MAGIC NUMBERS
   ❌ if (status == 2)
   ✅ if (status == UserStatus.ACTIVE)

6. ERROR HANDLING       → Throw meaningful exceptions; use Optional<> instead of null
```

---

## Testing Principles

### Q16: What is TDD?

TDD = Write the **test first**, then write the minimum code to make it pass.

```
Cycle: RED → GREEN → REFACTOR
  RED:     Write a failing test (class/method doesn't exist yet)
  GREEN:   Write minimum code to pass the test
  REFACTOR: Clean up code; test still passes
```

```java
// Step 1: Test first
@Test
void shouldCalculateTax() {
    assertEquals(18.0, new TaxCalculator().calculate(100.0, 0.18));
}

// Step 2: Write code
class TaxCalculator {
    double calculate(double amount, double rate) { return amount * rate; }
}
```

---

### Q17: What is the Testing Pyramid?

```
       /\        E2E Tests   (10%) — full user flows, slow
      /--\       Integration (20%) — API + DB interactions
     /----\      Unit Tests  (70%) — single method, fast, isolated
```

**Spring Boot examples:**
```java
// Unit Test (JUnit 5 + Mockito)
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository userRepository;
    @InjectMocks UserService userService;

    @Test
    void shouldReturnUserWhenFound() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(new User(1L, "Amith")));
        assertEquals("Amith", userService.findById(1L).getName());
    }
}

// Integration Test
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {
    @Autowired MockMvc mockMvc;

    @Test
    void shouldReturnUserList() throws Exception {
        mockMvc.perform(get("/api/users"))
            .andExpect(status().isOk());
    }
}
```

---

## Quick Revision Summary

### Scrum in One Table

| Item | Description |
|------|-------------|
| **Sprint** | 2-4 week iteration |
| **Product Backlog** | All features list (PO owns) |
| **Sprint Backlog** | This sprint's tasks (Team owns) |
| **Daily Standup** | 15 min, 3 questions, every day |
| **Sprint Review** | Demo to stakeholders |
| **Sprint Retro** | Improve the process |
| **Product Owner** | What to build |
| **Scrum Master** | Remove blockers |
| **Dev Team** | How to build it |

### SOLID in One Line Each

| Principle | One Line |
|-----------|---------|
| **S**RP | One class = one job |
| **O**CP | Extend without modifying |
| **L**SP | Subclass must work where parent works |
| **I**SP | Small interfaces > fat interfaces |
| **D**IP | Depend on interfaces, not implementations |

### Agile vs Scrum
- **Agile** = mindset/philosophy (the "what")
- **Scrum** = a framework that implements Agile (the "how")
- Scrum IS Agile, but Agile is NOT necessarily Scrum

### Common Interview Q&A

**Q: What is Agile?**
> Iterative software development that delivers working software frequently and welcomes changing requirements.

**Q: What is the role of Scrum Master?**
> Facilitates the Scrum process, removes blockers, coaches the team. Not a project manager.

**Q: What happens in Daily Standup?**
> 15-min meeting: What did I do yesterday? What will I do today? Any blockers?

**Q: What is a Sprint?**
> A fixed time-box (1-4 weeks) where the team delivers a potentially shippable product increment.

**Q: What is Definition of Done?**
> A shared checklist defining when a task is truly complete (code reviewed, tested, deployed, etc.).

**Q: Name the SOLID principles.**
> Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion.

**Q: What design pattern does Spring IoC use?**
> Dependency Inversion Principle + Factory Pattern — Spring creates and injects beans automatically.

---

*Last Updated: 2026-06-18*
